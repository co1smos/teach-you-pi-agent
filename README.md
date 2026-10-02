# Teach You Pi Agent

[中文](README.zh-CN.md)

Learn Pi architecture by reading source and answering questions with an AI tutor. This is a community learning fork, not an official Pi course. No earlier course or private project is required.

## Install & Quickstart

Open any agent you use (Codex, Claude Code, Hermes, Pi Agent, etc.) and paste the prompt below:

```text
Clone https://github.com/co1smos/teach-you-pi-agent.git into a new directory, or reuse an existing checkout without overwriting local work.
From the checkout root, read doc/questions.md and follow its teacher protocol. Create ../pi-learning-progress.md from it only if absent; otherwise resume that private workbook. Keep public files unchanged.
Tell me the checkout and progress paths, then immediately ask the first unfinished question in teaching order. Wait for my answer.
```

Your agent needs Git, network and local file/command access. No Pi installation is required. It downloads the source and starts **Block 1 / Q1**, or resumes your progress.

## Choose your documents

| Document | English | 中文 |
|---|---|---|
| Questions, teaching protocol, acceptance criteria, blank Q&A records | [questions.md](doc/questions.md) | [questions.zh-CN.md](doc/questions.zh-CN.md) |
| Reference answers and source evidence — spoilers | [answers.md](doc/answers.md) | [answers.zh-CN.md](doc/answers.zh-CN.md) |

Try each question before opening its reference answer. After PASS, your private workbook keeps your answer, the teacher's standard answer, and source evidence together.

## Fixed source baseline

The complete source in `pi/` is pinned to [`a32782520f69cd81b54814c3a13df4c7bd1f3ad7`](https://github.com/earendil-works/pi/tree/a32782520f69cd81b54814c3a13df4c7bd1f3ad7). Source paths start at this repository root (`pi/packages/...`). Use this version to match the workbook's references.

For running Pi, see its [upstream README](pi/README.md). Git-root-dependent development scripts and CI are not adapted to this teaching layout.

## Learning order

| Block | Topic | Order |
|---|---|---|
| 1 | Agent Loop & Lifecycle | Q1 → Q13 |
| 2 | State / Session | Q2 → Q3 → Q4 → Q6 → Q7 → Q14 |
| 3 | Messages / Model Context | Q5 → Q8 |
| 4 | Tool System | Q9 → Q10 → Q11 → Q12 |

Follow this order, not ascending Q numbers. Each of the **14 questions** includes source pointers and acceptance criteria.

## Self-study

Copy either question workbook to `../pi-learning-progress.md`, read the source, and answer in your own words. Then compare with the reference answer. Follow the workbook's **95% acceptance gate** and record each passed Q&A.

## Resume and finish

Reuse the prompt with the same checkout and private workbook path. Keep one progress file even when switching languages. Finish when all questions pass and have student/teacher answers plus evidence. Review personal content before publishing your workbook.

## Scope and attribution

Reference answers are AI-assisted; the pinned source is authoritative. No previous learner's private answers are included. Upstream source and its [MIT license](pi/LICENSE) are preserved unchanged.
