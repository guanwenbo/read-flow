# s04 Hooks — 挂在循环上，不写进循环里

> 📖 《Learn Claude Code》章节笔记｜[← 返回主笔记](../笔记.md)｜状态：✅ 已完成（2026-10-14）

## 一、本章讲了什么（充分展开，讲透为止）

### 1. 本章要解决的问题

s03 把 `check_permission()` 硬编码进了循环。能用，但每加一个横切逻辑——"记录每次 bash 调用"、"edit 之后自动 git add"、"出错时发通知"——都得再改 `agent_loop`。README 给了一段"循环膨胀预言"：`log_to_file(block)`、`check_permission(block)`、`notify_slack(block)`、`auto_git_add(block)` 一行行堆进去，循环很快认不出来。问题的本质是一句话：**你想扩展的是 Agent 的行为，但你改的却是循环本身**。循环应该是稳定的核心，扩展应该挂在外面。

### 2. 核心方案：注册表 + 触发器（观察者模式的 agent 版）

s03 的循环和权限逻辑完全保留，唯一变动是把 `check_permission()` 从循环体内搬到一个 hook 上，循环改为只调用 `trigger_hooks("PreToolUse", block)`，**由注册表决定跑什么**。四个事件覆盖一个完整的 agent cycle：

| 事件 | 触发时机 | 典型用途 |
|------|---------|---------|
| UserPromptSubmit | 用户输入提交后、进 LLM 前 | 输入验证、注入上下文 |
| PreToolUse | 工具执行前 | 权限检查、日志 |
| PostToolUse | 工具执行后 | 副作用（自动 git add）、输出检查 |
| Stop | 循环即将退出时 | 收尾统计、决定是否继续循环 |

机制本身只有 20 行：

```python
HOOKS = {"UserPromptSubmit": [], "PreToolUse": [], "PostToolUse": [], "Stop": []}

def register_hook(event, callback): HOOKS[event].append(callback)

def trigger_hooks(event, *args):
    for callback in HOOKS[event]:
        result = callback(*args)
        if result is not None:   # 返回值 ≠ None → hook 说"停"
            return result
    return None
```

**返回值协议是整个设计的枢纽**：hook 返回 `None` = "我只是看看"；返回非 None = "拦截/接管"。`PreToolUse` 返回字符串 → 本次工具执行被阻止（字符串作为 tool_result 回喂模型）；`Stop` 返回字符串 → 循环不退出，字符串作为 user 消息注入、继续循环。`UserPromptSubmit` 和 `PostToolUse` 的返回值不参与控制流（本章定位为纯观察点）。

### 3. 代码走读（code.py，280 行）

- **s03 的权限逻辑原样搬家**：`permission_hook` 注册为 PreToolUse 的第一个回调，deny list 命中返回 `"Permission denied by deny list"`，规则命中弹 `[y/N]` 审批、用户拒绝返回 `"Permission denied by user"`。循环里 s03 的 `if not check_permission(block)` 变成 `blocked = trigger_hooks("PreToolUse", block)`——**一行换一行，能力不变，结构变了**。
- **log_hook**：PreToolUse 的第二个回调，打印每次工具调用，返回 None（纯观察）。
- **large_output_hook**：PostToolUse，输出超 10 万字符时提醒。⚠️ 代码走读发现它是**事实上的死代码**：`run_bash` 输出截断在 50000 字符，永远到不了 100000 的阈值——钩子本身没毛病，但它守护的条件已被下游截断逻辑提前消化。这是个好教训：hook 的触发条件要和数据流的实际约束对账。
- **context_inject_hook**：UserPromptSubmit，打印工作目录（名字叫 inject 但实际只打印，真正的上下文注入留作示意）。
- **summary_hook**：Stop，统计本会话 tool_result 数量。注意它扫的是**整个 messages 历史**，所以多轮对话中计数是累计值（实测：1 → 2 → 3 递增）。
- **Stop 的"反悔"能力**值得单独记住：
  ```python
  if not tool_calls:
      force = trigger_hooks("Stop", messages)
      if force:
          messages.append({"role": "user", "content": force})
          continue   # hook 说"不许停"，注入消息继续循环
      return
  ```
  这是 hook 第一次获得**改变控制流方向**的能力——不只是拦截行动，还能拒绝结束。真实 Claude Code 的 Stop hook 正用来做"测试没过不许停"这类强制（见拓展资料 1）。

