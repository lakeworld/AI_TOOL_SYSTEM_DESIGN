# Teaching an AI to Use Tools — and Keeping It in Line: The Tool-System Design of an AI-Native ERP

> This article shares the complete design philosophy behind "how AI operates a business system" from Qihe OS · Cangji (启禾 OS · 仓迹), an AI-native ERP for inventory and sales management.
> No code, only ideas — because these ideas are independent of your language or framework. Any project that wants to give an AI "hands and feet" can use them.

---

## 1. Origins

Let me start with some background.

The interaction model of a traditional ERP is: open a menu → find a form → fill in fields → click save. The system trains the user to become a "form-filling operator".

We wanted to build the opposite: **a person says, in natural language, "place an order for 100 mosquito-killer lamps for Zhang San," and the system finds the customer, checks the product, calculates the amount, creates the order, and shows the result.** The person shouldn't have to learn the system; the system should understand the person.

That vision sounds beautiful, but the hard part of building an AI that can "get its hands dirty" is not the model itself. It's three things:

1. **How does the AI "know" what it can do?** — The system has dozens of capabilities. How does the AI discover them, choose among them, and invoke them correctly?
2. **How does the AI "do it right"?** — LLMs lie, guess, and invent parameters. How do you guarantee that every operation is backed by real data?
3. **How do you "keep the AI in line"?** — This is the hardest one. Once the AI has create/update/delete powers, what stops it from deleting data carelessly, exceeding its authority, or advancing a business state it shouldn't touch?

This article revolves around those three questions. Everything here comes from real pitfalls we hit and the designs we settled on. Think of it as a practical playbook for "giving your AI a tool system."

One note up front: **this article is about ideas, not code.** For each design point I'll explain *why it exists, what problem it solves, and what it costs* — but I won't paste implementations, because implementations drift as the project evolves, while ideas are stable.

---

## 2. Introducing the Project: Qihe OS · Cangji

Before diving into technical details, let me spend a minute introducing the project itself.

**Qihe OS · Cangji** is an **AI-native inventory-and-sales ERP** (进销存). Its core claim:

> This is not "an ERP with AI features bolted on." This is "an ERP where AI is the operating system."

What does that mean? A traditional ERP's main interface is menus and forms; Cangji's main interface is **conversation**. You tell it what you want, the way you'd tell a colleague, and it understands your intent, calls tools, performs operations, and returns results. The traditional console (tables, buttons, dashboards) still exists, as a supplementary entry point for high-frequency work.

It has four core differentiators:

- **Conversation is the UI**: CRUD happens through natural language. "Place an order for 100 mosquito-killer lamps for Zhang San" — the AI looks up the customer, checks the product, computes the amount, creates the order, and presents it.
- **Create a table and it goes live**: adding business capability doesn't require touching the frontend. Create a new table in the database, and the AI discovers it automatically through tools and learns to operate it. The cost of business expansion drops from "rewrite both frontend and backend" to "add one table."
- **The AI cannot fabricate data**: any answer that touches business data requires the AI to call a tool and get a real result first. There is no such thing as "the AI hallucinated fake data" — inventory, orders, customers, amounts are all backed by real database queries.
- **Informed consent**: irreversible operations (create, update, delete, batch import) must first pop up a confirmation card — the user sees "what's involved, what's affected" and clicks confirm before anything touches the database. The AI cannot silently modify or delete data.

Feature-wise it covers the full inventory-and-sales domain: products, inventory, barcode scanning, orders, purchasing, customers, returns, drop-shipping, waybill outbound, income/expense bookkeeping, a to-do board, import/export, an Excel report engine, permissions, and multi-tenancy. Warehouse staff can scan codes on a phone and even process outbound while offline; a boss can ask the AI to generate an accounts-receivable report while traveling.

Technically it's a decoupled frontend/backend app: Vue 3 on the frontend; the backend is built on PocketBase (an open-source realtime backend platform offering database, auth, REST API, realtime subscriptions, and hooks); the AI side is built on Eino (an open-source Go agent orchestration framework by ByteDance).

If you'd like to learn more, visit the official site: [https://www.qihebook.cloud](https://www.qihebook.cloud).

Everything below is the design of the "AI tool system" that powers the capabilities above.

---

## 3. Core Theses

Before the tool system, let me state three theses that run through the entire article. All designs derive from these.

### Thesis 1: Natural language is the operating system's interface

In traditional systems, the interface is a form; in an AI-native system, the interface is conversation. That means the system must both **understand intent** (natural-language understanding) and **perform actions** (tool calls). Understanding is only the precondition; execution is the main act — an AI that chats but can't do anything is worthless to a business system.

### Thesis 2: The AI cannot fabricate data

This is our most fundamental, non-negotiable discipline. Any answer that touches business data requires the AI to call a tool, get a real result, and answer based on that result. Inventory, orders, customers, amounts — all must be the outcome of real database queries.

"Don't fabricate" is easier said than done, because an LLM's default behavior is to "weave a plausible answer." Making it real isn't about writing "please don't fabricate" into the system prompt; it's about:

