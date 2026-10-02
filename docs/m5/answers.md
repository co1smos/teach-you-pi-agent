# M5 — Pi Architecture Reference Answers

[中文](answers.zh-CN.md) | [Questions](questions.md) | [How to use](../../README.md)

Spoilers: answer each question from source first. Teachers may consult the current answer privately for grading but reveal the standard answer only after PASS. These are reference explanations adapted from the original teacher answers, not a completed learner record; the pinned source wins if a discrepancy is found.

Source revision: `a32782520f69cd81b54814c3a13df4c7bd1f3ad7`. Keep answers under the same Q IDs as the question bank. Evidence paths are repository-relative. This edition removes the private cross-project synthesis and clarifies a few overbroad historical claims.

## Q1

`agentLoop()` is the streaming entry point, and `runAgentLoop()` initializes the prompt/context and lifecycle events, but `runLoop()` owns the repeated model -> tool -> model control flow. Each iteration calls `streamAssistantResponse()`, which transforms the current `AgentMessage[]` at the LLM boundary, invokes the configured stream function, appends or updates the streamed assistant message in the current context, and returns the final assistant message.

`runLoop()` extracts every `toolCall` content block from that assistant message. Unless the assistant output was truncated, it dispatches them through `executeToolCalls()`; the resulting `ToolResultMessage` objects are appended to both `currentContext.messages` and the invocation's `newMessages`. Because `currentContext` is reused, those observations enter the next provider request.

The inner loop continues when an executable tool batch leaves `hasMoreToolCalls` true or steering messages are pending. `finishTurn` may end immediately, request one explicit continuation, or leave normal scheduling unchanged. After the inner loop drains, follow-up messages restart processing; otherwise an explicit continuation performs one further request. With no tool work, steering, follow-up, or explicit continuation, the outer loop breaks and emits `agent_end`. Model error or abort responses are hard exits.

### Evidence
- `pi/packages/agent/src/agent-loop.ts::agentLoop`, `runAgentLoop`, and `runLoop` (lines 37-59, 101-125, 162-320 at the pinned revision).
- `pi/packages/agent/src/agent-loop.ts::streamAssistantResponse` (lines 380-466).
- `pi/packages/agent/src/agent-loop.ts::executeToolCalls` and tool-result construction (lines 505-520, 880-897).
- `pi/packages/agent/src/types.ts::FinishTurn` and `AgentTurnDecision` (lines 142-154).

---

## Q13

Termination and continuation in Pi operate across two architectural layers:

A. Low-level loop (`pi/packages/agent/src/agent-loop.ts::runLoop`):
1. Natural completion (no tool calls): when an assistant response emits text without tool calls, `hasMoreToolCalls = false`. The inner loop finishes and checks queues.
2. Tool batch termination hint: if every finalized tool call in a batch specifies `terminate: true` (or `beforeToolCall` blocks with terminate), `hasMoreToolCalls` is cleared to false, preventing further automatic tool loops for that turn.
3. Finish-turn decisions (`FinishTurn`): returning `{ action: "end" }` terminates the run immediately without consulting queues. Returning `{ action: "continue" }` forces one context-only next request if no messages were queued. Returning `undefined` preserves default queue-based scheduling.
4. Mid-run steering: `getSteeringMessages()` injects queued messages into the active inner loop before the next LLM call.
5. End-of-run follow-up: `getFollowUpMessages()` polls after the inner loop drains; if messages exist, they restart the outer loop.
6. Hard errors and aborts: an assistant message with `stopReason === "error"` or `"aborted"` causes an immediate exit from `runLoop()` without executing tool calls or polling queues.

B. Session orchestration (`pi/packages/coding-agent/src/core/agent-session.ts`):
7. Post-run retry and overflow recovery: `AgentSession` wraps the prompt lifecycle in a `while (!this._agentRunAbortRequested)` loop. When `runLoop()` completely finishes with an error or token-limit length stop, `_handlePostAgentRun()` inspects the outcome:
   - For retryable API errors, `_prepareRetry()` applies exponential backoff and omits the failed attempt from model projection.
   - For context overflow, `_checkCompaction()` triggers auto-compaction.
   In both cases, `AgentSession` does not resume the old loop; it starts a brand-new run by calling `agent.continue()`.

