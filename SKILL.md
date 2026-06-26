---
name: agentic-coding-loop
description: Run a structured 8-step agentic coding loop (Baseline → Plan → Search → Modify → Verify → Review → Repair → Summarize) with scope classification, plan confirmation, re-plan on search invalidation, baseline tracking, quantitative progress monitoring, reward hacking detection, and up to 3 auto-repair attempts. Supports subagent mode (Qwen Code) and inline mode (universal). Use when user says "run the agentic loop", "loop engineer this", wants plan→act→verify→repair, or invokes this skill.
---

# Agentic Coding Loop

A self-correcting workflow for non-trivial coding tasks. Forces verify-and-repair before claiming done. Patterns from: EPOCH (baseline-first), STOP (quantitative tracking), autoresearch (time budgets), AI Scientist (self-critique), SePO (archive policy), AlphaEvolve/AAR (reward hacking), Gödel Agent (action space).

## Inputs (auto-suggest, then ask)

Use `ask_user_question` to get four things. Auto-detect candidates, present as options, user picks or types their own.

**Skip rules:** If task is in the slash command, skip to verify. If task+verify are specified, skip to timeout. If all four specified, skip entirely. Execution mode is never skipped silently — always ask if subagent capability detected.

### 1. Task

Run in parallel: `git status --porcelain` + `git diff --stat HEAD`, `grep_search` for `TODO\|FIXME\|XXX` (limit 5), `git log -n 3 --oneline`, grep README/PRD for "Roadmap"/"TODO" sections.

> Failing tests surface in Baseline (Step 0) after verify command is selected — avoids circular dependency.

Present 2–4 options: "Continue uncommitted work in `app.py`", "Fix failing test in `tests/test_health.py`", "Implement TODO at `src/auth.py:42`", "Refactor `app.py`". If empty project: "Set up from scratch", "Add a feature", "Fix a bug".

### 2. Verify command

Auto-detect by project config, first match wins:

| Project signal | Verify command |
|---|---|
| `pyproject.toml` `[tool.pytest]` / `pytest.ini` / `conftest.py` | `pytest` |
| `package.json` `scripts.test` | `npm test` |
| `package.json` `scripts.build` + `tsconfig.json` | `npm run build && npm test` |
| `Makefile` `test:` | `make test` |
| `tsconfig.json` | `tsc --noEmit` |
| `pyproject.toml` `[tool.ruff]` | `ruff check . && pytest` |
| `requirements.txt` (no test config) | `python -m py_compile $(git diff --name-only HEAD)` |
| Nothing detected (JS/TS) | `node -c $(git diff --name-only HEAD)` |

Present top 2–3 candidates labeled with what was detected. Confirm once, reuse across all Repair iterations.

### 3. Time budget — optional

Options: "No timeout", "15 min", "30 min", "1 hour". If set, check at each Verify checkpoint. If exceeded → `TIMEOUT`.

### 4. Execution mode

If Qwen Code (has `agent` tool) → default **Subagent mode**, ask user. Otherwise silently use **Inline mode**. Store as `MODE`. Inline is the universal fallback; subagent is a Qwen Code optimization.

## The Loop

**Verify is gating** — only proceed to Review on pass, then Summarize.

### Scope Gate (before any work)

Classify the task. Inspect description + `git diff --stat HEAD` + `grep_search` for blast radius.

| Scope | Signal | Action |
|---|---|---|
| **Tactical** | Single file, clear fix, test exists/trivial | Proceed directly to Baseline |
| **Medium** | Multi-file, design choice needed | Run Steps 0–1, **ask user to confirm plan**. 1 re-plan allowed if Search invalidates. |
| **Strategic** | Feature-level, architectural, ambiguous | **STOP.** Recommend `/spec-to-ship`, `/grill-with-docs`, or `/to-prd` → `/to-issues`. Return on a single issue. |

Present classification + reasoning. Offer: "Proceed", "Override — just run the loop", "Escalate — /spec-to-ship first". Log overrides in Summarize.

