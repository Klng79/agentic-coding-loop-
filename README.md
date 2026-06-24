<p align="center">
  <h1 align="center">🔄 Agentic Coding Loop</h1>
  <p align="center">
    <strong>Stop shipping broken code. Start looping until it's right.</strong>
  </p>
  <p align="center">
    <a href="#compatibility"><img src="https://img.shields.io/badge/AI_Agent-Universal-blue?style=flat-square" alt="AI Agent Universal" /></a>
    <a href="https://github.com/Klng79/agentic-coding-loop/blob/main/LICENSE"><img src="https://img.shields.io/badge/license-MIT-green?style=flat-square" alt="License" /></a>
    <a href="https://github.com/Klng79/agentic-coding-loop/stargazers"><img src="https://img.shields.io/github/stars/Klng79/agentic-coding-loop?style=flat-square" alt="Stars" /></a>
  </p>
</p>

---

## The Problem

LLM coding agents are overconfident. They write code, declare victory, and move on — leaving you to discover the bugs they introduced. No verification. No repair. No accountability.

**The agentic coding loop fixes this** by forcing a disciplined Baseline → Plan → Search → Modify → Verify → Review → Repair cycle before any task is marked done. It won't say "done" until the tests actually pass.

## What You Get

```
/agentic-coding-loop add a /health endpoint
```

One command. The skill handles everything:

| Step | What happens |
|------|-------------|
| **Scope Gate** | Classifies your task as Tactical, Medium, or Strategic — and escalates before wasting your time |
| **Loop Pattern** | Detects feedback signal type (Test-Driven, Compiler-Driven, Runtime Debugging, Review-Driven, Product Iteration) and adapts the observation lens |
| **Baseline** | Runs your verify command *before* touching a single line, establishing a known-good reference |
| **Plan** | Decomposes the task into concrete, ordered steps you can review |
| **Search** | Maps the codebase — what's editable, what's read-only, what's off-limits |
| **Modify** | Makes the smallest coherent edit. No drive-by refactors. No "improvements" you didn't ask for |
| **Verify** | Runs your tests/build/lint and scores the result quantitatively |
| **Review** | Self-critiques for correctness, security, performance, coverage, and minimalism |
| **Repair** | On failure: diagnoses root cause, fixes it, re-verifies. Up to 3 attempts |
| **Summarize** | Structured report with progress log, deltas, and remaining risks |

If verify passes → review → summarize as **DONE**.
If verify fails → repair → re-verify. Up to 3 attempts before reporting **BLOCKED**.

## Why It's Different

### 📊 Quantitative Progress, Not Vibes

Every verify run produces a score:

```
| Attempt | Score  | Delta | Notes                    |
|---------|--------|-------|--------------------------|
| 0       | 3/10   | —     | baseline: 3 tests failing |
| 1       | 7/10   | +4    | fixed import, added test  |
| 2       | 10/10  | +3    | all tests passing         |
```

You can **see** the progress. You can **see** when a repair made things worse. No more "I think I fixed it."

### 🔒 Real Rollback, Not Promises

The skill snapshots your codebase at every improvement using `git diff` patches. If a repair introduces a regression, it automatically rolls back to the best-known-good state. You never lose ground.

### 🛡️ Reward Hacking Detection

LLMs will sometimes "fix" tests by deleting them, widening assertions, or modifying CI config. The agentic coding loop checks for:

- Diffs that touch files outside the action space
- Verify commands that were weakened (fewer tests, broader assertions)
- Suspiciously large diffs unrelated to the task

If detected → stops with `UNSAFE` and escalates to you.

### 🎯 Scope Classification Before Work Starts

| Scope | Signal | Action |
|-------|--------|--------|
| **Tactical** | Single file, clear fix | Run the loop immediately |
| **Medium** | Multi-file, design choice | Plan first, confirm with you |
| **Strategic** | Architectural, multi-concern | Stop. Recommend a planning skill first |

No more watching an agent barrel into a 15-file refactor it wasn't equipped to handle.

### 🔁 Loop Pattern Detection

Not all tasks use the same feedback signal. The skill detects the dominant loop pattern from your verify command and task description, then adapts how it observes and diagnoses:

| Pattern | When it applies | Observation lens |
|---------|----------------|-----------------|
| **Test-Driven** | Bug fix, feature, regression | Pass/fail counts, assertion failures, tracebacks |
| **Compiler-Driven** | Migration, dependency upgrade, refactor | Type errors, missing fields, incompatible signatures |
| **Runtime Debugging** | Runtime bug, integration issue | Logs, stack traces, HTTP responses — hypothesis → change → observe |
| **Review-Driven** | Post-PR fix, address review comments | Comment text categorized as bug, style, product question, or out-of-scope |
| **Product Iteration** | UI/frontend task, layout, accessibility | Screenshots, DOM state, console errors, viewport behavior |

The 8 steps stay the same — the *observation lens* changes.

### 🧪 Anti-Pattern Guards

The skill actively detects and handles:

- **Thrashing** — cycling between failure modes
- **Overfitting** — tests pass but behavior is wrong
- **Context drift** — plan no longer matches the code
- **Speculative rewrites** — large changes without a verify path
- **Endless polishing** — revising after verify already passes
- **Unsafe autonomy** — destructive commands, unrelated rewrites, or unreviewed pushes

## Installation

