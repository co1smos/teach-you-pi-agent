# M5 — Pi 架构参考答案

[English](answers.md) | [题目](questions.zh-CN.md) | [使用指南](../../README.zh-CN.md)

剧透提醒：请先根据源码自行回答每一道题。教师可以私下查阅当前题目的答案以评判作答，但只能在 PASS 之后展示标准答案。这些参考解释改编自原有的教师答案，不是已完成的学习记录；如有不一致，以固定版本的源码为准。

源码修订版本：`a32782520f69cd81b54814c3a13df4c7bd1f3ad7`。答案与题库使用相同的 Q 编号。证据路径均相对于仓库根目录。本版移除了私人的跨项目综合分析，并澄清了原有表述中过于宽泛的几处论断。

## Q1

`agentLoop()` 是流式入口，`runAgentLoop()` 负责初始化提示输入、上下文和生命周期事件，但反复执行的“模型 → 工具 → 模型”控制流由 `runLoop()` 负责。每次迭代都会调用 `streamAssistantResponse()`：它在 LLM 边界转换当前的 `AgentMessage[]`，调用配置的流式函数，在当前上下文中追加或更新流式助手消息，并返回最终的助手消息。

`runLoop()` 从该助手消息中提取所有 `toolCall` 内容块。只要助手输出没有被截断，它就会通过 `executeToolCalls()` 分派这些调用；产生的 `ToolResultMessage` 对象会同时追加到 `currentContext.messages` 和本次调用的 `newMessages` 中。由于 `currentContext` 会被复用，这些观测结果会进入下一次模型服务商请求。

当一个可执行的工具批次使 `hasMoreToolCalls` 保持为 true，或仍有待处理的引导消息时，内层循环会继续。`finishTurn` 可以立即结束运行、显式要求再继续一次，或保持正常调度不变。内层循环处理完毕后，后续消息会重新启动处理；否则，显式继续请求会再触发一次模型请求。当没有工具工作、引导消息、后续消息或显式继续请求时，外层循环退出并发出 `agent_end`。模型错误或中止响应会直接退出。

### 证据
- `pi/packages/agent/src/agent-loop.ts::agentLoop`、`runAgentLoop` 和 `runLoop`（固定修订版本的第 37-59、101-125、162-320 行）。
- `pi/packages/agent/src/agent-loop.ts::streamAssistantResponse`（第 380-466 行）。
- `pi/packages/agent/src/agent-loop.ts::executeToolCalls` 与工具结果的构造（第 505-520、880-897 行）。
- `pi/packages/agent/src/types.ts::FinishTurn` 和 `AgentTurnDecision`（第 142-154 行）。

---

## Q13

Pi 的终止与继续机制分布在两个架构层次：

A. 底层循环（`pi/packages/agent/src/agent-loop.ts::runLoop`）：
1. 自然完成（没有工具调用）：当助手响应只输出文本而没有工具调用时，`hasMoreToolCalls = false`。内层循环结束，随后检查队列。
2. 工具批次的终止提示：如果一个批次中所有已完成最终处理的工具调用都指定了 `terminate: true`（或被 `beforeToolCall` 阻止并指定 terminate），`hasMoreToolCalls` 就会被设为 false，从而阻止该轮次继续自动进入工具循环。
3. 结束轮次决策（`FinishTurn`）：返回 `{ action: "end" }` 会立即终止运行，不再检查队列。返回 `{ action: "continue" }` 会在没有排队消息时，强制发起一次仅使用现有上下文的后续请求。返回 `undefined` 则保留默认的队列调度方式。
4. 运行中的引导：`getSteeringMessages()` 会在下一次 LLM 调用之前，将排队的消息注入当前内层循环。
5. 运行结束时的后续消息：`getFollowUpMessages()` 会在内层循环处理完毕后检查队列；如果有消息，就重新启动外层循环。
6. 硬错误与中止：如果助手消息的 `stopReason === "error"` 或 `"aborted"`，就会立即退出 `runLoop()`，既不执行工具调用，也不检查消息队列。

