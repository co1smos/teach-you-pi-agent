# Pi 架构问题与学习手册

[English](questions.md) | [使用指南](../README.zh-CN.md)

通过源码阅读问题学习 Pi 架构。本手册不包含任何私人的学生答案或已完成的学习进度，也不需要先完成其他课程或访问私人项目。

## 源码基准

- 上游：https://github.com/earendil-works/pi
- 固定的源码修订版本：`a32782520f69cd81b54814c3a13df4c7bd1f3ad7`
- 下文中的源码路径均相对于本仓库根目录。请阅读 `main` 上的源码，而不是持续变化的上游 `main`。
- 目标：根据证据解释循环、状态与会话、消息、工具、终止条件以及外部状态边界，而不是背诵文件名。
- 这是源码阅读练习，不是要求重构 Pi。阅读代码不需要安装依赖、提供模型服务商凭据或发起真实的 LLM 调用。

## 学习导航

| 板块 | 重点 | 教学顺序 |
|---|---|---|
| 1 | 智能体循环与生命周期 | Q1 → Q13 |
| 2 | 状态与会话 | Q2 → Q3 → Q4 → Q6 → Q7 → Q14 |
| 3 | 消息与模型上下文 | Q5 → Q8 |
| 4 | 工具系统 | Q9 → Q10 → Q11 → Q12 |

Q 编号是固定的标识，不代表按数字递增的阅读顺序。每个板块都包含学习目标、主要源码入口以及各题的具体要求。板块导读不是参考答案。

# 教师协议

教师应当像一位要求严格的技术老师，而不是答案生成器。

### 导航与进度

按导航表中的板块顺序和题目顺序教学，不要按 Q 编号递增的顺序教学。恢复学习时，先阅读已有进度，再从该顺序中第一道尚未标记为 PASS 的题目开始；不要重新提问已经通过的题目。每次提问前，说明当前板块、主题和 Q 编号，一次只问一道题。一道题通过后，在学习者的本手册私人工作副本中，同时更新题库中的状态和对应的通过答案记录。重新编排教学顺序不会改变题目编号。

板块导读是学习目标，不是参考答案。不要把导航概述当作足以评判答案的证据。

## 95% 通过门槛

只有当学生的解释至少达到 95% 正确时，该题才能通过。

“95%”意味着：

- 核心机制正确；
- 重要的归属与职责边界正确；
- 相关的控制流与数据流正确；
- 没有遗漏题目要求的任何主要概念；
- 所有论断都与固定版本的源码一致；
- 剩余的错误或遗漏仅涉及细微的命名、措辞或非关键的边界情况。

即使答案的大方向正确，只要跳过了核心层次或混淆了两个抽象，也不能判定通过。

## 学生通过之前

当学生的答案尚未达到门槛时：

- **不要**给出正确答案。
- **不要**给出几乎完整的正确答案改述。
- 指出学生理解中哪些部分不完整、不一致或缺乏证据。
- 优先使用苏格拉底式提示和有针对性的追问。
- 指向一小段源码：文件 + 符号；必要时提供行号范围。
- 如果困惑属于概念问题，推荐一份一手文档，或一篇针对该问题的外部文章、文档页面。
- 如果可以进行网页搜索，而且补充说明能带来实质帮助，就使用搜索。优先选择一手资料。
- 要求学生修改同一道题的答案，不要进入下一道编号题目。

目标是帮助学生依据证据自行发现答案。

## 学生通过之后

当答案达到门槛时：

1. 明确说明该题通过。
2. 指出仍需修正的细节。
3. 编辑私人工作副本中对应的“通过答案记录”条目：
   - 记录学生通过时的答案，尽可能保留学生的原话；
   - 紧接其后写下教师的简明标准答案，并对照固定版本的源码核查 answers.zh-CN.md（英文版为 answers.md）中对应的部分；
   - 补充具有决定性的 Pi 源码证据（文件 + 符号；必要时提供行号范围）；
   - 将该题标记为 `PASS`。
4. 然后按板块的教学顺序提出下一道题。

在该题通过之前，绝不能把标准答案写入本文件。

## 评估维度

使用以下维度判断答案是否完整；如果缺少核心概念，不要机械地对各维度取平均分。

