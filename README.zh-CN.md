# Teach You Pi Agent — 跟着问题学习 Pi 源码

[English](README.md)

跟着 AI 老师阅读源码、回答问题，理解 Pi 架构。这是社区学习 fork，不是 Pi 官方课程，也不需要先完成其他课程或访问私人项目。

## Install & Quickstart（安装与快速开始）

打开你常用的 agent（Codex、Claude Code、Hermes、Pi Agent 等），粘贴下面的提示词：

```text
请将 https://github.com/co1smos/teach-you-pi-agent.git 克隆到新目录；如果已有本地副本就复用，不覆盖已有工作。
在仓库根目录阅读 doc/questions.zh-CN.md 并遵守其中的教师协议。仅当 ../pi-learning-progress.md 不存在时，从题目文档创建这个私人副本；否则恢复已有进度。保持公开文件不变。
告诉我仓库和进度文件的路径，然后立即按教学顺序提出第一道尚未通过的问题，等待我回答。
```

Agent 需要 Git、网络及本地文件和命令访问权限，无需安装 Pi。它会下载源码，直接开始**板块 1 / Q1**，或恢复已有进度。

## 选择文档

| 内容 | English | 中文 |
|---|---|---|
| 题目、提问协议、验收标准、空白问答记录 | [questions.md](doc/questions.md) | [questions.zh-CN.md](doc/questions.zh-CN.md) |
| 参考答案和源码依据（包含答案，请勿提前阅读） | [answers.md](doc/answers.md) | [answers.zh-CN.md](doc/answers.zh-CN.md) |

先自己回答，再看参考答案。每题 PASS 后，在私人练习册中一起保留你的答案、老师标准答案和源码依据。

## 固定源码版本

`pi/` 保存完整源码，固定在 [`a32782520f69cd81b54814c3a13df4c7bd1f3ad7`](https://github.com/earendil-works/pi/tree/a32782520f69cd81b54814c3a13df4c7bd1f3ad7)。源码路径相对于本仓库根目录（`pi/packages/...`）。使用这个版本，才能对应练习册中的引用。

实际运行 Pi 请参考[上游 README](pi/README.md)。依赖 Git 根目录的开发脚本与 CI 未适配此教学布局。

## 学习顺序

| 板块 | 主题 | 顺序 |
|---|---|---|
| 1 | Agent Loop 与生命周期 | Q1 → Q13 |
| 2 | 状态与会话 | Q2 → Q3 → Q4 → Q6 → Q7 → Q14 |
| 3 | 消息与模型上下文 | Q5 → Q8 |
| 4 | 工具系统 | Q9 → Q10 → Q11 → Q12 |

按上表顺序学习，不按 Q 编号递增。共 **14 题**，每题都提供源码入口和验收要求。

## 自学方式

将中英文任一题目文档复制到 `../pi-learning-progress.md`，读源码，用自己的话作答，再对照参考答案。遵守练习册的 **95% 验收门槛**，记录每道已通过的问答。

## 恢复与完成

复用同一提示词、仓库和私人练习册路径即可继续。切换语言也只维护一个进度文件。所有题目通过，并记录学生答案、老师答案和依据，才算完成。公开私人练习册前请检查个人信息。

## 范围与来源

参考答案由 AI 辅助整理，以固定版本源码为准。不包含原学习者的私人回答。上游源码及其 [MIT 许可证](pi/LICENSE) 原样保留。