- turning "look it up before you speak" into a mandatory tool-calling rule;
- making the tool's structured result enter the conversation context directly and become the *only* source of truth the AI answers from;
- restating and enforcing this discipline at every layer — persona, skills, and code.

### Thesis 3: Tools are the AI's only bridge to the data world

We wrap every data operation into a **tool**. The AI never reads or writes the database directly. It can only:

```
call a tool → the tool executes → return a structured result → the AI continues from the result
```

This "single bridge" design yields three huge benefits:

1. **Safe and controllable**: every operation passes through the same gate, where you can add permission checks, confirmation interception, and audit logging.
2. **Learnable by AI**: tools are the AI's capability list, and tool definitions are its learning manual.
3. **Data fidelity**: the AI never gets a chance to fabricate — any data it answers with comes from a tool's real return.

---

## 4. Overall Architecture

Here's a conceptual diagram of where the tool system sits in the architecture:

```
┌─────────────┐  natural language  ┌──────────────────────────────┐
│  Frontend   │ ──────────────────▶ │     Chat streaming API       │
│ (Vue/PWA)   │ ◀────────────────── │  (streaming, token by token) │
└─────────────┘     SSE stream      └──────────────┬───────────────┘
                                                   │
                                                   ▼
                            ┌──────────────────────────────┐
                            │          Agent loop          │
                            │ (understand → pick a tool →  │
                            │  execute → read result → go)  │
                            └──────────────┬───────────────┘
                                           │ tool call
                                           ▼
                            ┌──────────────────────────────┐
                            │        Tool execution        │
                            │  permission → confirm → audit │
                            │             → execute        │
                            └──────────────┬───────────────┘
                                           │
                                           ▼
                            ┌──────────────────────────────┐
                            │  Tool executor (separate CLI) │
                            │  params in via stdin,         │
                            │  JSON out via stdout          │
                            └──────────────┬───────────────┘
                                           │ REST / direct
                                           ▼
                            ┌──────────────────────────────┐
                            │       Data API + database     │
                            │ row-level rules + hooks + FSM │
                            └──────────────────────────────┘
```

One line of responsibility per layer:

| Layer | Responsibility |
|-------|----------------|
| Frontend | Show the conversation, render the stream, pop confirmation dialogs, render result cards |
| Chat streaming API | Receive user messages, push the AI's thoughts, tool calls, and results to the frontend as an SSE stream |
| Agent loop | LLM reasoning: understand intent, decide which tool to call, digest tool results, compose the reply |
| Tool execution | Run every tool call through an interceptor chain: permission → confirmation → audit → execute |
| Tool executor | A process separate from the main service that actually runs tool logic and talks to the data API |
| Data API + database | The final data reads/writes, with row-level isolation rules, business hooks, and state machines |

A principle runs through the whole architecture:

> **The AI only does what the platform can't.** LLM reasoning, intent understanding, streaming output, tool orchestration — those are AI-only. Permission, data validation, state transitions, transactional consistency — those belong to the platform layer (database rules + hooks). Never rely on the AI being "well-behaved."

The rest of this article revolves around the "tool layer."

---

## 5. Pillar One: Tools Defined by a Single Source of Truth

### 5.1 The problem

The first thing a tool system needs is a definition of "what tools exist and what each one looks like." If the tool list and descriptions are scattered (one copy in code, one in help docs, one in the prompt), they will inevitably drift — the "business card" the AI sees won't match the actual behavior, and the AI will pick the wrong tool.

### 5.2 The design

We converge **all tool definitions into one central table**. Each tool definition contains:

- **Name**: the unique identifier the AI uses to call it;
- **Summary**: one sentence (≤30 characters), injected into the system prompt so the AI can quickly see "what's available";
- **Full description**: when to use it, when not to, how to use it, with examples — fetched on demand through a help interface;
- **Parameters**: name, type, required or not, default value, description for each parameter;
- **Destructive flag**: whether it changes data (this drives confirmation and retry policy downstream);
- **Hidden flag**: tools exposed only to internal flows, never to the AI (e.g., saving chat messages, loading history).

### 5.3 Why this design

- **Single source of truth**: the tool catalog, help docs, permission mapping, and parameter validation all derive from this one table. Adding a tool = adding one row. The situation "the docs say the tool exists but it doesn't" can never happen.
- **Prompt economy**: the system prompt is a scarce resource. Each tool contributes only its one-line summary; the full description is fetched on demand. With dozens of tools, the prompt holds just a few lines, and the AI queries help when it wants details — this saves a huge amount of tokens versus stuffing every long description in, and it avoids confusing the AI with a wall of text.

This "summary in the prompt, details on demand" idea is the foundation of the whole tool UX. Chapter 8 explains how the AI does that "on-demand lookup."

---

## 6. Pillar Two: Two Tool Shapes

More tools are not better. We converge tools into two shapes, each with its own use case.

### 6.1 Macro tools: one action, one thing

One tool = one atomic action, few parameters, unambiguous semantics. Examples: "create a record," "delete a record," "describe a table schema."

Use cases: the action itself is indivisible, or decomposing it would add extra call rounds and more chances to fail.

### 6.2 Script-language tools: send a sequence of atomic instructions at once