| 维度 | 必须正确的内容 |
|---|---|
| 机制 | 实际发生了什么 |
| 归属 | 哪个抽象或函数负责该行为 |
| 数据流 | 各阶段之间传递什么数据 |
| 边界 | 运行时、会话、模型服务商和工具各自的职责 |
| 边界语义 | 重要的停止、错误与顺序行为 |
| 证据 | 论断能追溯到固定版本的源码 |

## 源码导航辅助工具

教师可以使用仓库图谱工具查找调用路径和架构边界：

- GitNexus（如果已安装或可用）。
- Graphify（如果已安装或可用）。
- 常规源码搜索（`grep`、`find`、编辑器符号搜索）始终可以使用。

使用 Graphify 时，可以尝试聚焦的概念查询、符号之间的路径查询或架构报告。使用 GitNexus 时，通过图谱或 MCP 查询发现符号和调用路径。无论使用哪种方式，都必须先在实际的 Pi 源码中核实结论，再评判答案。

如果直接阅读源码更快，就不要花大量时间安装或调试图谱工具。

### 答案分离与语言

公开的 `answers.md`（中文版为 `answers.zh-CN.md`）是独立的参考答案，不是学习进度。教师在需要评估学生作答时，只能私下查阅当前题目的参考答案；在 PASS 之前，绝不能透露或改述其中的答案。源码证据的优先级高于参考答案。只有在 PASS 之后，才能把学生答案和教师答案一起记录下来。选择一个私人工作副本（英文或中文）作为唯一的进度记录；切换教学语言不会产生第二份记录。保留学生的原话和原始语言。如另附译文，必须明确标注。95% 门槛不由自动评分器执行。

# 阅读地图

先从范围最小、足以解决问题的源码入手，不必从头到尾通读 Pi。

## 入门导览

- `pi/packages/coding-agent/docs/how-pi-works.md`
- `pi/packages/coding-agent/docs/session-format.md`

## 智能体循环

- `pi/packages/agent/src/agent-loop.ts`
  - `runAgentLoop()`
  - `runAgentLoopContinue()`
  - `runLoop()`
  - `streamAssistantResponse()`
  - `executeToolCalls()`
  - `executeToolCallsSequential()`
  - `executeToolCallsParallel()`

## 运行时状态封装

- `pi/packages/agent/src/agent.ts`
  - `createMutableAgentState()`
  - `Agent`
  - `Agent.prompt()`
  - `Agent.continue()`
  - `processEvents()` 周围的事件处理与状态归约逻辑

- `pi/packages/agent/src/types.ts`
  - `AgentContext`
  - `AgentState`
  - `AgentMessage`
  - `AgentTool`
  - `AgentToolResult`
  - `ToolExecutionMode`

## 持久化会话与上下文投影

- `pi/packages/coding-agent/src/core/agent-session.ts`
  - `AgentSession`
  - `prompt()`
  - `prepareNextTurn` 的集成方式
  - 仅在理解边界所必需时阅读上下文压缩的集成逻辑

- `pi/packages/coding-agent/src/core/session-manager.ts`
  - `SessionEntry` 的各个变体
  - `buildSessionPath()`
  - `buildContextEntries()`
  - `buildSessionProjection()`
  - `buildSessionContext()`
  - `SessionManager`
  - `appendMessage()`
  - `branch()`
  - `branchWithSummary()`

## 消息

- `pi/packages/coding-agent/src/core/messages.ts`
  - 自定义消息类型
  - `convertToLlm()`

- `pi/packages/ai/src/types.ts`
  - 仅在需要明确基础消息或内容类型时查阅

## 工具

- `pi/packages/coding-agent/src/core/tools/index.ts`
  - `createToolDefinition()`
  - `createTool()`
  - `createCodingToolDefinitions()`
  - `createCodingTools()`

- `pi/packages/agent/src/agent-loop.ts`
  - 执行分派与顺序

- `pi/packages/agent/src/harness/execution/tools.ts`
  - `prepareToolCall()`
  - `executeToolCall()`
  - `finalizeToolCall()`
  - 可选对照：这是独立的 AgentHarness 执行路径，不是 agent-loop.ts 调用的某个阶段

# 题库

逐个板块学习。在每个板块内按顺序回答问题，除非需要借助后面的题目来解决前面的题目。

四个核心板块覆盖 Pi 的架构：

1. **智能体循环与生命周期**
2. **状态与会话**
3. **消息与模型上下文**
4. **工具系统**

完成全部四个板块。本公开版仅包含 Q1–Q14。

