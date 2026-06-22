---
name: agentic-coding-loop
description: Run a structured 8-step agentic coding loop (Baseline → Plan → Search → Modify → Verify → Review → Repair → Summarize) on a task in the current project, with baseline tracking, quantitative progress monitoring, reward hacking detection, and up to 3 auto-repair attempts. Use when the user asks to "run the agentic loop", "loop engineer this", wants plan→act→verify→repair on a coding task, or invokes this skill directly.
---

# Agentic Coding Loop

A self-correcting workflow for non-trivial coding tasks. The first answer is rarely the final answer — this loop forces verify-and-repair before claiming done.

Incorporates patterns from frontier autonomous coding systems: baseline-first execution (EPOCH), quantitative progress tracking (STOP/autoresearch), reward hacking detection (AlphaEvolve/AAR findings), self-critique review (AI Scientist), and solution archiving (STOP/SePO).

## Inputs (auto-suggest, then ask)

Before starting, use `ask_user_question` to get three things from the user. Auto-detect candidates, present them as options, and let the user pick one or type their own — the tool always provides an **"Other"** option for custom input.

**Skip rules:**
- If the user already specified the task in their slash command (e.g. `/agentic-coding-loop add a /health endpoint`), skip the task question and go straight to verify.
- If both task and verify are already in the slash command, skip to the timeout question.
- If all three are already specified, skip the Inputs section entirely.

### 1. Task — auto-suggest candidates, then ask

Run these in parallel to find candidates:

| Signal | Command |
|---|---|
| Uncommitted changes | `git status --porcelain` and `git diff --stat HEAD` |
| Open TODOs / FIXMEs | `grep -rn 'TODO\|FIXME\|XXX' --include='*.py' --include='*.js' --include='*.ts' .` (limit 5) |
| Failing tests | run the detected verify command (Inputs §2) and surface failures |
| Recent commits | `git log -n 3 --oneline` |
| Project roadmap | grep README/PRD for "Next steps", "Roadmap", "TODO" sections |

Present 2–4 task options, e.g.:

- "Continue uncommitted work in `app.py`, `graph.py`"
- "Fix the failing test in `tests/test_health.py`"
- "Implement the open TODO at `src/auth.py:42`"
- "Refactor `app.py` (last touched 2 days ago)"

If the project is empty or has no detectable signals, present generic options:

- "Set up the project from scratch"
- "Add a new feature (specify below)"
- "Fix a bug (specify below)"

### 2. Verify command — auto-suggest, then ask

Auto-detect by inspecting project config. Use the first match in this order:

| Project signal | Proposed verify command |
|---|---|
| `pyproject.toml` with `[tool.pytest]`, or `pytest.ini`, or `conftest.py` | `pytest` |
| `package.json` with `scripts.test` | `npm test` |
| `package.json` with `scripts.build` and a `tsconfig.json` | `npm run build && npm test` |
| `Makefile` with `test:` target | `make test` |
| `tsconfig.json` | `tsc --noEmit` |
| `pyproject.toml` with `[tool.ruff]` | `ruff check . && pytest` |
| `requirements.txt` (no test config) | `python -m py_compile $(git diff --name-only HEAD)` |
| Nothing detected (JS/TS) | `node -c $(git diff --name-only HEAD)` |

Present the **top 2–3 candidates** as options, each labeled with what was detected, e.g.:

- `pytest` *(detected: pyproject.toml has [tool.pytest])*
- `ruff check . && pytest` *(detected: pyproject.toml has [tool.ruff] + pytest)*
- `python -m py_compile $(git diff --name-only HEAD)` *(minimal syntax-only)*

Confirm once, then reuse the same command across all Repair iterations — do not re-ask.

### 3. Time budget — optional

Ask if the user wants to set a time limit for the entire loop. Present options:

- "No timeout (run until done or blocked)"
- "15 minutes"
- "30 minutes"
- "1 hour"

If a timeout is set, track elapsed time and check at each Verify checkpoint. If exceeded, summarize with status `TIMEOUT`.

## The Loop

Run the 8 steps in order. **Verify is gating** — only proceed to Review on pass, then to Summarize.