B. 会话编排（`pi/packages/coding-agent/src/core/agent-session.ts`）：
7. 运行后的重试与上下文溢出恢复：`AgentSession` 把提示输入的生命周期包在 `while (!this._agentRunAbortRequested)` 循环中。当 `runLoop()` 因错误或 token 限制导致的长度停止而完全结束后，`_handlePostAgentRun()` 会检查结果：
   - 对于可重试的 API 错误，`_prepareRetry()` 采用指数退避，并从模型投影中排除失败的那次尝试。
   - 对于上下文溢出，`_checkCompaction()` 会触发自动上下文压缩。
   这两种情况下，`AgentSession` 都不会恢复旧循环，而是调用 `agent.continue()` 启动一次全新的运行。

### 证据
- `pi/packages/agent/src/agent-loop.ts::runLoop`（固定修订版本的第 178-320 行）：内外两层 while 循环、错误与中止退出、队列处理以及 `explicitContinuation`。
- `pi/packages/agent/src/agent-loop.ts::shouldTerminateToolBatch`（第 685-687 行）：批次终止语义。
- `pi/packages/agent/src/types.ts::FinishTurn`（第 142-154 行）和 `AgentToolResult.terminate`（第 427-431 行）。
- `pi/packages/coding-agent/src/core/agent-session.ts::_runAgentPrompt` 和 `_handlePostAgentRun`（第 1468-1528 行）：会话层的重试与压缩循环会调用 `agent.continue()`。

---

## Q2

`Agent` 是对底层循环调用的有状态内存封装。它的 `AgentState` 保存当前对话记录、模型、思考级别、可执行工具、流式状态、当前尚未完成的响应、待处理工具调用的 ID，以及最近一次运行的错误。该类还单独管理当前运行的控制器、事件监听器、可配置的循环钩子，以及引导消息和后续消息队列。

`Agent.prompt()` 将调用者提供的输入规范化为一条或多条新消息，然后调用 `runAgentLoop()`。`Agent.continue()` 通常使用现有对话记录和工具的快照调用 `runAgentLoopContinue()`，不添加新的提示输入；现有的最后一条消息必须适合再次向模型服务商发起请求。当对话记录以助手消息结尾时，`continue()` 也可以改为取出一条排队的引导消息或后续消息，并通过提示输入路径启动处理；否则，它会拒绝继续。

`runWithLifecycle()` 确保同一时间只存在一次活动运行，负责取消和运行结束的收尾处理，并把意外抛出的失败转换为正常的生命周期事件。`processEvents()` 根据循环事件更新实时的 `AgentState`，然后通知订阅者。这些职责使 `Agent` 可以在多次提示输入之间复用，同时把反复执行的模型与工具循环留给 `runLoop()`。持久化会话存储、树与分支历史、上下文投影和压缩、更高层的重试编排，以及重启与恢复，都不属于 `Agent` 的职责。

### 证据
- `pi/packages/agent/src/types.ts::AgentState`（固定修订版本的第 372-417 行）。
- `pi/packages/agent/src/agent.ts::createMutableAgentState`、`Agent` 和 `PendingMessageQueue`（第 82-110、142-250 行）。
- `pi/packages/agent/src/agent.ts::prompt`、`continue`、`runPromptMessages` 和 `runContinuation`（第 367-454 行）。
- `pi/packages/agent/src/agent.ts::createLoopConfig`、`runWithLifecycle` 和 `processEvents`（第 464-608 行）。

---

## Q3

`Agent` 负责通用的实时运行时：当前对话记录、模型与工具配置、流式输出与工具执行状态、取消、事件归约，以及引导消息和后续消息队列。它可以反复处理提示输入，但不具备持久化存储，也不了解会话树。

`AgentSession` 是 coding-agent 的编排层。它将 `Agent`、`SessionManager`、设置、工具、扩展、上下文压缩、重试与溢出恢复、请求与上下文投影，以及更高层的运行收尾处理结合起来。通过订阅 `Agent` 事件，它将通用运行时活动转换成 coding-agent 行为。例如，在收到 `message_end` 时，它会选择追加普通消息条目还是自定义消息条目。

`SessionManager` 每次负责一棵持久化、仅追加的会话树：会话文件与文件头、已建立索引的条目、父节点链接、当前叶节点、分支、恢复与加载，以及将所选分支投影为模型上下文。它执行持久化，但不执行模型、工具、重试或扩展。

这样的职责分离使通用运行时不依赖 coding-agent 的策略，使编排层能够协调实时执行与持久化历史，也使会话存储保持确定性，并独立于模型服务商与工具的执行。