---

# 板块 1 — 智能体循环与生命周期

## 本板块要回答什么

完成本板块后，你应当能够解释：

```text
提示输入
  ↓
模型请求
  ↓
助手响应
  ↓
工具调用
  ↓
工具结果
  ↓
下一次模型请求，或停止
```

重要的不只是“哪个函数在运行”，而是**谁负责反复执行的控制流，以及谁决定本次运行是否继续**。

### 主要源码路径

- `pi/packages/agent/src/agent-loop.ts`
  - `runAgentLoop()`
  - `runLoop()`
  - `streamAssistantResponse()`
  - `executeToolCalls()`
- `pi/packages/agent/src/types.ts::FinishTurn`

## Q1 — 究竟谁负责智能体循环？

解释 Pi 的一个正常轮次：从接受提示输入开始，到 Pi 决定是否需要再次请求模型为止。

你的答案必须指出：

- 哪个抽象或函数负责反复执行的模型与工具循环；
- 助手响应在哪里产生；
- 工具调用在哪里提取；
- 工具结果在哪里重新进入上下文；
- 什么会导致下一次迭代，什么会导致终止。

**主要源码线索**

- `pi/packages/agent/src/agent-loop.ts::runLoop`
- `pi/packages/agent/src/agent-loop.ts::streamAssistantResponse`
- `pi/packages/agent/src/agent-loop.ts::executeToolCalls`

**状态：** NOT STARTED

---

## Q13 — 谁决定一次运行已经结束？

找出所有在行为上有实质差异的终止与继续机制。

区分以下情况：

- 不再有工具工作需要处理；
- 显式的结束轮次行为；
- 队列中的引导消息（steering）或后续消息（follow-up）；
- 错误或中止；
- 工具给出的终止提示；
- coding-agent 或会话层的重试、恢复行为。

**主要源码线索**

- `runLoop()`
- `pi/packages/agent/src/types.ts` 中的 `FinishTurn`
- 工具的 `terminate` 语义
- 必要时查阅 `AgentSession` 中相关的恢复钩子

**状态：** NOT STARTED

---

# 板块 2 — 状态与会话

## 本板块要回答什么

本板块包含三个主要阅读主题，与前面的源码阅读指南对应：

- **智能体运行时：** `Agent` 负责什么，哪些内容会在多次调用之间保留？
- **状态与会话职责：** `Agent`、`AgentSession` 和 `SessionManager` 如何分工？
- **会话树：** 如何表示分支、持久化以及当前历史的选择？

然后检查它们与外部状态之间的边界。请根据源码补全每个组件的职责；本概述有意不提供这些答案。

核心问题是：

> 为什么 Pi 需要多个状态与会话抽象，而不是一个巨大的状态对象？

你还应当理解以下几种状态的区别：

```text
单次运行的执行状态
跨多次提示输入的实时对话状态
持久化会话状态
外部权威状态
```

### 主要源码路径

- `pi/packages/agent/src/agent.ts`
- `pi/packages/coding-agent/src/core/agent-session.ts`
- `pi/packages/coding-agent/src/core/session-manager.ts`
- `pi/packages/coding-agent/docs/session-format.md`

## Q2 — 如果循环由 `agent-loop.ts` 负责，`Agent` 的作用是什么？

解释为什么 Pi 除了底层循环函数之外，还需要 `Agent`。

涵盖以下内容：

- `Agent` 拥有哪些状态；
- 它负责哪些生命周期与事件行为；
- `prompt()` 和 `continue()` 分别意味着什么；
- 排队的引导消息与后续消息如何参与其中；
- 哪些职责仍在 `Agent` 之外。

**主要源码线索**

- `pi/packages/agent/src/agent.ts::createMutableAgentState`
- `pi/packages/agent/src/agent.ts::Agent`
- `pi/packages/agent/src/agent.ts::prompt`
- `pi/packages/agent/src/agent.ts::continue`
- `pi/packages/agent/src/agent.ts` 中的事件归约逻辑

**状态：** NOT STARTED

---

## Q3 — 运行时状态与持久化会话状态

解释以下三者的区别：

- `Agent`
- `AgentSession`
- `SessionManager`

要通过本题，必须解释为什么 Pi 同时需要这三者，而不是只使用一个“状态”对象。

还要说明哪些内容分别属于：

