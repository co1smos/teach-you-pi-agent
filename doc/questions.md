# Pi Architecture Questions & Learning Workbook

[中文](questions.zh-CN.md) | [How to use](../README.md)

Learn Pi architecture through source-reading questions. No private student answers or completed progress are included. No earlier course or private project is required.

## Source baseline

- Upstream: https://github.com/earendil-works/pi
- Pinned source revision: `a32782520f69cd81b54814c3a13df4c7bd1f3ad7`
- Source paths below are relative to this repository root. Read the source on `main`, not moving upstream `main`.
- Goal: explain loop, state/session, messages, tools, termination, and external-state boundaries from evidence, not file-name memorization.
- This is a source-reading exercise, not a request to refactor Pi. No dependencies, provider credentials, or live LLM calls are required to read the code.

## Learning navigation

| Block | Focus | Teaching order |
|---|---|---|
| 1 | Agent Loop & Lifecycle | Q1 → Q13 |
| 2 | State / Session | Q2 → Q3 → Q4 → Q6 → Q7 → Q14 |
| 3 | Messages / Model Context | Q5 → Q8 |
| 4 | Tool System | Q9 → Q10 → Q11 → Q12 |

Q numbers are stable IDs, not ascending reading order. Each block contains objectives, primary source entry points, and question-specific requirements. The block introductions are not answer keys.

# Teacher protocol

The teacher should behave like a demanding technical instructor, not an answer generator.

### Navigation and progress

Follow the block order and question order in the navigation table, not ascending Q numbers. On resume, read the existing progress and start at the first non-PASS question in that order; do not restart passed questions. Announce the current block, topic, and Q ID before asking one question at a time. After a question passes, update both its question-bank status and its matching accepted-answer record in the learner’s private working copy of this document. Question IDs remain stable across this reorganization.

The block introductions are learning objectives, not answer keys. Do not treat a navigation sketch as sufficient evidence for grading.

## 95% pass gate

A question passes only when the student's explanation is at least 95% correct.

"95%" means:

- the core mechanism is correct;
- the important ownership/responsibility boundaries are correct;
- the relevant control/data flow is correct;
- no major concept required by the question is omitted;
- claims are consistent with the pinned source;
- only small naming, phrasing, or nonessential edge-case details remain wrong or missing.

Do not pass an answer that is directionally correct but skips a core layer or confuses two abstractions.

## Before the student passes

When the student's answer is below the gate:

- Do **not** give the correct answer.
- Do **not** provide a near-complete paraphrase of the correct answer.
- State which part of the student's mental model is incomplete, inconsistent, or unsupported.
- Prefer Socratic hints and targeted follow-up questions.
- Point to a small source region: file + symbol, and line range when useful.
- If the confusion is conceptual, suggest a primary-source doc or a focused external article/documentation page.
- If web search is available, use it when a supplemental explanation would materially help. Prefer primary sources.
- Ask the student to revise the same answer. Do not move to the next numbered question.

The goal is to help the student discover the answer from evidence.

## After the student passes

When the answer reaches the gate:

1. Say that the question passes.
2. Mention any remaining minor correction.
3. Edit the private working copy under the corresponding `Accepted answer log` entry:
   - record the student's accepted answer, preserving the student's wording as much as practical;
   - immediately below it, write the teacher's concise standard answer, checking the matching section in answers.md against the pinned source;
   - add the decisive Pi source evidence (file + symbol; line ranges if useful);
   - mark the question `PASS`.
4. Then ask the next question in the block teaching order.

Never write the standard answer into this file before the question passes.

## Evaluation dimensions

Use these dimensions to judge completeness; do not mechanically average them if a core concept is missing.

| Dimension | What must be right |
|---|---|
| Mechanism | What actually happens |
| Ownership | Which abstraction/function owns the behavior |
| Data flow | What data moves between stages |
| Boundaries | Runtime vs session vs provider vs tool responsibilities |
| Edge semantics | Important stop/error/ordering behavior |
| Evidence | Claims can be traced to the pinned source |

