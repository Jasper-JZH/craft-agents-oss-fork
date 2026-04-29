# 2.2 AgentBackend 接口——所有引擎的共同契约

上一节我们知道了 Agent 是司机，后端是引擎。不管你开的是汽油车还是电动车，驾校教的东西都一样：起步、转弯、刹车。考官不在乎你脚下是什么引擎，只在乎你是不是按照大纲操作。AgentBackend 接口（Interface）就是这份"驾照考试大纲"——它规定了每一个后端**必须会做的事**，至于你怎么做到的，它不管。

## 什么是接口？

接口这个词听起来很技术，但概念其实很日常。想象一下卫生监督局给餐厅定的标准：必须有消毒柜、必须有健康证、生熟砧板必须分开。不管你是路边摊还是米其林三星，这些标准一视同仁。你用什么牌子的消毒柜、员工去哪家医院体检，标准不关心——它只关心**你有没有做到**。

在编程中，Interface（接口）就是这份"必须实现的清单"。它列出一组方法的名字、参数和返回类型，但不提供具体实现。任何声称"满足这个接口"的类，必须把清单上的每一项都实现出来，否则编译器直接报错——就像卫生检查不通过就不能开业。

在 Craft Agents 的源码中，这份清单定义在 `packages/shared/src/agent/backend/types.ts` 的 `AgentBackend` 接口里。ClaudeAgent 和 PiAgent 两个后端，不管内部差异多大，都必须把这份清单上的每一项逐个实现。

## 核心方法：驾驶类比

AgentBackend 的方法用驾驶来类比，会非常直观。

### chat() — 踩油门，开始对话

```typescript
chat(message: string, attachments?: FileAttachment[], options?: ChatOptions): AsyncGenerator<AgentEvent>;
```

你踩下油门，车就开始走。你发一条消息，`chat()` 就启动整个 Agent 循环——理解需求、选择工具、执行操作、观察结果，直到任务完成。

注意它的返回类型是 `AsyncGenerator<AgentEvent>`（异步生成器）。普通函数像倒水——一倒就完了。异步生成器更像一根持续供油的油管：Agent 每完成一步，就"吐"出一个事件（AgentEvent），你收到一个处理一个，直到 Agent 说"我做完了"或者被叫停。这种设计让你可以实时看到 Agent 的每一步进展，而不是等它全部干完才一次性返回。

### abort() — 踩刹车，优雅停止

```typescript
abort(reason?: string): Promise<void>;
```

你看到前方有情况，轻踩刹车让车缓缓停下。`abort()` 是"优雅停止"——告诉后端该停了，但允许它把手头的收尾工作做完，比如把当前的工具调用结束掉、保存好状态。它返回一个 Promise，意味着刹车踩下去之后，你需要等车真正停下来。

### forceAbort() — 紧急刹车，立即停下

```typescript
forceAbort(reason: AbortReason): void;
```

紧急情况！一脚踩死刹车，什么都不管了，立刻停下。和 `abort()` 不同，`forceAbort()` 不返回 Promise——它不等你确认，直接强制中断。什么时候用？用户点了"停止"按钮，或者系统需要立刻转向的时候。

### interruptForHandoff() — 靠边停车等乘客

```typescript
interruptForHandoff(reason: AbortReason): void;
```

想象你开着出租车，乘客说"等一下，我先确认个事"。你靠边停车，但不熄火，等乘客点头了再继续。`interruptForHandoff()` 是"协作式暂停"——Agent 主动把控制权交给用户，比如提交了一份执行计划等用户审批，或者需要用户授权某个操作。暂停的原因通过 `AbortReason` 枚举传递，比如 `PlanSubmitted`（计划已提交）或 `AuthRequest`（需要认证）。这和紧急刹车不同：车还在，发动机没关，随时可以重新出发。

### redirect() — 打方向盘，流中转向

```typescript
redirect(message: string): boolean;
```

你正在高速上跑着，导航突然说"前方拥堵，建议绕行"。你不需要先停车再重新出发，而是直接变道。`redirect()` 就是这种"行进中转向"的能力——当 AI 正在回复时，你发了新的消息，系统不会先中断再重新开始，而是把新消息注入当前流中，让 AI 调整方向继续工作。

