---
name: reflections
description: "Reflective memory consolidator. Use when: user asks for 'reflect', 'reflect now', 'reflect core/rem/deep', 'dream' (alias), 'consolidate memory', 'memory consolidation', 'show reflection config', 'set consolidation mode'."
---

# reflections — Manual Consolidation Skill

reflections is a reflective memory consolidator. Behaviorally equivalent to Reflections within the reference rubrics, profile thresholds, and verification suite — the control plane is prompt-based with no scripts. This skill covers operator-requested runs, checks, summaries, and targeted consolidation. Autonomous scheduling and setup are handled separately.

## When to Use

- Operator wants to run a reflection cycle now
- Operator wants a manual core/rem/deep pass
- Operator wants a status check or health summary
- Operator wants to see what changed in the last run
- Operator wants to inspect outputs after a run
- Operator wants to adjust consolidation modes or thresholds

## Manual Triggers

| Command | Action |
|---------|--------|
| "Consolidate memory" / "Reflect now" | Run full consolidation cycle (all due modes) |
| "Reflect core" / "Reflect rem" / "Reflect deep" | Run specific mode only |
| "Dream now" / "Dream core" / etc. | Backward-compatible aliases |
| "Show reflection config" | Display current `reflections.json` |
| "Set consolidation mode to core only" | Update `reflections.json` (confirm with user first) |

## Preconditions

Before manual execution:

1. Confirm workspace context — resolve paths dynamically, do not assume fixed install paths
2. Verify `reflections.json` exists with a selected profile
3. Verify required memory files are initialized (RTMEMORY.md, runtime/reflections-metadata.json, PROCEDURES.md)
4. If files are missing, point the operator to `INSTALL.md` — do not invent outputs

## Outputs

A manual run produces:

- **RTMEMORY.md** — updated long-horizon reflective memory
- **PROCEDURES.md** — updated reusable workflows
- **episodes/*.md** — updated project narratives (if applicable)
- **TRENDS.md** — updated recurring patterns (if applicable)
- **runtime/reflections-metadata.json** — updated index, stats, health history
- **memory/.reflections-log.md** — appended human-readable consolidation report
- **Telemetry** — one structured event appended to `$TELEMETRY_ROOT/memory-log-YYYY-MM-DD.jsonl`

## What the Agent Must Not Do

- Do not delete daily logs — only mark with `<!-- consolidated -->`
- Do not remove `⚠️ PERMANENT` entries
- Do not auto-install plugins or modify host config
- Do not assume hardcoded paths — resolve `SKILL_ROOT`, `WORKSPACE_ROOT`, `TELEMETRY_ROOT` dynamically
- Do not skip telemetry even when notification is silent
- Do not silently reconcile a runtime/reference rubric conflict — emit a blocker

## Language

All output uses the user's preferred language as recorded in USER.md.

## Boundaries

- This skill is for manual/operator-triggered use
- Scheduling and cron behavior are not owned by this file
- Setup and config internals are documented elsewhere
- Runtime orchestration lives in `runtime/reflections-prompt.md` and the references it cites

## See Also

- `INSTALL.md` — installation, configuration, profile selection, first-run bootstrap
- `README.md` — package overview, ownership boundary, memory-core coexistence
- `runtime/reflections-prompt.md` — the recurring 9-step cycle execution contract
- `runtime/first-reflections-prompt.md` — the one-time bootstrap
- `references/` — rubric authorities (scoring, durability, health, telemetry, etc.)