### 证据
- `pi/packages/agent/src/agent.ts::Agent`、`processEvents` 和生命周期方法（固定修订版本的第 181-250、503-608 行）。
- `pi/packages/coding-agent/src/core/agent-session.ts::AgentSession` 与构造函数中的连接逻辑（第 328-446 行）。
- `pi/packages/coding-agent/src/core/agent-session.ts::_handleAgentEvent`（第 894-974 行），包括在 `appendCustomMessageEntry()` 与 `appendMessage()` 之间作出选择。
- `pi/packages/coding-agent/src/core/agent-session.ts::_runAgentPrompt` 和 `_handlePostAgentRun`（第 1468-1528 行）。
- `pi/packages/coding-agent/src/core/session-manager.ts::SessionManager`（第 976-1129 行）和 `appendMessage()`（第 1196-1212 行）。

---

## Q4

不能把 Pi 作为一个整体笼统地称为“有状态”。状态分布在四种边界内，各自具有不同的生命周期：

1. 单次运行内的执行状态（内存中）：`runLoop()` 修改当前的 `AgentContext`，将流式助手消息与工具执行的观测结果追加到 `currentContext.messages`，使同一次运行中的后续模型迭代能够看到之前的步骤。
2. 跨多次提示输入的实时对话状态（内存中）：在运行中的 Node 进程内，`Agent._state.messages` 保留重放得到的对话记录，并连同提示配置在不同的 `prompt()` 和 `continue()` 调用之间维持实时对话状态。
3. 跨进程的持久化会话状态（持久存储中）：`SessionManager` 将会话条目持久化到磁盘上的仅追加 JSONL 文件中。条目通过 `id`、`parentId` 连接成无环树，并由当前叶节点指明活动位置。重启或恢复时，从磁盘重新读取会话树，再根据所选分支确定性地重建模型上下文，无需依赖临时的进程内存。
4. 外部环境状态（权威来源在 Pi 之外）：操作系统与文件系统是外部状态的权威来源。Pi 不拥有也不模拟这些状态；对话条目只表示过去某个时间点的观测。如果文件被外部修改，对话记录就会过时，直到 Pi 显式调用观测工具（`read`、`ls`、`bash`）获取当前外部状态。

### 证据
- `pi/packages/agent/src/agent-loop.ts::runLoop`（固定修订版本的第 170-277 行）：在内存中的 `currentContext.messages` 里累积消息。
- `pi/packages/agent/src/agent.ts::createMutableAgentState` 和 `processEvents`（第 82-111、561-599 行）：实时的 `AgentState.messages`。
- `pi/packages/coding-agent/src/core/session-manager.ts::SessionManager`（第 976-1129、1196-1212 行）和 `pi/packages/coding-agent/docs/session-format.md`：使用 `id` 与 `parentId` 的持久化 JSONL 树。
- `pi/packages/coding-agent/src/core/tools/read.ts`（第 44-48、103-136 行）和 `write.ts`：工具直接与外部文件系统 API（`fsReadFile`、`fsWriteFile`）交互，表明文件系统才是外部的权威来源。

---

## Q6

Pi 将会话保存为仅追加 JSONL 文件中的无环条目树，而不是扁平的线性列表：

1. 身份与层级：每个条目都有唯一的 `id` 和一个 `parentId`。根条目的 `parentId: null`。
2. 当前叶节点指针：`SessionManager` 跟踪 `leafId`，它表示当前活动分支的末端。每当记录一个新条目（消息、上下文压缩、模型变更）时，都会以 `parentId: this.leafId` 追加该条目，并推进 `leafId`。
3. 非破坏性分支：`SessionManager.branch(branchFromId)` 只是重新赋值 `this.leafId = branchFromId`。后续追加的条目会从那个较早的节点分叉。JSONL 文件中的任何行都不会因此被修改或删除。`branchWithSummary()` 还会在分叉点追加一个 `branch_summary` 条目，将离开的路径中的上下文衔接过来。
4. 模型上下文重建：`buildSessionPath(entries, leafId)` 从当前叶节点沿 `parentId` 指针回溯到根节点，然后反转路径。这只会选中当前叶节点的祖先路径；路径之外的原始条目不会被选入。不过，分支摘要仍然可以把已离开分支中的信息带入所选路径。之后，投影会应用上下文压缩和上下文编辑。

