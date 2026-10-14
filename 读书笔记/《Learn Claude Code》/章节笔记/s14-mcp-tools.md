# s14 MCP Tools — 发现并调用外部工具

> 📖 《Learn Claude Code》章节笔记｜[← 返回主笔记](../笔记.md)｜状态：✅ 已完成（2026-10-14）

## 一、本章讲了什么（充分展开，讲透为止）

### 1. 本章要解决的问题

s02 以来所有工具都直接写在 `code.py` 里：工具定义是 Python 字面量，handler 是本文件函数。接入文档系统、部署平台这类外部服务时，可以继续手写 `search_docs`、`deploy_status`、`trigger_deploy`——但每增加一个服务，就要亲自维护一份工具定义、参数格式和调用代码；外部系统改了工具，课程代码也得跟着改。**工具的生产和消费被焊死在同一个文件里**。本章引入 MCP（Model Context Protocol）把这两个角色拆开：**server 提供工具列表（tools/list）和调用入口（tools/call），Harness 负责连接、命名、权限检查，并把发现的工具交给模型**。

### 2. 核心方案：连接 → 发现 → 动态组装工具池

三个新组件：

- `MCPClient`：保存某个 server 的工具定义和调用入口（register/call_tool）；
- `connect_mcp(name)`：模型可调用的工具，连接 server 并拿到工具列表；
- `assemble_tool_pool()`：**每轮**调用模型前，把基础工具 + 所有已连接 server 的工具组装成当轮工具池。

课程里的 `docs` 和 `deploy` 是**进程内模拟 server**（`MOCK_SERVERS` 字典里的工厂函数）——真实的 MCP transport（stdio/SSE/HTTP、JSON-RPC 帧）不在本章实现，本章演示的是**协议边界**：发现与调用这两个动作长什么样、接入后 Harness 侧要处理什么。

### 3. 关键机制逐条走读（code.py，549 行）

**① 循环不变，工具池变活**（`agent_loop`，第 486-496 行）：s01 的循环骨架照旧，唯一变化是每轮开头 `tools, handlers = assemble_tool_pool()`。于是有了本章最优雅的性质：**`connect_mcp` 生效于下一轮**——模型第 N 轮调用 connect，Harness 保存 client，第 N+1 轮组装时新工具自动出现在模型输入里。不需要重启、不需要重注册循环，工具集随会话生长。这正是 s02 dispatch map 的运行时化：s02 加工具要改代码，本章加工具是**数据**。

**② 命名空间隔离**（`assemble_tool_pool`，第 323-366 行）：多个 server 都可能提供 `search`、`status`，Harness 用 `mcp__{server}__{tool}` 前缀区分——这正是 s02 拓展资料里 Anthropic 工具博客"用命名空间划清边界"建议的最小实现。细节三件套：`normalize_mcp_name()` 把模型工具名不允许的字符替换成下划线；组装时**检测规范化后的冲突**（`docs.one/get.version` 和 `docs_one/get_version` 不许悄悄映射成同名）；超过 64 字符直接报错（这是模型 API 的硬约束）。

**③ 一个经典 Python 坑的示范**（第 357-360 行）：

```python
handlers[prefixed] = (
    lambda *, client=server, tool=raw_name, **kwargs:
    client.call_tool(tool, kwargs)
)
```

模型看到带前缀的名字，handler 内部仍用 server 的原始工具名调 `call_tool`。**默认参数 `client=server, tool=raw_name` 是必须的**——否则循环里所有 lambda 闭包共享变量，全部指向最后一个 server 的最后一个工具。这是教学代码里藏的真实工程陷阱。

**④ 权限：授权来自宿主，不来自 server**（第 204-210、421-427 行）：MCP server 可以在工具定义里自报 `readOnlyHint`/`destructiveHint`（mock server 就报了），但这些**只是 server 的自称**，不能作为授权依据。本章用宿主侧策略表 `MCP_HOST_POLICY`（`(server, tool) → allow/confirm`），`permission_hook` 对 `mcp__` 前缀工具查表：**未配置的外部工具默认要求用户确认**——即使 description 写着 readOnly 也不放行。fail closed，与 s03 一脉相承，并且与 MCP 官方规范的安全条款逐字呼应（见拓展资料 1）。

**⑤ 错误留在工具边界内**（`MCPClient.call_tool`，第 190-197 行）：模型漏传参数、传错字段，异常被捕获返回 `MCP error: TypeError: ...` 文本——s02"错误文本化"哲学延伸到外部工具：server 出错是模型的上下文，不是进程的终止符。

**⑥ system prompt 也动态化**（`assemble_system_prompt`，第 369-372 行）：连接 server 后 prompt 尾部追加 "Connected MCP servers: docs, deploy"——模型的自我认知跟着工具池一起更新。

### 4. 实战观察（沙盒实测）

在 `/tmp/lcc-s14` 实测两条轨迹，与 README 的"试一下"完全吻合：

