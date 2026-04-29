# 5.4 Server Builder——如何组装一个服务器

上一节我们了解了会话（Session）的生命周期——从创建到运行再到销毁，SessionManager 像管家一样照看着每一个会话。但 SessionManager 不是凭空出现的，它只是整座"大楼"里的一个房间。这座大楼怎么盖起来的？各层楼、各房间之间怎么配合？这就引出了本节的主角：**Server Builder（服务器构建器）**。

## 乐高积木的哲学

你拼过乐高吗？一盒乐高积木里，有底板、有墙砖、有窗户、有屋顶，每块都是独立的。你可以按说明书一步一步拼，也可以自由组合换掉某些零件——把红墙换成蓝墙，房子还是房子，不会塌。

Craft Agents 的服务器构建也是这个思路。`bootstrapServer()` 就是一个泛型（Generic）构建器，每个组件——认证、RPC、Session——都是一块独立的积木。你按顺序把它们拼起来，一台完整的服务器就诞生了。更重要的是，因为每个组件都是独立的，所以你可以替换其中任何一块，而不会影响其他积木。

## 七步拼装说明书

打开 `packages/server-core/src/bootstrap/headless-start.ts`，你会看到 `bootstrapServer<TSessionManager, THandlerDeps>` 这个函数。它的工作像一份乐高说明书，按七个步骤把服务器拼装完成：

### 第一步：验证 Token 熵值（Entropy）

```
const entropy = validateTokenEntropy(serverToken)
if (!entropy.ok) {
  throw new Error(`Weak server token: ${entropy.error}`)
}
```

Token 是连接服务器的钥匙。这一步检查钥匙够不够复杂——太简单的钥匙容易被撬。具体来说，`validateTokenEntropy` 会做三重检查：长度不少于 16 个字符；不能是单个字符的重复（比如 `"aaaaaaaaaaaaaaaa"`）；唯一字符少于 8 个会发出警告。就像你家的门锁，如果密码是 `000000`，谁都能猜到；但如果是 `q7Km2xP9vR4sT8nL`，想猜中就难了。

### 第二步：创建 PlatformServices

```
const platform = options.platformFactory?.() ?? createHeadlessPlatform({ appVersion: options.serverVersion })
```

PlatformServices 是一组平台相关的服务接口，抽象了文件系统、窗口操作、图片处理、日志、通知等差异。这一步决定运行平台——无头（Headless）模式用 `createHeadlessPlatform`，Electron 模式用 `createElectronPlatform`。打个比方，就像盖楼前先选场地——是在空地上盖（无头模式，只靠命令行和文件系统），还是在现有建筑里装修（Electron 模式，有窗口管理、原生对话框等现成设施）。选好场地，后续的"施工队"才知道该用哪套工具。

### 第三步：初始化配置

```
bootstrapConfigArtifacts(platform)
ensureGlobalConfigExists(platform)
acquireServerLock(platform.logger)
```

加载用户配置、确保工作区设置存在、获取启动锁文件。这一步就像读取建筑图纸——楼要盖几层、每层什么用途，全在图纸上写着。同时，`acquireServerLock` 确保同一时间只有一个服务器实例在运行，就像工地门口的告示牌："施工中，请勿入内"。

### 第四步：创建 SessionManager

```
const sessionManager = options.createSessionManager()
```

装上会话管理系统。上一节我们已经认识了 SessionManager——它是大楼的物业，管理所有入住的会话（Session）。注意这里用的是 `options.createSessionManager()`，由调用方提供具体实现，而不是在构建器里写死。这就是泛型设计的好处：Electron 版可以传入带浏览器面板管理器的 SessionManager，无头版传入精简版——积木可以换。

### 第五步：创建 WsRpcServer

```
const wsServer = new WsRpcServer({ host, port, requireAuth: true, ... })
await wsServer.listen()
```

装上通信模块——WsRpcServer。它是大楼的电梯和电话系统，负责接收客户端的请求、推送服务器的消息。所有客户端的指令通过 WebSocket（网页套接字）进来，所有 Agent 的回复通过它推送出去。创建完毕后调用 `listen()`，服务器开始监听指定端口，等待连接。

