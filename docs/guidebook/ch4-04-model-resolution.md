# 4.4 模型解析——名字怎么变成 API 调用？

上一节我们了解了工具代理机制——子进程想执行工具，得请主进程代劳。但你有没有想过一个更根本的问题：当你在界面上选择了一个模型，比如"claude-sonnet"，系统是怎么知道该去哪个服务器、用哪个 API 密钥来发起调用的？毕竟，"claude-sonnet"只是个名字，又不是电话号码。

## 通讯录查找

想象你在手机里给张三打电话。你不需要记住他的电话号码——只要在通讯录里搜"张三"，手机就会帮你找到对应的号码并拨出去。如果你给张三设了"常用联系人"，那就更方便了，一键直达。

模型解析（Model Resolution）做的事一模一样。你在界面上选的模型名称——"claude-sonnet-4-6"、"gpt-5.2"、"gemini-pro"——都只是"人名"，不是"电话号码"。系统得在它的"通讯录"里找到这个名字对应的 Provider（提供者）和 API 端点（Endpoint），然后才能拨号通话。

## Provider：AI 世界的运营商

在搞清楚查找流程之前，先认识一下"通讯录"里的分组——Provider（提供者）。

Provider 就是 AI 服务的供应商，就像手机运营商：Anthropic 是"移动"，OpenAI 是"联通"，Google 是"电信"。不同的运营商有不同的号码段、不同的拨号规则。你要打"移动"的电话，就得走移动的网络；你要调用 Anthropic 的模型，就得走 Anthropic 的 API。

Pi SDK 支持的 Provider 比你想象的要多：

| Provider | 说明 |
|----------|------|
| `anthropic` | Anthropic 官方 API |
| `openai` | OpenAI 官方 API |
| `openai-codex` | ChatGPT Plus/Pro 的 OAuth 方式 |
| `github-copilot` | GitHub Copilot 的 OAuth 方式 |
| `google` | Google AI Studio |
| `amazon-bedrock` | AWS Bedrock 托管服务 |
| `azure-openai` | Azure 上的 OpenAI 服务 |
| `openrouter` | OpenRouter 聚合网关 |
| `minimax-cn` | MiniMax（国内） |
| `custom-endpoint` | 用户自定义的兼容端点 |

注意最后那个 `custom-endpoint`——它不是一个真正的"运营商"，更像你自己在通讯录里手动添加的联系人。如果你自己搭了一套兼容 OpenAI 接口的服务，就可以通过 custom-endpoint 把它注册进来。

## 两本核心账簿

有了 Provider 的概念，再看系统的两本"核心账簿"就水到渠成了。

**PiModelRegistry（模型注册表）**——就是那个通讯录。它管理着所有已知模型的目录：每个模型叫什么名字、属于哪个 Provider、上下文窗口多大、是否支持推理模式……全部登记在册。查找模型的时候，就是在这本通讯录里翻。

**PiAuthStorage（认证存储）**——就是你的钥匙串。它保存着每个 Provider 的 API Key 或 OAuth Token。找到模型只是第一步，还得有对应的"钥匙"才能进门。PiAuthStorage 在内存中按 Provider 分组存放凭证：Anthropic 的 API Key 放一格，OpenAI 的放一格，GitHub Copilot 的 OAuth Token 放一格。

两本账簿配合起来：通讯录负责"找到人"，钥匙串负责"开得了门"。

## 四步查找策略

现在进入正题：系统到底怎么把一个模型名称解析成一次 API 调用？答案是四步查找，优先级从高到低，就像你找人的时候有一套优先顺序。

### 第一步：优先自定义端点

如果你配置了 custom-endpoint（自定义端点），系统会先在这里找。就像你设了"常用联系人"——张三如果被加到了常用列表，直接从这里拨，不用翻通讯录。

这一步是可选的。只有当 `preferCustomEndpoint` 被设为 `true` 时才会触发。一旦在自定义端点里找到了匹配的模型，直接返回，不再往下走。

### 第二步：精确查找

按 Provider + Model 名精确匹配。就像在通讯录里搜"张三（公司）"——既看名字，也看分组。

系统此时已经知道你用的是哪个 Provider 登录的（比如你是用 OpenAI 的 API Key 还是 GitHub Copilot 的 OAuth），所以可以直接在对应的 Provider 下精确查找模型 ID。这步的好处是避免歧义——比如 `gpt-5.2` 这个模型 ID 同时存在于 `openai` 和 `azure-openai-responses` 两个 Provider 下，精确查找能确保不会"串线"。

有个有趣的细节：如果你用的是 MiniMax 国内版（`minimax-cn`），Pi SDK 的模型 ID 带有 `MiniMax-` 前缀（比如 `MiniMax-M2.5-highspeed`），但 MiniMax 的 API 实际上不认这个前缀。所以系统在找到模型后，会自动把前缀去掉再调用——就像通讯录里存的是"张三（公司）"，但拨号的时候只拨"张三"。

### 第三步：全模型扫描

如果精确查找没找到，系统会遍历所有 Provider 的所有模型列表，按 ID 或名称匹配。就像翻遍整个通讯录找人——不管他在哪个分组，只要名字对得上就行。

不过这里有个安全守卫：如果你已经通过某个 Provider 认证了，全模型扫描只会返回同一个 Provider 下的模型（或者 custom-endpoint 下的）。这避免了"串号"——你用 GitHub Copilot 的账号登录，系统不会把 Azure OpenAI 的模型返回给你，因为你没有 Azure 的钥匙，即使找到了也打不通。

### 第三步都找不到？第四步：通用回退

按 `[custom-endpoint, anthropic, openai, google]` 的顺序逐个尝试。就像找人的时候先打常用号码，不行再打手机，再打座机——总有一个能通。

同样有安全守卫：如果你已经认证了某个 Provider，回退只会尝试与该 Provider 兼容的选项（以及永远允许的 custom-endpoint），不会越界。

四步都走完还找不到？那就只能返回"查无此人"了——系统会告诉你模型不存在，请检查名称是否正确。

## 小结

模型解析的本质就是一个查找过程：把人类可读的模型名称，翻译成机器可执行的 API 调用。四步策略——优先自定义、精确匹配、全表扫描、通用回退——层层递进，既保证效率，又避免串线。而 PiModelRegistry 和 PiAuthStorage 这两本账簿，一个管"找到人"，一个管"开得了门"，缺一不可。

找到人了，门也开了，但 OAuth Token 过期了怎么办？下一节我们来看 Token 刷新——钥匙过期了，怎么换一把新的。