返回值是个布尔（boolean）：`true` 表示转向成功（事件继续在当前流中输出），`false` 表示这个后端不支持原地转向，只能先踩刹车再重新出发。目前 Pi 后端支持原地转向，Claude 后端则需要先中断再重发。

### runMiniCompletion() — 怠速运转，做简单任务

```typescript
runMiniCompletion(prompt: string): Promise<string | null>;
```

等红灯的时候，发动机不熄火，但也不全速运转——这就是怠速。`runMiniCompletion()` 用来做轻量级的文本任务，比如给会话起个标题、生成一段摘要、测试连接是否正常。它不走完整的 Agent 循环（不需要工具调用、不需要权限检查），只是简单地向 AI 模型发一条提示，拿到文本结果就返回。

### queryLlm() — 临时借用另一个引擎

```typescript
queryLlm(request: LLMQueryRequest): Promise<LLMQueryResult>;
```

你是开货车的司机，偶尔需要借一辆小轿车去办点私事。`queryLlm()` 允许后端发起一次独立的 LLM 调用，不走 Agent 主循环。它和 `runMiniCompletion()` 的区别在于，`queryLlm()` 更正式：可以指定模型、系统提示词（systemPrompt）、输出格式（outputSchema）等参数。这个方法主要用于 Agent 内部的"呼叫 LLM"工具，让 Agent 在执行任务的过程中临时借助 AI 能力做子任务。

### getModel() / setModel() — 换挡

```typescript
getModel(): string;
setModel(model: string): void;
```

开手动挡的车，上坡用低挡、高速用高挡。`getModel()` 和 `setModel()` 就是 Agent 的"挡位"——查询和切换当前使用的 AI 模型。你可以从 Claude Sonnet 换到 Claude Opus，就像从三挡换到五挡。`setModel()` 会验证新模型是否在当前后端的能力范围内，你不能给电动车加汽油。

### getPermissionMode() / setPermissionMode() — 调节安全模式

```typescript
getPermissionMode(): PermissionMode;
setPermissionMode(mode: PermissionMode): void;
cyclePermissionMode(): PermissionMode;
```

有些车有"运动模式"和"经济模式"。Agent 也有安全模式：`safe`（最严格，每个操作都要确认）、`ask`（按需询问）、`allow-all`（全部放行）。`getPermissionMode()` 查看当前模式，`setPermissionMode()` 设置模式，`cyclePermissionMode()` 则像按按钮循环切换——在三种模式之间轮流切换，返回切换后的新模式。

## 回调机制：Agent 可以"打电话回总部"

前面这些方法都是"你告诉 Agent 做什么"，但 Agent 有时候也需要主动联系你。就像外卖小哥遇到问题会打电话给你："门禁打不开，您下来接一下？"这就是 Callback（回调）机制。

回调是 Agent 在特定时刻主动调用的函数，由上层（门面层）在创建后端后"接线"——把函数赋值给回调属性，就像给手机存一个号码。AgentBackend 定义了几个关键的回调：

- **onPermissionRequest**：Agent 要执行一个需要授权的操作（比如运行一条 shell 命令），打电话问你"可以吗？"你回复同意或拒绝。
- **onPlanSubmitted**：Agent 制定了一份执行计划，靠边停车等你审批。你看了觉得没问题，Agent 才继续出发。
- **onAuthRequest**：Agent 需要访问某个需要登录的服务，打电话说"我需要认证，请帮我登录"。你完成登录后，Agent 拿到凭证继续工作。

这些回调的设计让 Agent 既拥有自主行动的能力，又在关键时刻把决定权交给用户。不是一味地闷头干，也不是每一步都来问——该自己做主的自己做主，该请示的请示。

## 小结

AgentBackend 接口是一份精密的契约，它确保了：不管底层用的是 Claude 还是 Pi，不管认证方式是 API Key 还是 OAuth，上层代码面对的始终是同一套方法、同一套回调、同一种工作方式。这正是接口的力量——**定义规则，不限制自由**。

下一节我们知道了接口长什么样，那实现这份清单得写多少代码？别担心，BaseAgent 会帮我们搭好脚手架——下一节我们看看它如何让我们少写代码。
