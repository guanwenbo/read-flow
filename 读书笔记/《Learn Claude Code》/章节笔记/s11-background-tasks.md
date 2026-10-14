# s11 Background Tasks — 慢操作放后台

> 📖 《Learn Claude Code》章节笔记｜[← 返回主笔记](../笔记.md)｜状态：✅ 已完成（2026-10-14）

## 一、本章讲了什么（充分展开，讲透为止）

### 1. 本章要解决的问题

到 s10 为止，所有工具调用都是**同步**的：`execute_tool` 不返回，循环就停在这一步。读文件、`git status` 很快，无所谓；但 `npm install`、全量测试、项目构建动辄几分钟——这段时间里 harness 既不能处理同一响应里的下一个 tool_use，也不能进入下一轮。如果后续工作并不依赖这条慢命令的输出（装依赖的同时完全可以去读文档、整理文件），同步等待就是纯粹的浪费。本章要解决：**让耗时命令在后台执行，Agent Loop 继续跑，结果在后续轮次收回来**。

### 2. 核心方案：显式后台化 + 占位结果 + 轮次收集

方案分四块，每一块都有一个关键设计决策：

**① 显式请求，不做关键词猜测**。bash 工具的 schema 增加一个 `run_in_background: boolean` 参数，`should_run_background()` 只在工具是 bash **且参数明确为 true** 时才走后台路径。README 专门点出反面教材：不再根据命令里有没有 "install"/"build"/"test" 等关键词猜测——猜测会把 harness 的语义判断变成不可靠的字符串匹配，而**是否后台是模型基于任务依赖关系做出的决策**（模型知道这个命令的结果后面要不要立刻用），决策权留在模型侧，机制侧只提供开关。

**② BackgroundManager：线程 + 锁 + 完成队列**。`start()` 在锁内登记任务（`bg_0001` 自增 ID、原始 `tool_use_id`、命令、status=running），然后起一个 **daemon 线程**跑 `_run()`，主线程立即拿到 `bg_id` 返回。`_run()` 里用 `subprocess.Popen` 执行命令（POSIX 下 `start_new_session=True`，让 shell 自成进程组），等 `communicate(timeout=120)` 结束后在锁内更新 status（exit 0 → completed，否则 failed）、存结果、把 task_id 追加进 `_ready` 完成队列。所有共享状态（`tasks`/`results`/`_ready`）的读写都在同一把 `threading.Lock` 内——线程模型朴素但纪律严格。

**③ 占位 tool_result，保住协议不变量**。后台任务的工具调用**当场**返回 `"[Background task bg_0001 started] The result will be collected on a later turn."` 作为正常 tool_result。为什么必须这样？因为 Anthropic API 的协议要求**一个 tool_use 必须恰好对应一个 tool_result**——如果后台任务不回结果，消息历史就不合法。真正的命令输出稍后以**独立的 `<task_notification>` 文本块**进入对话（不复用原 tool_use_id）。README 强调这一点：通知是一个新事件，不是迟到的 tool_result。

**④ 轮次开始处收集注入（拉取而非推送）**。`agent_loop` 每轮调 LLM 之前先 `inject_background_results(messages)`：`collect()` 在锁内清空 `_ready` 队列、弹出对应任务和结果，格式化成 `<task_notification>`（含 task_id/status/command/截断到 500 字符的 summary），然后**追加到 messages 末尾**——若最后一条是 user 消息就把文本块并进它的 content，否则新建一条 user 消息。这意味着后台结果**不会主动唤醒 agent**：如果模型已经停止调工具、回合结束了，完成通知要等下一个用户回合才被看到。README 如实写明这个限制。

**进程生命周期管理**是本章隐藏的硬功夫：`atexit` 注册 + `SIGTERM` handler 都会在退出时遍历 `_shell_processes` 集合，对每个进程组先发 SIGTERM、50ms 后补 SIGKILL（Windows 无进程组概念，降级为 `terminate()`/`kill()`）；`_run_bash_process` 的 `finally` 里正常结束后也会清进程组（防子进程泄漏）。README 诚实声明：这是生命周期清理，**不是沙箱**——主动 `setsid` 另立门户的进程仍能逃逸。

### 3. 代码走读要点（code.py，542 行）