- 当前运行时状态；
- coding-agent 编排；
- 持久化会话历史。

**主要源码线索**

- `pi/packages/agent/src/agent.ts`
- `pi/packages/coding-agent/src/core/agent-session.ts::AgentSession`
- `pi/packages/coding-agent/src/core/session-manager.ts::SessionManager`

**状态：** NOT STARTED

---

## Q4 — Pi 究竟是不是有状态的？

给出精确的答案；不要只说“有状态”而不指明具体层次。

分别解释 Pi 在以下范围内是否有状态：

- 一次模型与工具运行中的多个步骤之间；
- 同一个存活的智能体或会话中的多次用户提示之间；
- 进程重启或恢复之后；
- 文件系统等外部世界状态。

**主要源码线索**

- `Agent.state.messages`
- `Agent.prompt()`
- `SessionManager` 的持久化机制
- 会话 JSONL 文档

**状态：** NOT STARTED

---

## Q6 — 为什么会话是一棵树，而不是扁平的消息列表？

解释：

- `id` / `parentId` 分别表示什么；
- 当前叶节点意味着什么；
- 分支如何工作；
- 创建分支时是否会销毁旧条目；
- 这为什么会影响模型上下文的重建。

**主要源码线索**

- `pi/packages/coding-agent/docs/session-format.md`
- `pi/packages/coding-agent/src/core/session-manager.ts::buildSessionPath`
- `SessionManager.branch`
- `SessionManager.branchWithSummary`

**状态：** NOT STARTED

---

## Q7 — 当前上下文与审计、历史记录

解释 Pi 是否区分“发生过的一切”和“模型接下来会看到的内容”。

至少使用源码中的两个具体机制或例子，例如：

- 分支选择；
- 上下文压缩；
- 上下文编辑；
- 自定义条目或仅用于状态的条目。

重点讨论当前上下文与持久化历史之间的边界。

**主要源码线索**

- `session-manager.ts` 中 `SessionEntry` 的各个变体
- `buildContextEntries()`
- `CompactionEntry`
- `ContextEditEntry`
- `CustomEntry` 与 `CustomMessageEntry` 的区别

**状态：** NOT STARTED

---

## Q14 — 编码智能体中的外部权威状态是什么？

以文件系统和 shell 为具体例子，解释：

- 哪些状态存在于对话记录或会话之外；
- 工具结果向模型传达了什么；
- 对话记录本身是否是文件状态的事实依据；
- 当外部状态发生变化时，智能体如何重新与之同步。


**主要源码线索**

- 编码工具（`read`、`write`、`edit`、`bash`）
- 工具结果的行为
- 无需查看 UI 代码

**状态：** NOT STARTED

---

# 板块 3 — 消息与模型上下文

## 本板块要回答什么

追踪起点与终点如下的数据路径：

```text
起点：持久化的 SessionEntry 历史
        ↓
中间的选择、投影和转换阶段：自行从源码中查找
        ↓
终点：下一次模型服务商请求
```

自行识别中间阶段的实际顺序、输入输出类型以及职责归属。你应当明确知道**历史在哪里变成当前模型上下文**，以及哪些信息保持统一表示、哪些信息是模型服务商特有的。

### 主要源码路径

- `pi/packages/coding-agent/src/core/session-manager.ts`
  - `buildSessionPath()`
  - `buildContextEntries()`
  - `buildSessionProjection()`
  - `buildSessionContext()`
- `pi/packages/coding-agent/src/core/messages.ts::convertToLlm()`
- `pi/packages/agent/src/types.ts`
- `pi/packages/ai/src/types.ts`
- `pi/packages/agent/src/agent-loop.ts::streamAssistantResponse()`

## Q5 — 持久化会话如何变成下一次 LLM 调用的上下文？

追踪持久化会话条目如何逐步转换成最终适合模型使用的消息。

涵盖以下内容：

- 树、分支与叶节点的选择；
- 哪些会话条目会进入上下文；
- 投影与上下文构建；
- 自定义消息的转换；
- 最后的 `convertToLlm()` 边界。

**主要源码线索**

- `pi/packages/coding-agent/src/core/session-manager.ts::buildSessionPath`
- `buildContextEntries`
- `buildSessionProjection`
- `buildSessionContext`
- `pi/packages/coding-agent/src/core/messages.ts::convertToLlm`
- `pi/packages/agent/src/agent-loop.ts::streamAssistantResponse`

