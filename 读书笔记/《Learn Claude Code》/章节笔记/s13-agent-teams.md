# s13 Agent Teams — 团队运行时与协作协议

> 📖 《Learn Claude Code》章节笔记｜[← 返回主笔记](../笔记.md)｜状态：✅ 已完成（2026-10-14）

## 一、本章讲了什么（充分展开，讲透为止）

### 1. 本章要解决的问题

s06 的 subagent 解决了"一次性委派"，s10 的任务系统解决了"工作项的记录与依赖"，但面对"重构整个后端——清理配置加载、认证、测试"这类任务，单 Agent 仍有两个硬伤：**串行慢**，以及**早期细节随上下文膨胀逐渐丢失**。并行化是显而易见的答案，可用户只描述目标，不会替运行时设计团队。README 开篇列了六个相互关联的问题，它们其实就是本章的目录：谁判断并行有用、队友身份如何跨任务保留、结果如何自动回到 Lead（而不是让模型轮询）、空闲队友能否自己认领 ready task、并行修改用哪个工作目录、关机与审批如何成为可追踪的协议。

### 2. 核心方案：Lead + 持久队友 + 文件收件箱 + 原子认领 + 类型化协议

本章在 s10 的基础上增加一套**团队运行时**（Team 层），七个组件：

1. **Lead**：唯一与用户对话的 Agent，负责提出分工方案、创建任务图、spawn 队友、汇总结果；
2. **持久队友**：每个队友是独立线程里运行的完整 Agent Loop，在 WORK / IDLE 两态间循环直到收到关机指令；
3. **MessageBus**：`.mailboxes/<name>.jsonl` 文件收件箱，承载普通消息、结果、控制事件；
4. **运行时投递**：CLI 主循环同时等待终端输入和 Lead 收件箱，事件到达自动唤醒新一轮 Lead 调用；
5. **共享任务板**：IDLE 队友扫描 ready task 并在锁内原子认领；
6. **可选 worktree**：任务可绑定独立 Git 工作目录，队友的文件/Shell 工具 cwd 由任务决定；
7. **类型化协议**：关机握手和计划审批用带 `request_id` 的结构化消息，不接受自由文本猜测。

### 3. 关键机制逐条走读（code.py，1887 行，全书最重的一章）

**① 团队提案闸门是提示词契约，不是代码闸门**（第 641-655 行）：Lead 的 system prompt 明确写着"先提出小团队方案并等用户确认，确认前不许调 spawn_teammate"。注意这里作者选择了**软约束**——启动队友改变成本、并发度和工作区的写入者集合，这个决策交给"提示词 + 用户确认"而不是硬编码拦截，与 s03 的权限哲学一致：边界由人定，代码只提供执行点。

**② 队友 ≠ subagent**：README 给的对比表值得背下来——s06 subagent 一次调用即消亡、上下文随任务消失、单向返回结果；s13 队友 `WORK → IDLE → WORK` 持久循环、上下文跨任务保留、双向收发消息。**持久性的代价**就是本章后半部所有的复杂度：状态要管理、认领要原子、关机要握手。

**③ 通信放在模型上下文之外**（MessageBus，第 852-909 行）：Lead 和队友**不能共享 messages 数组**，否则一个队友的工具结果会污染另一个的推理上下文。收件箱是 append 的 jsonl 文件 + `Condition` 变量（消息到达即唤醒 IDLE 等待）；`read_inbox()` 是破坏性读（读完删文件）。`check_inbox` **不是模型工具**——消息的到达与消费属于运行时，模型只处理已被投递进上下文的事件。这是本章最重要的架构决策：通信层对模型不可见，避免了"模型忘了查收件箱"这一整类故障。

**④ 结果与空闲是两个事件**：队友完成任务后运行时依次发 `result`（"产出了什么"）和 `idle_notification`（"能否再接活"）。一个含糊的"完成了"无法同时表达两种状态——比如计划待审批的队友是"有产出但不可接活"。这个区分让 Lead 的任务板逻辑可以精确响应。

**⑤ IDLE 循环：先收件箱，再任务板**（`wait_for_work`，第 1322-1346 行）：队友空闲时每 2 秒先等消息（关机、审批、Lead 直派优先级最高），没消息才 `claim_next_task()` 扫描任务板。扫描（`scan_unclaimed_tasks`：pending + 无 owner + 依赖完成）和认领（`claim_task`）**分两步**：扫描结果只是快照，多个队友可能同时看到同一任务，所以认领必须在 `task_store_lock()` 里原子完成——这个锁是**线程锁 + fcntl 文件锁**双重（第 70-89 行），意味着即使另一个 Harness 进程共享同一 `.tasks/` 目录也不会双认领。认领同时把 cwd 绑定写进 `teammate_assignments` 注册表。

