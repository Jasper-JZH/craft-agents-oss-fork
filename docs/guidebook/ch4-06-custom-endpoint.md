# 4.6 自定义端点——接入你自己的模型

上一节我们了解了 OAuth Token 刷新——认证凭证怎么自动续签，让 Agent 不会因为"签证过期"而中断工作。但如果你压根不想走这些官方渠道呢？如果你想让 Agent 用本地跑的模型、用自己部署的服务、甚至用某个小众的 API 网关呢？这就需要本节的主角：自定义端点（Custom Endpoint）。

## 自建充电桩：官方充电站覆盖不到的地方

想象你买了一辆电动车。官方运营着一批充电站——Anthropic 充电站、OpenAI 充电站、Google 充电站——插上就能充，按度数收费。但问题来了：你家住在偏远小区，最近的充电站开车要半小时；或者你对电费敏感，想用自家屋顶太阳能发的电；又或者你的行车数据不想上传到任何云端。怎么办？自己建一个充电桩。

自定义端点就是这个"自建充电桩"。它让你可以指定任何一个 API 地址作为模型服务来源，不再被绑定在官方 Provider（提供者）的充电站里。你插上自家的充电桩，Agent 照样跑，而且用的是你自己选的电。

## 什么是自定义端点？

简单说，自定义端点就是一个**你自己指定**的 API 地址（baseUrl），Craft Agents 不会去官方的 Anthropic 或 OpenAI 服务器，而是直接跟你给定的地址通信。它不走官方 Provider 的认证流程，而是用你提供的 API Key（如果需要的话）直接调你的服务。

在源码里，自定义端点对应的 Provider 类型叫 `pi_compat`——"compat"是"compatible"（兼容）的缩写，意思是"兼容模式"。它的认证类型有两种：

- **api_key_with_endpoint**：需要 API Key + 自定义 API 地址（比如你买了 OpenRouter 的服务）
- **none**：不需要认证（比如本地跑的 Ollama）

## 支持的 API 格式

不同的模型服务说话的"方言"不一样。Craft Agents 目前支持两种 API 格式（CustomEndpointApi）：

- **openai-completions**：OpenAI 的聊天补全格式。这是最通用的格式，几乎所有的第三方模型服务都兼容它——你可以把它理解为充电行业的"国标接口"，绝大多数充电桩都支持。
- **anthropic-messages**：Anthropic 的消息格式。如果你部署的服务兼容 Anthropic 的协议，选这个。

选哪个格式取决于你的模型服务说的是哪种"方言"。大多数情况下选 `openai-completions` 就对了。

## 可以接入哪些模型服务？

既然自定义端点是一个通用的"充电接口"，那具体能插哪些"充电桩"呢？以下是几个常见的场景：

**Ollama**——在你自己电脑上跑开源模型。Llama、Mistral、Qwen……一条命令就能启动，不需要 API Key，不需要联网。对隐私敏感的数据处理场景，这是最安全的选择。baseUrl 通常是 `http://localhost:11434`。

**vLLM**——高性能推理服务器。如果你有一台带 GPU 的服务器，想自己部署模型并且跑得更快，vLLM 是个不错的选择。它兼容 OpenAI 格式，所以 API 格式选 `openai-completions`。

**LM Studio**——桌面应用，一键运行本地模型。有图形界面，对不熟悉命令行的用户很友好。同样兼容 OpenAI 格式，启动后会在本地开一个 API 端口。

**OpenRouter**——一个 API 访问数百个模型。它本身不是模型，而是一个"模型超市"——你用它的 API Key 就能调用各种模型，包括 Claude、GPT、Gemini 等等。通过 OpenRouter 使用模型时，模型 ID 格式是 `provider/model-name`，比如 `anthropic/claude-opus-4.7`。

**自建服务**——任何兼容 OpenAI 或 Anthropic API 格式的服务。无论你用的是 FastChat、Text Generation Inference，还是自己写的一个 API 转发层，只要它说对"方言"，Craft Agents 就能跟它对话。

## 注册自定义模型的流程

好了，知道能接什么了。那具体怎么接？核心流程分三步：

**第一步：在 PiModelRegistry 中注册 `custom-endpoint` Provider。** Pi SDK 的模型注册表（PiModelRegistry）需要一个 Provider 名称来组织模型。自定义端点统一使用 `custom-endpoint` 作为 Provider 名。源码中调用 `registerProvider('custom-endpoint', ...)` 来完成注册，传入的配置包括 baseUrl、apiKey、API 格式和模型列表。