**Re-plan:** If Search discovers plan is wrong, return to Step 1. Max 1 re-plan; if second plan also invalidated → `OUT_OF_SCOPE` + recommend `/grill-with-docs`. Applies to *Search invalidating Plan* only — implementation constraints during Modify/Repair are normal repair work.

### Loop Pattern (after Scope Gate, before Baseline)

Detect dominant feedback signal. The 8 steps stay the same, but the *observation lens* changes.

| Pattern | Trigger | Observation lens |
|---|---|---|
| **Test-Driven** | Test runner (`pytest`, `npm test`, `go test`) | Pass/fail counts, assertion failures, tracebacks. Confirm test fails for the *right reason*. |
| **Compiler-Driven** | Type checker (`tsc`, `cargo check`, `mypy`) | Type errors, missing fields, incompatible signatures. Structural correctness only. |
| **Runtime Debugging** | Runtime command (`npm run dev`, `python app.py`, `curl`) | Logs, stack traces, HTTP responses, exit codes. Hypothesis → change → observe → update. |
| **Review-Driven** | PR review comments | Categorize each as bug, product question, style preference, out-of-scope. Treat as requirements. |
| **Product Iteration** | Browser/preview (`playwright`, `cypress`, manual) | Screenshots, DOM state, console errors, viewport behavior. Compare against design constraints. |

If verify spans multiple categories, pick the **primary task signal**. State in Baseline output: `## Loop Pattern: <name> — Observation lens: <one-line>`

### State Management

Rollback via `git diff` patches in a temp dir:

```bash
SNAP_DIR=$(mktemp -d /tmp/acl-snapshots-XXXXXX)
```

| Action | Command / When |
|--------|---------|
| Save snapshot | `git add -A && git diff --cached > $SNAP_DIR/<label>.patch && git reset -q` |
| Rollback to snapshot | `git checkout -- . && git clean -fd && git apply $SNAP_DIR/<label>.patch` |
| Rollback to clean | `git checkout -- . && git clean -fd` |
| Cleanup | `rm -rf $SNAP_DIR` (in Summarize) |
| **When to snapshot** | After Baseline → `baseline.patch`; After each Verify → `attempt-N.patch` **only if score improved** |
| **Best-known-good** | Highest-score attempt. If repair #N regresses, rollback to better patch before next fix. |

> `git add -A` stages all changes including untracked files, `git diff --cached` captures the full diff, `git reset -q` restores the index. Plain `git diff` misses untracked files.

> **External state:** `git checkout` only rolls back files — NOT DB migrations, caches, installed packages, or seed data. Log external mutations and warn in Summarize. Scope verify to avoid side-effects when possible.

Progress log format:
```
| Attempt | Score | Delta | Patch file       | Notes |
|---------|-------|-------|------------------|-------|
| 0       | 3/10  | —     | baseline.patch   | reference |
| 1       | 5/10  | +2    | attempt-1.patch  | fixed import |
| 2       | 4/10  | -1    | (not saved)      | regression → rollback to attempt-1 |
```

**Context broker (subagent mode):** Main agent is sole keeper of loop state. Subagents are stateless. Pass forward: Search Results → Modify briefing, Modify Results → Review briefing, Verify Results → Repair diagnosis, Review Findings → Summarize. Store in context or `todo_write`.

## Subagent Protocols (Subagent Mode Only)

Four steps delegate. Each subagent starts with zero context — main agent passes a complete briefing and receives structured markdown. Main agent acts as **context broker**.

### Search (`Explore` subagent)

**Briefing:** Task, project root, plan steps. Find: files, symbols, tests, conventions, patterns to respect, tests covering area, files not to touch, blockers. Thoroughness: medium.

**Return:** `## Search Results` → Relevant files (path — why), Patterns & conventions, Tests covering area, Action space (Editable/Read-only/Off-limits), Blockers (or "none").

Main agent: validate plan against action space + blockers → re-plan if invalidated.

### Modify (`general-purpose` subagent)

**Pre-condition:** Take git snapshot before spawning.