### 0. Baseline
Run the verify command **before any edits**. Record:
- Exit code
- Output (test count, error count, coverage %, or other measurable signal)
- Timestamp

This is your reference point. All subsequent repairs must improve relative to this baseline. If baseline already passes, note it and proceed to Plan with a "green field" status.

**Output:**
```
## Baseline
**Verify command:** `<command>`
**Exit code:** <code>
**Score:** <metric> (e.g., "3/10 tests passing", "0 errors", "85% coverage")
**Status:** PASS | FAIL
```

### 1. Plan
Decompose the task into concrete, ordered steps. Output a numbered list. Each step = one coherent edit OR one verification action. Do not start editing until the plan could paste cleanly into a PR description.

### 2. Search
Find files, symbols, tests, and conventions relevant to the plan. Use `grep_search`, `glob`, `read_file`. Note:
- Existing patterns the change must respect
- Tests that cover the area
- Files that should NOT be touched

**Define the action space** — explicitly classify files into three categories:
- **Editable:** files you will modify
- **Read-only:** files you need for context but won't touch
- **Off-limits:** files that must not be modified (test infrastructure, CI config, safety tooling, unrelated modules)

Output:
```
## Action Space
**Editable:** <list>
**Read-only:** <list>
**Off-limits:** <list>
```

### 3. Modify
Make the smallest coherent set of edits. One diff per logical change. Use `edit` for surgical changes, `write_file` only for new files. Match existing style — do not "improve" adjacent code.

**Checkpoint:** If this is a repair iteration and the previous repair made things worse, rollback to the best-known-good state before applying the new fix.

### 4. Verify
Run the user-supplied command via `run_shell_command`. Capture stdout, stderr, exit code.

**Parse the output into structured feedback:**
- **Errors:** list of error messages with file:line references
- **Test results:** pass/fail/skip counts
- **Warnings:** non-fatal issues
- **Score:** quantitative metric (e.g., "7/10 tests passing", "2 errors remaining")

**Judge:**
- **Pass** — exit 0 AND output matches expected behavior
- **Fail** — anything else

**Check for reward hacking:**
- Did the diff touch files outside the action space?
- Did verify pass but the diff is suspiciously large or unrelated to the task?
- Did the repair weaken the verify command (fewer tests, broader assertions, disabled checks)?
- Did the repair modify test files or CI config to make tests pass?

If reward hacking is detected, flag it and stop with status `UNSAFE`.

**Track progress:**
```
## Progress Log
| Attempt | Result | Score | Delta | Notes |
|---------|--------|-------|-------|-------|
| Baseline | FAIL | 3/10 | — | reference |
| 1 | FAIL | 5/10 | +2 | fixed import error |
| 2 | PASS | 10/10 | +7 | all tests passing |
```

### 5. Review (only on Verify Pass)
Self-critique the solution. Re-read the diff and ask:
- Does the solution actually solve the task, or just make tests pass?
- Is the diff minimal and focused, or does it include unrelated changes?
- Are there any edge cases or failure modes not covered by tests?
- Would this change pass a human code review?

If the review finds issues, either:
- Fix them immediately (if small) and re-verify
- Flag them in the Summarize as "Remaining risks"
- Stop with status `OUT_OF_SCOPE` if the issues require significant additional work