### 4. 实战观察（沙盒实测，/tmp/lcc-s04）

两轮会话实测，四个事件全部触发，另有三个文档没写的发现：

- **正常链路**：每次提问先看到 `[HOOK] UserPromptSubmit`，工具执行前看到 `[HOOK] write_file([...])` 日志，循环退出时看到 `[HOOK] Stop: session used N tool calls`。✅
- **权限照常工作**：`rm` 触发 `[permission] Potentially destructive command` 审批，答 y 执行；`sudo` 被 deny list 拦截。✅（s03 能力无损迁移）
- **发现 1：注册顺序 = 优先级，短路导致"被拒的操作没有日志"**。`permission_hook` 注册在 `log_hook` 之前，当 rm 被拦截时 `trigger_hooks` 提前 return，**log_hook 根本没机会跑**——输出里能看到 `[permission]` 提示，但看不到 `[HOOK] bash(['rm ...'])` 日志。也就是说：审计最想要的"用户拒绝了什么"恰恰没被记录。短路语义把"顺序"变成了隐式的优先级配置。
- **发现 2：s03 的两个缺陷原样继承**。审批 `input()` 在 stdin 耗尽时 EOFError 崩溃（实测 Traceback 复现）；用户拒绝后模型依旧换姿势重试——答了一次 n 之后，模型连续尝试了 `rm && ls`、`rm test2.txt`、`rm -f ...; echo exit=$?` 共 4 个变体，每次都弹审批。**hook 化解决的是"代码组织"问题，不改变行为语义**——这一点本身也是对 hook 能力的边界说明。
- **发现 3（对照）**：`s04` 的 SYSTEM prompt 从 s03 的 "All destructive operations require user approval" 换成了 "Use tools to solve tasks. Act, don't explain."——行为引导更激进，但权限 hook 兜底还在，"prompt 鼓励行动 + hook 负责设卡"的分工很清晰。

## 二、与前面章节的关系

**这是全书第一次"架构重构"而不是"功能叠加"**。

- **承 s03 的引线**：s03 章末自问"想加日志？想自动 git commit？循环很快膨胀"——本章直接回答：把扩展逻辑从循环里抽出来挂到事件上。而 s03 的 `check_permission` 成为这次重构的**第一个受益者**：它没有被删除，而是改头换面成为 `permission_hook`。前一章的机制是本章最好的测试用例——重构后权限行为实测无损。
- **验证 s01 断言（第三个证据）**：s01 笔记问题 1 里我预测"hooks 的注入点是循环的边缘"——完全命中。循环骨架（发消息→查 tool_use→执行→回喂）第四次原样保留；这次连"执行前检查"这一行都被参数化成了 `trigger_hooks("PreToolUse", ...)`。循环里剩下的全是抽象调用，没有任何具体策略。
- **与 s02 的对照**：s02 证明"加能力不用改循环"（dispatch map），s04 证明"改组织方式也不用改循环语义"（事件注册表）。一个是横向扩展点（多工具），一个是纵向扩展点（横切关注）。
- **埋下新问题**：hook 多了之后顺序敏感、短路跳过后续 hook（实测发现 1）——这正是 AOP 三十年前就被批评的"obliviousness"问题（见拓展资料 3）。

## 三、实战场景（怎么用）

1. **审计日志不落陷阱**：把日志类 hook 和拦截类 hook 分开注册——日志用"总是执行"的通道（或注册在最前、返回 None），拦截钩子放后面。本章实测的短路问题说明：把所有 hook 塞进一个短路列表，被拒操作就会漏记，审计日志恰恰会丢掉最关键的记录。
2. **CI 质量门**：注册一个 Stop hook——循环想退出时检查"测试跑了吗？通过了吗？"，没跑就返回字符串 `"Tests have not been run yet. Run pytest first."` 强制继续。这是 Stop hook "拒绝结束"能力的杀手级用法，真实 Claude Code 官方文档也这么推荐。
3. **自动副作用**：PostToolUse 监听 `edit_file`/`write_file`，触发后自动 `git add` 或跑 prettier/black——不用碰循环一行。
4. **给自己的系统画事件模型前先抄这四个**：输入前（UserPromptSubmit）、行动前（PreToolUse）、行动后（PostToolUse）、结束前（Stop）——这是一个 agent cycle 的最小完备切分，官方 Claude Code 也是同族事件再加细（SessionStart、SubagentStop 等）。