```bash
# Global install (available in all projects)
cd ~/.qwen/skills
git clone https://github.com/Klng79/agentic-coding-loop.git agentic-coding-loop

# Per-project install
cd your-project/.qwen/skills
git clone https://github.com/Klng79/agentic-coding-loop.git agentic-coding-loop
```

Restart Qwen Code. Done.

## Usage

```bash
# Interactive — auto-detects tasks from git state, TODOs, recent commits
/agentic-coding-loop

# Direct — pass the task inline
/agentic-coding-loop add a /health endpoint
/agentic-coding-loop fix the failing test in auth.py
/agentic-coding-loop implement rate limiting on API routes
```

The skill auto-detects your project's verify command from `package.json`, `pyproject.toml`, `Makefile`, etc. You can also specify one manually.

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  Inputs                                             │
│  Task · Verify command · Timeout (optional)         │
└──────────────────────┬──────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────┐
│  Scope Gate                                         │
│  Tactical → proceed  │  Medium → confirm  │  Stop  │
└──────────────────────┬──────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────┐
│  Loop Pattern                                        │
│  Test-Driven · Compiler · Runtime · Review · Product │
└──────────────────────┬──────────────────────────────┘
                       ▼
  ┌─────────────────────────────────────────────────┐
  │  0. Baseline     run verify, snapshot state      │
  │  1. Plan         decompose into ordered steps    │
  │  2. Search       map files, tests, conventions   │
  │  3. Modify       smallest coherent edit          │
  │  4. Verify       score & judge result            │
  │  5. Review       self-critique (on pass)         │
  │  6. Repair       diagnose & fix (on fail, ×3)   │
  │  7. Summarize    structured final report         │
  └─────────────────────────────────────────────────┘
```

### Stopping Conditions

| Status | Meaning |
|--------|---------|
| ✅ `DONE` | Verify passes, review finds no critical issues |
| 🚫 `BLOCKED` | Same failure 3×, missing data, or out-of-scope repair |
| ⚠️ `OUT_OF_SCOPE` | Next repair would touch off-limits files |
| 🔴 `UNSAFE` | Destructive action required or reward hacking detected |
| ⏱️ `TIMEOUT` | Time budget exceeded |

## Built On Frontier Research

This isn't a naive retry loop. Every design decision traces back to published research on autonomous coding systems:

| Pattern | Source | What it does for you |
|---------|--------|---------------------|
| Baseline-first execution | **EPOCH protocol** | Never lose sight of where you started |
| Quantitative utility functions | **STOP (Self-Taught Optimizer)** | Measure real progress, not perceived progress |
| Solution archiving & rollback | **STOP / SePO** | Regressions are caught and reversed automatically |
| Fixed time budgets | **autoresearch (Karpathy)** | Optional time limits prevent infinite loops |
| Automated self-critique | **AI Scientist (Sakana AI)** | Review catches issues before you do |
| Reward hacking detection | **AlphaEvolve / AAR** | Prevents the agent from cheating its own tests |
| Action space boundaries | **Gödel Agent** | Keeps edits surgical and within declared bounds |

## Examples

<details>
<summary><strong>Tactical: Add a /health endpoint</strong> — single file, 1 attempt</summary>

```
Scope: Tactical (single file, clear spec)
Baseline: pytest → 0/1 passing (file not found)
Plan → Search → Modify → Verify: 0/1 (import error)
Repair: fix import → Verify: 1/1 passing ✅
Status: DONE (baseline 0/1 → final 1/1, 1 repair)
```

</details>

<details>
<summary><strong>Medium: Add rate limiting</strong> — multi-file, re-plan triggered</summary>

```
Scope: Medium (multi-file, design choice)
Baseline: npm test → 45/45 passing (green field)
Plan v1: create middleware → Search finds plugin architecture
Re-plan v1→v2: enable existing throttle plugin, configure, add tests
Modify → Verify → 45/45 ✅
Status: DONE (re-plan noted in report)
```

</details>

<details>
<summary><strong>Strategic: Refactor auth to JWT</strong> — escalated before work starts</summary>

```
Scope: Strategic (spans auth, sessions, tokens, DB, frontend)
Action: STOP. Recommend /grill-with-docs → /to-issues → loop per slice
```

</details>

<details>
<summary><strong>Blocked with rollback: Fix integration tests</strong></summary>

```
Baseline: pytest → 4/8 passing
Attempt 1: 6/8 passing (+2) — saved as best-known-good
Repair 1: changed conftest fixture → 5/8 (-1, regression)
  → automatic rollback to attempt-1 state (6/8)
Repair 2-3: fails, root cause needs fixture refactor (off-limits)
Status: BLOCKED (best state preserved at 6/8)
```

</details>

## Requirements

- **Any AI coding agent** — works with Qwen Code, Claude Code, Cursor, Cline, Aider, or any tool that can read and follow structured skill documents
- **Git** — snapshotting and rollback via `git diff` patches

## Compatibility

The skill is defined as a structured `SKILL.md` document that any AI coding agent can read and follow. While originally developed for Qwen Code, it works with any agent that supports:

- Reading markdown skill/instruction files
- Running shell commands (`git`, your verify command)
- File editing with surgical precision
- Tool-based workflows (if available)

**Tested with:** Qwen Code, Claude Code, Cursor, Cline

**Should work with:** Aider, Copilot, Windsurf, and other agentic coding tools

## Contributing

Issues and PRs welcome. This skill is designed to be forked and customized — the `SKILL.md` file is the single source of truth that your AI agent reads.

## License

MIT