### 证据
- `pi/packages/coding-agent/docs/session-format.md`：树结构规范（`id`、`parentId`、叶节点导航、JSONL 存储）。
- `pi/packages/coding-agent/src/core/session-manager.ts::buildSessionPath`（固定修订版本的第 390-416 行）：从叶节点向根节点回溯。
- `pi/packages/coding-agent/src/core/session-manager.ts::branch`（第 1572-1577 行）：非破坏性地重新指向叶节点。
- `pi/packages/coding-agent/src/core/session-manager.ts::branchWithSummary`（第 1593-1620 行）：记录所离开分支的摘要。

---

## Q7

Pi 明确分离持久化审计记录（“发生过的一切”）与投影后的提示内容（“模型接下来会看到的内容”）：

1. 持久化审计日志：常规会话历史以仅追加的 JSONL 条目保存，包括 `SessionMessageEntry`、`ThinkingLevelChangeEntry`、`ModelChangeEntry`、`UsageEntry`、`CustomEntry`、`LabelEntry`、`SessionInfoEntry`、`CompactionEntry`、`BranchSummaryEntry` 和 `ContextEditEntry`。分支、上下文压缩和上下文编辑都会保留原始历史条目。这并不意味着每个流式事件都会被保存，也不意味着任何维护操作都绝不可能重写会话文件。
2. 上下文投影：准备模型请求时，`SessionManager.buildSessionProjection()` 和 `buildContextEntries()` 通过确定性的筛选，将原始历史投影为上下文：
   - 分支筛选：只处理从根节点到当前叶节点路径上的条目（`buildSessionPath`）。
   - 压缩筛选：用 `CompactionEntry` 的摘要替换 `firstKeptEntryId` 之前的条目，使模型不再看到较早的对话轮次，但这些条目仍保留在文件中。
   - 上下文编辑：`ContextEditEntry` 可以在投影上下文中覆盖较早消息的内容，或将其完全省略（`replacement: null`），而不修改历史条目。
   - 按语义类型筛选：`UsageEntry`（费用与 token 遥测）和 `CustomEntry`（扩展内部状态）等条目会保留在持久化历史中，但不会进入模型上下文；而 `CustomMessageEntry` 则会明确投影为消息。

这种分离支持检查保留的历史、跟踪费用以及非破坏性地创建对话分支（并非回滚文件系统），同时让当前模型上下文保持精简且与任务相关。

### 证据
- `pi/packages/coding-agent/src/core/session-manager.ts::SessionEntry` 的各个变体（固定修订版本的第 120-194 行）：参与和不参与上下文的条目定义（`UsageEntry`、`CustomEntry`、`CustomMessageEntry`、`ContextEditEntry`）。
- `pi/packages/coding-agent/src/core/session-manager.ts::buildContextEntries`（第 476-512 行）：压缩筛选，用 `CompactionEntry` 替换较早条目。
- `pi/packages/coding-agent/src/core/session-manager.ts::projectContextEntry`（第 519-540 行）：通过 `ContextEditEntry` 进行非破坏性替换。
- `pi/packages/coding-agent/src/core/session-manager.ts::sessionEntryToContextMessages`（第 439-460 行）：过滤不参与上下文的条目（`usage`、`custom`、`label`）。

---

## Q14

在编码智能体中，外部环境（操作系统文件系统、进程和网络）才是外部现实唯一的权威依据。

1. 外部权威来源与对话记录：coding-agent 默认的 `read`、`write`、`edit` 和 `bash` 实现操作的是文件系统或进程 API；通过注入操作实现，也可以委托其他环境执行。Pi 不维护一套虚拟的影子文件系统。
2. 工具结果是特定时间点的观测：对话记录中的 `ToolResultMessage` 对象记录工具在执行时观测或修改了什么。如果外部进程或人在之后修改了文件，对话记录就会过时。
3. 重新同步：对话记录不是文件系统的实时镜像。在模型与工具循环中，新观测按需发生：模型通过发出观测工具调用（`read`、`ls`、`bash`）采样环境；或者在修改失败时（例如 `edit` 因内容不匹配而失败），重新读取目标文件。