- **循环骨架只加一行**：`while True:` 里第一句变成 `inject_background_results(messages)`，其余（发消息、查 tool_use、回喂结果）与 s04 内核逐行一致。分叉点在 `execute_tool` 内部：PreToolUse hooks 照常在主线程执行（权限检查不因后台化而跳过！），之后才按 `should_run_background` 分流到 `start_background_task` 或同步的 `call_tool`——s03 的权限、s04 的 hooks 与后台执行完全正交。
- 同步/后台两条路径最终共用 `_run_bash_process` 和 `_format_bash_result`，行为一致（120s 超时、50000 字符截断、非零退出码包装成 Error 文本）。
- `start()` 里线程启动失败有兜底：从 `tasks` 里摘除已登记的记录再抛出——不留僵尸登记。
- 通知 summary 只截 500 字符：通知的职责是"告诉模型发生了什么"，需要全文时模型可以自己再想办法——返回值克制哲学（s02）的延续。

### 4. 实战观察（沙盒实测）

在 `/tmp/lcc-s11` 沙盒实测：

- **回合 1**：要求"后台跑 `sleep 8 && echo background-job-finished`（用 run_in_background），等的同时写 notes.txt"。模型一轮响应里发出两个 tool_use：`bash(command=..., run_in_background=True)` → 立即打印 `[background] started bg_0001` 并返回占位结果；紧接着 `write_file` 同步执行成功。**8 秒的 sleep 没有阻塞写文件**——主循环继续运行的核心承诺验证成功。
- **回合 2**：问"后台任务完成了吗？"。模型先做了一次同步 bash 检查，而循环顶部打印 `[background] collected bg_0001: completed`——`<task_notification>` 已随本轮注入，模型据此报告 status=completed、输出 `background-job-finished`。**收集发生在轮次边界**的机制验证成功。

一个细节：模型在回合 1 末尾说 "Let me know when you want to check on the background job's output"——它准确理解了"结果不会自动来找我，要后续轮次收集"这个协议，说明占位 tool_result 的措辞（"will be collected on a later turn"）有效传达了机制语义。

## 二、与前面章节的关系

**对执行模型的第一次"时间维度"扩展**。此前所有章节的工具执行都是同步的：s02 的 dispatch map、s03 的权限、s04 的 hooks 都假设"调用→等待→返回"一步完成。本章在这个链条中间插入异步分支，而**既有机制无一需要修改**：权限 hook 仍在主线程先于执行跑完，dispatch map 照常分发，循环骨架照常判断 tool_use——"机制长在循环上"的架构纪律经受住了并发这个最挑剔的考验。

与 s10 的衔接是问题驱动的：任务系统让 agent 能管理"建 schema → 写 API → 跑全量测试"这样的图，但"跑全量测试"本身可能几分钟——没有本章，任务图越实用，被慢命令卡死的时间就越长。README 的"接下来"也指明了方向：本章解决"慢操作不阻塞"，下一个自然问题是"定时触发"（s12 Cron）。另外注意本章**没有**和 s10 的 task 打通（后台任务不会自动更新 .tasks/ 状态）——两套系统目前是正交的，组合留给使用者。

## 三、实战场景（怎么用）

1. **开发流里的标准动作**：让 agent "后台跑完整测试套件，同时继续按任务清单改下一个文件"——测试完成通知到达时再回来修失败用例。这是 Claude Code 真实用户每天都在用的模式（Ctrl+B 后台化）。
2. **启动长服务类命令**：本地起 dev server、数据库容器后立刻继续干别的，不用干等 "listening on port 3000"。
3. **设计自己的异步工具时的检查清单**：①占位结果必须立刻返回以保住 tool_use↔tool_result 一一对应；②完成事件用独立通知格式，不复用原 ID；③权限检查要在分叉前完成；④进程退出要有清理路径（atexit/信号/进程组）；⑤明确告知模型"结果不会主动来，后续轮次才收"。五条全中才算抄对了本章。

## 四、我的思考

*（待补充——你的联想、认同与质疑写在这里）*

## 五、我的问题与解答