## 四、我的思考

*（待补充——你的联想、认同与质疑写在这里）*

## 五、我的问题与解答

- ❓**为什么本章只有 PreToolUse 和 Stop 的返回值参与控制流，UserPromptSubmit / PostToolUse 只能"看"不能"拦"？**
  → ✅解答：这是克制的 API 设计——先给每个事件开放最小语义（观察 + 副作用），控制流能力只给确实有拦截需求的两个点。好处是 hook 作者的心智负担小：写 UserPromptSubmit hook 不用担心误伤输入流。但这个边界在实践中被突破了：真实 Claude Code 的 UserPromptSubmit 后来也支持 block（拦截提示词并反馈原因），PostToolUse 也能给模型返回追加反馈。**结论：事件语义会随需求生长，本章给的是最小可行集，不是终态**（已于官方文档验证）。
- ❓**trigger_hooks 的短路语义（第一个非 None 就 return）和"全部跑完收集结果"哪个对？**
  → ✅解答（实测佐证）：短路的好处是"拦截"语义自然——一个 hook 说不，后面不用表态；代价是**顺序变成隐式优先级**，且观察型 hook 会被拦截型 hook 跳过（实测：rm 被拒时 log_hook 没执行）。更稳的设计是分层：observer 类 hook 无条件全部执行，interceptor 类 hook 走短路表决；或在注册时声明类型。本章的 20 行实现选择了简单，把坑留给了使用者去发现——教学上这反而有效，亲手踩过"被拒操作没日志"才理解为什么要分层。
- ❓**hook 化之后，"循环里还有什么"？如果一切皆可 hook，循环本身会不会变成空壳？**
  → ✅解答：不会。循环保留的是**不可替代的控制流**：LLM 调用、tool_use 判定、结果回喂、终止条件——这些是"业务无关的骨架"；hook 接管的是**策略**：查不查权限、记不记日志、准不准停。判据很清晰：换个 agent 产品也要有的，留在循环；因团队/项目/部署环境而异的，挂到 hook。这和 Web 框架的分层同构（框架核心 vs 中间件）。

## 六、关键摘记

⭐ "挂在循环上，不写进循环里"——扩展挂事件，循环保持干净｜⭐ 返回值协议：None = 观察，非 None = 拦截/接管；Stop hook 返回字符串可拒绝结束｜⭐ 注册顺序 = 隐式优先级，短路会跳过后续 hook（被拒操作因此没有日志）｜💡 本章是全书第一次架构重构而非功能叠加：能力不变，结构变｜💡 large_output_hook 阈值 10 万 > bash 截断 5 万，是守护永远触不到条件的死钩子——hook 条件要和数据流约束对账｜💡 hook 化只改组织不改语义：s03 的 EOFError 和拒绝重试原样继承

## 七、代码实战

跑通 ✅（沙盒 /tmp/lcc-s04 实测：四个事件全部触发；权限 hook 无损承接 s03 能力；Stop hook 累计计数 1→2→3；实测发现"短路导致被拒操作无日志""注册顺序即优先级""EOFError 与拒绝重试缺陷原样继承"）　改造实验 ☐（待做：注册一个 Stop hook——todo 未完就返回字符串强制继续，验证"拒绝结束"能力）

## 八、📚 本章拓展资料与对照（一搜，每条详细展开）

**1. Claude Code Hooks 官方文档（code.claude.com/docs/en/hooks，已于 2026-10-14 curl 验证）——赞同：本章的生产放大版**

