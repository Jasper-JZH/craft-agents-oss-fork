# 2.3 BaseAgent 模板方法——造一个新引擎有多难？

上一节我们了解了 AgentBackend 接口——所有引擎必须遵守的"宪法"。接口定好了规矩，但光有规矩还不够：如果每个引擎都要从零实现权限管理、Source 追踪、Prompt 拼接这些重复逻辑，那造一个新引擎就像从挖地基开始盖房子，费时费力还容易出错。BaseAgent 就是来解决这个问题的。

## 毛坯房——水电管道已经铺好

想象你买了一套毛坯房。墙壁是灰色的水泥，地面是粗糙的找平层，但水管已经走好了，电线已经埋好了，燃气管道已经入户了。你要做的不是重新铺设基础设施，而是刷墙、铺地板、装灯具——把这套房子装修成你喜欢的样子。

BaseAgent 就是这个毛坯房。它已经把所有引擎都需要的"水电管道"铺好了：

- **PermissionManager**——门禁系统。工具执行之前先问它"这个操作允许吗"，它根据当前的权限模式（safe / ask / allow-all）给出答案。没有门禁，谁都能随便执行危险命令，整栋楼都不安全。
- **SourceManager**——通讯录。追踪哪些外部服务（MCP 服务器、REST API）处于激活状态，哪些处于离线状态。就像手机里的通讯录，Agent 随时能查到"我有哪些朋友可以联系"。
- **PromptBuilder**——说明书编写器。把工作区信息、用户偏好、会话状态等碎片拼成一份完整的系统提示（System Prompt），告诉 AI 模型"你在什么环境下工作、有什么工具可以用"。
- **UsageTracker**——电表。精准记录每一轮对话消耗了多少 Token（令牌），包括输入、输出、缓存命中等详细数据。就像电表一样，让你知道"这个月用了多少电"。
- **PrerequisiteManager**——入场检查。有些工具必须先读完说明文档才能使用——就像进实验室之前必须先读完安全须知。PrerequisiteManager 追踪哪些文件已被读取，在条件不满足时阻止工具执行。
- **AutomationSystem**——自动化管家。处理工作区级别的自动化规则（来自 `automations.json`），比如"每次工具执行后自动发一条 Webhook 通知"。它默默在后台干活，不打扰主流程。

这六个模块在 BaseAgent 的构造函数中一次性初始化完毕。任何继承 BaseAgent 的子类自动拥有它们，不需要写一行重复代码。

## 填空题——Template Method 设计模式

理解了毛坯房提供了什么，接下来看一个关键问题：BaseAgent 是怎么让子类"只管装修"的？

答案是一种叫 Template Method（模板方法）的设计模式。听起来很学术，但核心思想简单得像做填空题：

老师出了一份试卷，题目框架和答题格式都已经印好了，你只需要在留空的地方填上答案。**题目的结构是固定的，答案因人而异。**

在 BaseAgent 里，`chat()` 方法就是那份"已填好的试卷"，`chatImpl()` 就是那道"留空的填空题"。

看看 `chat()` 做了什么（简化版）：

```typescript
async *chat(message, attachments, options) {
  // 1. 提取技能引用（@skill:xxx）
  const { skillPaths, cleanMessage } = this.extractSkillPaths(message);

  // 2. 注册前置条件——必须先读 SKILL.md 才能用工具
  this.prerequisiteManager.registerSkillPrerequisites([...skillPaths.values()]);

  // 3. 拼接分支上下文、会话迁移摘要、技能指令
  const effectiveMessage = [branchSeedContext, transferredSessionContext, directive, cleanMessage]
    .filter(Boolean).join('\n\n');

  // 4. 调用子类实现——"填空题"的答案
  yield* this.chatImpl(effectiveMessage, attachments, options);
}
```

前 3 步是通用逻辑——提取技能、注册前置条件、拼接上下文——不管是 Claude、Pi 还是未来的什么引擎，这些事都得做。BaseAgent 替你做了。第 4 步才是每个引擎自己的事：怎么把消息发给 AI 模型、怎么处理流式响应、怎么执行工具回调。这些差异全部封装在 `chatImpl()` 里。

所以，当你想造一个新引擎时，不需要关心技能怎么提取、前置条件怎么注册，只需要实现 `chatImpl()`——把对话循环写好就行了。

## 抽象方法清单——七道填空题

除了 `chatImpl()`，BaseAgent 还定义了其他必须由子类实现的 Abstract Method（抽象方法）。它们是新引擎必须完成的七道填空题：

| 方法 | 类比 | 干什么 |
|------|------|--------|
| `chatImpl()` | 引擎的主循环 | 核心对话实现——怎么发消息、怎么收响应、怎么执行工具 |
| `abort()` | 踩刹车 | 用户点了"停止"之后，如何优雅地中止当前对话 |
| `forceAbort()` | 拉手刹 | 强制停止，用于用户强制中断或系统需要立刻回收资源的场景 |
| `isProcessing()` | 看转速表 | 查询引擎是否正在工作中 |
| `respondToPermission()` | 对讲机应答 | 当 UI 返回权限决定（允许/拒绝）时，如何把结果传回正在等待的引擎 |
| `runMiniCompletion()` | 快递小包裹 | 不带工具、不带系统提示的纯文本补全，用于生成标题、摘要等轻量任务 |
| `queryLlm()` | 直拨电话 | 直接调用 LLM 的接口，用于 `call_llm` 工具和子会话等独立查询场景 |

每道题都对应引擎之间的真实差异。比如 `abort()`——Claude 引擎通过取消 SDK 的 HTTP 请求来中止，Pi 引擎通过向子进程发送中止信号来中止。方式不同，但 BaseAgent 不关心，它只要求子类"给我一个能用的 `abort()` 方法"。

## 组合优于继承——毛坯房的"模块化装修"

你可能注意到了，BaseAgent 的六个核心模块不是用继承层层叠加的，而是通过 Composition（组合）——在构造函数里 `new` 出来，赋给属性，需要时调用。PermissionManager 是独立的类，SourceManager 也是独立的类，它们各管各的事，互不干扰。

这种"组合"的方式就像毛坯房的模块化装修：你可以换一种地板，但不影响墙壁的涂料；你可以升级门禁系统，但不影响通讯录的运作。如果所有功能都堆在一个巨大的类里，改一处就可能牵动全身。而组合让每个模块独立演进、独立测试，改 PermissionManager 的逻辑完全不用担心搞坏 SourceManager。

BaseAgent 把"所有引擎都要做的事"和"每个引擎自己决定的事"分得清清楚楚。前者写死在基类里，后者留给抽象方法。正因如此，造一个新引擎的难度从"盖一栋楼"降到了"装修一套房"——你需要实现七个方法，把对话循环跑通，剩下的基础设施全都现成可用。

---

有了毛坯房，接下来看看 ClaudeAgent 是怎么装修的——它如何在 BaseAgent 的基础上，用 Claude Agent SDK 实现那七道填空题。
