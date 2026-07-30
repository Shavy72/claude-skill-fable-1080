---
name: fable-1080
description: >-
  Mandatory protocol when the session model is a frontier model (e.g. Claude Fable 5) and a dev task comes in (implement, build, fix, refactor, feature, migration). The orchestrator works only at the front (plan/architecture) and at the back (review/verification) — the execution in the middle is done by the subagents executor-opus (complex) and executor-sonnet (routine). Triggers: any implementation request on a frontier session model, "10-80-10", "/fable-1080", "build token-efficiently". NOT for pure lookups, questions, single-file mini edits (<10 lines) or planning discussions without implementation.
---

# fable-1080 — the frontier model plans, others implement, the frontier model verifies

> **Calibrated for:** Fable 5 · Opus 5 · Sonnet 5 · Claude Code ≥ 2.1.219 · as of 2026-07-30 — on a new model/Claude Code release: run the update-drift protocol (below).

Goal: frontier-model quality at a fraction of the frontier-model tokens. Frontier tokens flow only into the two phases where frontier intelligence pays off (architecture + review). The execution — where 80 % of the tokens burn — runs on Opus 5/Sonnet 5. Opus 5 delivers frontier intelligence at half the frontier price and is strong even at low effort tiers — so the delegation threshold is LOW: when in doubt, delegate.

## Phase 1 — PLAN (orchestrator, you, ~10 %)

1. Classify the task: complex (multi-file, logic, debugging) → `executor-opus`. Routine (boilerplate, tests, mechanical edits) → `executor-sonnet`. Mini edit under ~10 lines in 1 file → do it yourself, no overhead.
2. Read only as much context as the plan needs (Grep/Glob first, Read with offset/limit). Always spawn recon subagents (Explore etc.) with an explicit `model: "sonnet"` — since Claude Code 2.1.199, Explore inherits the session model (capped at Opus) instead of running cheaply.
2b. Apply the Lazy Ladder to the plan BEFORE commissioning new code: does this need to exist (YAGNI)? Does it already exist in the codebase? Stdlib/native platform feature/already-installed dependency instead of building it? The result feeds into CONTRACT/OUT-OF-SCOPE of the handoff.
3. **Write the handoff** (definition of ready) — without a complete handoff you do NOT delegate:

```
GOAL: <what works at the end, 1-2 sentences>
WHY: <intent/context — prevents wrong directions>
FILES: <affected paths + what happens there>
CONTRACT: <signatures, data models, names that are fixed>
ACCEPTANCE: <checkable criteria, including which test/command must be green>
OUT-OF-SCOPE: <what is explicitly NOT touched>
```

For `executor-sonnet` (routine) the short form is enough: GOAL + FILES + ACCEPTANCE — WHY/CONTRACT/OUT-OF-SCOPE only when truly needed.

## Phase 2 — EXECUTE (subagent, ~80 %)

- Delegate via the Agent tool with `subagent_type: "executor-opus"` or `"executor-sonnet"`. **NEVER spawn an executor without an explicit model assignment** — an agent without a model override inherits the frontier session and burns the limit. Executors no longer inherit effort from the session: it is fixed in the agent frontmatter (`executor-opus: medium`, `executor-sonnet: low`).
- **Keep the subagent alive:** send fixes and follow-up work to the same agent via SendMessage — it resumes with full history instead of a cold start (subagent cache: 5-min TTL, so send follow-ups promptly). Spawn a new one only for a new, independent work package.
- **Fork instead of a named agent** when a side task needs the full session context (e.g. analysis over what has already been read): `subagent_type: "fork"` (fallback: `/subtask`) inherits both the main session's conversation AND prompt cache — no cold start. Still runs on the frontier model though: only for context-heavy read/analysis tasks, never for bulk execution.
- **Workflow tool from ~3 independent increments on** (only if orchestration has been approved): keep intermediate results in the script instead of the orchestrator's context; set `model`/`effort` explicitly per stage in the script — otherwise every stage runs on the frontier model.
- Split large tasks into bounded increments (1 increment = 1 handoff = 1 checkable result); do not dump "the whole feature" into one agent.
- Independent, parallelizable increments: start several executors at once (one message block, multiple Agent calls). Be aware: parallelizing saves wall-clock time, NOT tokens — all subagents draw from the same limit pool. Size the count to the task, never fan out wider "just in case".
- While an executor is running: do NOT read the same files in parallel or duplicate the work.

## Phase 3 — REVIEW (orchestrator, you, ~10 %)