### 证据
- `pi/packages/coding-agent/src/core/tools/read.ts`（固定修订版本的第 44-48、103-136 行）：`ops.readFile` 通过 `fsReadFile` 直接查询操作系统文件系统。
- `pi/packages/coding-agent/src/core/tools/edit.ts`（第 92-96、174-198 行）：`edit` 读取磁盘上的实时内容，并将 `oldText` 与当前实际内容比较；不同步时会失败。
- `pi/packages/coding-agent/src/core/tools/write.ts`（第 34-37、75-88 行）：`write` 通过 `fsWriteFile` 修改磁盘状态。
- `pi/packages/agent/src/agent-loop.ts::createToolResultMessage`（第 880-893 行）：返回 `ToolResultMessage`，将观测记录到对话中。

---

## Q5

加载后的 `SessionEntry[]` 通过 Map（不是 Set）按 ID 建立索引。`buildSessionPath()` 从所选叶节点沿父节点链接回溯，再将结果反转为从根到叶的顺序。`buildContextEntries()` 选取最近一次上下文压缩、该压缩从 `firstKeptEntryId` 起保留的非系统条目，以及之后的条目；原始历史保持不变。

`buildSessionProjection()` 收集适用的上下文编辑，并通过 `projectContextEntry()` 和 `sessionEntryToContextMessages()` 投影所选条目。编辑会替换投影内容，或在 replacement 为 null 时省略目标，但不会改变原始条目。不参与上下文的条目类型不会产生消息。投影保留对源条目的溯源信息，并将消息展开为 `AgentMessage[]`；`buildSessionContext()` 返回其中的消息、模型和思考级别。`AgentSession` 在准备运行时请求时使用该投影。

在 `streamAssistantResponse()` 中，可选的 `transformContext` 会先作用于 `AgentMessage[]`，然后才调用 `convertToLlm`。coding-agent 的转换逻辑把自定义消息和摘要消息映射为用户消息；摘要的前后缀是文本，而非 JSON 包装。Bash 执行记录会转换为用户文本，除非被排除。标准的系统、用户、助手和工具结果消息则直接通过。得到的面向模型的 `Message[]` 会经过规范化，再传给配置的流式函数；此时它还不是模型服务商特有的传输请求载荷。

### 证据
- 固定版本的 `session-manager.ts`：`buildSessionPath`（390-416）、`sessionEntryToContextMessages`、`buildContextEntries`、`projectContextEntry`、`buildSessionProjection`、`buildSessionContext`（439-583），位于 `pi/packages/coding-agent/src/core/` 下。
- `pi/packages/coding-agent/src/core/agent-session.ts:608-630`：请求投影的集成。
- `pi/packages/coding-agent/src/core/messages.ts:148-196`：`convertToLlm`。
- `pi/packages/agent/src/agent-loop.ts:380-406`：变换、转换、规范化和流式调用。

---

## Q8

`AgentMessage` 在 Pi 共享的、面向模型的 `Message` 联合类型基础上，扩展了应用特有的消息类型。`convertToLlm()` 将这些自定义类型映射为模型兼容的消息，并让标准助手消息直接通过。这些共享类型既不是模型服务商的原始响应，也不是其特有的请求载荷；记录一条构造后的助手消息，不意味着记录了原始响应中的每个字段。

助手消息会保留来源元数据（`provider`、`api`、`model`），并在内容块中保留部分模型服务商续接字段。因此，共享表示并没有抹去所有模型服务商特有的信息。后续的适配器转换会决定哪些内容可以针对目标模型服务商、API 和模型进行重放。

在 `transformMessages` 中，三个来源字段必须全部匹配，`isSameModel` 才成立。对于同一模型，即使可见文本为空，带签名的思考块也会被保留。跨模型转换时，未被隐去且非空的思考内容会变成不带签名的纯文本；空的思考内容会被省略。被隐去的思考内容只在同一模型下保留，否则会被完全移除，而不是作为空块发送。跨模型处理还会视情况去除文本签名和工具调用的思考签名。

在 OpenAI Responses 适配器中，`thinkingSignature` 包含序列化后的推理条目，其中可能含有加密内容。适配器解析该字段，并将该条目追加到请求中。如果没有这个字段，该分支就不会为这个块输出推理条目。因此，可见文本为空不代表没有续接数据。其他模型服务商对签名字段的解释不同；这里并不是说每个签名都是加密的推理内容。