**Briefing:** Task, project root, plan steps, Search summary (pasted), specific edits per step, conventions (from Search), files to create (or "none"). Constraints: `edit` for surgical changes, `write_file` only for new files, match existing style, no drive-by refactors, don't touch off-limits.

**Return:** `## Modify Results` → Files changed (file — description), Edits applied (numbered), Issues encountered (or "none"), Files created (or "none").

Main agent: proceed to Verify.

### Verify (`general-purpose` subagent)

**Briefing:** Verify command, scoring formula (from Loop Pattern), loop pattern name, signals to parse.

**Return:** `## Verify Results` → Exit code, Status (PASS/FAIL), Score (e.g. "7/10"), Errors (file:line — message), Test results (pass/fail/skip), Warnings, Reward hacking flags (or "none").

Main agent: Review (pass) or Repair (fail), track progress, save snapshot if improved, check reward hacking → UNSAFE if detected.

### Review (`code-reviewer` subagent)

**Briefing:** Task, project root, changed files (from Modify Results), action space (from Search Results), verify status. Review for: Correctness, Security, Performance, Coverage parity, Minimalism.

**Return:** `## Review Findings` → per-dimension assessment (or "no issues"), Severity summary: critical | warning | clean.

Main agent: clean → DONE, warning → flag in Summarize or fix+re-verify, critical → `OUT_OF_SCOPE` if significant, or fix+re-verify.

## The 8 Steps

### 0. Baseline

Run verify command **before any edits**. Record exit code, output, timestamp. Take baseline snapshot:
```bash
SNAP_DIR=$(mktemp -d /tmp/acl-snapshots-XXXXXX)
git add -A && git diff --cached > $SNAP_DIR/baseline.patch && git reset -q
```
Reference point for all repairs. If baseline passes, note "green field". If failing tests appear, consider "fix N failing tests" as a task candidate.

Output: `## Baseline` — Loop pattern, Verify command, Exit code, Score, Status (PASS/FAIL).

### 1. Plan

Decompose into concrete ordered steps. Each step = one coherent edit OR one verification action. Plan should paste cleanly into a PR description. Use `todo_write` to track.

**Medium scope confirmation:** After plan, `ask_user_question`: "Proceed", "Revise — <instructions>", "Abort". Max 1 revision — further changes → recommend `/grill-with-docs`.

**Re-plan:** If Search reveals plan is wrong, return here. Max 1 re-plan; second invalidation → `OUT_OF_SCOPE` + `/grill-with-docs`.

### 2. Search

**Subagent:** Spawn `Explore` with Search Protocol briefing. Validate plan against return → re-plan if invalidated. Store results for Modify.

**Inline:** Use `grep_search`, `glob`, `read_file` for files, symbols, tests, conventions. Note patterns to respect, tests covering area, files not to touch.

**Action space:** Editable (will modify) / Read-only (context only) / Off-limits (CI config, safety tooling, unrelated modules).

> **Escape hatch:** If task explicitly targets test infrastructure ("fix fixtures", "refactor conftest"), those files move from Off-limits to Editable. Declare in action space output.

### 3. Modify

**Subagent:** Take snapshot. Spawn `general-purpose` with Modify Protocol briefing + Search Results. Wait for return. Proceed to Verify — don't inspect diff.

**Inline:** Smallest coherent edits. One diff per logical change. `edit` for surgical changes, `write_file` only for new files. Match existing style — no improving adjacent code, no new abstractions.

**Checkpoint (repair iterations):** If previous score regressed, rollback before applying:
```bash
git checkout -- . && git clean -fd && git apply $SNAP_DIR/attempt-<best>.patch
```

### 4. Verify

**Subagent:** Spawn `general-purpose` with Verify Protocol briefing. Use return to decide next step. Save snapshot if improved. Reward hacking flags → `UNSAFE`.

**Inline:** Run verify command via `run_shell_command`. Capture stdout, stderr, exit code.

**Parse into:** Errors (file:line — message), Test results (pass/fail/skip), Warnings, Score.