## Source-navigation aids

The teacher may use repository graph tools to find call paths and architecture boundaries:

- GitNexus, if installed/available.
- Graphify, if installed/available.
- Normal source search (`grep`, `find`, editor symbol search) is always acceptable.

For Graphify, useful operations may include focused concept queries, path queries between symbols, or architecture reports. For GitNexus, use graph/MCP queries to discover symbols and call paths. In all cases, verify conclusions in the actual Pi source before grading.

Do not spend substantial time installing or debugging a graph tool if direct source inspection is faster.

### Answer separation and language

The published `answers.md` is a separate reference key, not learner progress. The teacher may consult only the current question’s reference privately when necessary to assess an attempt; never reveal or paraphrase it before PASS. Source evidence outranks the key. Record student and teacher answers together only after PASS. Choose one private working copy (English or Chinese) as the sole progress record; switching teaching language does not create a second record. Preserve the student’s original wording and language. Optional translations must be labeled. No automated grader enforces the 95% gate.

# Reading map

Start with the smallest useful sources instead of reading Pi front-to-back.

## Orientation

- `pi/packages/coding-agent/docs/how-pi-works.md`
- `pi/packages/coding-agent/docs/session-format.md`

## Agent loop

- `pi/packages/agent/src/agent-loop.ts`
  - `runAgentLoop()`
  - `runAgentLoopContinue()`
  - `runLoop()`
  - `streamAssistantResponse()`
  - `executeToolCalls()`
  - `executeToolCallsSequential()`
  - `executeToolCallsParallel()`

## Runtime state wrapper

- `pi/packages/agent/src/agent.ts`
  - `createMutableAgentState()`
  - `Agent`
  - `Agent.prompt()`
  - `Agent.continue()`
  - event/state reduction around `processEvents()`

- `pi/packages/agent/src/types.ts`
  - `AgentContext`
  - `AgentState`
  - `AgentMessage`
  - `AgentTool`
  - `AgentToolResult`
  - `ToolExecutionMode`

## Persistent session and context projection

- `pi/packages/coding-agent/src/core/agent-session.ts`
  - `AgentSession`
  - `prompt()`
  - `prepareNextTurn` integration
  - compaction integration only as needed to understand boundaries

- `pi/packages/coding-agent/src/core/session-manager.ts`
  - `SessionEntry` variants
  - `buildSessionPath()`
  - `buildContextEntries()`
  - `buildSessionProjection()`
  - `buildSessionContext()`
  - `SessionManager`
  - `appendMessage()`
  - `branch()`
  - `branchWithSummary()`

## Messages

- `pi/packages/coding-agent/src/core/messages.ts`
  - custom message types
  - `convertToLlm()`

- `pi/packages/ai/src/types.ts`
  - inspect only when needed to resolve base message/content types

## Tools

- `pi/packages/coding-agent/src/core/tools/index.ts`
  - `createToolDefinition()`
  - `createTool()`
  - `createCodingToolDefinitions()`
  - `createCodingTools()`

- `pi/packages/agent/src/agent-loop.ts`
  - execution dispatch and ordering

- `pi/packages/agent/src/harness/execution/tools.ts`
  - `prepareToolCall()`
  - `executeToolCall()`
  - `finalizeToolCall()`
  - optional comparison: this is the separate AgentHarness execution path, not a stage called by agent-loop.ts

# Question bank

Work block-by-block. Within each block, answer the questions in order unless a later question is needed to resolve an earlier one.

The four core blocks cover Pi's architecture:

1. **Agent Loop & Lifecycle**
2. **State / Session**
3. **Messages / Model Context**
4. **Tool System**

Complete all four blocks. This public edition contains Q1–Q14 only.

---

# Block 1 — Agent Loop & Lifecycle