**状态：** NOT STARTED

---

## Q8 — Pi 的消息模型中，哪些是统一表示，哪些是模型服务商特有的信息？

解释：

- `AgentMessage` 的作用；
- 面向模型的 `Message` 类型的作用；
- 转换发生在哪里；
- 助手消息是否保留模型服务商和模型的元数据；
- Pi 是完全抹去模型服务商特有的续接信息，还是保留其中一部分。

如果最后一点因模型服务商或字段而异，请准确说明，不要强行给出非此即彼的答案。

**主要源码线索**

- `pi/packages/agent/src/types.ts`
- `pi/packages/coding-agent/src/core/messages.ts::convertToLlm`
- `pi/packages/ai/src/types.ts`
- 仅在核实续接字段所必需时查阅模型服务商适配器

**状态：** NOT STARTED

---

# 板块 4 — 工具系统

## 本板块要回答什么

追踪完整的工具处理流程：

```text
工具定义与参数模式
        ↓
向模型开放的当前工具集合
        ↓
模型输出 toolCall
        ↓
查找与校验
        ↓
执行
        ↓
ToolResult
        ↓
工具结果消息
        ↓
下一次模型上下文
```

本板块还涵盖多个工具的执行顺序，以及**可恢复的工具失败**与会改变或停止本次运行的失败之间的区别。

### 主要源码路径

- `pi/packages/coding-agent/src/core/tools/index.ts`
- 一个具体工具，例如 `read.ts` 或 `bash.ts`
- `pi/packages/agent/src/agent-loop.ts::executeToolCalls()`
- `pi/packages/agent/src/harness/execution/tools.ts`
- `pi/packages/agent/src/types.ts::ToolExecutionMode`

## Q9 — 工具如何声明并向模型开放？

追踪：

- 一个编码工具的实现；
- 它的参数模式与描述；
- 构造与注册过程；
- 当前启用的工具集合如何进入模型请求。

解释工具定义与可执行行为之间的边界。

**主要源码线索**

- `pi/packages/coding-agent/src/core/tools/index.ts`
- 一个具体工具，例如 `core/tools/read.ts` 或 `core/tools/bash.ts`
- `Agent.state.tools`
- 智能体核心中 `toToolDeclaration` 的用法

**状态：** NOT STARTED

---

## Q10 — 模型的工具调用如何变成执行，再变成观测结果？

端到端追踪一次工具调用。

涵盖以下内容：

- 工具查找；
- 参数校验；
- 执行前的检查或钩子；
- 执行；
- 结果与错误的表示；
- 工具结果消息的创建；
- 如何重新加入后续的模型上下文。

**主要源码线索**

- `pi/packages/agent/src/agent-loop.ts::executeToolCalls`
- `pi/packages/agent/src/harness/execution/tools.ts`
- `AgentToolResult` / `ToolResultMessage`

**状态：** NOT STARTED

---

## Q11 — 工具的顺序执行与并行执行

解释 Pi 如何执行一条助手消息中的多个工具调用。

涵盖以下内容：

- 在哪里选择执行模式；
- Pi 何时采用顺序执行；
- 何时允许并发执行；
- 完成顺序与模型可见的结果顺序是否必然相同；
- 这种区别为什么重要。

先解释当前行为，再提出修改建议。

**主要源码线索**

- `pi/packages/agent/src/types.ts::ToolExecutionMode`
- `pi/packages/agent/src/agent-loop.ts::executeToolCalls`
- `executeToolCallsSequential`
- `executeToolCallsParallel`

**状态：** NOT STARTED

---

## Q12 — 工具失败与模型、运行时失败

解释 Pi 如何处理：

- 无效或未知的工具调用；
- 工具执行异常；
- 被阻止的工具执行；
- 模型或模型服务商的错误响应；
- 被中止的运行。

哪些失败会变成模型能够据此作出反应的观测结果，哪些会停止或改变本次运行？

**主要源码线索**

- `pi/packages/agent/src/agent-loop.ts`
- `pi/packages/agent/src/harness/execution/tools.ts`
- 仅在必要时查阅 `pi/packages/coding-agent/src/core/agent-session.ts` 中的重试与上下文溢出处理

**状态：** NOT STARTED

# 通过答案记录

使用私人工作副本；公开模板的记录初始为空。每题 PASS 之后，在这里同时记录学生通过时的答案与教师标准答案，并更新上方该题的状态。不要覆盖公开的参考答案。