### Evidence
- `pi/packages/agent/src/agent-loop.ts::runLoop` (lines 178-320 at pinned revision): inner and outer while loops, error/abort exits, queue draining, and `explicitContinuation`.
- `pi/packages/agent/src/agent-loop.ts::shouldTerminateToolBatch` (lines 685-687): batch termination semantics.
- `pi/packages/agent/src/types.ts::FinishTurn` (lines 142-154) and `AgentToolResult.terminate` (lines 427-431).
- `pi/packages/coding-agent/src/core/agent-session.ts::_runAgentPrompt` and `_handlePostAgentRun` (lines 1468-1528): session-level retry and compaction loops calling `agent.continue()`.

---

## Q2

`Agent` is the stateful in-memory wrapper around the low-level loop invocation. Its `AgentState` holds the active transcript, model, thinking level, executable tools, streaming status, current partial response, pending tool-call IDs, and latest run error. The class separately owns the active-run controller, event listeners, configurable loop hooks, and steering/follow-up queues.

`Agent.prompt()` normalizes caller-supplied input into one or more new messages and invokes `runAgentLoop()`. `Agent.continue()` normally invokes `runAgentLoopContinue()` with a snapshot of the existing transcript and tools, adding no new prompt; the existing final message must be suitable for another provider request. When the transcript ends in an assistant message, `continue()` may instead drain a queued steering or follow-up message and start it through the prompt path, but otherwise rejects the continuation.

`runWithLifecycle()` enforces one active run, owns cancellation and settlement, and converts unexpected thrown failures into normal lifecycle events. `processEvents()` reduces loop events into the live `AgentState` and then notifies subscribers. These responsibilities make `Agent` reusable across prompts while leaving the repeated model/tool cycle to `runLoop()`. Durable session persistence, tree/branch history, context projection and compaction, higher-level retry orchestration, and restart/resume remain outside `Agent`.

### Evidence
- `pi/packages/agent/src/types.ts::AgentState` (lines 372-417 at the pinned revision).
- `pi/packages/agent/src/agent.ts::createMutableAgentState`, `Agent`, and `PendingMessageQueue` (lines 82-110, 142-250).
- `pi/packages/agent/src/agent.ts::prompt`, `continue`, `runPromptMessages`, and `runContinuation` (lines 367-454).
- `pi/packages/agent/src/agent.ts::createLoopConfig`, `runWithLifecycle`, and `processEvents` (lines 464-608).

---

## Q3

`Agent` owns the generic live runtime: the active transcript and model/tool configuration, streaming and tool-execution status, cancellation, event reduction, and steering/follow-up queues. It can run prompts repeatedly, but it has no durable storage or session-tree knowledge.

`AgentSession` is the coding-agent orchestration layer. It combines an `Agent`, a `SessionManager`, settings, tools, extensions, compaction, retry and overflow recovery, request/context projection, and higher-level run settlement. By subscribing to `Agent` events, it translates generic runtime activity into coding-agent behavior. For example, on `message_end` it chooses whether to append a normal message entry or a custom-message entry.

`SessionManager` owns one durable append-only session tree at a time: the session file/header, indexed entries, parent links, current leaf, branching, resume/loading, and projection of the selected branch into model context. It performs persistence but does not execute models, tools, retries, or extensions.

Keeping these separate lets the generic runtime operate without coding-agent policies, lets orchestration coordinate live execution with durable history, and lets session storage remain deterministic and independent of provider/tool execution.

### Evidence
- `pi/packages/agent/src/agent.ts::Agent`, `processEvents`, and lifecycle methods (lines 181-250, 503-608 at the pinned revision).
- `pi/packages/coding-agent/src/core/agent-session.ts::AgentSession` and constructor wiring (lines 328-446).
- `pi/packages/coding-agent/src/core/agent-session.ts::_handleAgentEvent` (lines 894-974), including selection of `appendCustomMessageEntry()` versus `appendMessage()`.
- `pi/packages/coding-agent/src/core/agent-session.ts::_runAgentPrompt` and `_handlePostAgentRun` (lines 1468-1528).
- `pi/packages/coding-agent/src/core/session-manager.ts::SessionManager` (lines 976-1129) and `appendMessage()` (lines 1196-1212).

---

## Q4

Pi is not monolithically "stateful"; state exists across four distinct boundaries with different lifecycles:

1. Intra-run execution state (in-memory): `runLoop()` mutates an active `AgentContext`, appending streamed assistant messages and tool execution observations into `currentContext.messages` so subsequent model iterations within that run observe prior steps.
2. Live multi-prompt conversation state (in-memory): `Agent._state.messages` retains the replayed transcript and prompt configuration across distinct `prompt()` and `continue()` calls in the running Node process.
3. Durable cross-process session state (persistent): `SessionManager` persists session entries into append-only JSONL files on disk. Entries are linked as an acyclic tree (`id`, `parentId`) pointing to an active leaf. On restart or resume, the session tree is re-read from disk, and model context is reconstructed deterministically from the selected branch without depending on transient process memory.
4. External environment state (authoritative outside Pi): The operating system and filesystem are the authoritative external state. Pi does not own or simulate this state; transcript entries represent historical point-in-time observations. If files are changed externally, the transcript becomes stale until Pi explicitly invokes observation tools (`read`, `ls`, `bash`) to sample the current external state.

### Evidence
- `pi/packages/agent/src/agent-loop.ts::runLoop` (lines 170-277 at pinned revision): in-memory accumulation in `currentContext.messages`.
- `pi/packages/agent/src/agent.ts::createMutableAgentState` and `processEvents` (lines 82-111, 561-599): live `AgentState.messages`.
- `pi/packages/coding-agent/src/core/session-manager.ts::SessionManager` (lines 976-1129, 1196-1212) and `pi/packages/coding-agent/docs/session-format.md`: durable JSONL tree with `id` and `parentId`.
- `pi/packages/coding-agent/src/core/tools/read.ts` (lines 44-48, 103-136) and `write.ts`: tools interact directly with external filesystem APIs (`fsReadFile`, `fsWriteFile`), proving the filesystem is the external authority.

---

## Q6

Pi stores sessions as an acyclic tree of entries in an append-only JSONL file rather than a flat linear list:

1. Identity and hierarchy: every entry has a unique `id` and a `parentId`. A root entry has `parentId: null`.
2. Active leaf pointer: `SessionManager` tracks `leafId`, representing the tip of the currently active branch. Whenever a new entry (message, compaction, model change) is recorded, it is appended with `parentId: this.leafId` and advances `leafId`.
3. Non-destructive branching: `SessionManager.branch(branchFromId)` simply reassigns `this.leafId = branchFromId`. Subsequent appends branch off from that earlier node. No lines in the JSONL file are modified or deleted. `branchWithSummary()` additionally appends a `branch_summary` entry at the fork point to bridge context from the abandoned path.
4. Model context reconstruction: `buildSessionPath(entries, leafId)` traverses backward from the active leaf via `parentId` pointers to the root and reverses the path. This selects only the active ancestral path; off-path raw entries are excluded. A branch summary can still carry information from an abandoned branch into the selected path. Projection then applies compaction and context edits.

### Evidence
- `pi/packages/coding-agent/docs/session-format.md`: tree structure specification (`id`, `parentId`, leaf navigation, JSONL storage).
- `pi/packages/coding-agent/src/core/session-manager.ts::buildSessionPath` (lines 390-416 at pinned revision): backward traversal from leaf to root.
- `pi/packages/coding-agent/src/core/session-manager.ts::branch` (lines 1572-1577): non-destructive leaf repointing.
- `pi/packages/coding-agent/src/core/session-manager.ts::branchWithSummary` (lines 1593-1620): capturing abandoned branch summary.

---

## Q7

Pi strictly decouples the durable audit record ("everything that happened") from the projected prompt ("what the model sees next"):

1. Durable audit log: normal session history is stored as append-only JSONL entries, including `SessionMessageEntry`, `ThinkingLevelChangeEntry`, `ModelChangeEntry`, `UsageEntry`, `CustomEntry`, `LabelEntry`, `SessionInfoEntry`, `CompactionEntry`, `BranchSummaryEntry`, and `ContextEditEntry`. Branching, compaction, and context edits preserve the original historical entries. This is not a claim that every streaming event is stored or that no maintenance operation can ever rewrite a session file.
2. Context projection: when preparing a model request, `SessionManager.buildSessionProjection()` and `buildContextEntries()` project this raw history through deterministic filters:
   - Branch filter: only entries on the path from root to current leaf are evaluated (`buildSessionPath`).
   - Compaction filter: entries prior to `firstKeptEntryId` are replaced by the `CompactionEntry` summary, hiding obsolete conversation turns from the model while retaining them in the file.
   - Context edits: `ContextEditEntry` can override content or completely omit (`replacement: null`) an earlier message in the projected context without modifying the historical entry.
   - Semantic type filtering: entries such as `UsageEntry` (cost/token telemetry) and `CustomEntry` (internal extension state) are preserved in durable history but filtered out from model context, whereas `CustomMessageEntry` is explicitly projected into a message.

