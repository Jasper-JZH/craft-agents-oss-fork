# 4.3 工具代理——子进程的工具怎么在主进程跑？

上一节我们了解了 JSONL 通信协议——主进程和子进程靠一行行 JSON 消息来对话。但有一个问题还没回答：子进程里的 AI 想用工具时，有些工具自己并没有，只能请主进程代劳。这种"你下单、我来做"的模式，就是本节的主角——工具代理（Tool Proxy）。

## 外卖代购：子进程想吃火锅

想象你住在一个没有厨房的宿舍里，突然特别想吃火锅。怎么办？你打电话给外面的火锅店，告诉他们你要什么锅底、什么配菜（下单），店里做好后给你送过来（收货）。你自己没有锅，但你照样吃上了火锅。

子进程的处境一模一样。AI 模型决定调用一个工具（想吃火锅），但这个工具的执行环境在主进程那边（锅在火锅店），子进程自己没法做。于是它通过 JSONL 发一条 `tool_execute_request` 消息（打电话下单），主进程执行完毕后回一条 `tool_execute_response`（外卖送到），子进程把结果交给 AI，对话继续。

## 为什么锅在火锅店？

你可能会问：干嘛不把工具直接放进子进程里？非得大老远让主进程代劳？原因有三个：

**第一，MCP 连接池在主进程。** 所有的外部服务连接——Gmail、Slack、Notion——都由主进程的 `McpClientPool`（MCP 客户端连接池）统一管理。连接是稀缺资源：建立要时间、维护要心跳、断线要重连。如果每个子进程各建一套连接，既浪费又混乱。集中管理就像所有外卖单都走同一家火锅店的后厨——一个厨房服务所有客人。

**第二，权限检查在主进程。** 上一章详细介绍的 PreToolUse 六步安检管道，运行在主进程里。权限策略、Source 激活状态、用户确认弹窗——这些都需要访问主进程的状态。如果子进程自己检查权限，要么把所有安全逻辑复制一份（容易不同步），要么每次都要跟主进程通信（那还不如直接让主进程做）。

**第三，会话级工具的逻辑在主进程。** 像 SubmitPlan（提交计划）、call_llm（调用另一个模型）、spawn_session（启动子会话）这些工具，它们的执行依赖主进程的会话上下文和 UI 回调。子进程没有这些上下文，没法独立完成。

## 工具代理的完整流程

让我们跟着一次完整的工具调用走一遍流程：

**第 1 步：AI 决定使用工具。** AI 模型在生成回复时，决定调用 `mcp__gmail__sendEmail` 这个代理工具。注意，子进程只拿到了这个工具的"菜单"（名称、参数描述），并没有真正的执行代码。

**第 2 步：子进程发送 `tool_execute_request`。** 子进程通过 JSONL 向主进程发一条消息，包含请求 ID、工具名和参数：

```json
{ "type": "tool_execute_request", "requestId": "proxy-1714...", "toolName": "mcp__gmail__sendEmail", "args": { "to": "...", "subject": "...", "body": "..." } }
```

**第 3 步：主进程通过 `routeToolCall()` 路由。** 主进程的 `handleToolExecuteRequest()` 收到请求后，调用 `routeToolCall()` 来决定由谁来执行这个工具。路由规则很清晰：

- 以 `mcp__` 开头的工具名（如 `mcp__gmail__sendEmail`）→ 交给 `McpClientPool.callTool()` 执行
- 会话级工具（如 SubmitPlan、config_validate）→ 交给 `SESSION_TOOL_REGISTRY` 里的 Handler（处理器）
- `call_llm` → 走 `preExecuteCallLlm()` 管道
- `spawn_session` → 走 `preExecuteSpawnSession()` 管道
- `browser_tool` → 走 `executeBrowserToolCommand()`

就像火锅店的前台接到订单后，根据菜品分到不同的后厨窗口——涮锅归涮锅区，烤串归烤串区。