Some needs are **multi-step compositions**: "query customers → filter balance > 0 → sum amounts → sort → take top 10." If every step were a tool, the AI would make five round-trips, wasting tokens and latency each time, and intermediate state is easy to lose.

Our approach: **the CLI is a scripting language.** Treat a family of capabilities as "a small language": the AI sends an ordered array of atomic instructions (`steps`) in one call, and the system executes them sequentially:

```
[{op: from, collection: customers, filter: balance>0},
 {op: aggregate, func: sum, field: balance, as: total_owed},
 {op: sort, by: total_owed, order: desc},
 {op: limit, n: 10}]
```

Two canonical script-language tools:

- **The query language**: `from (fetch rows) → search (fuzzy/exact) → group (group by) → aggregate (sum/count/avg/max/min) → sort → limit → select (project columns)`. The AI assembles queries like building blocks; every step is deterministic computation.
- **The Excel-building language**: `create → write_header → write_row → data_import (pull data from the database) → style (beautify) → save (persist and return a download link)`. The AI says "generate an accounts-receivable report," composes a sequence of atomic instructions itself, and submits it; the system executes in pure memory, zero external dependencies.

### 6.3 Common design of the script engine

Both script languages share the same engine design:

- **Op docs live next to the implementation**: each op's name, parameters, and examples are written beside its implementing code; at startup the system assembles them into a complete language manual. When the AI queries help, it sees the full op set, language rules, and copy-pasteable examples.
- **Pre-validation with typo correction**: op names are validated before execution; an unknown op is rejected with a "closest by edit distance" suggestion (if the AI types `sotr`, the system says "did you mean `sort`?").
- **Resource limits**: a cap on step count and a wall-clock timeout for the whole script. Prevents an AI from submitting 100,000 steps and stalling the system.
- **Atomic semantics**: if any step fails, the whole script fails. All-or-nothing — never a half-executed dirty state.
- **Diagnostics**: on failure, returns "how many steps executed, which step failed, and why," so the AI can fix just that step and resubmit the whole script.

### 6.4 Why two shapes

Behind this is an iron rule: **tool count is an LLM cognitive burden.** Every extra tool raises the odds the AI picks the wrong one. Absorbing all multi-step composition needs into script languages keeps the tool count bounded — we once went from 20 tools down to 18, and the system became *stronger*, by merging "get one record / list records / aggregate / sort" into one query-language tool instead of adding three new ones.

---

## 7. Pillar Three: The Tool Executor Process

### 7.1 Why run tools in a separate process

Tool logic does not run inside the main service; it runs in a **separate CLI process**. Why?

1. **Fault isolation**: if a tool execution (parsing a 50 MB Excel file, running a complex aggregation) crashes or hangs, only that child process is affected; the main service survives. The main service gives the child a timeout and simply kills and restarts it on expiry.
2. **Language freedom**: a CLI process is a black-box interface. In theory any language can implement tools as long as it follows the same stdin/stdout protocol. We use Go throughout, but the architecture doesn't care.
3. **Process-level privilege boundary**: the tool process doesn't need the main service's full authority; it holds only the current user's credentials and does only what those allow.

### 7.2 The protocol: JSON in, JSON out

The tool process protocol is extremely simple:

- **Input**: parameters arrive via stdin (one-shot mode) or line by line (persistent mode). Parameters are JSON.
- **Output**: the process writes **one line of JSON** to stdout, with a uniform structure (success flag + structured data + error message + error category + next-step suggestion).
- **stderr discipline**: stderr carries debug logs only and never participates in result output.

That "one JSON line on stdout" discipline seems trivial, but it once saved our lives: early on, error messages were written to stderr; in a log-mixed environment the result parser would read log lines as JSON, and the AI got a pile of "parse failed" errors — one of the most insidious and deadly bugs in an AI tool system.

### 7.3 Persistent mode (--serve)

Forking a process per call is expensive (process creation + Go runtime init + connection-pool rebuild). To squeeze that overhead, we added a persistent mode:

- the main service keeps a **pool of resident worker processes** (a connection pool);
- it sends commands line by line through stdin (one JSON per line: command name + identity + parameters) and reads results line by line from stdout;
- used processes are not destroyed; they return to the pool for reuse.

This drops per-call overhead from "millisecond-scale process creation" to "microsecond-scale I/O," so the AI can make dozens of tool calls across a conversation without pressure.

### 7.4 Tiered timeouts

Execution times vary wildly across tools: looking up one record is milliseconds; generating a complex Excel is seconds; batch-importing waybills can take minutes. So timeouts are not one-size-fits-all; they're tiered:

- normal tools: 60 seconds by default;
- long-running tools (batch import, etc.): relaxed to 10 minutes.

The lesson came from an incident: early on, every tool shared a uniform 60-second timeout, so batch imports were killed halfway through — the user stared at "import failed" while data had already been partially written. Timeouts aren't "bigger is better" or "smaller is better"; they must be tiered by tool nature and combined with atomicity (Pillar Two) so no dirty data survives.

---

## 8. How the AI Operates Tools: a Discover–Choose–Call–Recover Loop

