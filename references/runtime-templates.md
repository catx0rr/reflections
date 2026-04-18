# Runtime Templates — Chat Messages, Log Format, Weekly Math, Milestones, Streak

This document is the **rubric authority** for all human-facing output (chat notifications, skip messages, log entries) and the deterministic math that backs them. The runtime cites it for templates and field provenance.

---

## 1. Mode-due check (Step 0 timing rubric)

The runtime decides which modes are due by comparing each mode's `lastRun` timestamp against hardcoded elapsed-time intervals.

### 1.1 Intervals (hardcoded, never read from config)

| Mode | Interval | Notes |
|------|----------|-------|
| `rem` | 6 hours | Lightest cycle |
| `deep` | 12 hours | Mid-tier |
| `core` | 24 hours | Daily reflective pass |

These intervals are **independent of `dispatchCadence`**. The cron may fire 4×/day, but a mode only runs if its elapsed-time threshold is met.

### 1.2 Computation

For each mode in `activeModes`:

```
elapsed = now - lastRun.<mode>
if lastRun.<mode> is null OR elapsed >= interval:
  due_modes.append(mode)
```

If `lastRun.<mode>` is null (never run), the mode is due. If the file is missing entirely, all enabled modes are due.

### 1.3 Output variables (in-memory)

```
MODE_CSV       = comma-joined due_modes (e.g. "core,rem")
STRICT_MODE    = config.strictMode (default false)
DURABILITY_ON  = STRICT_MODE AND config.durability.enabled
SCAN_DAYS      = config.scanWindowDays (default 7)
```

---

## 2. Skip-with-recall message template (Step 0-B)

When no work is found OR no modes are due, the runtime emits the skip message **only if** `reflections.reporting.sendReport == true`.

### 2.1 Template

```
💭 No modes due — skipped reflection
✨ From your memory:
   {N} days ago ({date}), {stale_summary}.
📈 Memory: {total_after} entries · Health {health_score}/100 · Streak: {streak}
⚙️ Active modes: {active_modes_csv} · Next due: {next_due.mode} in {next_due.in}
```

If the stale-recall fields are not available (no stale entry found), omit the entire `✨ From your memory:` block.

### 2.2 Field provenance — skip

| Placeholder | Source |
|-------------|--------|
| `{N}` | `stale.days_stale` (top-1 stale result) |
| `{date}` | derived from stale entry's `lastReferenced` (formatted as YYYY-MM-DD) |
| `{stale_summary}` | `stale.summary` (one-liner) |
| `{total_after}` | count of active (non-archived) entries in the index |
| `{health_score}` | `index.stats.healthScore` |
| `{streak}` | computed per §6 |
| `{active_modes_csv}` | comma-joined `config.activeModes` |
| `{next_due.mode}` | mode with the smallest remaining time per §1 |
| `{next_due.in}` | formatted remaining time (e.g. `"2.5h"`, `"45m"`, `"now"`) |

If any field is null/unavailable, omit the line containing it.

---

## 3. Cycle log entry (committed by Step 3.10)

Append to `memory/.reflections-log.md`:

```markdown
## 2026-04-18 22:30 [Asia/Manila] · core+rem

**Status:** ok
**Modes fired:** core, rem
**Logs scanned:** 3 (memory/2026-04-16.md, memory/2026-04-17.md, memory/2026-04-18.md)
**Logs marked consolidated:** 2

**Cycle counters:**
- Extracted: 12 candidates
- Qualified (gate): 6
- Deferred (gate): 6
- Routed (durability): 3 promote · 1 merge · 1 compress · 1 defer · 0 reject

**Surface deltas:**
- RTMEMORY.md: 142 → 145 entries (+3 new, 1 updated, 0 archived)
- PROCEDURES.md: unchanged
- TRENDS.md: 1 reinforced, 0 new

**Health:** 78/100 (good) · freshness 0.82 · coverage 0.70 · coherence 0.45 · efficiency 0.62 · reachability 0.58

**Insights:**
- Decisions and lessons drove most promotions; no new procedures this cycle.
- Reachability still below 0.6 — consider linking related entries.

**Stale (top 3):**
- (45d) [mem_022] Q2 priority alignment with operator
- (32d) Stripe webhook retry policy decision
- (28d) [mem_055] Dev server noon-restart pattern

🪙 Tokens: 4320 in / 812 out / 5132 total

---
```

Headers and dividers are markdown. `---` separates cycles. Always append; never edit prior cycles.