### 证据
- `pi/packages/agent/src/types.ts:347-370`：可扩展的 `AgentMessage` 联合类型。
- `pi/packages/coding-agent/src/core/messages.ts:148-196`：`convertToLlm`。
- `pi/packages/ai/src/types.ts`：`AssistantMessage`、`ThinkingContent`、`TextContent`、`ToolCall`。
- `pi/packages/ai/src/api/transform-messages.ts:92-134`：模型身份与续接字段的处理。
- `pi/packages/ai/src/api/openai-responses-shared.ts:261-266,539-547`：推理条目的重放与加密内容的保留。

---

## Q9

编码工具定义将参数模式、描述与本地可执行行为结合起来。内置定义在工具模块中构造；`_buildRuntime` 填充 `_baseToolDefinitions` 并初始化扩展运行器。`_refreshToolRegistry` 将允许使用的内置工具与已注册的扩展工具、SDK 自定义工具合并，封装为 AgentTool 对象，并填充 `_toolRegistry`。允许与排除筛选影响的是工具是否进入注册表，而不仅仅是是否启用。

选择启用哪些工具是单独的一步：显式的 activeToolNames 或 previousActiveToolNames 提供初始选择；允许列表、includeAllExtensionTools 以及处理新注册工具的分支还可以添加名称。`setActiveToolsByName` 从注册表中解析选中的名称，并替换 `agent.state.tools`。默认初始化会构造所有内置工具定义，但只选择 read/bash/edit/write 以及扩展工具；因此，grep/find/ls/powershell 可以已注册但未启用。这不同于一种假设情况：在原注册表为空时，首次进行不带选项的刷新，此时所有新的注册表条目都会被启用。

循环执行时使用 `currentContext.tools`，即在适当的轮次边界从 Agent 状态刷新的快照，而不是直接查询原始定义或会话注册表。快照中的工具保留 execute。`toToolDeclaration` 去掉可执行行为和仅用于展示的行为，保留 name、description、parameters，以及可选的 constrainedSampling。循环通过 toolsAdded/toolsRemoved 记录声明的变化；模型服务商适配过程生成实际的 API 请求。不要把这些对话记录更新误解为发送 execute 函数，也不要假设每个模型服务商都接受增量形式的工具字段。

### 证据
- `pi/packages/coding-agent/src/core/tools/read.ts`：具体的 read 参数模式、描述与实现。
- `pi/packages/coding-agent/src/core/tools/index.ts:182-192`：createAllToolDefinitions。
- `pi/packages/coding-agent/src/core/agent-session.ts:1279-1290,3144-3288`：启用工具的选择、注册表刷新和运行时构造。
- `pi/packages/agent/src/agent.ts:457-461`：上下文快照。
- `pi/packages/agent/src/agent-loop.ts:184-210,322-361,703-788`：上下文刷新、声明、运行时查找与执行。
- `pi/packages/ai/src/utils/transcript.ts:123-129`：toToolDeclaration。

---

## Q10

循环在 currentContext.tools 中查找请求的工具。prepareToolCallArguments 可以先规范化参数，然后 validateToolArguments 根据参数模式进行校验。可选的 beforeToolCall 钩子接收已校验的参数，并可以阻止执行；工具缺失、校验或准备失败、执行被阻止，以及在检查点发现中止，都会立即产生错误结果，而不是调用该工具。

对于准备完成的调用，executePreparedToolCall 会调用本地 execute 函数，传入调用 ID、已校验的参数、中止信号以及部分更新回调。执行成功时返回结果，并设置 isError 为 false；抛出的执行错误会由 createErrorToolResult 转换为文本错误结果，isError 为 true。最终处理可以应用可选的 afterToolCall 覆盖，对 content、details、usage、terminate 和 isError 进行调整；最终处理钩子抛出的错误也会表示为错误结果。对报告结果的覆盖本身不会撤销工具已经造成的外部影响。

createToolResultMessage 将最终处理后的内容与错误状态关联到该工具调用。随后发出工具结果消息事件，并将批次结果追加到 currentContext.messages 和 newMessages，以供后续模型上下文使用。这种观测是执行情况的报告，并非外部文件系统状态的权威来源。顺序与并发问题在 Q11 中单独评估。