## Q1 记录

**状态：** NOT STARTED

### 学生通过时的答案

（仅在 PASS 之后记录；保留原话和原始语言。）

### 教师标准答案

（仅在 PASS 之后记录。）

### 证据

（固定修订版本中的文件、符号和相关行号。）

---

## Q13 记录

**状态：** NOT STARTED

### 学生通过时的答案

（仅在 PASS 之后记录；保留原话和原始语言。）

### 教师标准答案

（仅在 PASS 之后记录。）

### 证据

（固定修订版本中的文件、符号和相关行号。）

---

## Q2 记录

**状态：** NOT STARTED

### 学生通过时的答案

（仅在 PASS 之后记录；保留原话和原始语言。）

### 教师标准答案

（仅在 PASS 之后记录。）

### 证据

（固定修订版本中的文件、符号和相关行号。）

---

## Q3 记录

**状态：** NOT STARTED

### 学生通过时的答案

（仅在 PASS 之后记录；保留原话和原始语言。）

### 教师标准答案

（仅在 PASS 之后记录。）

### 证据

（固定修订版本中的文件、符号和相关行号。）

---

## Q4 记录

**状态：** NOT STARTED

### 学生通过时的答案

（仅在 PASS 之后记录；保留原话和原始语言。）

### 教师标准答案

（仅在 PASS 之后记录。）

### 证据

（固定修订版本中的文件、符号和相关行号。）

---

## Q6 记录

**状态：** NOT STARTED

### 学生通过时的答案

（仅在 PASS 之后记录；保留原话和原始语言。）

### 教师标准答案

（仅在 PASS 之后记录。）

### 证据

（固定修订版本中的文件、符号和相关行号。）

---

## Q7 记录

**状态：** NOT STARTED

### 学生通过时的答案

（仅在 PASS 之后记录；保留原话和原始语言。）

### 教师标准答案

（仅在 PASS 之后记录。）

### 证据

（固定修订版本中的文件、符号和相关行号。）

---

## Q14 记录

**状态：** NOT STARTED

### 学生通过时的答案

（仅在 PASS 之后记录；保留原话和原始语言。）

### 教师标准答案

（仅在 PASS 之后记录。）

### 证据

（固定修订版本中的文件、符号和相关行号。）

---

## Q5 记录

**状态：** NOT STARTED

### 学生通过时的答案

（仅在 PASS 之后记录；保留原话和原始语言。）

### 教师标准答案

（仅在 PASS 之后记录。）

### 证据

（固定修订版本中的文件、符号和相关行号。）

---

## Q8 记录

**状态：** NOT STARTED

### 学生通过时的答案

（仅在 PASS 之后记录；保留原话和原始语言。）

### 教师标准答案

（仅在 PASS 之后记录。）

### 证据

（固定修订版本中的文件、符号和相关行号。）

---

## Q9 记录

**状态：** NOT STARTED

### 学生通过时的答案

（仅在 PASS 之后记录；保留原话和原始语言。）

### 教师标准答案

（仅在 PASS 之后记录。）

### 证据

（固定修订版本中的文件、符号和相关行号。）

---

## Q10 记录

**状态：** NOT STARTED

### 学生通过时的答案

（仅在 PASS 之后记录；保留原话和原始语言。）

### 教师标准答案

（仅在 PASS 之后记录。）

### 证据

（固定修订版本中的文件、符号和相关行号。）

---

## Q11 记录

**状态：** NOT STARTED

### 学生通过时的答案

（仅在 PASS 之后记录；保留原话和原始语言。）

### 教师标准答案

（仅在 PASS 之后记录。）

### 证据

（固定修订版本中的文件、符号和相关行号。）

---

## Q12 记录

**状态：** NOT STARTED

### 学生通过时的答案

（仅在 PASS 之后记录；保留原话和原始语言。）

### 教师标准答案

（仅在 PASS 之后记录。）

### 证据

（固定修订版本中的文件、符号和相关行号。）

---

# 完成规则

当 Q1–Q14 的每一道题都满足自身要求并达到 95% 通过门槛，且学生答案、教师答案和源码证据均已记录在同一份私人学习手册中时，本练习才算完成。恢复学习时继续遵循板块顺序。无需完成额外的比较里程碑，也不需要外部项目。