**Token line** — per `recurring-rules.md` §7e: use `🪙 Tokens:` when `source == "exact"`, `🪙 Tokens (approx):` when `source == "approximate"`, omit the line entirely when `source == "unavailable"`. Same compact variants as the notification (full triple or total-only).

---

## 4. Cycle notification template (Step 4.2)

Emitted to chat **only when** `reflections.reporting.sendReport == true`.

### 4.1 Main template

```
💭 Reflection #{reflection_count} complete ({modes_fired} cycle)

📥 Consolidated: {entries_delta} entries from {logs_count} logs
   new: {new} · updated: {updated} · archived: {archived}
📈 Total: {total_before} → {total_after} entries ({percent_growth} growth)
🧠 Health: {health_score}/100 — {rating}

✨ Highlights:
   • {LLM-composed change bullet from this cycle's promoted entries}
   • {LLM-composed change bullet}

💡 Insight: {top insight from Step 3.5}

⏳ Stale: {one stale result from Step 2.8 if any}

{milestones[0] if any}
{milestones[1] if any}
💬 {LLM-composed warm closing line — vary wording, never include numbers}
```

### 4.2 Field provenance — cycle

All numeric/state placeholders come from deterministic computation by Step 4.2 (the runtime), per §5–§6 below. The LLM composes only four fields.