**Score by verify type:**
| Type | Formula | Example |
|---|---|---|
| Test runner | `passed / total` | 7/10 |
| Linter/formatter | `1 / (1 + error_count)` | 1/(1+3)=0.25 → 0.5 |
| Compiler/type checker | `1 / (1 + error_count)` | 1/(1+5)=0.17 |
| Coverage tool | `coverage_percentage` | 85% |
| Build command | Binary 1/0 | 0 → 1 |
| Custom script | `1 / (1 + meaningful_error_count)` | — |

**Judge:** Pass = exit 0 AND output matches expected. Fail = anything else.

**Reward hacking check:** Diff outside action space? Suspiciously large/unrelated diff? Weakened verify (fewer tests, broader assertions)? Modified test/CI files to pass? If detected → `UNSAFE`.

Track in progress log. After each Verify, save snapshot if score improved:
```bash
git add -A && git diff --cached > $SNAP_DIR/attempt-<N>.patch && git reset -q
```

### 5. Review (only on Verify Pass)

**Subagent:** Spawn `code-reviewer` with Review Protocol briefing. clean → DONE, warning → flag in Summarize or fix+re-verify, critical → `OUT_OF_SCOPE` if significant, or fix+re-verify.

**Inline:** Self-critique against:
- **Correctness:** Solves task or just passes tests? Edge cases uncovered?
- **Security:** Hardcoded secrets, open endpoints, injection? Handles boundary input?
- **Performance:** Debug artifacts, unused imports, algorithmic regressions?
- **Coverage parity:** New feature → matching test? Reduced coverage?
- **Minimalism:** Focused diff? Would pass human review?

If issues: fix small + re-verify, flag in Summarize, or `OUT_OF_SCOPE` if significant.

### 6. Repair (only on Verify Fail)