**⑥ 工作目录由任务决定**（第 744-773 行）：队友的 bash/read/write/edit/glob 全部经过 `assignment_cwd()` 解析目录——绑定 worktree 的任务解析到 `.worktrees/<name>`，未绑定的回 WORKDIR，**没认领任务的队友直接用不了工作区工具**（返回 "Claim a Task first"）。worktree 的创建只允许 Lead 调 `create_worktree`（校验任务 pending 且未绑定 → 校验名称/分支/Git 注册 → `git worktree add` → 最后才写任务绑定，失败留现场报 partial operation）；**移除则连模型都不给**——`remove_worktree` 是宿主保留函数。README 特别警告：worktree 只是目录隔离，不是安全沙箱。

**⑦ 类型化协议的三重校验**（`match_response`，第 944-967 行）：关机和计划审批不靠自由文本，走 `ProtocolState`（request_id / type / sender / target / status）。响应生效需要同时满足：类型匹配（shutdown 配 shutdown_response）、方向匹配（响应者必须是原 target）、状态是 pending（防重复生效）。计划闸门更狠——**拦在工具分发层**（`_run_teammate_tool`，第 1038-1057 行）：`plan_gates` 状态为 required/pending/rejected 时，`bash`/`write_file`/`edit_file` 直接返回 "Blocked: plan status is ..."。审批还绑定 `work_version`：队友认领/释放任务会推进版本号，使旧审批自动失效——防止"批准的是上一个任务的计划"这类错配。

### 4. 实战观察（沙盒实测）

在 `/tmp/lcc-s13` 让 Lead "把创建 hello_a.py 和 hello_b.py 拆到任务板并行完成"。实测全链路：

1. **提案闸门生效**：Lead 收到需求后只提出分工方案（teammate-a/teammate-b），结束回合等待；我回复"开始吧"之后它才 `create_task` × 2 → `spawn_teammate` × 2。spawn 时**先 claim 初始任务再启动线程**（输出里 `[claim] 创建 hello_b.py -> in_progress` 先于 `[teammate] teammate-b spawned`）；
2. **真并行**：两个队友线程的 hook 输出交错出现在终端（两个 `bash` 写文件钩子几乎同时）；
3. **队友会主动协作**：teammate-b 完成 `complete_task` 之外还主动用 `send_message` 给 Lead 发了一条进度汇报——系统提示词里 "Use send_message only for intermediate coordination" 引导出的行为；
4. **事件唤醒**：`[wake: 3 team event(s) -> new turn]`，Lead 收件箱的 result + idle_notification 被格式化成 `[Team events]` 注入，Lead 醒后**亲自用 bash 复验**了两个文件内容并运行（`cat hello_a.py hello_b.py && python3 ...`），然后才发起 `request_shutdown` × 2；
5. **关机握手**：`req_254277`/`req_170579` 两个 shutdown_request 进入队友收件箱，队友回 shutdown_response，运行时打印 `[protocol] req_xxx -> approved`，队友线程退出，`[all teammates shut down]`；
6. **落盘验证**：`.tasks/task_*.json` 持久化记录了 owner 和 completed 状态；`.mailboxes/` 目录因破坏性读已清空——通信是临时的，任务板才是档案。

## 二、与前面章节的关系

**直接基座是 s10（Task System）**：任务的两阶段契约（先 `create_task` 拿运行时 ID，再 `update_task(addBlockedBy=...)` 补依赖边）原样保留，但本章加了团队语义——**只有 Lead 能改任务图结构**，队友只能列举、认领、完成。共享任务板 + 原子认领，是 s10 的单 Agent 任务工具在并发场景下的正确化：s10 不需要锁（单线程），本章引入了双重锁。

**对 s06（Subagent）的升级**：从一次性委派到持久协作。这个升级的必要性恰好由 Cognition 的反方文章（见拓展资料 2）来拷问。

**不带入 s11/s12**：README 明说后台任务和定时任务不参与本章——它们与队友通信、任务认领、计划审批正交。再次体现"每章只叠加一个维度"。

**循环骨架依然不变**：这是最值得强调的一点——**每个队友的 `work()` 方法就是 s01 的那个循环**（发 messages → 查 tool_use → 执行 → 回喂），只是外套了 WORK/IDLE 状态机；Lead 的 `agent_loop` 更是逐行同构。多 Agent 系统没有发明新的循环，只是**让同一个循环跑在多个线程里，用文件收件箱交换消息**。所谓"团队"，在代码层面 = N 个循环 + 1 个消息总线 + 1 块共享任务板。