1. **docs 链路**：`connect_mcp(name="docs")` → 打印 `[mcp] connected: docs -> search, get_version` → 模型随后调用 `mcp__docs__search(query="agent hooks")` 和 `mcp__docs__get_version()`（前缀名正确出现）→ 汇总"找到 3 条结果，API v2.1.0"。**三轮三个 tool_use**，发现到调用无缝衔接；
2. **deploy 链路 + 权限分级**：`connect_mcp(name="deploy")` → `mcp__deploy__status(service="web")` **按宿主策略 allow 直接放行**（`[deploy] web: running (v1.4.2)`）→ `mcp__deploy__trigger(service="web")` **触发交互确认**：终端弹出 `[permission] External tool mcp__deploy__trigger({'service': 'web'}) Allow? [y/N]`，输入 y 后才执行。同一会话里两条 MCP 工具走出了两条权限路径——策略表的差异化控制实测生效。

一个值得记录的行为细节：模型的调用顺序是"先 connect 再使用"，这是 system prompt 里 "Call connect_mcp before using a server" 一句话引导的——模型第一轮只看到 5 个基础工具 + connect_mcp，它无法也无需"猜"有哪些外部工具存在。

## 二、与前面章节的关系

**基座回到 s04**（不是紧邻的 s13）：本章从 s04 的五工具 + Hooks + 权限出发，是一条独立分支——Task（s10）、Background（s11）、Cron（s12）、Team（s13）都**刻意不带入**，它们要在 s15 Integrated Harness 才与 MCP 会师。这个安排再次印证全书结构：每条机制线独立讲透，最后合并。

**对 s02 的回应与超越**：s02 说"加工具只加一行"，但那还是要改代码；本章把工具注册变成了**运行时的数据流**——工具池从固定常量变成每轮组装的函数结果。s02 拓展资料里讨论的"工具膨胀"问题在这里获得一个新维度：MCP 让工具数量可以无限增长，命名空间前缀和宿主策略表就是对膨胀的第一道管理。

**对 s03/s04 的延伸**：权限钩子第一次面对"工具来自不可信外部"的场景，答案是策略表 + 默认 confirm——把 s03 的"执行前加一道门"从本地命令推广到外部能力。

**哲学上与 s07（Skill Loading）同源**：都是"能力按需进入上下文"——s07 是用到时加载提示词知识，本章是连接时才注入工具定义。

**循环骨架**：第四次验证 s01——`agent_loop` 的结构一行未改，只是 tools/system 两个参数从常量变成了每轮计算的函数。

## 三、实战场景（怎么用）

1. **给团队内部系统接 agent**：把内部的部署平台、工单系统、知识库各包一个 MCP server，agent harness 只需维护一份策略表；新服务上线 = 启动新 server，harness 代码零改动。本章的 `MCP_HOST_POLICY` 直接对应生产里的"工具审批矩阵"。
2. **安全的第三方工具接入**：接第三方 MCP server 时照搬本章原则——**不信 server 自报的 readOnlyHint**，宿主侧默认 confirm，逐工具评审后才改 allow；配合 description 审计（防 tool poisoning，见拓展资料 2）。
3. **设计自己的工具协议时的命名纪律**：`mcp__{server}__{tool}` 前缀 + 规范化冲突检测 + 长度上限检查，这套组合拳可直接复用到任何多来源工具系统。

## 四、我的思考

*（待补充——你的联想、认同与质疑写在这里）*

## 五、我的问题与解答

- ❓**为什么工具池要每轮重新 `assemble_tool_pool()`，而不是 connect 时一次性更新全局 TOOLS？**
  → ✅解答：① **正确性**：MCP 协议允许 server 动态增删工具（规范里有 `listChanged` 通知能力），每轮组装保证模型看到的永远是当前真实工具集；如果只在 connect 时快照，server 侧后续变更就丢了；② **一致性**：tools 数组和 handlers 字典、policies 表由同一次组装产出，三者不会错位。代价是每轮重复构造——对教学规模是微秒级，换来的是状态单一时点的一致性，值。
- ❓**为什么不能信任 server 自报的 `readOnlyHint`？它撒谎的动机是什么？**
  → ✅解答：两个层面的威胁。① **server 可能是恶意的**：MCP 生态鼓励"即插即用"地连接第三方 server，恶意 server 可以把窃取数据的工具标成 readOnly 诱使宿主放行——这就是 tool poisoning 攻击（见拓展资料 2）；② **server 可能只是写错了**：hint 是文档性元数据，没有强制执行机制。授权决策必须基于**宿主自己评审过的策略**，server 元数据至多作为评审参考。MCP 官方规范把这点写成了 MUST 级条款（"clients MUST consider tool annotations to be untrusted unless they come from trusted servers"）——本章的实现与规范逐字对齐。