This separation enables inspection of retained history, cost tracking, and non-destructive conversation branching (not filesystem rollback) while keeping the active model context compact and relevant.

### Evidence
- `pi/packages/coding-agent/src/core/session-manager.ts::SessionEntry` variants (lines 120-194 at pinned revision): definitions of context vs non-context entries (`UsageEntry`, `CustomEntry`, `CustomMessageEntry`, `ContextEditEntry`).
- `pi/packages/coding-agent/src/core/session-manager.ts::buildContextEntries` (lines 476-512): compaction filtering replacing older entries with `CompactionEntry`.
- `pi/packages/coding-agent/src/core/session-manager.ts::projectContextEntry` (lines 519-540): non-destructive replacement via `ContextEditEntry`.
- `pi/packages/coding-agent/src/core/session-manager.ts::sessionEntryToContextMessages` (lines 439-460): filtering out non-context entries (`usage`, `custom`, `label`).

---

## Q14

In a coding agent, the external environment (the OS filesystem, processes, and network) is the sole authoritative ground truth.

1. External authority vs transcript: The default coding-agent `read`, `write`, `edit`, and `bash` implementations operate on filesystem/process APIs; injected operations can delegate to other environments. Pi does not maintain a virtual shadow filesystem.
2. Tool results as point-in-time observations: `ToolResultMessage` objects in the transcript record what the tool observed or mutated at the moment of execution. If an external process or human modifies a file afterwards, the transcript becomes stale.
3. Re-synchronization: The transcript is not a live filesystem mirror. In the model/tool loop, fresh observations occur on demand: the model samples the environment by issuing observation tools (`read`, `ls`, `bash`), or reacts to mutation failures (such as `edit` failing due to mismatched content) by re-reading the target file.

### Evidence
- `pi/packages/coding-agent/src/core/tools/read.ts` (lines 44-48, 103-136 at pinned revision): `ops.readFile` queries OS filesystem directly via `fsReadFile`.
- `pi/packages/coding-agent/src/core/tools/edit.ts` (lines 92-96, 174-198): `edit` reads live disk content and checks `oldText` against current reality, failing if out of sync.
- `pi/packages/coding-agent/src/core/tools/write.ts` (lines 34-37, 75-88): `write` mutates disk state via `fsWriteFile`.
- `pi/packages/agent/src/agent-loop.ts::createToolResultMessage` (lines 880-893): returns `ToolResultMessage` capturing observation into transcript.

---

## Q5

The loaded `SessionEntry[]` is indexed by ID in a Map (not a Set). `buildSessionPath()` follows parent links from the selected leaf and reverses the result into root-to-leaf order. `buildContextEntries()` selects the latest compaction, its retained non-system entries starting at `firstKeptEntryId`, and subsequent entries; raw history remains intact.

`buildSessionProjection()` collects applicable context edits and projects selected entries through `projectContextEntry()` and `sessionEntryToContextMessages()`. Edits replace projected content, or omit the target when replacement is null, without changing the raw entry. Non-context entry types yield no messages. The projection preserves source-entry provenance and flattens messages to `AgentMessage[]`; `buildSessionContext()` returns its messages, model, and thinking level. `AgentSession` uses the projection when preparing runtime requests.

At `streamAssistantResponse()`, optional `transformContext` runs on `AgentMessage[]` before `convertToLlm`. Coding-agent conversion maps custom and summary messages to user messages; summary prefixes/suffixes are text, not JSON wrappers. Bash execution is converted to user text unless excluded. Standard system/user/assistant/tool-result messages pass through. The resulting model-facing `Message[]` is normalized and passed to the configured stream function; it is not yet a provider-specific wire payload.

### Evidence
- Pinned `session-manager.ts`: `buildSessionPath` (390-416), `sessionEntryToContextMessages`, `buildContextEntries`, `projectContextEntry`, `buildSessionProjection`, `buildSessionContext` (439-583), under `pi/packages/coding-agent/src/core/`.
- `pi/packages/coding-agent/src/core/agent-session.ts:608-630`: request projection integration.
- `pi/packages/coding-agent/src/core/messages.ts:148-196`: `convertToLlm`.
- `pi/packages/agent/src/agent-loop.ts:380-406`: transformation, conversion, normalization, and stream invocation.

---

## Q8

