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

## 做出来能完成什么 · What you can build

把这个设计落地后，你的系统会获得这些能力：

- **对话即操作**：用户一句话完成查询/创建/修改/删除，结果以结构化卡片呈现，不需要教用户学菜单
- **建表即上线**：数据库新增一张表，AI 自动发现并学会操作，业务扩展不用改前端
- **AI 可以安全地接触真实数据**：不编造（所有回答基于工具真实结果）、不可逆操作先确认、权限按角色拦截、全程审计——达到"敢放进生产环境"的标准
- **批量脏活交给 AI**：文件批量导入、数据清洗、Excel 报表生成，AI 编排原子指令完成
- **人永远有最终决定权**：AI 只能创建草稿、删除已取消/已拒绝的终态记录；所有业务状态推进由人在操作台完成
- **多租户天然隔离**：每个租户的 AI 助手只能操作自己租户的数据，AI 本身无需感知租户
- **长对话不劣化**：上下文超限自动降级（落盘 / 按需读回），任务永不中断
- **每一步都可追溯**：所有工具调用进入审计——谁、何时、做了什么、结果如何，全程留痕

一句话：把"会聊天的 AI"，变成"敢让它碰生产数据的 AI"。

In short, this design turns "an AI that chats" into "an AI you trust with production data."

- **Conversation is the operation**: one sentence completes query/create/update/delete, with results rendered as structured cards
- **Create a table, it goes live**: add a table and the AI discovers and learns it — no frontend changes
- **The AI can safely touch real data**: no fabrication (answers come from real tool results), confirmation before irreversible actions, role-based permission gates, full auditing
- **Batch grunt work goes to the AI**: file imports, data cleaning, Excel report generation via atomic-instruction scripts
- **Humans keep the final say**: the AI can only create drafts and delete cancelled/rejected terminal records; every business state transition happens in the console, by a human
- **Tenants are isolated by default**: each tenant's AI can only touch its own data, and the AI never even needs to know tenancy exists
- **Long sessions don't degrade**: context overflow degrades gracefully (offload / read back on demand), tasks are never interrupted
- **Every step is traceable**: all tool calls are audited — who, when, what, and the outcome

---

## 背景项目 · Background project

思路来自 [启禾 OS · 仓迹](https://www.qihebook.cloud) —— 一个 AI 原生的进销存 ERP：自然语言是主要操作界面，传统表单是辅助通道。

The ideas come from [Qihe OS · Cangji](https://www.qihebook.cloud), an AI-native ERP where natural language is the primary interface and traditional forms are the auxiliary channel.

---

## 版权 · Copyright

文档内容可自由引用、转载与再创作，注明出处即可。思路可复用，代码会过时。

The content is free to quote, repost, and remix with attribution. Ideas are reusable; code goes stale.