| Placeholder | Source |
|-------------|--------|
| `{reflection_count}` | `len(index.stats.healthHistory)` (after this cycle's entry is appended) |
| `{modes_fired}` | `MODE_CSV` joined with `+` (e.g. `"core+rem"`) |
| `{entries_delta}` | `total_after - total_before` |
| `{logs_count}` | count of unconsolidated logs processed this cycle |
| `{new}` | count of `route == "promote"` (or all consolidated in parity flow) |
| `{updated}` | count of `route == "merge"` (parity flow: 0) |
| `{archived}` | count of entries archived this cycle (Step 3) |
| `{total_before}` | active-entry count from Step 0.5 snapshot |
| `{total_after}` | active-entry count from Step 2.5 snapshot |
| `{percent_growth}` | computed per §5.1 |
| `{health_score}` | `index.stats.healthScore` |
| `{rating}` | from `health.md` rating bands |
| `{milestones[i]}` | computed per §7 |

### 4.3 LLM-authored fields (only four)

| Placeholder | LLM task |
|-------------|----------|
| `✨ Highlights` (1–2 bullets) | Read this cycle's `route == "promote"` candidates; distill into short bullets describing what changed |
| `💡 Insight` | The top insight from Step 3.5 (one sentence) |
| `⏳ Stale` | One stale result from Step 2.8 (one line) — omit the `⏳ Stale:` line if no stale |
| `💬 closing line` | A warm closing sentence inviting the operator to flag anything missed. Vary wording per cycle. Never include numeric/state fields. Examples: "Let me know if anything was missed." / "Flag anything that didn't land." / "Tell me if I missed a beat." |

### 4.4 Omission rules

- If a placeholder's source is `null`, omit the entire line containing it.
- If `weekly_snapshot_available == false`, omit the weekly block entirely (per §8).
- Never invent a number to fill a missing field.

### 4.5 Weekly block (prepended to main template when triggered)

Triggered when **either**:
- It is Sunday AND local time >= 18:30, OR
- `reflection_count` is divisible by 7 (every 7th cycle)

```
📊 Weekly Report ({weekly.date_range})

🧠 This week: +{weekly.weekly_new} new · {weekly.weekly_updated} updated · {weekly.weekly_archived} archived
   {weekly.total_before_week} → {weekly.total_after} entries ({weekly.percent_growth} growth)

🌟 Biggest memories this week:
   1. {weekly.biggest_memories[0].summary}
   2. {weekly.biggest_memories[1].summary}
   3. {weekly.biggest_memories[2].summary}
```

If fewer than 3 biggest-memories returned, render only the entries present.

---

## 5. Growth math

### 5.1 Percent growth

```
delta = total_after - total_before
if total_before > 0:
  percent_growth = "+{pct}%" or "{pct}%" depending on sign
  where pct = round((delta / total_before) * 100, 1)
else:
  percent_growth = "+∞%" if delta > 0 else "0.0%"
```

Examples:

| total_before | total_after | percent_growth |
|--------------|-------------|----------------|
| 142 | 145 | `+2.1%` |
| 100 | 100 | `0.0%` |
| 100 | 95 | `-5.0%` |
| 0 | 5 | `+∞%` |
| 0 | 0 | `0.0%` |

---

## 6. Streak computation

Counts consecutive distinct calendar days with at least one cycle, working backward from the most recent cycle. A gap of ≥ 2 days breaks the streak.

```
dates = sorted unique dates in index.stats.healthHistory, descending
if dates is empty: streak = 0
else:
  streak = 1
  for i in 1 to len(dates) - 1:
    if (dates[i-1] - dates[i]).days == 1:
      streak += 1
    else:
      break
```

Examples:

| healthHistory dates | Streak |
|---------------------|--------|
| (empty) | 0 |
| `[2026-04-18]` | 1 |
| `[2026-04-16, 2026-04-17, 2026-04-18]` | 3 |
| `[2026-04-10, 2026-04-11, 2026-04-13]` | 1 (gap broke it) |
| `[2026-04-15, 2026-04-15, 2026-04-16]` | 2 (duplicates collapsed) |

---

## 7. Milestones

After computing reflection_count and streak, append milestone banners (in order) to the cycle notification:

| Condition | Banner |
|-----------|--------|
| `reflection_count == 1` | `🎉 First reflection complete!` |
| `reflection_count == 7` OR `streak == 7` | `🏅 One week streak!` |
| `reflection_count == 30` OR `streak == 30` | `🏆 One month streak!` |
| Entry-count crossed 100 this cycle (`total_before < 100 <= total_after`) | `📊 Memory milestone! 100 entries.` |
| Entry-count crossed 200 this cycle | `📊 Memory milestone! 200 entries.` |
| Entry-count crossed 500 this cycle | `📊 Memory milestone! 500 entries.` |
| Entry-count crossed 1000 this cycle | `📊 Memory milestone! 1000 entries.` |

Multiple banners may fire in one cycle; emit each on its own line.

---

## 8. Weekly math

Triggered per §4.5. Computes a 7-day rollup from the index.

### 8.1 Date range

```
end_date = today (YYYY-MM-DD)
start_date = end_date - 7 days
date_range = "<start_date> – <end_date>"
```

### 8.2 Counters

```
active = entries where archived != true

weekly_new      = count of active where created >= start_date
weekly_updated  = count of active where lastReferenced >= start_date AND created < start_date
weekly_archived = count of all entries (incl. archived) where archived_at >= start_date
total_after     = len(active)
```

### 8.3 total_before_week

If `index.stats.healthHistory` has an entry from ≥ 7 days ago, derive structurally. Otherwise approximate:

```
total_before_week = total_after - weekly_new + weekly_archived
```

### 8.4 percent_growth (weekly)

Same formula as §5.1, applied to `total_before_week → total_after`.

### 8.5 biggest_memories

```
recent_touched = active where (created >= start_date OR lastReferenced >= start_date)
sort recent_touched by importance descending
biggest_memories = first 3 entries (or fewer if not enough)

each entry: { "id": "mem_NNN", "summary": "...", "importance": 0.78 }
```

### 8.6 weekly_snapshot_available

```
weekly_snapshot_available = len(active) > 0 AND (weekly_new + weekly_updated + weekly_archived) > 0
```

If false, omit the weekly block.

### 8.7 Weekly token rollup (optional)

See `recurring-rules.md` §7f. Summed only over cycles whose telemetry carries `token_usage` with `source ∈ {exact, approximate}`. If any cycle in the window is `"approximate"`, the whole line is labeled `(approx)`. If no cycle in the window carries token data, omit the line. Never backfill historical counts.

---

## 9. Token-usage provenance

Token numbers follow the `recurring-rules.md` §7 rule across all surfaces (telemetry envelope, cycle log entry, notification line, weekly rollup):

- Runtime-provided usage metadata → `source: "exact"`
- Character-count approximation (`chars / 4`, §7c) → `source: "approximate"`, label line with `(approx)`
- Neither available → `source: "unavailable"`, **omit the line** (telemetry envelope still carries nulls)

**Never fabricate `"exact"`.** Never invent missing values. The 🪙 emoji marks the line consistently.

---

## 10. Authority and conflict handling

If this rubric appears to disagree with the runtime's Step 0-B / 4.1 / 4.4 instruction, **stop and emit a blocker** per the runtime's precedence rule. Do not silently reconcile.

This doc owns the templates, math, and field provenance. The runtime owns the *when* and *whether*.