`AgentMessage` extends Pi's shared model-facing `Message` union with application-specific message types. `convertToLlm()` maps those custom types to model-compatible messages and passes standard assistant messages through. These shared types are not the raw provider response or the provider-specific request payload; capturing a constructed assistant message does not mean capturing every raw response field.

Assistant messages retain origin metadata (`provider`, `api`, `model`) and selected provider continuation fields in their content blocks. The shared representation therefore does not erase all provider-specific information. Subsequent adapter conversion decides what can be replayed for the target provider/API/model.

In `transformMessages`, all three origin fields must match for `isSameModel`. Signed thinking blocks are retained for the same model even with empty visible text. On cross-model conversion, non-redacted nonempty thinking becomes plain text without its signature; empty thinking is omitted. Redacted thinking is retained only for the same model and otherwise removed entirely, not sent as an empty block. Cross-model handling also strips text signatures and tool-call thought signatures as applicable.

In the OpenAI Responses adapter, `thinkingSignature` contains a serialized reasoning item, potentially including encrypted content. The adapter parses that field and appends the item to the request. Without the field, this branch emits no reasoning item for that block. Thus empty visible text does not imply absence of continuation data. Other providers interpret their signature fields differently; this is not a claim that every signature is encrypted reasoning.

### Evidence
- `pi/packages/agent/src/types.ts:347-370`: extensible `AgentMessage` union.
- `pi/packages/coding-agent/src/core/messages.ts:148-196`: `convertToLlm`.
- `pi/packages/ai/src/types.ts`: `AssistantMessage`, `ThinkingContent`, `TextContent`, `ToolCall`.
- `pi/packages/ai/src/api/transform-messages.ts:92-134`: model identity and continuation-field handling.
- `pi/packages/ai/src/api/openai-responses-shared.ts:261-266,539-547`: reasoning item replay and encrypted-content preservation.

---

## Q9

Coding tool definitions combine schemas/descriptions with local executable behavior. Built-in definitions are constructed in the tools module; `_buildRuntime` populates `_baseToolDefinitions` and initializes the extension runner. `_refreshToolRegistry` combines allowed built-ins with registered extension and SDK custom tools, wraps them as AgentTool objects, and populates `_toolRegistry`. Allowed/excluded filtering affects registry membership, not just activation.

Active selection is separate: explicit activeToolNames or previousActiveToolNames seed the selection; allow-list, includeAllExtensionTools, and newly registered tool branches can add names. `setActiveToolsByName` resolves selected names from the registry and replaces `agent.state.tools`. Default initialization constructs all built-in definitions, but selects read/bash/edit/write plus extension tools; grep/find/ls/powershell can therefore remain registered but inactive. This differs from a hypothetical first no-options refresh with an empty prior registry, which activates all new registry entries.

The loop executes against `currentContext.tools`, a snapshot refreshed from Agent state at the appropriate turn boundary, not by directly consulting original definitions or the session registry. Its tools retain execute. `toToolDeclaration` strips executable/display-only behavior and preserves name, description, parameters, and optional constrainedSampling. The loop records declaration changes through toolsAdded/toolsRemoved; provider adaptation produces the actual API request. These transcript updates should not be confused with sending execute functions or assuming every provider accepts delta-shaped tool fields.

### Evidence
- `pi/packages/coding-agent/src/core/tools/read.ts`: concrete read schema, description, and implementation.
- `pi/packages/coding-agent/src/core/tools/index.ts:182-192`: createAllToolDefinitions.
- `pi/packages/coding-agent/src/core/agent-session.ts:1279-1290,3144-3288`: active selection, registry refresh, runtime construction.
- `pi/packages/agent/src/agent.ts:457-461`: context snapshot.
- `pi/packages/agent/src/agent-loop.ts:184-210,322-361,703-788`: context refresh, declarations, runtime lookup and execution.
- `pi/packages/ai/src/utils/transcript.ts:123-129`: toToolDeclaration.

---

## Q10

The loop looks up the requested tool in currentContext.tools. prepareToolCallArguments optionally normalizes arguments, then validateToolArguments checks the schema. The optional beforeToolCall hook receives validated arguments and can block execution; missing tools, validation/preparation failures, blocks, and checked aborts produce immediate error outcomes instead of invoking that tool.

