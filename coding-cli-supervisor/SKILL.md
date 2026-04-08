---
name: coding-cli-supervisor
description: Use when delegating a long-running coding task to Codex CLI, Claude Code, or Gemini CLI inside tmux and you need unattended execution with crash recovery and iterative review
---

# Coding CLI Supervisor

## Two Roles

| Role | Who | Where | Responsibility |
|------|-----|-------|----------------|
| **Coding CLI** (Worker) | Codex CLI, Claude Code, or Gemini CLI | Runs inside a tmux session | Writes code autonomously for extended periods |
| **Supervisor** (You) | OpenClaw | Runs outside tmux | Launches the worker, monitors progress, reviews changes, sends feedback |

The Coding CLI does the heavy lifting — it can run for hours writing code, running tests, and iterating. tmux keeps it alive regardless of what happens to the Supervisor's own process.

The Supervisor's job is a loop: **launch → monitor → wait for completion → review → send feedback → wait for fixes → review again → repeat until consensus**.

## When to Use

- The coding task will run for an extended period (minutes to hours)
- You want crash recovery and iterative review without manual babysitting
- The task is concrete enough to hand off with a clear goal and constraints

**Not for:** Quick one-shot prompts, tasks you can do directly, or non-coding CLI work.

## Core Rules

- One tmux session per task. Name descriptively: `codex-feature-auth`, `claude-refactor-payments`.
- Default to full-access, unattended mode unless the user explicitly restricts it.
- State clearly to the user that this skill defaults to high-permission execution.
- Never assume the Coding CLI is healthy — verify with pane output.
- Never skip the review loop just because the Coding CLI says it finished.
- Monitor every 10 minutes by default. Adjust based on task complexity.

## Workflow

### 1. Gather task contract

Confirm before launch: **target directory**, **CLI choice**, **coding goal**, **definition of done** (if available). Ask only for genuinely missing items. Do not ask about permission mode unless the user requested restrictions.

### 2. Choose CLI

If the user named a CLI, use it. Otherwise, pick based on task type:

| CLI | Best for |
|-----|----------|
| Claude Code | Architecture, planning, logic-heavy code, long reasoning chains |
| Codex CLI | Broad code generation, refactoring, code review, unit tests |
| Gemini CLI | Frontend/UI, visual design, e2e testing with browser-use |

### 3. Launch the Coding CLI in tmux

See `references/tmux-commands.md` for exact commands.

Create a tmux session, cd to the repo, start the CLI with its strongest available model + highest reasoning + full access, then paste the task prompt. Do not hardcode model names — detect the latest model before launching.

Example task prompt to send to the Coding CLI:

```
Implement <goal> in <repo path>.

Constraints:
- <any constraints>

Work autonomously unless you are blocked and need human input.
Summarize your progress every 10-15 minutes.
Run all relevant tests and linters before claiming completion.
Leave the repo in a clean, reviewable state (no debug prints, no commented-out code).
If you encounter repeated failures on the same issue, document what you tried and move on.
```

### 4. Monitor periodically

Check the tmux pane every **10 minutes** (adjust for task complexity). Each check answers:

- **Alive?** — Is the CLI process still running?
- **Progressing?** — Is it making forward progress?
- **Blocked?** — Is it stuck on an error or waiting for input?
- **Finished?** — Has it claimed completion?

Send the user a compact status update after each check.

### 5. Recover on failure

See `references/tmux-commands.md` for recovery patterns and resume prompt template.

Triggers: session disappeared, CLI exited, no output for extended period, repeated failures without progress. Restart the Coding CLI proactively — do not wait for the user.

### 6. Review loop

When the Coding CLI claims completion, the Supervisor reviews:

```
┌─────────────────────────────────────┐
│  Coding CLI claims "done"           │
└──────────────┬──────────────────────┘
               ▼
┌─────────────────────────────────────┐
│  Supervisor reviews diffs, tests,   │
│  and generated notes                │
└──────────────┬──────────────────────┘
               ▼
          ┌─────────┐
          │ Issues?  │
          └────┬─────┘
         yes   │   no
          ▼    │    ▼
┌──────────┐   │  ┌────────────────┐
│ Send     │   │  │ Consensus      │
│ feedback │   │  │ reached → Done │
│ into     │   │  └────────────────┘
│ tmux     │   │
└────┬─────┘   │
     │         │
     ▼         │
┌──────────┐   │
│ Coding   │   │
│ CLI fixes│   │
└────┬─────┘   │
     │         │
     └─────────┘ (back to review)
```

What to look for during review:
- Failing or missing tests
- Broken edge cases
- Unsafe or destructive operations
- Unnecessary complexity or scope creep
- Acceptance criteria not actually met (just claimed)
- Poor error handling

Send concrete, specific feedback into tmux. The Coding CLI may push back — that's fine. The goal is **consensus**, not compliance. If the Coding CLI makes a valid argument, accept it.

### 7. Finish

Report to the user: implementation status, what changed, test/validation results, remaining caveats, whether the tmux session is still available.

## References

- `references/tmux-commands.md` — concrete tmux commands for launching, monitoring, recovery, and sending review feedback
- [用tmux让AI Agent写几个小时的代码](https://mp.weixin.qq.com/s/ZQ0nc2NpXD76VLoan-QIFQ) — 刘小排's walkthrough of using tmux to supervise coding CLIs for long-running tasks with progress monitoring, crash recovery, and iterative code review