**对 s02 的延伸**：`safe_path` 的单 WORKDIR 假设被打破，文件工具通过 `teammate_assignments` 注册表实现"按 owner 路由 cwd"——同一把工具，不同队友看到不同工作目录。

## 三、实战场景（怎么用）

1. **多模块并行重构**：真实对应本章示例——config/auth/tests 三个方向互不依赖时，spawn 三个队友 + 给可能冲突的任务绑 worktree，Lead 最后汇总并跑回归测试。**前提判断**：模块间接口稳定、改动可隔离；不满足就退回单 Agent 串行。
2. **代码评审角色分工**：一个队友写实现、一个队友专职写测试并试图打脸、Lead 当裁判——利用 plan gate 让"写实现"的队友先交计划获批。
3. **长任务的身份延续**：运维巡检类工作，队友 IDLE 挂起等新任务而不是每次重建上下文——上下文里积累的"上次这台机器是什么问题"就是持久队友的价值。
4. **评估多 agent 框架的 checklist**：找到它的收件箱在哪、认领是否原子、结果和空闲是否分事件、关机有没有握手——四问缺一个，生产上就会踩对应的坑。

## 四、我的思考

*（待补充——你的联想、认同与质疑写在这里）*

## 五、我的问题与解答

- ❓**为什么 Agent 间通信走文件收件箱（jsonl）而不是内存队列？**
  → ✅解答：① **跨进程安全**：文件 + fcntl 让另一个 Harness 进程也能参与同一任务板/收件箱（`claim_task` 的注释明说这点）；② **可观测可调试**：`.mailboxes/` 和 `.tasks/` 目录随时可以人工检查——本章实测中我就是靠看 `.tasks/*.json` 验证状态的；③ **与全书哲学一致**：s09 记忆、s10 任务都是"文件即状态"，文件是最小公分母的持久化。代价是 IO 开销和锁复杂度，对教学规模可忽略。生产系统会换 Redis/消息队列，但**协议语义（request_id、类型、状态机）原样适用**。
- ❓**计划闸门为什么拦在工具分发层（代码），而不是只在队友的提示词里说"先交计划"？**
  → ✅解答：**提示词是软约束，代码闸门是硬保证**。模型可能误解、遗忘或被注入内容诱导绕过提示词指令；而 `_run_teammate_tool` 里的 `plan_gates` 检查在每次工具调用时强制执行，模型没有任何路径绕过。这是 s03 权限章"在执行前加一道门"思想在多 agent 场景的复用，也是 defense in depth：提示词负责引导模型走正路（少撞门），代码负责兜底（撞了也过不去）。
- ❓**审批绑定 `work_version` 是为了什么攻击/故障场景？**
  → ✅解答（代码走读得出）：防止**审批错配**。场景：队友提交任务 A 的计划 → Lead 还没批 → 队友释放了 A 认领了任务 B → 此时 Lead 的批准到达。没有版本绑定时，"批准 A 的计划"会被错误地应用到 B 的执行上。`advance_assignment_version()` 在每次认领/释放时推进版本并让旧闸门回到 required，批准只在 `work_version` 和 `task_id` 双一致时生效。这本质是分布式系统里**租约（lease）+ fencing token** 思想的迷你版。

## 六、关键摘记

⭐ 团队 = N 个 s01 循环 + 1 个 MessageBus + 1 块任务板——多 agent 没有新循环｜⭐ 通信在模型上下文之外：check_inbox 不是工具，运行时负责投递｜⭐ result 和 idle_notification 是两个事件："产出了什么" ≠ "能否再接活"｜⭐ 扫描与认领分离，认领必须原子（线程锁 + 文件锁）｜💡 spawn 前先 claim——认领失败的队友根本不该出生｜💡 计划闸门拦在工具分发层：提示词引导，代码兜底｜💡 worktree 是目录隔离不是沙箱；移除权保留给宿主

## 七、代码实战

跑通 ✅（沙盒 `/tmp/lcc-s13`：提案闸门 → 确认后 create_task × 2 + spawn × 2 → 两队友并行写文件 → result + idle_notification 经 MessageBus 回 Lead → wake 注入 → Lead 亲自 bash 复验 → shutdown 握手 req_254277/req_170579 双双 approved → `.tasks/` 落盘 completed）　改造实验 ☐（待做：git init 沙盒后测 worktree 绑定；对队友发 `request_plan` 验证计划闸门拦截 bash）

## 八、📚 本章拓展资料与对照（每条结合本章展开）

**1. Anthropic《How we built our multi-agent research system》（2025-06，Anthropic Engineering）——赞同 + 量化代价**