Max **3** attempts. Context refresh: if verify reveals uncovered files/modules, re-run targeted search (subagent: spawn `Explore`; inline: `grep_search`/`read_file`). Diagnose root cause (don't patch symptoms) — diagnosis stays with main agent.

**Apply fix:**
- **Subagent:** Rollback to best-known-good if regressed. Spawn `general-purpose` with fix (Modify Protocol). Spawn Verify subagent. Check result.
- **Inline:** Smallest fix via `edit`/`write_file`. Re-run Verify directly.

**Sandbox:** Never modify CI config or safety tooling. If fix requires files outside action space (unless escape hatch) → `OUT_OF_SCOPE`.

**Human checkpoint after attempt 2 fails** — `ask_user_question` with summary (attempt 1+2 results, root cause, best-known-good). Options: "Try different approach — <instructions>", "Stop — I'll take over" (→ BLOCKED), "One more attempt — specific fix" (→ attempt 3). If 3 attempts fail → BLOCKED.

**Solution archive:** Before Summarizing (especially BLOCKED), rollback to best-known-good:
```bash
git checkout -- . && git clean -fd && git apply $SNAP_DIR/attempt-<best>.patch
```
If no attempt improved over baseline → `git checkout -- . && git clean -fd`.

### 7. Summarize

Stop early on any condition:

| Status | Trigger |
|---|---|
| **DONE** | Verify passes AND review clean |
| **BLOCKED** | 3× failures, OR user stopped, OR missing credentials/data, OR out-of-scope repair |
| **OUT_OF_SCOPE** | Next repair touches unrelated/off-limits files (unless escape hatch) |
| **UNSAFE** | Destructive action (force-push, drop tables, `rm -rf`) OR reward hacking — escalate, don't execute |
| **TIMEOUT** | Time budget exceeded |

Format:
```
## Summary
**Status:** DONE | BLOCKED | OUT_OF_SCOPE | UNSAFE | TIMEOUT
**Task:** <one-line>  |  **Verify:** `<cmd>`  |  **Time:** <duration>
**Verify result:** PASS (attempts: N) | FAIL (attempts: 3/3)

### Baseline: Score <metric>, Status PASS|FAIL
### Progress: | Attempt | Result | Score | Delta | Patch | Notes |
### State: Final patch — <attempt-N.patch | rolled back to baseline | attempt-K.patch (best)>
### Changes: `<file>`: <description>
### Review notes: <observations, edge cases, risks>
### Remaining risks: <uncertainties that passed verify>
```

**Cleanup:** `rm -rf $SNAP_DIR`

## Anti-Patterns

| Pattern | Signal | Action |
|---|---|---|
| **Thrashing** | Cycling between two failure modes | Narrow diff, human checkpoint before attempt 3 |
| **Overfitting to tests** | Tests pass but behavior wrong | Re-read task, verify from different angle |
| **Context drift** | Plan no longer matches code | Re-run Search, update plan |
| **Speculative rewrites** | Large changes without small-step verify | Split work, loop on each piece |
| **Endless polishing** | Continuing after verify+review clean | Stop, ship it |
| **Unsafe autonomy** | Destructive commands, unrelated rewrites, push without review | Enforce action space, require human approval, use UNSAFE |

## Examples

### Subagent mode flow

When MODE = subagent, steps 2–5 delegate. Main agent: classifies (Scope Gate) → runs verify (Baseline) → decomposes + confirms (Plan) → spawns `Explore` (Search) → snapshots + spawns `general-purpose` (Modify) → spawns `general-purpose` (Verify) → diagnoses + spawns fix + Verify (Repair) → spawns `code-reviewer` (Review) → owns report (Summarize). Context stays clean: only structured summaries, not raw results/diffs/output.

### Tactical

*"Add a `/health` endpoint returning 200 `{ok: true}`. Verify: `pytest tests/test_health.py`."*

1. **Scope Gate:** Tactical (single file, clear spec, test given)
2. **Baseline:** 0/1 (file not found) → 3. **Plan:** add handler, add test, verify → 4. **Search:** editable=[`app.py`, `tests/test_health.py`], off-limits=[`pytest.ini`] → 5. **Modify:** add handler + test → 6. **Verify:** 0/1 (import error) → 7. **Repair:** fix import → 1/1 (+1) → 8. **Review:** minimal, covers happy path → 9. **Summarize:** DONE, 0/1 → 1/1

### Medium (re-plan)

*"Add rate limiting to all API routes. Verify: `npm test`."*

1. **Scope Gate:** Medium (multi-file, design choice) → 2. **Baseline:** 45/45 PASS (green field) → 3. **Plan v1:** create `middleware/rateLimit.ts`, apply globally, add config, tests → 4. **Search:** discovers plugin architecture, existing disabled `plugins/throttle.ts` → **Re-plan** → 5. **Plan v2:** enable throttle plugin, configure limits, add tests → **User confirms** → 6–9. Standard loop → **Summarize:** DONE, note re-plan

### Strategic (escalated)

*"Refactor auth from sessions to JWT with refresh tokens."*

**Scope Gate:** Strategic (spans middleware, session store, token management, DB schema, routes, frontend) → **STOP.** Recommend `/grill-with-docs` for token strategy → `/to-issues` for vertical slices → return to `/agentic-coding-loop` per slice.

### BLOCKED with rollback

*"Fix failing integration tests. Verify: `pytest tests/integration/`."*

1. **Scope Gate:** Tactical → 2. **Baseline:** 4/8, save `baseline.patch` → 3–4. **Modify+Verify attempt 1:** 6/8 (+2), save `attempt-1.patch` → 5. **Repair attempt 2:** change fixture in `conftest.py` (off-limits) → 6/8 → **Verify:** 5/8 (−1, regression) → 7. **Rollback** to `attempt-1.patch` (6/8) → 8. **Human checkpoint:** root cause — remaining tests need fixture refactor (off-limits). User: "Stop" → 9. **Summarize:** BLOCKED, `attempt-1.patch` (6/8), recommend `/grill-with-docs`

## References

EPOCH (baseline-first, round tracking) · STOP (quantitative utility, archiving, budgets) · autoresearch (time budgets, metric iteration) · AI Scientist (automated review, self-critique) · SePO (regression avoidance, archive admission) · AlphaEvolve/AAR (reward hacking, evaluation integrity) · Gödel Agent (action space boundaries).