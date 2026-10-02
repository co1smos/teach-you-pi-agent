# Teach You Pi Agent

[中文](README.zh-CN.md)

Learn Pi architecture by reading source and answering questions with an AI tutor. This is a community learning fork, not an official Pi course. No earlier course or private project is required.

## Install & Quickstart

Open a coding assistant with Git, network access, and permission to read/write local files and execute commands. Paste this prompt; it asks the assistant to download the source and begin teaching in the same session:

```text
Clone https://github.com/co1smos/teach-you-pi-agent.git into a new directory, or reuse an existing checkout without overwriting local work.
From the checkout root, read doc/questions.md and follow its teacher protocol. Create ../pi-learning-progress.md from it only if absent; otherwise resume that private workbook. Keep public files unchanged.
Tell me the checkout and progress paths, then immediately ask the first unfinished question in teaching order. Wait for my answer.
```

No Pi installation, npm dependencies, or Pi API keys are needed. The tutor needs its own configured model access and may ask you to approve file/command access. A chat assistant without these tools cannot perform the download. The teaching protocol already contains the 95% gate, hints, and post-PASS student/teacher answer recording, so the launch prompt does not repeat them.

Prefer to clone manually? Run the following, open your assistant in the new directory, and paste the same prompt (it can reuse this checkout):

```sh
git clone https://github.com/co1smos/teach-you-pi-agent.git
cd teach-you-pi-agent
```

Successful startup ends with **Block 1 / Q1**, awaiting your answer—not a setup summary or a reference solution. When resuming, it starts at the first unfinished question instead.

## Choose your documents

| Document | English | 中文 |
|---|---|---|
| Questions, teaching protocol, acceptance criteria, blank Q&A records | [questions.md](doc/questions.md) | [questions.zh-CN.md](doc/questions.zh-CN.md) |
| Reference answers and source evidence — spoilers | [answers.md](doc/answers.md) | [answers.zh-CN.md](doc/answers.zh-CN.md) |

Questions and reference answers are separate so you can investigate before seeing a solution. Your **personal working workbook** still keeps the Q&A format: after a question passes, record your accepted answer, the teacher's standard answer, and decisive source evidence together. The public template contains no previous learner's answers or progress.

## Fixed source baseline

The source is pinned to upstream commit [`a32782520f69cd81b54814c3a13df4c7bd1f3ad7`](https://github.com/earendil-works/pi/tree/a32782520f69cd81b54814c3a13df4c7bd1f3ad7). This fork’s `main` contains the complete pinned source tree under `pi/` and the learning materials outside it. Upstream history is retained; the fork HEAD is not the source baseline. Do not switch to moving upstream `main` and expect identical line numbers.

```sh
git diff a32782520f69cd81b54814c3a13df4c7bd1f3ad7 HEAD:pi
```

The last command should print no source-tree changes. Source reading requires no dependency installation, builds, API keys, or paid model calls. An optional AI tutor may have its own costs. The unchanged upstream [README](pi/README.md) is under `pi/`. Run upstream package commands from `pi/`, not this repository root. Its Git-root-dependent development scripts and CI are not adapted to this teaching layout; upstream workflows are preserved under `pi/.github/` and are not active root workflows.

## Learning order

| Block | Topic | Order |
|---|---|---|
| 1 | Agent Loop & Lifecycle | Q1 → Q13 |
| 2 | State / Session | Q2 → Q3 → Q4 → Q6 → Q7 → Q14 |
| 3 | Messages / Model Context | Q5 → Q8 |
| 4 | Tool System | Q9 → Q10 → Q11 → Q12 |

There are **14 questions**. IDs are stable references, not a numeric reading order. The original cross-project synthesis questions are excluded. Each question lists what a passing answer must explain and where to start reading.

## Self-study

1. Pick English or Chinese. Copy its question workbook to a private writable location outside this repository, for example `../pi-learning-progress.md`. Resolve source pointers from the teaching repository root (`pi/packages/...`), not the copy's directory.
2. Read the teacher protocol and the first question in teaching order. Leave the reference answer file closed.
3. Trace definitions and callers. Explain the mechanism, ownership, data flow, and edge cases in your own words, citing files and symbols. Read beyond a line range if necessary.
4. Compare your attempt with the source and then the matching reference answer. Revise until all requirements meet the **95% gate**. This is a qualitative correctness bar, not an automated numeric score.
5. After PASS, update the question status and its Q&A record in your private workbook. Preserve your answer; put the standard answer immediately below, followed by evidence. Continue in block order.

## Resume and finish

In a new assistant session, reuse the same prompt and private workbook path. Resume at the first non-PASS question in teaching order; do not restart completed questions. Choose **one** English or Chinese working copy, not two competing progress records. Student answers may be in either language; preserve the original wording and label optional translations.

Finish when every question has a PASS and a student answer, teacher answer, and evidence record. Reading all reference answers alone is not completion. A private workbook may contain personal details: review it deliberately before choosing to publish it.

## Scope and attribution

The material is adapted from a learner/AI-teacher source-reading workbook. Reference answers are AI-assisted explanations, not upstream specifications; verify disputed claims in the pinned source. The original learner's answers and local progress are not included. Source ownership and the upstream [MIT license](pi/LICENSE) remain unchanged. The learning commit reorganizes the repository without editing the pinned source contents.
