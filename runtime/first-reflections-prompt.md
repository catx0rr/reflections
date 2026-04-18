# reflections — First Reflection (Initial Memory Scan)

One-time bootstrap. Read USER.md first. All output in user's language.

## Precedence

1. Runtime prompts define execution order.
2. Reference docs define formulas / routing law.
3. Worked examples are illustrative only.
4. On runtime/reference conflict → stop, emit blocker.

## Paths

```
SKILL_ROOT     = parent of runtime/
WORKSPACE_ROOT = current working directory
CONFIG_PATH    = $REFLECTIONS_CONFIG | ~/.openclaw/reflections/reflections.json
TELEMETRY_ROOT = $REFLECTIONS_TELEMETRY_ROOT | $MEMORY_TELEMETRY_ROOT | ~/.openclaw/telemetry
```

## Gate bypass

All quality gates bypassed. Every extracted entry consolidates. Set `promotedBy: "first-reflection"` on each new index entry. Read `$CONFIG_PATH` — missing → blocker.

## Guardrails

- Execute every phase in the current agent. No sub-agent.
- No chat narration. Internal details → telemetry / log files only.
- Chat emits exactly one of: Phase 4 report, Phase 5 fresh-instance report, blocker, or nothing.
- `.bak runtime/reflections-metadata.json` before mutation.
- `.bak RTMEMORY.md` if existing content present (this run can change > 30%).
- Never delete daily logs. Never remove `⚠️ PERMANENT` entries.

---

## Phase 1 — Snapshot BEFORE

Count and hold in memory:

```
RTMEMORY_LINES        = wc -l RTMEMORY.md (0 if missing)
RTMEMORY_SECTIONS     = count of "^## " (0 if missing)
DECISIONS             = count of "^- " in "Key Decisions and Rationale" section
LESSONS               = count of "^- " in "Lessons and Patterns" section
PROCEDURES_LINES      = wc -l PROCEDURES.md (0 if missing)
OPEN_THREADS          = count of "^- \[" in "Open Threads" section
DAILY_LOGS            = count of memory/YYYY-MM-DD.md files
UNCONSOLIDATED        = count of daily logs without <!-- consolidated -->
EPISODES              = count of episodes/*.md files
```

Fresh-instance check:

```
DAILY_LOGS == 0 AND RTMEMORY_LINES < 10  → Phase 5
```

## Phase 2 — Collect

Read **all** unconsolidated daily logs (full history). Extract per `references/durability.md` taxonomy: decisions, lessons, preferences, procedures, obligations, relationships, identity shifts, architecture conclusions, observations, todos, workflows. Skip small talk and content already in RTMEMORY.md.

Per candidate:

```json
{
  "summary": "...",
  "source": "memory/YYYY-MM-DD.md",
  "category": "decision|lesson|preference|procedure|obligation|relationship|identity|architecture|observation",
  "marker": null|"HIGH"|"PERMANENT"|"PIN"|"PREFERENCE"|"ROUTINE"|"PROCEDURE",
  "target_section": "<RTMEMORY section name>",
  "tags": []
}
```

## Phase 3 — Consolidate

Compare each candidate against RTMEMORY.md:

| Case | Action |
|------|--------|
| New | Append to destination (RTMEMORY section / PROCEDURES.md / `episodes/<name>.md`) |
| Updated (newer metric) | Update in place |
| Duplicate (semantic) | Skip |

For every new entry:
1. Apply `add_entry` per `references/index-operations.md` §3.1 with `promotedBy: "first-reflection"`.
2. Append `- [mem_NNN] (YYYY-MM-DD) <summary>` to destination section.

Procedures → `### Procedure: <name>` blocks in PROCEDURES.md.
Multi-event narratives → create or append to `episodes/<name>.md`.

Update `_Last updated:` date in RTMEMORY.md and PROCEDURES.md.

Mark each processed daily log `<!-- consolidated -->` after all its candidates handled.

`.bak runtime/reflections-metadata.json` → save.

## Phase 4 — Snapshot AFTER + Report

Recompute Phase 1 metrics → `*_AFTER`. Compute:

- `NEW_ENTRIES` = items added
- `UPDATED_ENTRIES` = items updated
- `STALE_COUNT` = entries unreferenced > 30 days

Append cycle report to `memory/.reflections-log.md` (include the 🪙 Tokens line per `references/recurring-rules.md` §7e when available; omit if unavailable).

Append telemetry per `references/telemetry-schema.md` §4.5 with `mode: "first-reflection"` and `entries_consolidated`. Resolve `token_usage` per `references/recurring-rules.md` §7b (exact → approximate → unavailable).

Compose final reply (this is your only chat output):

```
Reflections — First Memory Scan Complete!

📦 Your memory assets:
   • {DAILY_LOGS} daily logs ({earliest_date} ~ {latest_date}, spanning {days} days)
   • {RTMEMORY_LINES} lines of long-term memory (RTMEMORY.md)
   • {PROCEDURES_LINES} lines of workflow procedures
   • {EPISODES} project narratives

🔍 Scan results:
   • Extracted {NEW_ENTRIES} new entries from {UNCONSOLIDATED} logs
   • Updated {UPDATED_ENTRIES} existing entries
   • Found {STALE_COUNT} items stale for 30+ days

📊 Before → After:
   ┌─────────────────┬────────┬────────┐
   │                 │ Before │ After  │
   ├─────────────────┼────────┼────────┤
   │ Long-term memory│ {B}    │ {A}    │
   │ Key decisions   │ {B}    │ {A}    │
   │ Lessons learned │ {B}    │ {A}    │
   │ Procedures      │ {B}    │ {A}    │
   │ Open threads    │ {B}    │ {A}    │
   └─────────────────┴────────┴────────┘

🔮 Insights:
   1. {insight_1}
   2. {insight_2}
   3. {insight_3}

🪙 Token Usage: {per recurring-rules.md §7d — omit line entirely if source == "unavailable"}

⏰ Scheduled auto-reflection is now set up.
   You'll receive reports on the configured schedule.

💬 Let me know if anything was missed.

💭 After reading through {days} days of your history:
   {2-3 sentence personalized summary — mention specific projects by name,
   growth numbers, patterns you observed. End with one sentence about what
   reflections will do going forward. Reference real content from the logs.}
```

Translate to user's language before sending.

## Phase 5 — Fresh-instance report

```
💭 reflections Initialized!

✅ Memory architecture is ready:
   • 📝 Long-term memory (RTMEMORY.md)
   • 🔄 Workflow procedures (PROCEDURES.md)
   • 📁 Project narratives (episodes/)
   • 📊 Reflection reports (memory/.reflections-log.md)
   • 📦 Archive (memory/.reflections-archive.md)
   • 📈 Trend tracking (TRENDS.md)

🌱 Starting from zero — and that's fine.
   From now on, every conversation is remembered.
   On the configured schedule, I'll consolidate your daily logs
   into structured long-term memory.

⏰ Auto-reflection scheduled on the configured cadence.
   Your first real report will come after a few runs.

💬 Just chat naturally — I'll handle the rest.
```

Append fresh-instance telemetry per `telemetry-schema.md` §4.5 with zero counters. Translate to user's language before sending.

---

## Telemetry payload

```json
{
  "event": "run_completed",
  "status": "ok",
  "mode": "first-reflection",
  "details": {
    "logs_scanned": N,
    "entries_extracted": N,
    "entries_consolidated": N
  },
  "token_usage": {
    "prompt_tokens": <int|null>,
    "completion_tokens": <int|null>,
    "total_tokens": <int|null>,
    "source": "exact|approximate|unavailable"
  }
}
```

(Phase 5: all counters 0.) Use `entries_consolidated` (not `entries_qualified`) — first reflection has no gate. Resolve `token_usage` per `references/recurring-rules.md` §7b.

## Blocker Handling

On any failure, missing config, or unwritable workspace:

1. Append `run_failed` telemetry per `references/telemetry-schema.md` §4.6.
2. If `sendReport == true`: emit one short blocker message.
3. Do NOT mark daily logs. Do NOT partial-commit the index — `.bak` is recovery target.