## What this block is trying to answer

By the end of this block, you should be able to explain:

```text
prompt
  ↓
model request
  ↓
assistant response
  ↓
tool calls
  ↓
tool results
  ↓
next model request OR stop
```

The important question is not just "which function runs", but **who owns the repeated control flow and who decides whether the run continues**.

### Main source path

- `pi/packages/agent/src/agent-loop.ts`
  - `runAgentLoop()`
  - `runLoop()`
  - `streamAssistantResponse()`
  - `executeToolCalls()`
- `pi/packages/agent/src/types.ts::FinishTurn`

## Q1 — Who actually owns the agent loop?

Explain one normal Pi turn from the point a prompt is accepted until Pi decides whether another model request is needed.

Your answer must identify:

- the abstraction/function that owns the repeated model/tool cycle;
- where the assistant response is produced;
- where tool calls are extracted;
- where tool results re-enter context;
- what causes another iteration versus termination.

**Primary pointers**

- `pi/packages/agent/src/agent-loop.ts::runLoop`
- `pi/packages/agent/src/agent-loop.ts::streamAssistantResponse`
- `pi/packages/agent/src/agent-loop.ts::executeToolCalls`

**Status:** NOT STARTED

---

## Q13 — Who decides that a run is finished?

Identify all materially different termination/continuation mechanisms you find.

Distinguish among:

- no more tool work;
- explicit finish-turn behavior;
- queued steering/follow-up work;
- errors/abort;
- tool-driven termination hints;
- coding-agent/session-level retry or recovery behavior.

**Primary pointers**

- `runLoop()`
- `FinishTurn` in `pi/packages/agent/src/types.ts`
- tool `terminate` semantics
- relevant `AgentSession` recovery hooks if needed

**Status:** NOT STARTED

---

# Block 2 — State / Session

## What this block is trying to answer

This block has three main reading topics, matching the earlier source-reading guide:

- **Agent runtime:** what does `Agent` own, and what survives between calls?
- **State/session responsibilities:** how do `Agent`, `AgentSession`, and `SessionManager` divide work?
- **Session tree:** how are branching, persistence, and active-history selection represented?

Then check the boundary with external state. Fill in each component's responsibilities from source; the overview deliberately does not supply those answers.

The core question is:

> Why does Pi need multiple state/session abstractions instead of one big state object?

You should also understand the difference between:

```text
single-run execution state
live multi-prompt conversation state
durable session state
external authoritative state
```

### Main source path

- `pi/packages/agent/src/agent.ts`
- `pi/packages/coding-agent/src/core/agent-session.ts`
- `pi/packages/coding-agent/src/core/session-manager.ts`
- `pi/packages/coding-agent/docs/session-format.md`

## Q2 — What is the role of `Agent` if `agent-loop.ts` owns the loop?

Explain why Pi needs `Agent` in addition to the low-level loop functions.

Cover:

- what state `Agent` owns;
- what lifecycle/event responsibilities it owns;
- what `prompt()` and `continue()` mean;
- how queued steering/follow-up messages fit;
- what remains outside `Agent`.

**Primary pointers**

- `pi/packages/agent/src/agent.ts::createMutableAgentState`
- `pi/packages/agent/src/agent.ts::Agent`
- `pi/packages/agent/src/agent.ts::prompt`
- `pi/packages/agent/src/agent.ts::continue`
- event reduction in `pi/packages/agent/src/agent.ts`

**Status:** NOT STARTED

---

## Q3 — Runtime state vs persistent session state

Explain the difference among:

- `Agent`
- `AgentSession`
- `SessionManager`

A passing answer must explain why Pi has all three instead of one "state" object.

Also classify what belongs to:

- active runtime state;
- coding-agent orchestration;
- durable session history.

**Primary pointers**

