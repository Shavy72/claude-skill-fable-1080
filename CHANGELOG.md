# Changelog

All notable changes to this project are documented in this file.

## 1.1.0 — 2026-07-30

- Recalibrated for the Claude 5 family (Opus 5/Sonnet 5).
- Fixed an effort-inheritance leak via explicit `effort:` frontmatter on the executors (`executor-opus: medium`, `executor-sonnet: low`).
- Hard subagent damping via a `tools:` allowlist on both executors.
- Removed over-verification instructions from the protocol.
- Slimmed the executor report format down to `VERIFICATION:` + `DEVIATIONS:`.
- Documented cache rules: an effort switch is a cache miss, named subagents have a 5-min TTL, forks inherit the session cache.
- Added an Opus pre-review step for large diffs (≥ ~200 lines or ≥ 5 files).
- Added the update-drift protocol for keeping the calibration current across model releases.
- Measured result: −63 % execution tokens (33k vs. 90k in an A/B test); a follow-up via resume costs ≈ 3.5k tokens.

## [1.0.0] - 2026-07-24

Initial public release.

- `fable-1080` skill: 10-80-10 delegation protocol (plan → execute → review) for Claude Code.
- `executor-opus` and `executor-sonnet` subagents for the execute phase.
- Claude Code plugin manifest (`.claude-plugin/plugin.json`) and single-plugin marketplace (`.claude-plugin/marketplace.json`).
- Installable via `npx skills add`, `/plugin marketplace add`, or manual `git clone`.
