# Agents Rule of Two：AI Agent 安全的实用方法

![Cover Image](https://ai.meta.com/blog/practical-ai-agent-security/)

> **原文**：https://ai.meta.com/blog/practical-ai-agent-security/  
> **作者**：Meta AI  
> **翻译**：由 OpenClaw 翻译  
> **图片说明**：本文图片托管于 Facebook CDN，因防盗链保护无法下载到本地，已改用原文链接。

**2025年10月31日 | 14分钟阅读**

---

想象一个个人 AI Agent——Email-Bot，旨在帮助你管理收件箱。为了提供价值并有效运作，Email-Bot 可能需要：

- 访问未读邮件内容，从各发件人处获取有用的摘要
- 阅读现有收件箱，跟踪重要更新、提醒或上下文
- 代表你发送回复或跟进邮件

虽然自动邮件助手可以提供很大帮助，但这个假设的 Bot 也展示了 AI Agent 如何引入新型风险。值得注意的是，对行业来说最大的挑战之一是 Agent 对 **prompt injection** 的敏感性。

Prompt injection 是所有 LLM 中一个根本性的、未解决的弱点。通过 prompt injection，某些类型的不可信字符串或数据片段——当传入 AI Agent 的上下文窗口时——可能导致意外后果，例如忽略开发者提供的指令和安全指南，或执行未授权的任务。这个漏洞足以让攻击者控制 Agent 并伤害 AI Agent 的用户。

以 Email-Bot 为例，如果攻击者在发给目标用户的电子邮件中放置 prompt injection 字符串，一旦该邮件被处理，他们可能劫持 AI Agent。示例攻击可能包括窃取敏感数据（如私人邮件内容），或执行不想要的操作（如向目标用户的朋友发送网络钓鱼消息）。

与许多行业同行一样，我们对 Agentic AI 改善人们生活和提高生产力的潜力感到兴奋。实现这一愿景的路径涉及授予 AI Agent（如 Email-Bot）更多能力，包括访问：

- **由未知方创作的数据源**，如入站邮件或从互联网查询的内容
- **Agent 被允许使用以进行规划并实现更高个性化的私人或敏感数据**
- **可以自主调用以代表用户完成任务的工具**

在 Meta，我们正在深入思考如何在平衡这一产品愿景所需的实用性和灵活性的同时，最大程度减少 prompt injection 带来的不良后果，如私人数据泄露、强制代表用户执行操作或系统中断。为了最好地保护人们和我们的系统免受这一已知风险，我们开发了 **Agents Rule of Two**。当遵循此框架时，安全风险的严重性会被确定性降低。

受为 Chromium 开发的同名策略以及 Simon Willison 的"lethal trifecta"启发，我们的框架旨在帮助开发者理解并驾驭当今这些强大 Agent 框架中存在的权衡。

---

## Agents Rule of Two

高层次来看，Agents Rule of Two 指出，在稳健性研究允许我们可靠地检测和拒绝 prompt injection 之前，Agent 必须在会话中满足以下**三个属性中不超过两个**，以避免 prompt injection 的最高影响后果。

| 属性 | 描述 |
|------|------|
| **[A]** | Agent 可以处理不可信输入 |
| **[B]** | Agent 可以访问敏感系统或私人数据 |
| **[C]** | Agent 可以更改状态或对外通信 |

这三个属性对于执行请求仍然可能是必要的。如果 Agent 需要所有三个属性而不启动新会话（即使用新的上下文窗口），则不应允许 Agent 自主操作，至少需要监督——通过 human-in-the-loop 批准或其他可靠的验证方式。

---

## Agents Rule of Two 如何阻止攻击

让我们回到 Email-Bot 示例，看看应用 Agents Rule of Two 如何防止数据泄露攻击。

**攻击场景**：垃圾邮件中的 prompt injection 包含一个字符串，指示用户的 Email-Bot 收集用户收件箱的私人内容，并通过调用 Send-New-Email 工具将其转发给攻击者。

此攻击成功是因为：

- [A] Agent 可以访问不可信数据（垃圾邮件）
- [B] Agent 可以访问用户的私人数据（收件箱）
- [C] Agent 可以对外通信（通过发送新邮件）

通过 Agents Rule of Two，可以通过几种不同方式防止此攻击：

- 在 **[BC]** 配置中，Agent 只能处理来自可信发件人（如密友）的邮件，防止初始 prompt injection 有效载荷进入 Agent 的上下文窗口。
- 在 **[AC]** 配置中，Agent 不会访问任何敏感数据或系统（例如在训练测试环境中运行），因此到达 Agent 的任何 prompt injection 都不会产生有意义的影响。
- 在 **[AB]** 配置中，Agent 只能将新邮件发送给可信收件人，或者在人类验证草稿消息内容后发送，防止攻击者最终完成攻击链。

通过 Agents Rule of Two，Agent 开发者可以比较不同设计及其相关的权衡（如用户摩擦或能力限制），以确定哪种选项最适合其用户需求。

---

## Agents Rule of Two 的假设示例和实现

让我们看另外三个假设的 Agent 用例，了解它们如何选择满足框架。

### 旅行 Agent 助手 [AB]

这是一个面向公众的旅行助手，可以回答问题并代表用户行事。它需要搜索网络以获取旅游目的地的最新信息 [A]，并可以访问用户的私人信息以进行预订和购买体验 [B]。

为了满足 Agents Rule of Two，我们通过以下方式对其工具和通信 [C] 设置预防性控制：
- 请求任何操作的真人确认，如预订或支付押金
- 将网络请求限制为仅来自可信来源返回的 URL，而不访问由 Agent 构造的 URL

### 网络浏览研究助手 [AC]

这个 Agent 可以与网络浏览器交互，代表用户执行研究。它需要填写表单并向任意 URL 发送大量请求 [C]，并必须处理结果 [A] 以根据需要重新规划。

为了满足 Agents Rule of Two，我们通过以下方式对其访问敏感系统和私人数据 [B] 设置预防性控制：
- 在限制性沙箱中运行浏览器，不预加载会话数据
- 限制 Agent 访问私人信息（除初始 prompt 外），并告知用户其数据可能如何共享

### 高速内部编码器 [BC]

这个 Agent 可以通过在组织内部基础设施上生成和执行代码来解决工程问题。为了解决有意义的问题，它必须能够访问生产系统的子集 [B]，并能够对这些系统进行状态更改 [C]。虽然 human-in-the-loop 可以是 valuable defense-in-depth，但开发者的目标是通过最小化人工干预来解锁规模化运营。

为了满足 Agents Rule of Two，我们通过以下方式对任何不可信数据源 [A] 设置预防性控制：
- 使用作者 lineage 来过滤 Agent 上下文窗口内处理的所有数据源
- 提供人工审查流程以标记误报，并 enable Agent 访问数据

---

与通用框架一样，细节决定成败。为了启用额外的用例，Agent 在同一会话中从 Agents Rule of Two 的一种配置转换到另一种配置可以是安全的。一个具体示例是在 [AC] 状态下访问互联网，然后在访问内部系统时通过禁用通信完成到 [B] 的单向切换。

---

## 局限性

重要的是要注意，满足 Agents Rule of Two 不应被视为足以防范 Agent 其他常见威胁向量（如攻击者提升、垃圾邮件扩散、Agent 错误、幻觉、过度权限等）或 prompt injection 的较低后果影响（如 Agent 回复中的错误信息）。

同样，应用 Agents Rule of Two 不应被视为缓解风险的终点线。满足 Agents Rule of Two 的设计仍然可能容易失败（如用户盲目确认警告插入），纵深防御是缓解最高风险场景的关键组成部分，当单层失败可能发生时。Agents Rule of Two 是补充——而非替代——如最小权限等通用安全原则。

---

## 现有解决方案

了解补充 Agents Rule of Two 的更多 AI 保护解决方案，请阅读我们的 Llama Protections。产品包括用于编排 Agent 保护的 Llama Firewall、用于分类潜在 prompt injection 的 Prompt Guard、用于减少不安全代码建议的 Code Shield，以及用于分类潜在有害内容的 Llama Guard。

---

## 下一步

我们相信 Agents Rule of Two 对今天的开发者是一个有用的框架。我们也对其在规模化安全开发方面的潜力感到兴奋。

随着通过 Model Context Protocol (MCP) 等协议采用即插即用 Agent 工具调用，我们看到了新兴的风险和机会。虽然盲目地将 Agent 连接到新工具可能是灾难的配方，但有可能通过内置的 Rule of Two 意识实现安全默认。例如，通过在支持工具调用中声明 Agents Rule of Two 配置，开发者可以更有信心地知道操作将根据其策略成功、失败或请求额外批准。

我们还知道，随着 Agent 变得更有用和能力增长，一些广受欢迎的用例将难以干净地适应 Agents Rule of Two，例如人工介入具有破坏性或无效的后台进程。虽然我们相信传统的软件护栏和人工批准仍然是满足当前用例中 Agents Rule of Two 的首选方法，但我们将继续通过对齐控制（如监督 Agent 和开源 LlamaFirewall 平台）追求满足 Agents Rule of Two 的监督批准检查。我们期待在未来分享更多。