- `pi/packages/agent/src/agent.ts`
- `pi/packages/coding-agent/src/core/agent-session.ts::AgentSession`
- `pi/packages/coding-agent/src/core/session-manager.ts::SessionManager`

**Status:** NOT STARTED

---

## Q4 — Is Pi actually stateful?

Give a precise answer; do not use "stateful" without naming the layer.

Explain separately whether Pi is stateful across:

- steps within one model/tool run;
- multiple user prompts in the same live agent/session;
- process restart / resume;
- external world state such as the filesystem.

**Primary pointers**

- `Agent.state.messages`
- `Agent.prompt()`
- `SessionManager` persistence
- session JSONL docs

**Status:** NOT STARTED

---

## Q6 — Why is the session a tree instead of a flat message list?

Explain:

- what `id` / `parentId` represent;
- what the current leaf means;
- how branching works;
- whether old entries are destroyed when branching;
- why this matters for reconstructing model context.

**Primary pointers**

- `pi/packages/coding-agent/docs/session-format.md`
- `pi/packages/coding-agent/src/core/session-manager.ts::buildSessionPath`
- `SessionManager.branch`
- `SessionManager.branchWithSummary`

**Status:** NOT STARTED

---

## Q7 — Active context vs audit/history

Explain whether Pi distinguishes "everything that happened" from "what the model sees next".

Use at least two concrete mechanisms/examples from the source, such as:

- branch selection;
- compaction;
- context edits;
- custom/state-only entries.

Keep this focused on the active-context versus durable-history boundary.

**Primary pointers**

- `SessionEntry` variants in `session-manager.ts`
- `buildContextEntries()`
- `CompactionEntry`
- `ContextEditEntry`
- `CustomEntry` vs `CustomMessageEntry`

**Status:** NOT STARTED

---

## Q14 — What is authoritative external state in a coding agent?

Using the filesystem/shell as the concrete example, explain:

- which state lives outside the transcript/session;
- what tool results tell the model;
- whether the transcript itself is the source of truth for files;
- how the agent can re-synchronize with changed external state.


**Primary pointers**

- coding tools (`read`, `write`, `edit`, `bash`)
- tool-result behavior
- no need to inspect UI code

**Status:** NOT STARTED

---

# Block 3 — Messages / Model Context

## What this block is trying to answer

Trace a data path with these endpoints:

```text
Starting point: durable SessionEntry history
        ↓
Intermediate selection, projection, and conversion stages: find in source
        ↓
End point: the next provider request
```

Identify the actual order, input/output types, and ownership of the intermediate stages yourself. You should know exactly where **history becomes active model context**, and which information stays canonical versus provider-specific.

### Main source path

- `pi/packages/coding-agent/src/core/session-manager.ts`
  - `buildSessionPath()`
  - `buildContextEntries()`
  - `buildSessionProjection()`
  - `buildSessionContext()`
- `pi/packages/coding-agent/src/core/messages.ts::convertToLlm()`
- `pi/packages/agent/src/types.ts`
- `pi/packages/ai/src/types.ts`
- `pi/packages/agent/src/agent-loop.ts::streamAssistantResponse()`

## Q5 — How does a persisted session become the next LLM context?

Trace the transformation from durable session entries to the messages that are finally suitable for the model.

Cover:

- tree/branch/leaf selection;
- which session entries contribute to context;
- projection/context-building;
- custom message conversion;
- the final `convertToLlm()` boundary.

**Primary pointers**

- `pi/packages/coding-agent/src/core/session-manager.ts::buildSessionPath`
- `buildContextEntries`
- `buildSessionProjection`
- `buildSessionContext`
- `pi/packages/coding-agent/src/core/messages.ts::convertToLlm`
- `pi/packages/agent/src/agent-loop.ts::streamAssistantResponse`

**Status:** NOT STARTED

---

## Q8 — What exactly is canonical versus provider-specific in Pi's message model?

Explain:

- the role of `AgentMessage`;
- the role of the model-facing `Message` types;
- where conversion happens;
- whether assistant messages retain provider/model metadata;
- whether Pi fully erases provider-specific continuation information or preserves some of it.

If the last point differs by provider or field, say so precisely rather than forcing a binary answer.

**Primary pointers**

- `pi/packages/agent/src/types.ts`
- `pi/packages/coding-agent/src/core/messages.ts::convertToLlm`
- `pi/packages/ai/src/types.ts`
- provider adapters only as needed to verify continuation fields

**Status:** NOT STARTED

---

# Block 4 — Tool System

## What this block is trying to answer

Trace the full tool pipeline:

```text
tool definition/schema
        ↓
active tools exposed to model
        ↓
model emits toolCall
        ↓
lookup + validation
        ↓
execution
        ↓
ToolResult
        ↓
tool-result message
        ↓
next model context
```

This block also owns multiple-tool ordering and the distinction between **recoverable tool failures** and failures that alter/stop the run.

### Main source path

- `pi/packages/coding-agent/src/core/tools/index.ts`
- one concrete tool such as `read.ts` or `bash.ts`
- `pi/packages/agent/src/agent-loop.ts::executeToolCalls()`
- `pi/packages/agent/src/harness/execution/tools.ts`
- `pi/packages/agent/src/types.ts::ToolExecutionMode`

## Q9 — How are tools declared and exposed to the model?

Trace:

- a coding tool implementation;
- its schema/description;
- construction/registration;
- how the active tool set reaches the model request.

Explain the boundary between a tool definition and executable behavior.

**Primary pointers**

- `pi/packages/coding-agent/src/core/tools/index.ts`
- one concrete tool such as `core/tools/read.ts` or `core/tools/bash.ts`
- `Agent.state.tools`
- `toToolDeclaration` usage in the agent core

**Status:** NOT STARTED

---

## Q10 — How does a model tool call become execution and then an observation?

Trace one tool call end-to-end.

Cover:

- tool lookup;
- argument validation;
- pre-execution checks/hooks;
- execution;
- result/error representation;
- tool-result message creation;
- reinsertion into subsequent model context.

**Primary pointers**

- `pi/packages/agent/src/agent-loop.ts::executeToolCalls`
- `pi/packages/agent/src/harness/execution/tools.ts`
- `AgentToolResult` / `ToolResultMessage`

**Status:** NOT STARTED

---

## Q11 — Sequential vs parallel tool execution

Explain Pi's execution semantics for multiple tool calls from one assistant message.

Cover:

- where execution mode is selected;
- when Pi uses sequential behavior;
- when concurrent execution is allowed;
- whether completion order and model-visible result order are necessarily the same;
- why this distinction matters.

Explain current behavior before proposing changes.

**Primary pointers**

- `pi/packages/agent/src/types.ts::ToolExecutionMode`
- `pi/packages/agent/src/agent-loop.ts::executeToolCalls`
- `executeToolCallsSequential`
- `executeToolCallsParallel`

**Status:** NOT STARTED

---

## Q12 — Tool failure vs model/runtime failure

Explain how Pi treats:

- invalid/unknown tool calls;
- tool execution exceptions;
- blocked tool execution;
- model/provider error responses;
- aborted runs.

Which failures become observations that the model can react to, and which stop or alter the run?

**Primary pointers**

- `pi/packages/agent/src/agent-loop.ts`
- `pi/packages/agent/src/harness/execution/tools.ts`
- `pi/packages/coding-agent/src/core/agent-session.ts` retry/overflow handling only where necessary

**Status:** NOT STARTED

# Accepted answer log

Use a private working copy; the public template starts blank. Record both the student's accepted answer and the teacher's standard answer here after each PASS, and update that question's status above. Do not overwrite the public reference key.

## Q1 record

**Status:** NOT STARTED

### Student accepted answer

(To be recorded only after PASS; preserve original wording and language.)