- ❓**为什么用线程 + 轮次收集，而不是 asyncio 或回调主动唤醒 agent？**
  → ✅解答：三个层面。①适配性：agent loop 本质是与 LLM API 的同步请求-响应循环，"下一轮开始前收结果"是最小侵入的注入点——异步回调要在中途打断/修改消息历史，协议上更复杂。②协议约束：通知只能在"下一条 user 消息"里出现，而 user 消息只在轮次边界产生，所以收集点天然就在轮次开头。③成本：线程 + 一把锁的模型 100 行内可读完，asyncio 会要求全链路改写。代价是已知的：回合结束后完成的任务要**等下一个用户回合**才被模型看到（README 明言）。生产系统（如 Claude Code）也是同样的轮询收集模型，可见这个取舍不是教学偷懒。
- ❓**daemon 线程 + 进程组清理，会不会误杀或漏杀？**
  → ✅解答（代码分析）：设计上分三层：正常结束——`finally` 里清进程组（防 shell 已退出但子进程还活着）；进程退出——atexit + SIGTERM handler 遍历 `_shell_processes` 逐个 SIGTERM→SIGKILL；崩溃（如 SIGKILL 打在本进程上）——什么都没法跑，daemon 线程随进程消失，但**子进程可能变孤儿**留在系统里。README 的诚实声明"这不是沙箱，setsid 的进程会逃逸"正是指这类边界。教学实现覆盖了 95% 的路径，剩下 5% 需要容器/命名空间级别的隔离，那是部署问题不是 harness 问题。
- ❓**`inject_background_results` 把通知并进已有的最后一条 user 消息，这会篡改历史吗？**
  → ✅解答（待验证方向）：不会破坏协议但值得警惕语义。代码只在最后一条消息是 user 角色时**追加**文本块（原内容保留），API 层面合法；且注入发生在发送给 API 之前，历史里这条消息从此就带着通知——后续所有轮次都会重复看到它（token 成本）但也获得了"何时收到通知"的时序记录。真正要验证的边界情况：如果通知到达时模型正处在"连续 tool_use"的中间，最后一条消息是 tool_result 组成的 user 消息——追加文本块到 content 列表是否影响 tool_result 块的完整性？从代码看是 `content.extend(blocks)`，往数组尾部加 text 块，不触碰已有 tool_result 块，合法。待观察极端情况下模型是否会混淆通知与工具结果。

## 六、关键摘记

⭐ 是否后台 = 模型的显式决策（run_in_background），不是 harness 的关键词猜测——决策归模型，机制归代码｜⭐ 占位 tool_result：一个 tool_use 必须恰好对应一个 tool_result，后台结果以独立 `<task_notification>` 事件进入对话，不复用原 ID｜⭐ 拉取而非推送：结果在轮次开始处收集，后台任务不会主动唤醒 agent｜⭐ 权限检查（PreToolUse）在同步/后台分叉之前执行——后台化不绕过任何既有安全机制｜💡 daemon 线程 + atexit/SIGTERM + 进程组 SIGTERM→SIGKILL：生命周期清理不是沙箱｜💡 通知 summary 只截 500 字符——通知的职责是"告诉模型发生了什么"，不是搬运全文

## 七、代码实战

跑通 ✅（沙盒实测：后台 `sleep 8` 启动后立刻返回 bg_0001 占位结果，write_file 同步完成未被阻塞；下一回合轮次开头打印 `[background] collected bg_0001: completed`，模型从 `<task_notification>` 读到输出 `background-job-finished`）　改造实验 ☐（待做：后台跑一条 `exit 1` 的失败命令，观察 status=failed 的通知格式与模型反应）

## 八、📚 本章拓展资料与对照（每条结合本章展开）

**1. Claude Code 官方文档《Interactive mode — Background Bash commands》——赞同，本章机制的生产对照**

它讲了什么：Claude Code 真实产品的后台任务机制：后台运行命令时**立即返回一个 background task ID**，命令继续异步执行，Claude 可以同时响应新 prompt；触发方式可以是模型自己决定后台跑，也可以用户按 Ctrl+B 把正在运行的 Bash 调用转入后台；命令超时时自动转入后台而非掐断；输出写入文件、可用 Read 工具取回；退出时自动清理后台任务，macOS/Linux 上连 setsid 脱离进程组的进程也会被处理；输出超过 5GB 自动终止并在 stderr 说明。（已于 2026-10-14 用 curl 抓取 docs.claude.com 原文验证）

