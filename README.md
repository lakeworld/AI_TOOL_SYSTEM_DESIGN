# AI 工具系统设计 · AI Tool System Design

> 让 AI 学会"用工具"，又管住它——一个 AI 原生 ERP 的工具系统设计思路
>
> Teaching an AI to use tools — and keeping it in line: the tool-system design of an AI-native ERP.

本仓库收录一篇关于「AI 如何安全地操作系统」的设计思路文章，源自 **启禾 OS · 仓迹**（AI 原生进销存 ERP）的真实工程实践。不贴代码，只讲思路。

This repository hosts a design-essay on "how AI can safely operate a business system," drawn from the real engineering practice of **Qihe OS · Cangji**, an AI-native inventory-and-sales ERP. Ideas only, no code.

---

## 文档 · Documents

| 语言 | 文件 |
|------|------|
| 中文 | [AI_TOOL_SYSTEM_DESIGN.md](./AI_TOOL_SYSTEM_DESIGN.md) |
| English | [AI_TOOL_SYSTEM_DESIGN.en.md](./AI_TOOL_SYSTEM_DESIGN.en.md) |

---

## 这篇文章讲什么 · What's inside

- **AI 如何操控工具**：发现 → 选择 → 调用 → 纠错闭环（help 目录按需查询、技能文档、参数宽容、错误即教学）
- **怎么限制 AI 做下一步**：三层约束模型——提示词"教" / 工具契约"纠" / 代码硬拦截"锁"
- **工具系统三大支柱**：元数据单一事实源、两种工具形态（宏工具 + 脚本语言）、独立执行进程
- **人机确认机制**：不可逆操作必须人拍板，确认号 5 分钟批量复用、单独一轮、上下文豁免
- **多租户隔离 / 上下文经济性 / 工具演进方法论 / 踩过的坑**

How the AI operates tools (discover → choose → call → recover), how to keep it in line (a three-layer constraint model: prompts teach, contracts correct, code locks), the three pillars of the tool system, human confirmation design, multi-tenant isolation, context economy, and hard-won pitfalls.

**适合谁**：任何想给 AI 接上"手脚"、让 AI 进入严肃生产环境（ERP、医疗、物流、金融等不容许幻觉的场景）的项目与团队。

**Who it's for**: any project or team that wants to give an AI "hands and feet" and bring it into high-stakes production environments where hallucination is unacceptable.

---

## 背景项目 · Background project

思路来自 [启禾 OS · 仓迹](https://www.qihebook.cloud) —— 一个 AI 原生的进销存 ERP：自然语言是主要操作界面，传统表单是辅助通道。

The ideas come from [Qihe OS · Cangji](https://www.qihebook.cloud), an AI-native ERP where natural language is the primary interface and traditional forms are the auxiliary channel.

---

## 版权 · Copyright

文档内容可自由引用、转载与再创作，注明出处即可。思路可复用，代码会过时。

The content is free to quote, repost, and remix with attribution. Ideas are reusable; code goes stale.