### Teacher standard answer

(To be recorded only after PASS.)

### Evidence

(File, symbol, and relevant lines at the pinned revision.)

---

## Q13 record

**Status:** NOT STARTED

### Student accepted answer

(To be recorded only after PASS; preserve original wording and language.)

### Teacher standard answer

(To be recorded only after PASS.)

### Evidence

(File, symbol, and relevant lines at the pinned revision.)

---

## Q2 record

**Status:** NOT STARTED

### Student accepted answer

(To be recorded only after PASS; preserve original wording and language.)

### Teacher standard answer

(To be recorded only after PASS.)

### Evidence

(File, symbol, and relevant lines at the pinned revision.)

---

## Q3 record

**Status:** NOT STARTED

### Student accepted answer

(To be recorded only after PASS; preserve original wording and language.)

### Teacher standard answer

(To be recorded only after PASS.)

### Evidence

(File, symbol, and relevant lines at the pinned revision.)

---

## Q4 record

**Status:** NOT STARTED

### Student accepted answer

(To be recorded only after PASS; preserve original wording and language.)

### Teacher standard answer

(To be recorded only after PASS.)

### Evidence

(File, symbol, and relevant lines at the pinned revision.)

---

## Q6 record

**Status:** NOT STARTED

### Student accepted answer

(To be recorded only after PASS; preserve original wording and language.)

### Teacher standard answer

(To be recorded only after PASS.)

### Evidence

(File, symbol, and relevant lines at the pinned revision.)

---

## Q7 record

**Status:** NOT STARTED

### Student accepted answer

(To be recorded only after PASS; preserve original wording and language.)

### Teacher standard answer

(To be recorded only after PASS.)

### Evidence

(File, symbol, and relevant lines at the pinned revision.)

---

## Q14 record

**Status:** NOT STARTED

### Student accepted answer

(To be recorded only after PASS; preserve original wording and language.)

### Teacher standard answer

(To be recorded only after PASS.)

### Evidence

(File, symbol, and relevant lines at the pinned revision.)

---

## Q5 record

**Status:** NOT STARTED

### Student accepted answer

(To be recorded only after PASS; preserve original wording and language.)

### Teacher standard answer

(To be recorded only after PASS.)

### Evidence

(File, symbol, and relevant lines at the pinned revision.)

---

## Q8 record

**Status:** NOT STARTED

### Student accepted answer

(To be recorded only after PASS; preserve original wording and language.)

### Teacher standard answer

(To be recorded only after PASS.)

### Evidence

(File, symbol, and relevant lines at the pinned revision.)

---

## Q9 record

**Status:** NOT STARTED

### Student accepted answer

(To be recorded only after PASS; preserve original wording and language.)

### Teacher standard answer

(To be recorded only after PASS.)

### Evidence

(File, symbol, and relevant lines at the pinned revision.)

---

## Q10 record

**Status:** NOT STARTED

### Student accepted answer

(To be recorded only after PASS; preserve original wording and language.)

### Teacher standard answer

(To be recorded only after PASS.)

### Evidence

(File, symbol, and relevant lines at the pinned revision.)

---

## Q11 record

**Status:** NOT STARTED

### Student accepted answer

(To be recorded only after PASS; preserve original wording and language.)

### Teacher standard answer

(To be recorded only after PASS.)

### Evidence

(File, symbol, and relevant lines at the pinned revision.)

---

## Q12 record

**Status:** NOT STARTED

### Student accepted answer

(To be recorded only after PASS; preserve original wording and language.)

### Teacher standard answer

(To be recorded only after PASS.)

### Evidence

(File, symbol, and relevant lines at the pinned revision.)

---

# Completion rule

Complete the exercise when every Q1–Q14 question passes its requirements and the 95% gate, with student answers, teacher answers, and source evidence recorded in the same private workbook. Preserve block order on resume. No comparison milestone or external project is required.