它讲了什么：官方 hooks 系统与本章几乎同构：同名事件（PreToolUse / PostToolUse / UserPromptSubmit / Stop）之外还有 SessionStart、SessionEnd、SubagentStop、TaskCreated 等更细的事件族，文档明确按频率分三档——**per session（SessionStart/End）、per turn（UserPromptSubmit、Stop）、每次工具调用（PreToolUse、PostToolUse）**，与本章四个事件的定位一一对应。生产化差异：hook 不是进程内 Python 回调，而是**外部 shell 命令 / HTTP 端点 / MCP 工具**——事件上下文以 JSON 经 stdin 传入，hook 用 **exit code 2 表示阻断**、或输出 JSON 做细粒度 decision 控制；另有超时、失败策略（hook 崩溃时是否放行）、`once` 一次性 hook 等工程选项。

与本章的对照：**一致**：事件模型、拦截语义、"决定权在 hook 不在循环"全部对应——本章是官方系统的 20 行白盒版。**差异**：① 官方 hook 是独立进程（隔离性、语言无关），本章是进程内函数（简单但 hook 崩溃会带走整个 agent——实测的 EOFError 正是这种耦合的代价）；② 官方 UserPromptSubmit 也能 block、PostToolUse 也能反馈模型，突破了本章"只有两个事件参与控制流"的边界。我的判断：本章教的是**事件切分和决策协议**，官方文档展示的是把它**产品化需要的隔离与容错**；读完本章再看官方文档的 exit code / JSON decision 部分，理解成本几乎为零。

**2. 面向切面编程 AOP（Kiczales et al., ECOOP 1997 提出；Wikipedia "Aspect-oriented programming" 条目，已于 2026-10-14 curl 验证）——学术源头**

它讲了什么：AOP 是 1997 年 Xerox PARC 提出的编程范式，目标是分离**横切关注点**（cross-cutting concerns）——日志、权限、事务这类"散落在每个业务函数里却与业务无关"的逻辑。做法是：不修改原代码，而是在外部声明 **pointcut**（在哪些连接点切入）+ **advice**（切入后做什么，before/after/around）。

与本章的对照：概念映射严丝合缝——`PreToolUse`/`PostToolUse` 就是 before/after advice，事件名就是 pointcut，权限检查/日志就是教科书级的横切关注点。README 里"循环膨胀预言"那段代码（`log_to_file`/`check_permission`/`notify_slack` 堆在循环里）正是 AOP 要消灭的 **scattering**（同一关注点散落多处）与 **tangling**（多种关注点缠在一处）的现场样本。本章等于说：**30 年前 AOP 对业务代码的诊断，原样适用于 agent loop**。我的判断：这不是巧合而是必然——agent loop 就是一种新的"业务主流程"，权限/日志/统计这些横切需求必然出现，解决思路必然趋同。学这一章顺手补了半个世纪的软件工程史。

**3. AOP 的批评（Wikipedia 同条目 "Criticism" 节，已于 2026-10-14 curl 验证）——反方：hook 的隐性控制流是有代价的**

它讲了什么：对 AOP 最基本的批评是**控制流被遮蔽**（control flow is obscured）——批评者认为它比 GOTO 更糟，近乎笑话里的 COME FROM 语句：代码自己看不出会在哪里被插入行为。根源是 **obliviousness**（无意识性）：被切入的代码没有任何迹象表明 advice 会生效；另有 **fragile pointcut problem**（脆弱的切点）：重构改了函数名/结构， silently 导致 advice 失配或误配，问题被认为是难解的。

与本章的对照与冲突：本章的卖点"循环保持干净"在批评者眼里正是缺点——读 `agent_loop` 只看到 `trigger_hooks("PreToolUse", block)`，**实际会跑什么、跑几个、什么顺序，全部不可见**，要去注册处翻，还要理解短路语义。而且这不是理论担忧：本章实测的"被拒操作没有日志"就是 obliviousness 的直接后果——循环的作者不知道 permission_hook 会短路掉 log_hook，log_hook 的作者也不知道自己被排在了会短路的钩子后面。我的辩证判断：**批评成立但有剂量关系**——四五个 hook 时，"循环干净"的收益远大于"控制流隐性"的成本；hook 多到互相依赖顺序、互相短路时，就必须引入显式优先级、observer/interceptor 分层、或官方那样的"JSON decision 协议 + 文档化事件语义"。换句话说：hook 不是免费的解耦，它把"显式 but 混乱"换成了"整洁 but 隐性"，工程成熟度的标志是知道什么时候该把隐性重新显性化。