Using a tool is not a single action; it's a full loop. Each stage has a matching design.

### 8.1 Discover: an on-demand help catalog

How does the AI know what tools exist and how to use each one?

- The system prompt holds only a **one-line summary** per tool (Pillar One), so the AI knows "these capabilities exist."
- For details, the AI calls the help system: list tools, search tools by intent, get a tool's full usage, or read a script language's op table and examples.

This help system is **generated dynamically** — when tool definitions change, the help changes automatically; no human maintains the docs. The AI never holds stale instructions.

### 8.2 Choose: skill documents teach the AI to pick the right tool

Choosing the wrong tool ruins even the best execution. We teach the AI to choose through **skill documents**.

A Skill is a piece of business knowledge stored in the database — for example "order handling," "inventory counting," "customer analysis," "Excel reports." Each skill document states:

- which tools to use for this kind of task and in what order;
- which tools **not** to use (to stop the AI from misusing them);
- typical examples.

The AI loads skills on demand during conversation (the user can trigger one directly with `/skill-name`, or the AI decides it needs one). Skills are **readable, updatable, versionable knowledge** — giving the AI a new capability doesn't require code changes; just write a new skill document.

### 8.3 Call: parameter validation + type tolerance

When calling tools, the AI's most common error is unstable parameter formatting: it passes a string where an object is declared, or a comma-separated text where an array is expected.

Our strategy is **system-side tolerance**:

- the server runs a **parameter pre-validation** before execution (required fields present, types legal); violations return a structured "parameter error" for the AI to fix — bad parameters never reach business logic;
- structural parameters (objects, arrays) accept **two shapes**: native JSON or a JSON string. Either works; the system parses uniformly.

### 8.4 Recover: errors as teaching

The AI will inevitably call the wrong tool, pass bad parameters, or trip a business rule. The key is not "avoid errors" but "let the AI recover by itself." That's the subject of the next chapter.

---

## 9. Errors as Teaching

This chapter is, in my view, the most worth-sharing design in the whole tool system.

### 9.1 Core insight: an error is not a failure, it's data

In traditional systems, errors are for developers (stack traces, status codes, logs). In AI systems, errors are **instructions for the next step of the AI**.

LLMs have no "debugging ability"; they only have text context. When a tool fails, the AI's only basis for action is the error text the tool returned. So the design standard for error messages is:

> **Error messages must let the LLM self-correct** — state clearly "why it failed, what to do instead, and what to do next."

### 9.2 A uniform result contract

Every tool returns a uniform structure:

```
- success          whether the call succeeded
- data             structured result data
- error            the natural-language error message for the AI (why + how to fix)
- error_category   machine-readable category (for the orchestration layer)
- suggestion       the recommended next action for the AI
```

On success the AI takes `data`; on failure it takes `error + suggestion` and decides how to fix and retry. The orchestration layer (retry, circuit breaker, audit) consumes `error_category` for machine decisions — the category is not for humans, it's for the system.

### 9.3 Translating "typical AI mistakes" into targeted hints

This is the most valuable part. We anticipated the typical mistakes an AI makes and designed error messages specifically for them:

| Typical AI mistake | The system's response |
|--------------------|------------------------|
| Misspelled tool name | "Unknown tool — did you mean xxx?" (edit-distance suggestion) |
| Missing required parameter | Lists which one is missing and suggests checking help |
| Passing a "business number" as a record ID for a relation field | Intercepts: "that's an order number — you need the record ID returned at creation," and teaches the correct way |
| Writing sorting as a standalone op (`{op: order}`) | "There is no `order` op — `order` is a parameter of `sort`, not an op," with a corrected `{op: sort, by: ...}` example |
| Writing a date by intuition, `= '2026-07-29'`, and getting no hits | The system rewrites the pure-date comparison into a time range and explains why |
| Step N of a script fails | "Failed at step N; the failing op is xxx; reason: ...; fix that step and resubmit the whole script" |
| Trying to advance an order's status (not allowed) | States plainly "the AI cannot advance business status," and gives the alternative path (create a draft; the user advances it in the console) |

These messages weren't written on a whim; they came from real conversation logs between users and the AI. Each one answers three questions: **what went wrong, why, and what to do next.**

### 9.4 Three audiences for one failure

- the natural-language error → the AI reads it and self-corrects;
- the machine category → the orchestration layer decides (e.g., "don't retry validation errors; auto-retry network errors");
- the full raw error → the audit log, for human troubleshooting.

One failure event, and the AI, the system, and the human each get exactly the information they need.

---

## 10. The Three-Layer Constraint Model: How to Keep the AI in Line

Now to the core of this article. Once the AI has tools, what stops it from running wild?

Our answer is the **three-layer constraint model**. Each layer answers a different question:

| Layer | Carrier | Question it answers | Nature |
|-------|---------|---------------------|--------|
| Layer 1: prompt/persona | persona definition + skill documents | How the AI **should** behave | Soft — relies on model discipline |
| Layer 2: tool contract | result contract + confirm-ID protocol | When the AI errs, **how it gets pulled back** | Semi-hard — self-correction driven by errors |
| Layer 3: hard code interception | permission middleware + data rules + circuit breaker | Which actions are **never allowed**, ever | Hard — independent of model behavior |