**第二步：提供连接参数。** 具体来说需要：
- **baseUrl**：你的 API 地址，比如 `http://localhost:11434/v1`
- **apiKey**：API 密钥。本地服务不需要，远程服务通常需要
- **api**：API 格式，`openai-completions` 或 `anthropic-messages`
- **models**：模型列表，告诉系统你有哪些模型可用

**第三步：定义模型参数。** 每个模型定义包含：
- **名称**（id）：模型的标识符，比如 `llama3`、`mistral-7b`
- **上下文窗口大小**（contextWindow）：默认 131,072（131K）Token。如果你知道你的模型支持更大的上下文，可以手动指定
- **最大输出 Token**（maxTokens）：默认 8,192（8K）
- **费用**（cost）：本地模型全部为 0——你的电，当然不算钱
- **输入类型**（input）：默认只有 `text`；如果模型支持图片，需要显式开启 `supportsImages`

源码中 `buildCustomEndpointModelDef()` 函数负责构造这些模型定义。它很务实：既然自定义端点的模型能力无法自动探测（不像官方 API 可以查询），那就用合理的默认值，同时允许你在连接级别或单个模型级别上覆盖。

## 本地模型不需要 API Key

这是一个贴心的细节。当你配置的 baseUrl 是 localhost 地址（`localhost`、`127.0.0.1` 或 `::1`）时，系统知道这是本地服务，不需要认证。但 Pi SDK 内部要求每个注册的 Provider 都有一个 apiKey 字段，否则会报错。怎么解决？用一个占位符 `'not-needed'`。

源码中的逻辑是：检测到本地地址时，自动把 apiKey 设为 `'not-needed'`；如果既不是本地地址又没有 API Key，则打一条警告日志——"non-localhost endpoint without API key, requests will likely fail"。这个判断用的是 `isLocalhostUrl()` 函数，解析 URL 的 hostname 后判断是否为回环地址。

## 模型解析的优先级

上一节我们讲了模型解析，但没有提到自定义端点的优先级。当用户选择了一个模型 ID 时，`resolvePiModel()` 按以下顺序查找：

1. 如果 `preferCustomEndpoint` 为 true（即当前连接有自定义端点配置），优先在 `custom-endpoint` Provider 里找
2. 用 `piAuthProvider` 做精确匹配
3. 全量扫描所有模型
4. 按 `custom-endpoint`、`anthropic`、`openai`、`google` 的顺序逐个 Provider 尝试

注意，`custom-endpoint` 始终在回退列表里。这意味着即使你用的是官方 Provider，系统也不会遗漏自定义端点中注册的同名模型。

另外，自定义端点还有一个动态注册的能力：如果你在对话中途切换到一个新的模型 ID，而模型注册表里没有它，系统会自动把它注册到 `custom-endpoint` Provider 下，然后切换过去。这对于探索性的使用场景非常方便——你不需要预先把所有可能用到的模型都列出来。

## 为什么自定义端点重要？

聊了这么多"怎么做"，最后回答一个更根本的问题：为什么？

**不被绑定。** 如果 Agent 只能用 Anthropic 或 OpenAI 的模型，那你就被锁死在它们的生态里——定价变了你得接受，服务挂了你也只能等。自定义端点让你随时可以换一家"充电站"。

**数据隐私。** 敏感数据不想出本机？用 Ollama 跑本地模型，数据从你的电脑进来、在你的电脑上处理，一根网线都不出。

**成本控制。** 开源模型不要钱（或者说只要电费），对于高频调用的场景，省下的 API 费用不是小数目。

**灵活组合。** 你可以在同一个 Craft Agents 里配多个连接——重要对话用 Claude Opus，日常任务用本地 Llama，特殊需求用 OpenRouter 上的某个小众模型。每个连接各取所需，互不干扰。

---

了解了自定义端点，Pi Agent Server 的所有内部机制我们就都摸透了。从进程隔离、JSONL 通信、工具代理、模型解析、Token 刷新到自定义端点——这些"幕后工作"让 Agent 能稳定、安全、灵活地运行。最后一章，我们来看看如何把 Agent 从桌面搬到云端。
