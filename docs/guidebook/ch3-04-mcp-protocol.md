# 3.4 MCP 协议——Agent 如何连接外部世界？

上一节我们拆解了工具的定义，知道了每个工具就是一个"能做的事"。但你可能已经发现一个问题：Agent 想用的工具散落在四面八方——Linear 管项目、Gmail 管邮件、Slack 管消息、GitHub 管代码……每家的接口都不一样，难道 Agent 要为每个服务单独写一套对接代码？

## 万能插头——一个标准解决所有接口问题

你出国旅行时一定遇到过这个麻烦：中国的插头到日本插不上，日本的插头到欧洲插不上，欧洲的插头到英国又插不上。每个国家一套插座标准，出门得带一堆转换器。

没有统一标准之前，AI Agent 连接外部服务也是这个局面。想用 Linear 的 API？写一套专用代码。想接 Slack？再写一套。每接一个服务就要写一套"转换器"，工作量爆炸，而且极难维护。

MCP（Model Context Protocol，模型上下文协议）就是来解决这个问题的。它定义了一套统一的"插头标准"——不管你的电器是什么牌子，只要插头符合标准，就能插上用。服务端只要按 MCP 协议暴露接口，Agent 就能直接调用，不需要任何专用代码。

## MCP 的三个核心角色

理解 MCP，只需要搞清楚三个角色：

**MCP Server（服务器）**——提供工具的一方。它就像电力公司，拥有资源，按标准接口提供电力。在 Craft Agents 里，每一个 Source（数据源）就是一个 MCP Server。Linear 是一个 Server，Gmail 也是一个 Server，它们各自拥有自己的数据和功能，但对外都走同一套协议。

**MCP Client（客户端）**——使用工具的一方。它就像电器，通过标准插头获取电力。Agent 本身就是 Client，它不需要知道 Server 内部怎么运作，只需要知道"有哪些插头可以插"。

**Tool（工具）**——服务器暴露的能力。它就像插座上的每个插孔，每个插孔提供一种服务。一个 MCP Server 可以暴露多个 Tool，比如 Linear 的 Server 可能同时提供 `createIssue`、`listIssues`、`updateIssue` 等好几个工具。

三者关系很清晰：Client 找到 Server，发现上面有哪些 Tool，然后按需调用。

## Craft Agents 的 MCP 架构

知道概念之后，我们来看看 Craft Agents 是怎么组织这些连接的。

### McpClientPool——一个智能插线板

你家里可能有一个插线板，上面同时插着台灯、手机充电器、电风扇。Craft Agents 里的 `McpClientPool`（客户端连接池）就扮演这个角色——它集中管理所有 MCP 连接，让 Agent 一次插着多个"电器"。

当你在 Craft Agents 里添加一个 Source，Pool 就为它建立连接、缓存工具列表、把工具注册给 Agent。移除 Source 时，Pool 断开连接、清理工具。整个过程动态进行——不需要重启 Agent，随时插拔。

### 两种传输方式——Wi-Fi 和 USB

Pool 支持两种 Transport（传输方式）来连接 MCP Server：

- **HTTP/SSE**：像 Wi-Fi 连接——Agent 通过网络 URL 访问远程的 MCP Server。大部分在线服务（如 Linear、Slack）用这种方式。
- **stdio**：像 USB 直连——Agent 启动一个本地子进程，通过标准输入输出通信。适合在本机运行的 MCP Server，比如文件系统工具。

不管用哪种方式连接，连上之后的行为完全一样——都是发现工具、调用工具。传输方式只是"线缆"不同，插头标准是一样的。

## 工具命名规则——一眼看出"谁家的什么"

当 Pool 把一个 MCP Server 的工具注册给 Agent 时，它不会直接用原名，而是套上一个命名规则：

```
mcp__{sourceSlug}__{toolName}
```

其中 `sourceSlug` 是服务的缩写名，`toolName` 是工具的功能名。比如：

- `mcp__linear__createIssue` —— Linear 服务的"创建 Issue"工具
- `mcp__slack__sendMessage` —— Slack 服务的"发消息"工具
- `mcp__github__createPR` —— GitHub 服务的"创建 PR"工具

这个命名规则让 Agent 和开发者一眼就能看出一个工具属于哪个服务、做什么事。Pool 内部维护了一个从代理名称到原始名称的映射——Agent 调用 `mcp__linear__createIssue`，Pool 自动把它翻译成 Linear Server 上的 `createIssue`，然后把请求路由过去。

## API Source 也走 MCP——统一入口

有趣的是，Craft Agents 里不只是原生支持 MCP 的服务才走 MCP 协议。REST API（如 Gmail、Slack）也被包装成了 MCP Server。

`ApiSourcePoolClient` 通过内存传输通道连接到进程内的 MCP Server 实例，走的是和远程 MCP Server 完全一样的协议。这意味着，不管底层是原生 MCP 还是普通 REST API，对 Agent 来说入口完全统一——都是 `mcp__{slug}__{tool}` 的形式，同一套调用流程。

这也是 MCP 的精妙之处：它不只是一种对接方式，而是一种统一的"接口语言"。只要你能用 MCP 的语法描述能力，Agent 就能用 MCP 的方式调用你。

## 内置的 craft-agents-docs 服务器

在所有 MCP Server 中，有一个特别的存在：`craft-agents-docs`。这是一个永远可用的内置服务器，提供文档查询工具——不需要配置，不需要认证，每个工作区自动连接。当 Agent 需要查询"怎么配置 Linear"这类信息时，就会调用它。在源码中，它被硬编码在 Agent 配置里，作为始终在线的基础设施。

---

MCP 协议解决了"怎么连"的问题，但 Agent 的工具箱里还有一种特殊的工具——`call_llm`。它让 Agent 能请"同事"帮忙：遇到复杂问题时，调用另一个 LLM 来做子任务，就像项目经理把一块工作交给更擅长的同事。下一节我们来看看这个"请同事帮忙"的机制是怎么回事。