### 10.1 Layer 1: the prompt/persona layer (teach)

Write behavioral rules into the AI's "persona" and skill documents. For example:

- **No fabricating data**: answers touching business data must call a tool first;
- **Look before you answer**: when unsure, query; don't guess;
- **Informed consent**: irreversible operations require confirmation first;
- **Stop after repeated failures**: if the same tool keeps failing, stop that thread and tell the user, instead of grinding on;
- **Ask when information is insufficient**: if intent is incomplete, ask first; don't guess and proceed.

This layer's strengths: readable and iterable (change a document, change behavior). Its weakness: **it relies on model discipline and will be violated.** That's why it's layer one, not the whole story.

### 10.2 Layer 2: the tool-contract layer (correct)

When the AI breaks a rule, the tool's structured error pulls it back on track (see the previous chapter). The key is that the error must be *actionable* — the AI knows what to do next after reading it.

The confirm-ID protocol lives here too: high-risk tools require a confirmation ID before they'll execute; without one, the tool refuses and tells the AI the correct flow.

### 10.3 Layer 3: hard code interception (lock)

This is the real defense line; it relies on no prompt at all. Every tool call passes through an interceptor chain:

```
parameter validation → permission check → confirmation gate → audit → retry policy → execute
```

Walking through each:

**① Permission check**: each tool maps to permission points (read/write/manage), enforced by the user's role (founder / admin / employee). No permission → immediate refusal, never entering business logic. The mapping is explicitly maintained — adding a tool requires declaring its permission.

**② System-table blacklist**: a set of internal tables (users, audit logs, sessions, confirmation records, etc.) is fully hidden from the AI — the AI can't even *see* them, let alone operate on them. Hiding isn't a prompt instruction like "don't touch these tables"; the tool layer refuses outright.

**③ Multi-tenant isolation**: the tenant identifier flows from the request into every write; the AI doesn't pass it and cannot pass it (see Chapter 12).

**④ Central write policy**: not every "update" is allowed. A central set of write rules constrains what the AI may change. The whole policy condenses into one sentence:

> **The AI can never advance business status: it can only create drafts and delete records in a cancelled/rejected terminal state. Every other state transition is performed by a human in the console.**

Breaking that down:

- **Drafts only**: when creating a document, status fields are always rejected — anything the AI creates is a draft. The AI can't "place the order," "ship it," or "complete it"; it can only prepare the draft, and the user advances it in the console.
- **Terminal states only for deletion**: deletion is equally constrained — only "cancelled/rejected" terminal states (and draft states) can be deleted by the AI; in-flight business documents are untouchable. So even a "wrong delete" is limited to already-finished business.
- **No state-machine advancement**: "draft → confirmed → shipped" for orders, "draft → ordered → received" for purchasing — any update carrying a status field is rejected outright. The AI can edit non-status fields (customer name, quantities) but cannot make business decisions on the user's behalf.
- **Protected fields**: status-class fields can never be changed by the AI, not even on creation.
- **Parent-record validation**: writes to line items must verify the parent record exists and belongs to the current tenant.
- **Relation fields must take IDs**: to relate a record, the AI must pass the system-generated record ID; passing a "business number" is intercepted (see the errors-as-teaching chapter).

**⑤ Confirmation gate**: high-risk operations are forced through confirmation (next chapter).

**⑥ Consecutive-failure circuit breaker**: after a tool fails consecutively up to a limit, it trips and returns "this path is dead; try another way or terminate the task" — preventing the AI from re-calling the same broken tool in a loop and burning tokens.

**⑦ Iteration cap**: the whole conversation has a reasoning-step cap; beyond it, execution stops. Prevents infinite tool-call loops.

**⑧ Context-overflow degradation**: when the AI's context window fills up, the task is not refused — large content is offloaded to disk and replaced with a placeholder the AI can read back on demand (Chapter 13).

### 10.4 How the three layers cooperate

One example to tie it together:

The user says, "Cancel order NO.20260715001."

1. **Layer 1**: the AI's persona demands "confirm before irreversible operations," and the skills teach it "look up the order → call the confirm tool → wait for the user → then execute."
2. **Layer 2**: if the AI skips confirmation and directly calls "update order," the tool contract refuses: "this operation requires confirmation," and tells it the correct flow.
3. **Layer 3**: even with a confirmation ID and fixed parameters, the hard layer still checks: changing `status` is **state-machine advancement**, never allowed for the AI — refused, with a hint "complete the cancellation in the console." Even if the cancellation is business-legitimate, a human must do it by hand in the UI.