### 6. Repair (only on Verify Fail)
- Attempts so far: 0 → max **3**
- Diagnose the root cause from structured verify output (don't patch symptoms)
- Make the smallest fix that addresses the cause
- **Sandbox rules:** Repair must never modify test files, CI config, or safety tooling. If the fix requires changing these, stop with status `OUT_OF_SCOPE`.
- Re-run Verify
- If 3 attempts all fail → jump to Summarize with status `BLOCKED`

**Solution archive:** Keep the best-known-good state (the attempt with the highest score, even if it didn't fully pass). If repair #3 makes things worse than repair #1, rollback to repair #1's state before summarizing.

### 7. Summarize
Final report. Use this exact format:

```
## Summary

**Status:** DONE | BLOCKED | OUT_OF_SCOPE | UNSAFE | TIMEOUT
**Task:** <one-line restatement>
**Verify command:** `<the command>`
**Time elapsed:** <duration>
**Verify result:** PASS (attempts: N) | FAIL (attempts: 3/3)

### Baseline
- **Score:** <baseline metric>
- **Status:** PASS | FAIL

### Progress
| Attempt | Result | Score | Delta | Notes |
|---------|--------|-------|-------|-------|
| Baseline | <status> | <score> | — | reference |
| 1 | <status> | <score> | <delta> | <notes> |
| ... | ... | ... | ... | ... |

### Changes
- `<file>`: <one-line description>
- ...

### Review notes
- <self-critique observations>
- <edge cases or risks identified>

### Remaining risks
- <anything that passed verify but is still uncertain>
```

## Progress Tracking

Maintain a running log of all verify attempts. This enables:
- **Delta tracking:** see if each repair actually improves the score
- **Regression detection:** if score drops, rollback to best-known-good
- **Reward hacking detection:** if score improves but diff is suspicious, flag it

Update the progress log after every Verify call.

## Stopping Rules

Stop early and summarize with the matching status if:

- **DONE** — verify passes AND review finds no critical issues
- **BLOCKED** — same failure 3× in a row, OR missing credentials/data, OR repair requires out-of-scope changes
- **OUT_OF_SCOPE** — next repair would touch unrelated files or off-limits files (test infrastructure, CI config, safety tooling)
- **UNSAFE** — required action is destructive (force-push, dropping tables, `rm -rf`, etc.) OR reward hacking detected — escalate to user, do not execute
- **TIMEOUT** — time budget exceeded (if set by user)

## Anti-Patterns

- **Thrashing** — cycling between two failure modes → narrow the diff, ask the user
- **Overfitting to tests** — tests pass but user-visible behavior is wrong → re-read the task, verify with a different angle (browser check, manual test, different test)
- **Context drift** — plan no longer matches the code → re-run Search, update the plan
- **Speculative rewrites** — large multi-file changes without a small-step verify path → split the work, run the loop on each piece
- **Reward hacking** — agent finds unintended ways to maximize score without fulfilling the true objective (e.g., disabling tests, crashing the scorer, weakening assertions) → flag and stop with status `UNSAFE`
- **Evaluator drift** — the verify command itself degrades over iterations (tests get weaker, assertions get broader) → compare current verify output to baseline, flag if verify is easier now
- **Endless polishing** — continuing to revise after verify passes and review is clean → stop, ship it
- **Sandbox bypass** — repair attempts to disable safety flags or modify test infrastructure "for efficiency" → stop with status `UNSAFE`

## Example

User: *"Add a `/health` endpoint that returns 200 with `{ok: true}`. Verify with `pytest tests/test_health.py`."*

1. **Baseline:** run pytest — fails (0/1 tests, file not found). Score: 0/1.
2. **Plan:** (1) add route handler, (2) add test, (3) verify
3. **Search:** read existing route patterns, find test fixtures. Action space: editable=[`app.py`, `tests/test_health.py`], read-only=[`conftest.py`], off-limits=[`pytest.ini`, other test files]
4. **Modify:** add the handler, add the test
5. **Verify:** run pytest — fails (0/1 tests, import error). Score: 0/1. Delta: 0.
6. **Repair:** fix the import, re-run pytest — passes (1/1 tests). Score: 1/1. Delta: +1.
7. **Review:** re-read diff — handler is minimal, test covers the happy path, no edge cases. Diff is clean.
8. **Summarize:** status DONE, baseline 0/1, final 1/1, files changed, review notes

## References

This skill incorporates patterns from:
- **EPOCH protocol** — baseline-first execution, round-level tracking
- **STOP (Self-Taught Optimizer)** — quantitative utility functions, solution archiving, budget constraints
- **autoresearch (Karpathy)** — fixed time budgets, metric-based iteration
- **AI Scientist (Sakana AI)** — automated review, self-critique feedback loops
- **SePO** — regression-avoidance, archive admission policy
- **AlphaEvolve/AAR findings** — reward hacking detection, evaluation integrity
- **Gödel Agent** — action space boundaries, self-referential improvement