- ❓**本章没有实现真实 transport（stdio/HTTP），那学到的有多少能迁移到真实 MCP？**
  → ✅解答：迁移率很高。真实 MCP 客户端只是在本章 `MCPClient` 外面包一层 JSON-RPC over stdio/SSE/HTTP 的传输，`tools/list` 与 `tools/call` 的语义、schema 形状、错误返回约定完全相同；Harness 侧的全部工作——前缀命名、冲突检测、宿主策略、权限钩子、错误文本化——**一字不改地适用**。本章省掉 transport 是把"协议语义"和"网络细节"分离的教学取舍：前者是 harness 作者真正要懂的，后者交给官方 SDK。

## 六、关键摘记

⭐ connect_mcp 生效于下一轮——工具池每轮重组装，工具集随会话生长｜⭐ `mcp__{server}__{tool}` 前缀 + 规范化冲突检测 + 64 字符上限 = 多来源工具的命名纪律｜⭐ 授权来自宿主配置，不来自 server 的 annotations（规范级 MUST）｜💡 handler 里 lambda 的默认参数绑定（client=server）——闭包陷阱的教科书示范｜💡 MCP 把"工具生产"和"工具消费"拆成两个角色，harness 从此只维护策略不维护实现

## 七、代码实战

跑通 ✅（沙盒 `/tmp/lcc-s14`：connect docs → mcp__docs__search / get_version 正常调用；connect deploy → status 按策略 allow 直放行、trigger 触发 `[permission] Allow? [y/N]` 交互确认，输 y 后执行；宿主策略分级实测生效）　改造实验 ☐（待做：构造带非法字符名的 mock server 工具，验证 normalize 冲突检测报错路径）

## 八、📚 本章拓展资料与对照（每条结合本章展开）

**1. MCP 官方规范 2025-06-18《Tools》章节（modelcontextprotocol.io）——权威源头 + 安全条款互证**

它讲了什么：规范把 Tools 定义为 **model-controlled**——"模型基于上下文理解和用户提示自动发现并调用工具"，协议本身不强制特定交互模式。两个协议动作正是本章演示的：client 发 `tools/list` 发现工具（支持分页）、发 `tools/call` 调用（JSON-RPC，`params` 里是 name + arguments）；server 可声明 `listChanged` 能力以便工具列表变化时通知 client。安全部分有两条硬条款：① **"SHOULD always be a human in the loop with the ability to deny tool invocations"**——建议宿主提供工具暴露的可见 UI、调用指示和确认提示；② **"clients MUST consider tool annotations to be untrusted unless they come from trusted servers"**——`readOnlyHint` 等 annotations 对不可信 server 一律视为不可信。

与本章的对照：逐条吻合。本章的 permission_hook（外部工具未配置默认 confirm）就是第 ① 条的实现；`MCP_HOST_POLICY` 宿主策略表、明确不信 `readOnlyHint`，就是第 ② 条的逐字落地——读规范原文之前我以为"不信 server 自称"只是作者的谨慎，读完发现这是**规范级要求**，本章的选型不是风格而是合规。差异：真实规范还有分页、listChanged 通知、结构化 content 返回等本章未实现的细节——都不影响协议心智模型。（已于 2026-10-14 查证原文，https://modelcontextprotocol.io/specification/2025-06-18/server/tools ）

**2. Invariant Labs《MCP Security Notification: Tool Poisoning Attacks》（2025-04）——反方/批判视角：本章没讲的攻击面**

它讲了什么：安全公司 Invariant 披露的 MCP 根本性漏洞——**Tool Poisoning Attack（TPA）**：恶意 MCP server 在**工具描述（description）里嵌入恶意指令**，这些指令对用户不可见（UI 通常只显示工具名和参数摘要）但对模型完全可见，可操纵模型执行未授权操作：窃取敏感文件内容、把数据外泄到攻击者服务器，甚至**覆盖其他可信 server 提供的指令**、劫持整个 agent 的行为。实验表明 Anthropic、OpenAI 生态及 Cursor 等主流 MCP client 都受影响。本质是间接提示注入（indirect prompt injection）在工具元数据通道上的变种。配套发布了 MCP-Scan 扫描工具。

与本章的对照：本章的 mock server 是善意的，所以攻击面没有展开——但**本章代码里 `description` 原样进入模型上下文**（`assemble_tool_pool` 直接透传 `tool_def.get("description")`），这正是 TPA 的注入通道。本章做对了另一半：授权决策不依赖 description/annotations（宿主策略表）——TPA 能骗模型"想"做坏事，但过不了宿主策略的门（未配置默认 confirm）。我的辩证判断：**宿主策略是必要的但不够**——TPA 的完整缓解还需要三层本章没有的东西：连接前用 MCP-Scan 类工具审计 description、UI 向用户展示完整工具描述、限制 server 工具结果里夹带指令（结果也是注入通道）。接任何第三方 MCP server 前，本章的 `MCP_HOST_POLICY` 应配合描述审计一起用。（已于 2026-10-14 查证原文，https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks ）