### 第六步：注册 RPC Handlers

```
options.registerAllRpcHandlers(wsServer, deps, serverHandlerContext)
```

Handler（处理器）是服务器的业务逻辑——处理认证、管理会话、读写文件、查询模型列表……每个 Handler 就像给办公室分配职能：201 房间负责认证，202 房间管文件，203 房间查模型。核心处理器（Core Handlers）包含认证、会话、文件、工作区等基础功能；Electron 版额外注册了窗口管理、浏览器面板等 GUI 专属处理器。同样，具体注册哪些处理器，由调用方通过 `options.registerAllRpcHandlers` 决定。

### 第七步：启动模型刷新

```
modelRefreshService.startAll()
```

定期更新可用模型列表。LLM 提供商的模型目录不是一成不变的——新模型上线、旧模型下架、价格调整，都在变。模型刷新服务就像定期更新的商品目录，确保服务器始终知道"现在有哪些模型可以用、用什么密钥能访问"。

## 泛型的意义：同一份蓝图，不同的实现

回头看 `bootstrapServer<TSessionManager, THandlerDeps>` 的签名，两个泛型参数 `TSessionManager` 和 `THandlerDeps` 是整台服务器的灵魂。

在无头模式下（`packages/server/src/index.ts`），调用方这样传入：

```typescript
bootstrapServer<SessionManager, HandlerDeps>({
  createSessionManager: () => new SessionManager(),
  createHandlerDeps: ({ sessionManager, platform, oauthFlowStore }) => ({
    sessionManager, platform, oauthFlowStore, messagingRegistry: ...
  }),
  registerAllRpcHandlers: registerCoreRpcHandlers,
  ...
})
```

在 Electron 模式下（`apps/electron/src/main/index.ts`），同样是这个函数，但传入的实现不同：

```typescript
bootstrapServer<SessionManager, HandlerDeps>({
  platformFactory: () => platform,  // Electron 平台
  createSessionManager: () => {
    const sm = new SessionManager()
    sm.setBrowserPaneManager(browserPaneManager!)  // 带浏览器面板
    return sm
  },
  createHandlerDeps: ({ sessionManager, platform, oauthFlowStore }) => ({
    sessionManager, platform,
    windowManager,       // Electron 独有
    browserPaneManager,  // Electron 独有
    oauthFlowStore, messagingRegistry: ...
  }),
  registerAllRpcHandlers: isHeadless
    ? registerCoreRpcHandlers         // 无头：只有核心处理器
    : registerAllRpcHandlers,         // 有界面：核心 + GUI 处理器
  ...
})
```

同一份构建逻辑，两种运行环境，各自的差异通过泛型参数注入——这就是 Builder 模式的威力。你不需要为每个环境写一套启动流程，只需要换几块积木。

## 模块独立性：换一块不影响其他

七个步骤中，每一步产出的组件都是独立的。PlatformServices 不关心 SessionManager 的内部实现，SessionManager 不关心 WsRpcServer 怎么通信，WsRpcServer 不关心 Handler 处理什么业务。这种独立性带来两个好处：

**独立测试**——你可以单独测试 SessionManager 的会话创建逻辑，不需要真的启动 WebSocket 服务器；也可以单独测试 Handler 的业务逻辑，只需要 mock（模拟）一个 SessionManager 接口。

**灵活替换**——想换一种通信方式？用新的 WsRpcServer 实现替换即可，其他模块不受影响。想给 SessionManager 加新功能？改它自己的代码，Handler 和 PlatformServices 完全不用动。就像乐高积木——换一块红色的砖不会让旁边的窗户掉下来。

---

七步拼装完成，一台完整的服务器就跑起来了。从 Token 安全检查到平台选择，从配置加载到通信建立，每一步都像一块积木，拼在一起构成了一座可以运行 Agent 的大楼。但服务器跑起来只是第一步——怎么把它打包、部署、让它在任何环境里稳定运行？最后一节，我们看看如何用 Docker 把整个服务器打包部署。