与本章的对照：几乎逐项对应——立即返回任务 ID（bg_0001）、执行期间继续工作、退出时进程组清理，全部吻合，说明本章不是简化玩具而是生产机制的忠实缩微。生产版多出来的恰好是本章 README 坦白的边界：输出落盘供事后读取（本章只在通知里截 500 字符）、超时自动后台化（本章超时直接报错）、5GB 输出熔断（本章只有 50000 字符截断）。判断：学本章再看 Claude Code 文档会有"每个功能都认识"的体验——教学模型抓住了骨架，生产版加的是容量与边界情况的肉。

**2. Python 官方文档《threading — Thread-based parallelism》——机制地基，含 GIL 这个关键前提**

它讲了什么：Python 标准库线程模块的官方参考。两个与本章直接相关的事实：①**daemon 线程**的语义——主程序退出时 daemon 线程直接被回收，不等它们跑完；②**GIL**——CPython 中同一时刻只有一个线程执行 Python 字节码，官方建议计算密集型用 multiprocessing，但明确说"想同时跑多个 I/O 密集型任务时，threading 仍是合适的模型"。（已于 2026-10-14 用 curl 抓取原文验证）

与本章的对照：本章选择线程的合法性正是 GIL 论证的教科书案例——后台线程 99% 的时间阻塞在 `process.communicate()` 上（等待子进程 I/O，C 层释放 GIL），主线程的 API 调用也是 I/O，两者几乎不争抢解释器；真正并行的是**子进程**（独立进程，不吃 GIL）。daemon=True 则是双刃剑：它保证 agent 退出时线程不拖住进程，但也意味着崩溃时线程静默死亡——所以本章才需要 atexit/信号处理做进程级清理来补 daemon 线程不管的部分（子进程）。判断：本章用对了工具；如果哪天后台任务变成 CPU 密集的 Python 计算（而非跑子进程），这套线程方案就会撞上 GIL 天花板，那时才需要 multiprocessing。

**3. Nathaniel J. Smith《Notes on structured concurrency, or: Go statement considered harmful》（2018，vorpus.org）——反方：裸线程 fire-and-forget 是 1960 年代的 goto**

它讲了什么：Trio 作者的著名檄文，把"裸的 `go`/spawn 一个并发任务"类比为 goto 语句的历史错误：fire-and-forget 的并发原语破坏了控制流的可见性——你看不出一个函数会不会派生出并发任务、任务什么时候结束、失败了谁负责，错误处理和资源清理因此系统性出错。他提出的替代是**结构化并发**：所有并发任务必须挂在显式的作用域（Trio 的 nursery）里，作用域退出前所有任务必须结束，错误沿调用栈传播、取消有明确语义。"不允许在没有 nursery 的情况下 spawn"这个看似别扭的约束，正是保证"程序的控制流图仍然是一棵树"的关键。（已于 2026-10-14 用 curl 抓取原文验证）

与本章的冲突：本章的 `threading.Thread(daemon=True).start()` 正是 Smith 批评的裸 spawn——没有 nursery，没有结构化生命周期：线程完成只往队列里塞个 ID，失败只改个 status 字段，**没有任何机制保证 agent 一定会来收尸**（用户直接退出会话，完成通知永远无人读取；daemon 线程让进程能退出，但这恰恰是以"丢弃结果"为代价）。而且"轮次开始处收集"意味着通知的可见性依赖对话继续——结构化并发视角下，这是把任务生命周期托付给了不受控的外部事件。

我的辩证判断：批评成立但有缓解因素。①本章的任务是**子进程包装器**，真正的资源（进程组）有 atexit/SIGTERM/killpg 三层清理兜底，泄漏的不是内存锁这类线程资源，风险等级低于裸线程共享内存的场景；②结果丢弃在 agent 语义下是可接受的——后台任务的价值在完成那一刻已部分实现（测试跑完了就是跑完了），通知只是锦上添花；③Smith 的方案在"同步 LLM 循环"这个宿主结构里没有干净的落地方式——你没法让 nursery 的 `async with` 作用域横跨多个 API 请求轮次。结论：本章是该约束条件下的合理妥协，但如果 harness 演进为全异步架构（asyncio 化），结构化并发就是必须补的课——线程可以裸，**生命周期不能裸**，至少要有本章这样的登记-收集-清理闭环。