Any layer can stop the AI, but the semantics differ: layer 1 *persuades*, layer 2 *corrects*, layer 3 *locks*. Remove any layer and the system is unsafe; keep only layer 3 and the system becomes unusable (the AI is constantly rejected and can't self-correct); keep only layers 1–2 and the system runs out of control.

---

## 11. Human Confirmation (Confirm)

The confirmation mechanism is the most delicate, and the easiest to screw up, part of "constraining the AI." The typical screw-ups: the AI spams confirmation dialogs until the user goes crazy, or the dialog becomes a rubber stamp. Our design goal: **critical operations require a human decision, but the friction of the flow must be minimal.**

### 11.1 The basic flow

1. The AI judges an operation high-risk (create / update / delete / batch import) → first calls the "confirm" tool and submits a confirmation request (what it will do, which records are affected, severity);
2. The frontend pops a confirmation card; the user sees "which table, which record, what changes";
3. The user clicks "confirm and execute" → the system issues a **confirmation ID** (valid 5 minutes);
4. The AI uses that ID to perform the real operation; the tool verifies the ID is valid before letting it through;
5. The user can also click "reject" → the AI receives the rejection signal and abandons the operation.

### 11.2 Four subtle details

**① Confirmation IDs are reusable for a batch.** One confirmation ID can be used multiple times within 5 minutes. Why? Because one business action often splits into multiple record writes (a purchase order = header + line items + linked expense). If each record needed its own confirmation, the AI would pop three dialogs and the user would lose it. Confirm once and execute the same batch — safe *and* efficient.

**② The confirm tool must be called in a round by itself.** In one round the AI may call several tools at once. If the confirm tool appears alongside others, the system **strips the others and executes only the confirmation**. Why so strict? Because with parallel calls, if the other tools' parameters get lost to context compression, the AI may rebuild them and execute the wrong object — the user confirmed "operation A" but "operation B" actually ran. That defeats the entire point of confirmation. Better to spend one extra round than to have "what you confirmed" ≠ "what ran."

**③ Confirmation context is exempt from cleanup.** In long conversations, context gets compressed and cleaned (Chapter 13). But confirmation-related records are **exempt** — if the AI "loses its memory," it will spam confirmation dialogs. Keeping that context means the AI knows "I already got confirmation — just execute with the ID."

**④ One sentence from the user releases or aborts.** After confirmation, if the user says "confirm / go ahead," the AI executes directly without re-confirming; if the user says "forget it / cancel," the AI abandons it. The gate serves people; it's not decoration on a process.

### 11.3 The confirmation state machine

Confirmation requests have a full lifecycle: pending → confirmed / rejected; a confirmed request is reusable within its validity window and expires automatically; orphaned pending requests are cleaned up by a background job. Every confirmation action is audited — who confirmed, when, and what — fully traceable.

---

## 12. Multi-Tenant Isolation: the AI Doesn't Even Need to Know

The nightmare of multi-tenant systems is cross-tenant leakage — Company A's AI modifies Company B's orders. Our isolation design has a distinctive feature: **the AI doesn't need to know tenancy exists at all.**

### 12.1 Full-chain injection

- at login, the system resolves the user's tenant identifier;
- on every tool call, that identifier travels end-to-end: request → tool adapter → executor process → data API;
- on writes, the system **unconditionally overrides** the tenant identifier — even if the AI passes one, it's ignored; the system always uses its own;
- on reads, the tenant filter is **appended automatically** — the AI's hand-written filter doesn't need a tenant condition and isn't allowed to write one.

### 12.2 A database-layer backstop

Even if the AI layer leaks, the database layer is the last line: data tables have access rules that force "the record's tenant = the current user's tenant." This rule is a platform-level capability, not application code — even someone bypassing the AI and calling the API directly can't read another tenant's data.

### 12.3 Why the AI should be "ignorant"

If the AI were tenant-aware, it would make two kinds of mistakes: omit the tenant condition (reading others' data) or write it wrong (failing to read its own). Removing tenancy from the AI's cognition entirely is a "less is more" move — the platform layer absorbs the complexity, and the AI focuses on business.

---

## 13. Context Economy: the AI's Memory Is a Scarce Resource

The LLM's context window is finite, and every tool call consumes it. Without context management, the system slowly "gets dumber" across long conversations. Our principle:

> **Never refuse; only degrade.** No matter how full the context gets, the task must not be interrupted — degrade the content, and the task continues.

### 13.1 Save at the source

- query tools return small result sets by default (say 50 rows), never pulling whole tables;
- oversized results are **auto-slimmed**: only key columns are kept; the AI asks explicitly when it needs full fields;
- even larger results (say a 10,000-row export) become **files** with a download link — the AI's context only ever holds the link, not the content.

### 13.2 Degrade when over the limit

- a single tool result over a threshold (say 8 KB of characters) is offloaded to disk; the context keeps a placeholder (path and size), and the AI pages it back on demand with a "read back" tool;
- when cumulative context approaches the ceiling, the earliest tool calls' parameters and results are cleaned and offloaded, **keeping the most recent few rounds**;
- critical content is exempt: confirmation context (so the AI doesn't lose memory and re-spam dialogs), and the read-back tool's own results (preventing a read-back-that-gets-offloaded-again infinite loop).

### 13.3 Why "degrade" instead of "refuse"

Refusing = task interruption, broken UX. Degrading = the task continues; the AI just needs one extra "read back on demand" step. We take the burden of "remembering everything" off the model and hand it to deterministic systems (disk, files, links); the model fetches only when needed. That's what makes long sessions and big data feasible.

---

## 14. Tool Evolution Methodology

A tool system isn't built once; it grows and shrinks as the business evolves. We've distilled a methodology for managing that.

### 14.1 Three iron rules

**Anti-complexity**: if a script language can extend an existing tool's capability, don't add a new tool. Tool count is an LLM cognitive burden; every extra tool raises the odds of picking wrong. We went from 20 tools to 18 and got *stronger*, precisely by merging instead of adding.

**Anti-architecture-debt**: retire tools by **removing them outright** — no soft "hidden" shutdowns, no compatibility layers. The AI has no "muscle memory"; migrating means editing one skill document. Keeping dead tools only bloats the help catalog and makes mis-selection more likely.

**Single source of truth**: maintain the tool list in exactly one place (Pillar One). Help, permissions, validation all derive from it; never maintain a second copy.

### 14.2 Checklist for adding a tool

1. First ask: can a script language extend an existing tool? Only if not, add a new one;
2. Define the tool metadata (summary ≤30 chars; description includes "when to use / when not to" + examples);
3. Declare its permission mapping;
4. Declare its destructive level (high-risk? requires confirmation?);
5. Write a skill document teaching the AI how to use it;
6. Full-chain tests + update the tool-count stats.

### 14.3 Checklist for removing a tool

1. Remove the tool definition and implementation;
2. Remove it from permission mappings, audit categories, and constant lists;
3. Rewrite skill documents, replacing call examples with the alternative;
4. Grep the whole codebase to confirm no lingering references;
5. Test + deploy + smoke-verify.

This methodology keeps the tool system "small and sharp" through long-term evolution rather than "big and messy."

---

## 15. Integration with the Agent Framework

The tool system ultimately runs on an agent framework. We went from a hand-written loop to a framework; here are a few key takeaways.

### 15.1 Hand-written loop vs. framework

The early version used a hand-written ReAct loop (model generates → execute tools → feed results back → generate again); later we migrated to an open-source agent orchestration framework (Eino ADK). The framework took over the loop control, streaming output, and context management; we kept only the business parts (the tool layer, the confirm protocol, the SSE protocol).

**Key principle: the framework owns the generic; the business layer owns the specific.** The tool execution layer (permission/confirmation/audit) and the external protocol (streaming frames) are our business assets — the framework shouldn't touch them. Loops, retries, and model switching are generic mechanisms — hand them to the framework.

### 15.2 Model failover

AI capability depends on model services, and model services can rate-limit or go down. Our strategy is a **model chain**: when the primary model fails or is rate-limited, fall back to alternates automatically. Switching isn't just "swap one for another"; it also involves cost-awareness (prefer cheaper fallbacks) and recovery (new requests start from the primary again).

### 15.3 Two technical routes for confirmation

Human confirmation can be implemented in an agent framework two ways, and we made an explicit trade-off:

- **Route 1 (current)**: after the confirm tool runs, it **terminates the current reasoning round**; the AI waits for the user; once confirmed, the AI retries the original operation in the next round carrying the confirmation ID. Simple and stable.
- **Route 2 (future optimization)**: use the framework's suspend/resume — when the confirm tool runs, the whole session state is archived; after the user confirms, the **same session resumes** and continues. Better UX (the AI doesn't "lose memory") but depends on the framework's checkpoint mechanism — more complex.

**Advice: get Route 1 working first; it's good enough. Move to Route 2 only when confirmation becomes a bottleneck.** Don't chase the perfect architecture from day one.

### 15.4 The SSE streaming protocol

The frontend needs to see, in real time, the AI's thinking, its tool calls, and confirmation dialogs. We define a handful of streaming frame types: text tokens, info notices, tool-call starts, tool results, confirmation requests, errors, and session end. The frontend renders by frame type — that's the low-level backbone of "conversation is the UI."

---

## 16. A Blueprint You Can Take Away

If you want to give your own AI a tool system, here's our advice.

### 16.1 The minimal viable combination

Don't build everything at once. Start with these five pieces and you'll have the backbone of "an AI safely operating a system":

1. **Tool metadata table**: one central definition of tools (name/summary/parameters/destructive flag) — the starting point of everything;
2. **Uniform result contract**: all tools return the same structure (success/data/error/category/suggestion) — the basis of AI self-correction;
3. **Permission gate**: every tool declares required permissions, enforced before execution — the safety floor;
4. **Confirmation gate**: high-risk operations confirm first, then execute with a confirmation ID — the guarantee that "a human decides";
5. **Context management**: summaries in the prompt, details on demand, overflow degrades instead of refusing — the guarantee that long sessions don't degrade.

### 16.2 A checklist of decision points

- Tool shape: macro or script language? — prefer script language for multi-step compositions;
- Execution location: in-process or separate process? — a separate process buys fault isolation and language freedom, at the cost of process management;
- Error design: who are errors for? — always remember errors are for the AI, and must be actionable;
- Confirmation granularity: single confirmation or batch reuse? — think first about what "a batch" is for your business actions;
- Permission granularity: tool-level or field-level? — start tool-level; add field-level when needed;
- Framework choice: hand-written or framework? — framework for the generic, your own code for the business parts.

### 16.3 The three most important things

If you remember only three sentences:

1. **Tools are the AI's only bridge to data** — every operation goes through the same gate, and safety, audit, and confirmation all live on that gate;
2. **Error messages must let the LLM self-correct** — an error is not an endpoint; it's the AI's next instruction;
3. **Lock critical permissions in code, don't persuade with prompts** — the AI may violate prompts, but it can't break code.

---

## 17. Pitfalls We Hit

Finally, some real lessons. Every one cost us, and every one left a corresponding design in the system.

1. **A uniform timeout killed long tasks**: early on, every tool shared a 60-second timeout; batch imports were killed halfway, leaving half-dirty data. → Tier timeouts by tool nature (see 7.4).
2. **stderr mixed with results**: error messages were written to stderr; in mixed logs the result parser read non-JSON, and the AI got a pile of "parse failed." → stdout carries only result JSON; stderr carries only logs (see 7.2).
3. **Audit recorded only successes**: early on, only successful operations were audited; failed operations (possibly attacks or probes) left no trace. → Record successes and failures alike (see 10.3).
4. **Errors leaked internal details**: on tool failure, the full internal error (with paths and stack traces) went back to the AI. → Truncate outward-facing output; keep only an actionable summary for the AI (see 9.4).
5. **Orphaned tool-call records**: after confirmation, context was compressed, the AI forgot it had already asked for confirmation, and it kept re-popping dialogs. → Confirmation context is exempt from cleanup (see 11.2).
6. **Date-by-intuition never matched**: the AI wrote `date = '2026-07-29'` by intuition and got zero hits (the database stores full timestamps). → The system rewrites pure-date comparisons into time ranges and tells the AI (see 9.3).
7. **Tenant identifiers could be forged by clients**: early on we trusted the tenant ID sent by the client. → The server re-proves access via membership; the client can only express intent (see 12.1).
8. **The hand-written loop became unmaintainable**: the manual ReAct loop accumulated streaming assembly, retries, and context compression; touching one thing broke three. → Migrate to an agent framework; keep only the business assets (see 15.1).

---

## 18. Appendix

### Appendix A: A tool-call sequence for one conversation

The user says: "Create a customer profile for Zhang San."

```
User ──▶ AI
         │
         ├─ calls the "query" tool: search customers by name, confirm Zhang San doesn't exist yet
         │     ← result: 0 hits
         │
         ├─ calls the "confirm" tool: submit a confirmation request
         │     action: create customer "Zhang San"
         │     target: customers table
         │     severity: medium
         │     ← result: confirmation ID #abc123 (waiting for user)
         │
User sees the confirmation card, clicks "confirm and execute" ──▶ system
         │
         ├─ calls the "create" tool: create the customer, carrying confirmation ID #abc123
         │     ← result: new record {id: "xxx", name: "Zhang San", ...}
         │
         ├─ calls the "card" tool: format the record into a result card
         │     ← result: card rendering data
         │
User ◀── card shown: "Customer Zhang San created successfully"
```

Key points: the confirmation ID and the actual execution are two separate steps; the confirmation request lets the user see exactly "what's being created"; after execution, the result is shown as a card, not raw JSON.

A conceptual success-result shape (not real data):

```json
{
  "success": true,
  "data": { "id": "xxx", "name": "Zhang San", "code": "KH-0001" },
  "error_category": "",
  "suggestion": ""
}
```

A conceptual error-result shape:

```json
{
  "success": false,
  "error": "This operation requires confirmation. Call the confirm tool to get a confirmation ID (valid for 5 minutes); an existing one can be reused and the operation retried directly.",
  "error_category": "confirm_required",
  "suggestion": "Call the confirm tool (action='create customer Zhang San', target='customers') to get a confirmation ID, then retry"
}
```

### Appendix B: Glossary

| Term | Meaning |
|------|---------|
| Tool | The AI's only entry point to the data world — one callable unit of action |
| Macro tool | A simple tool: one action, one thing |
| Script-language tool | A composite tool: one call carries a sequence of atomic instructions executed in order |
| Confirmation (confirm) | The human gate before high-risk operations; the human decides and a confirmation ID is issued |
| Confirmation ID | A temporary credential issued after user confirmation, valid for 5 minutes, reusable across a batch |
| Skill | A business-knowledge document stored in the database that teaches the AI how to handle a class of tasks |
| Subject | The concept of tenant/company; each subject is an independent business world |
| Console | The traditional form UI — the auxiliary entry for high-frequency work, running alongside conversation |
| FSM / state machine | The status-transition engine for documents such as orders and purchases; the AI cannot advance it directly — it can only create drafts and delete cancelled/rejected terminal-state records |
| Context degradation | When context overflows, content is offloaded to disk and read back on demand; the task is not interrupted |
| Error category | The machine-readable class of a tool error, consumed by the orchestration layer for decisions (retry / circuit-break / audit) |

---

*Compiled from the real engineering practice of Qihe OS · Cangji. Ideas are reusable; code goes stale — may these designs help you give your AI a pair of safe hands in your own project.*