0. **Opus pre-review for large diffs** (≥ ~200 changed lines or ≥ 5 files): spawn a read-only review agent (`subagent_type: "ecc:code-reviewer"`, `model: "opus"` — NOT executor-opus, whose contract is implementation; if the ECC plugin is disabled: general-purpose with `model: "opus"` and a read-only mandate) with a bounded mandate: "only real bugs in the diff, 1 line per finding, no style comments." The orchestrator then checks findings + acceptance instead of the raw diff. Small diffs: go straight to step 1.
1. Review from the diff (`git diff`), do not re-read whole files.
2. Review against the ACCEPTANCE criteria of the handoff — not against taste. No post-hoc refactoring for stylistic reasons.
2b. **Complexity pass** (ponytail review format, 1 line per finding): `<file>:L<n>: <tag> <what> → <replacement>` with tags `delete:` (dead/speculative) · `stdlib:` (hand-rolled what stdlib does) · `native:` (dependency for a platform feature) · `yagni:` (abstraction with a single implementation) · `shrink:` (same logic, fewer lines). Close with `net: -N lines possible` or `Lean already. Ship.` Never flag safety-floor code (validation, error handling, the one mini check) as bloat.
3. Check the executor's verification evidence; when in doubt, run the decisive check yourself (test, smoke, curl).
4. Small review findings: hand them back as a follow-up task via SendMessage to the living executor. Step in yourself only for architectural errors.
5. Then the project duties: commit/push or deploy according to project doctrine.

## Token hygiene (applies in all phases)

- **Effort discipline (orchestrator):** medium is the working mode; high only for architecture decisions and stuck debugging; max never as a default. The user sets effort via `/effort` — on an obvious mismatch, suggest it briefly. **Do NOT switch effort in the main session mid-task:** the prompt cache is keyed on model + effort — every switch is a full cache miss on the entire history.
- **Cache reality:** the main session caches with a 1-hour TTL (subscription), named subagents always only 5 minutes, forks inherit the session cache. Bundle follow-up work to living executors promptly for that reason.
- Executor effort lives in the agent frontmatter (opus: medium, sonnet: low) — don't ask for "think harder" per task in the prompt, that does not change the effort.
- Outcome-first, answer tersely; never request or deliver reasoning narratives. Do not stack verification extras into handoffs ("double-check", "verify with a subagent") — Opus 5 verifies on its own, such instructions cause over-verification.
- Act, don't plan: when there is enough information to act, act — no option surveys.
- No re-reads of files already read; summarize tool outputs instead of passing them through.
- For large multi-increment tasks: put the handoff/plan state in a file (HANDOFF.md) — auto-compaction summarizes old history, files survive that losslessly. Same at session end mid-work.

## Anti-patterns (forbidden)

- The orchestrator writes boilerplate, tests or mass edits itself.
- Executor spawn without a handoff ("just do feature X") — underspecified tasks are the second biggest token waste after blind exploration.
- Several frontier-model instances in parallel (Agent calls without model/subagent_type on a frontier session).
- Review as a complete re-read of the repo instead of a diff.
- Stacking verification instructions into handoffs ("double-check", "verify subagent") — over-verification on Opus 5.
- Switching effort in the main session mid-task (cache miss on the entire history).

## 🔄 Update-drift protocol (a fixed part of this skill)

Models and Claude Code change — this skill is a calibration to a point in time, not a law of nature. **Trigger:** a new model release (Opus/Sonnet/Fable), a Claude Code major update, OR the symptom "token usage/behavior suddenly different from usual".

1. **Docs sweep** (Firecrawl/WebFetch): read `platform.claude.com/docs/…/whats-new-<model>` + `prompting-claude-<model>` + `code.claude.com/docs/en/model-config.md` + `sub-agents.md` + `prompt-caching.md`. Focus: effort defaults, thinking behavior, subagent/cache mechanics, model alias mappings (`opus:`/`sonnet:` → which version?), pricing relations (does the frontier↔Opus boundary still pay off?), new delegation features.
2. **Reconcile against this file + executor-opus.md + executor-sonnet.md:** which assumption in the calibration stamp no longer holds? Fix every outdated rule, don't invent prophylactic ones.
3. **Effort sweep:** check the executor frontmatter effort against the new recommendations (never carry old defaults forward unchecked — Anthropic changes effort semantics between generations). While at it, verify that `effort:` frontmatter still takes effect in the current harness: run an identical mini task as an A/B (executor vs. general-purpose at the same model tier), compare the token delta from the task notifications — reference value 2026-07-30: −63 % (33k vs. 90k).
4. **Update the calibration stamp above** (models, Claude Code version, date) and update this repo accordingly.