For a prepared call, executePreparedToolCall invokes the local execute function with call ID, validated arguments, abort signal, and a partial-update callback. Successful execution returns its result with isError false; a thrown execution error is converted by createErrorToolResult into a textual error result with isError true. Finalization can apply optional afterToolCall overrides to content, details, usage, terminate, and isError; a thrown finalization-hook error is also represented as an error result. Reporting overrides do not themselves undo external effects already performed by the tool.

createToolResultMessage associates the finalized content and error status with the tool call. Tool-result message events are emitted, and the batch results are appended to currentContext.messages and newMessages for subsequent model context. The observation is a report of execution, not the authoritative external filesystem state. Ordering and concurrency are assessed separately in Q11.

### Evidence
- `pi/packages/agent/src/agent-loop.ts:703-770`: prepareToolCall, lookup, validation, blocking, and immediate outcomes.
- `pi/packages/agent/src/agent-loop.ts:773-814`: executePreparedToolCall and error conversion.
- `pi/packages/agent/src/agent-loop.ts:816-860`: finalizeExecutedToolCall and optional result overrides.
- `pi/packages/agent/src/agent-loop.ts`: createErrorToolResult, createToolResultMessage, emitToolResultMessage, executeToolCallsSequential/Parallel, and runLoop result reinsertion.

---

## Q11

executeToolCalls selects sequential execution when config.toolExecution is sequential OR any requested tool has executionMode sequential; otherwise it selects the parallel path. This per-tool override serializes the whole batch, not just that tool. Sequential execution awaits preparation, execution, finalization, and result emission for each call before proceeding, checking abort between calls.

The parallel path prepares calls in original order and stores immediate outcomes or deferred execution functions. Promise.all invokes those functions and preserves input order in its returned array despite differing completion times. Execution-end events occur within each execution/finalization task, so they need not follow call order; final tool-result messages are emitted afterward by iterating orderedFinalizedCalls and therefore retain original call order. End-event timing also includes finalization, not just the raw execute completion.

Stable transcript ordering does not serialize side effects: a concurrent read may observe a file before another call writes it, even if the write result precedes the read result in the transcript. The model chooses which calls to batch; runtime config and tool metadata choose scheduling. Dependent operations can be requested across turns so the model sees one result before choosing the next call and its arguments. Sequential scheduling alone does not let a model revise arguments for calls it already emitted together.

### Evidence
- `pi/packages/agent/src/agent-loop.ts:505-519`: global and per-tool mode selection.
- `pi/packages/agent/src/agent-loop.ts:527-580`: sequential execution.
- `pi/packages/agent/src/agent-loop.ts:583-656`: preparation, deferred parallel execution, end events, Promise.all, and ordered result messages.
- `pi/packages/agent/src/types.ts`: ToolExecutionMode.

---

## Q12

Unknown tools, argument preparation/validation failures, blocked execution, and caught tool execution exceptions become error tool-result observations. Such errors do not by themselves mean a provider failure or automatically terminate the low-level run; termination hints, aborts, and other loop controls are separate.

An assistant response whose stopReason is error or aborted is retained as an assistant message. runLoop calls finishTurn, emits turn_end with an empty toolResults array and agent_end, then returns before tool-call handling. This path executes none of the response's requested tools and creates no per-call tool-result messages. An assistant error message is not a tool-result observation.

For stopReason length, tool arguments may be truncated even when parsable. failToolCallsFromTruncatedMessage executes none of the calls and creates an isError tool-result message for each one. Those observations are emitted and reinserted into context, allowing the model to re-issue complete calls through the normal continuation path. This differs from the early return for error/aborted.

Abort during tool handling also prevents further execution at the checked boundaries and passes cancellation to executing tools; it does not undo completed external effects. Session-level retry or overflow handling occurs after the low-level run completes. When recovery proceeds, agent.continue() starts a fresh low-level run with prepared context rather than resuming the returned runLoop invocation. Not every error is retryable, and cancellation is not a promise of automatic retry.

### Evidence
- `pi/packages/agent/src/agent-loop.ts:244-275`: assistant error/aborted early return versus length handling and result reinsertion.
- `pi/packages/agent/src/agent-loop.ts:440-465`: final assistant response retention.
- `pi/packages/agent/src/agent-loop.ts:468-499`: failToolCallsFromTruncatedMessage.
- `pi/packages/agent/src/agent-loop.ts:703-860`: preparation failures, execution exceptions, cancellation checks, and finalization.
- `pi/packages/coding-agent/src/core/agent-session.ts`: _handlePostAgentRun, _prepareRetry, and recovery via agent.continue(); fresh-run boundary already established in Q13.