它讲了什么：Anthropic 官方复盘其 Research 功能的多 agent 架构——**orchestrator-worker 模式**：LeadResearcher 分析查询、制定策略、spawn 子 agent 并行搜索、汇总结果，与本章的 Lead/队友结构完全同构。最有价值的是量化数据与失败教训：① 内部评估中多 agent 系统以 90.2% 的相对优势超过单 agent，但 **token 消耗约为普通对话的 15 倍**（单 agent 约 4 倍）；② 分析发现 token 用量单独解释 80% 的性能方差——多 agent 有效的本质原因是"用更多并行上下文窗口烧更多 token"；③ 早期失败模式包括"为简单查询 spawn 50 个 agent""子 agent 任务描述太简略导致重复劳动"，解法是**教 orchestrator 详细委派**（目标、输出格式、工具边界）；④ 明确说**需要共享上下文或强依赖的任务（如多数编码任务）目前不适合多 agent**。

与本章的对照：本章的"Lead 先建任务图、给每个 spawn 传详细 task_id + prompt"就是文章"详细委派"教训的产品化；本章提案闸门（等用户确认再 spawn）也呼应了"简单查询不该 spawn 50 个 agent"的成本意识。冲突点在适用面：文章明说编码任务并行度低、不适合多 agent，而本章的示例恰恰是编码重构——我的判断：本章是教学演示（演示机制用两个独立文件足够），生产决策应采纳 Anthropic 的门槛："任务价值要配得上 15 倍 token"。（已于 2026-10-14 查证原文，https://www.anthropic.com/engineering/multi-agent-research-system ）

**2. Cognition《Don't Build Multi-Agents》（Walden Yan，2025-06）——反方，必须正面回答**

它讲了什么：Devin 制造商 Cognition 的立场文章，点名批评 OpenAI Swarm 和微软 AutoGen 推动的多 agent 架构"是构建 agent 的错误方式"。核心论证：多 agent 拆分任务的架构**很脆弱**——用"造 Flappy Bird"举例：subagent 1 把背景做成了马里奥风格、subagent 2 做的鸟不像游戏素材，最后合并者面对两堆互相误解的产出。根源是**子任务间的隐式假设无法传递**：每个"行动都携带隐式决策"，拆给不同上下文执行就会发散。由此提出两条上下文工程原则：**Share context**（共享完整执行轨迹，而非只传结论）和 **Actions carry implicit decisions**（冲突的隐式决策导致坏结果）。推荐架构是单线程线性上下文的 agent。

与本章的冲突与辩证：本章的 MessageBus 恰好就是 Cognition 反对的架构——队友之间**刻意不共享上下文**（README 明说共享 messages 会互相污染），只传 result 摘要。但细看本章的缓解设计：任务板要求 Lead 写清 subject+description（对抗误解）、Lead 收到结果后亲自复验（本章实测中 Lead 真的 `cat` 了两个文件并运行）、worktree 隔离冲突修改。我的判断：Cognition 批评的是"**高耦合任务**强行拆分"，本章适合的是"**可独立验证的子任务**"——本章实测的两个 hello 文件互不依赖，天然安全；若任务变成"队友 A 写接口、队友 B 写调用方"，Flappy Bird 式发散立刻会出现。**架构没有对错，任务耦合度决定一切**；本书 s16 标题"模型决定单步，脚本决定编排"暗示作者最终也会承认编排的边界。（已于 2026-10-14 查证原文，https://cognition.ai/blog/dont-build-multi-agents ）

**3. Anthropic《Building Effective Agents》orchestrator-workers 章节（2024-12）——赞同，定位本章在光谱中的位置**

它讲了什么：这份报告的 workflow 光谱里，**orchestrator-workers** 的定义是"中央 LLM 动态拆解任务、委派给 worker、综合结果"，适用条件明确写着"子任务无法预先预测的复杂任务——例如编码中，需要改哪些文件、每个文件怎么改都取决于任务本身"；并强调它与 parallelization 的区别在于子任务是 orchestrator 根据输入**动态确定**的。

与本章的对照：本章 Lead 正是按这个模式工作——用户只说"重构后端"，拆成 config/auth/tests 是 Lead 的动态决策而非硬编码路径，完全一致。有意思的张力：Anthropic 把它归为 **workflow**（预定义代码路径编排）范畴而非 autonomous agent，而本书把本章放在 harness 机制演进线上——分类学不同，机制相同。我的判断：本章代码里"运行时投递事件唤醒 Lead"这一段，其实就是 workflow 的边（控制流）以事件驱动形式实现——workflow 与 agent 的边界在实践中比教科书上模糊得多。（已于 2026-10-14 查证原文，https://www.anthropic.com/research/building-effective-agents ）
