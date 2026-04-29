# 1.2 项目全景——代码仓库长什么样？

刚才我们了解了 Craft Agents 是什么，现在打开代码仓库看看它的"骨架"。

想象你走进一家公司的总部大楼：一楼是门店，客户在这里和产品打交道；楼上几层是工厂和车间，生产核心产品；还有一间公共资料室，所有部门共享同一套术语和规范。Craft Agents 的代码仓库就是这栋大楼的蓝图——`apps/` 是门店，`packages/` 是工厂。

## apps/——面向用户的门店

### apps/cli/——终端收银台

这是一款命令行工具（CLI，Command Line Interface），程序员的最爱。它通过 WebSocket 连接到服务器，可以列出会话、发送消息、流式接收 AI 回复，甚至一键启动一个自带服务器的完整对话。如果你写脚本自动化、做 CI/CD 流水线、或者就是离不开终端，`cli` 就是你的入口。

### apps/electron/——旗舰店

这是大多数用户接触 Craft Agents 的地方——一个基于 Electron（电子框架）的桌面应用。Electron 把网页技术"包装"成原生桌面软件，而 Craft Agents 在这里分成了三个各司其职的层：

- **main（主进程）**：相当于门店经理，掌管窗口创建、应用生命周期、会话管理、系统菜单，以及和操作系统的所有交互。
- **preload（桥接层）**：相当于前台对讲机，在主进程和渲染界面之间安全地传递消息。它通过 Electron 的 Context Bridge（上下文桥接）机制，只暴露经过筛选的 API，防止渲染界面越权访问系统资源。
- **renderer（渲染器）**：相当于门店的展示厅，用 React 搭建用户界面——聊天窗口、会话列表、设置面板、权限请求弹窗，所有你看得见摸得着的东西都在这里。它使用 shadcn/ui 组件库和 Tailwind CSS 来保证界面美观一致。

> 除了 `cli` 和 `electron`，`apps/` 下还有 `viewer`（共享链接查看器）和 `webui`（浏览器端界面），它们是门店的线上分店。

## packages/——生产车间

### packages/core/——公司词典

每个公司都需要一本词典，确保大家说同一种话。`core` 就是这本词典——它定义了整个项目通用的 TypeScript 类型：Workspace（工作区）、Session（会话）、Message（消息）、AgentEvent（代理事件）……所有模块在传递数据时，都以这里的类型为准。它还提供几个轻量工具函数，比如生成唯一消息 ID。这个包故意保持精简，几乎没有运行时依赖，改起来牵一发动全身，所以非常稳定。

### packages/shared/——生产车间

如果说 `core` 是词典，`shared` 就是真正的生产车间。这里是核心业务逻辑的集中地，包含：

- **agent/**——CraftAgent 代理核心，权限模式（Explore / Ask to Edit / Auto）在这里实现
- **auth/**——OAuth 认证、令牌管理
- **sessions/**——会话的创建、持久化、恢复
- **sources/**——MCP 服务器、REST API、本地文件系统等外部数据源的连接
- **credentials/**——用 AES-256-GCM 加密存储密钥
- **config/**——配置读写、偏好设置、主题
- **skills/**——技能（Skill）的定义与管理
- **automations/**——事件驱动的自动化规则
- **prompts/**——系统提示词模板

桌面应用和远程服务器都依赖这个车间来干活。它是最"厚"的一个包，也是你阅读源码时最常造访的地方。

### packages/session-tools-core/——工具清单

每次会话开始，Agent 需要知道"我手头有哪些工具可以用"。`session-tools-core` 就是一份工具注册表——它定义了会话级别的可用工具列表，负责工具的注册、筛选和校验。你可以把它想象成车间墙上挂的那块工具板，上面列着当前工序可以取用的工具名称和用法。

### packages/pi-agent-server/——外加工车间

Craft Agents 同时支持两套 AI 后端：一套是 Anthropic 的 Claude Agent SDK，另一套是 Pi SDK。Pi SDK 以 Subprocess（子进程）的方式运行，负责处理 Google AI Studio、ChatGPT Plus、GitHub Copilot 等连接。`pi-agent-server` 就是这个子进程的入口——像一个外加工车间，接收主车间的订单，用不同的工艺完成加工，再把结果送回来。它还负责模型选择和端点解析，确保请求被路由到正确的 AI 提供商。

### packages/server-core/——外卖厨房

"外卖厨房"只做后厨的事，不接客。`server-core` 封装了无头服务器（Headless Server）的基础设施：WebSocket RPC 传输、平台服务契约、通用处理器依赖、启动编排逻辑。它不包含任何 UI 或窗口管理代码，纯粹是"厨房设备"——灶台、管线、出餐口。无论是 Electron 内嵌的服务器还是独立服务器，都复用这套基础设施。

### packages/server/——外卖门店

有了厨房，还需要一个门店来接单。`server` 是一个独立的可执行服务器，它基于 `server-core` 搭建，加上会话管理、凭证读取、消息网关、Web UI 托管等具体业务。你可以把它部署到一台远程 Linux 服务器上，然后用桌面应用或 CLI 作为"瘦客户端"连接过去——所有重活都在服务器端完成，本地只负责展示。

> 除了以上这些，`packages/` 下还有 `ui`（共享 UI 组件库）、`messaging-gateway`（消息网关）、`messaging-whatsapp-worker`（WhatsApp 消息处理）和 `session-mcp-server`（MCP 服务器子进程）等包，分别负责特定的通信和协议任务。

## 一张图看清全貌

```
craft-agents-oss/
├── apps/                    # 门店：面向用户的应用
│   ├── cli/                 #   终端客户端（程序员用）
│   ├── electron/            #   桌面应用（普通用户用）
│   │   └── src/
│   │       ├── main/        #     主进程（窗口、生命周期）
│   │       ├── preload/     #     桥接层（安全通信）
│   │       └── renderer/    #     渲染界面（React UI）
│   ├── viewer/              #   共享链接查看器
│   └── webui/               #   浏览器端界面
├── packages/                # 工厂：核心能力模块
│   ├── core/                #   公司词典（共享类型）
│   ├── shared/              #   生产车间（业务逻辑）
│   ├── session-tools-core/  #   工具清单（会话级工具注册）
│   ├── pi-agent-server/     #   外加工车间（Pi SDK 子进程）
│   ├── server-core/         #   外卖厨房（无头服务器基础设施）
│   └── server/              #   外卖门店（独立服务器二进制）
```

下次打开仓库时，先想想"这是门店还是工厂"——你就能快速定位到想看的代码。下一节我们来看看这些目录里用到了哪些技术，搞清楚这栋大楼是用什么材料盖起来的。