### 证据
- `pi/packages/agent/src/agent-loop.ts:703-770`：prepareToolCall、查找、校验、阻止执行和立即产生的结果。
- `pi/packages/agent/src/agent-loop.ts:773-814`：executePreparedToolCall 与错误转换。
- `pi/packages/agent/src/agent-loop.ts:816-860`：finalizeExecutedToolCall 与可选的结果覆盖。
- `pi/packages/agent/src/agent-loop.ts`：createErrorToolResult、createToolResultMessage、emitToolResultMessage、executeToolCallsSequential/Parallel，以及 runLoop 将结果重新加入上下文的逻辑。

---

## Q11

当 config.toolExecution 为 sequential，或任意一个被请求工具的 executionMode 为 sequential 时，executeToolCalls 选择顺序执行；否则选择并行路径。这个按工具设置的覆盖会让整个批次串行执行，而不只是该工具。顺序执行会依次等待每个调用完成准备、执行、最终处理和结果发出，然后才处理下一个调用，并在调用之间检查中止状态。

并行路径按原始顺序准备调用，并保存立即产生的结果或延后执行的函数。Promise.all 调用这些函数；即使完成时间不同，它返回的数组仍保留输入顺序。执行结束事件在各自的执行与最终处理任务内部发出，因此不一定遵循调用顺序；最终的工具结果消息则在之后通过遍历 orderedFinalizedCalls 发出，因此保留原始调用顺序。结束事件的时机也包含最终处理所需时间，而不只是原始 execute 完成的时刻。

稳定的对话记录顺序不意味着副作用被串行化：并发的读取可能在另一个调用写入文件之前就读到该文件，即使对话记录中的写入结果排在读取结果之前。模型决定把哪些调用放在同一批次；运行时配置和工具元数据决定调度方式。存在依赖关系的操作可以分散到多个轮次请求，使模型先看到一个结果，再选择下一次调用及其参数。单凭顺序调度，并不能让模型修改它已经一起发出的调用参数。

### 证据
- `pi/packages/agent/src/agent-loop.ts:505-519`：全局模式与按工具设置的模式选择。
- `pi/packages/agent/src/agent-loop.ts:527-580`：顺序执行。
- `pi/packages/agent/src/agent-loop.ts:583-656`：准备、延后的并行执行、结束事件、Promise.all，以及按顺序发出的结果消息。
- `pi/packages/agent/src/types.ts`：ToolExecutionMode。

---

## Q12

未知工具、参数准备或校验失败、执行被阻止，以及被捕获的工具执行异常，都会变成错误形式的工具结果观测。这类错误本身不意味着模型服务商失败，也不会自动终止底层运行；终止提示、中止以及其他循环控制是独立的机制。

stopReason 为 error 或 aborted 的助手响应会作为助手消息保留。runLoop 调用 finishTurn，发出带有空 toolResults 数组的 turn_end 以及 agent_end，然后在处理工具调用之前返回。这条路径不会执行该响应请求的任何工具，也不会为各个调用创建工具结果消息。助手错误消息不是工具结果观测。

对于 stopReason 为 length 的情况，即使工具参数仍可解析，也可能已经被截断。failToolCallsFromTruncatedMessage 不执行任何调用，而是为每个调用创建一条 isError 工具结果消息。这些观测会被发出并重新加入上下文，使模型能够沿正常的继续路径重新发出完整调用。这与 error/aborted 的提前返回不同。

在工具处理期间中止，也会在检查边界阻止后续执行，并将取消信号传给正在执行的工具；它不会撤销已经完成的外部影响。会话层的重试或溢出处理发生在底层运行结束之后。如果继续恢复，agent.continue() 会使用准备好的上下文启动一次新的底层运行，而不是恢复已经返回的 runLoop 调用。并非所有错误都可以重试，取消也不保证会自动重试。

### 证据
- `pi/packages/agent/src/agent-loop.ts:244-275`：助手 error/aborted 的提前返回，与 length 的处理和结果重新加入上下文之间的区别。
- `pi/packages/agent/src/agent-loop.ts:440-465`：保留最终助手响应。
- `pi/packages/agent/src/agent-loop.ts:468-499`：failToolCallsFromTruncatedMessage。
- `pi/packages/agent/src/agent-loop.ts:703-860`：准备失败、执行异常、取消检查和最终处理。
- `pi/packages/coding-agent/src/core/agent-session.ts`：_handlePostAgentRun、_prepareRetry，以及通过 agent.continue() 进行恢复；Q13 已说明这是一次全新运行的边界。