**第 4 步：主进程执行工具，获得结果。** 对应的 Handler 执行完毕，返回结果（可能是成功内容，也可能是错误信息）。

**第 5 步：主进程发送 `tool_execute_response`。** 主进程把结果打包回送：

```json
{ "type": "tool_execute_response", "requestId": "proxy-1714...", "result": { "content": "Email sent successfully", "isError": false } }
```

**第 6 步：子进程把结果交给 AI。** 子进程收到响应，把内容交给 AI 模型，AI 根据结果继续生成后续回复。

## 权限检查也是代理的

上面说的是工具执行阶段的代理。但在执行之前，还有一道关卡——权限检查，同样是子进程拜托主进程做的。

流程类似，但消息类型不同：

1. 子进程在执行任何工具前，先发一条 `pre_tool_use_request`（包含工具名和参数）
2. 主进程运行 PreToolUse 六步管道
3. 主进程回复 `pre_tool_use_response`，结果有三种：
   - **allow**（允许）：放心执行
   - **modify**（修改）：参数需要调整后再执行（比如把 `path` 规范化为 `file_path`）
   - **block**（阻止）：操作不允许，附带拒绝原因
4. 子进程根据结果决定继续、修改参数、还是中止

在 `ask` 模式下，如果权限检查需要用户确认，主进程会弹出确认窗口，等用户选择后再把结果送回子进程。

## wrapToolsWithHooks()：统一代理包装

你可能会想：每个工具都要自己写权限检查和代理逻辑，岂不是太麻烦？确实，所以源码里有一个 `wrapToolsWithHooks()` 函数，自动给所有工具套上一层代理包装。它做了三件事：

**执行前——权限检查。** 调用 `requestPreToolUseApproval()`，把 `pre_tool_use_request` 发给主进程，等待审批。如果被 block，直接抛错；如果被 modify，用修改后的参数继续。

**执行后——大响应自动摘要。** 有些工具的输出特别长（比如读了一个大文件），AI 处理起来既慢又浪费 Token。包装器会检测输出大小，超过阈值时用 Haiku 模型自动压缩超长输出——只保留关键信息，丢掉冗余内容。就像火锅店送的菜单太长，前台帮你划出重点菜。

**参数名规范化。** Pi SDK 里的工具参数叫 `path`，但主进程的权限系统期望 `file_path`。包装器在权限检查前自动把 `path` 重命名为 `file_path`，确保两边对得上。

## 推测性预取（Speculative Prefetch）：并行下单不等串行上菜

还有一个精巧的优化值得一提。

正常情况下，AI 模型发出的工具调用是串行执行的——第一个做完才做第二个。但如果 AI 一次发出多个 `call_llm` 调用（比如同时问三个不同的问题），串行等待就太慢了。

子进程的做法是：当它检测到 AI 在一条消息里发出了两个以上的 `call_llm` 调用时，在 `message_end` 事件触发的那一刻，**并行**发送所有 `tool_execute_request` 给主进程，不等前一个完成再发下一个。每个请求的结果缓存在 `prefetchCache` 里，等工具按顺序执行到那一步时，直接从缓存取结果，而不是重新发请求。

这就像你同时点了三道菜，三道菜同时做，而不是等第一道做完才点第二道。总等待时间从"三道菜依次做"变成了"最慢的那道菜做完"。

## 小结

工具代理的核心思想很简单：子进程负责"想"（AI 决定用什么工具），主进程负责"做"（执行工具、检查权限、管理连接）。两者通过 `tool_execute_request` / `tool_execute_response` 和 `pre_tool_use_request` / `pre_tool_use_response` 这两对消息完成协作。`wrapToolsWithHooks()` 让这套机制对每个工具都透明生效，推测性预取则让并行调用不再被串行拖慢。

那么，用户在界面输入一个模型名（比如 "Claude Sonnet 4" 或 "GPT-4o"），系统怎么知道该去哪个 API 地址、用哪个 Provider？下一节我们来看模型解析（Model Resolution）。
