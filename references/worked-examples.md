# Worked Examples — End-to-End Cycle Traces (Debug / Human Reference Only)

> **NOT loaded on the recurring cron path.** This document is for humans reading the package, debugging unexpected behavior, or studying the design — never cited by the runtime prompt. The recurring runtime consults `references/recurring-rules.md` (the compact card) for all rules; long-form theory and these examples stay out of the hot path.
>
> Per the precedence rule:
> - Runtime owns execution order
> - `recurring-rules.md` owns rules for the cron path
> - Long-form docs (scoring.md / durability.md / health.md) own derivations and full theory — for humans
> - These worked examples are illustrative only and never override either

These traces show how a personal-assistant parity cycle and a business-employee strict+durability cycle execute end-to-end, with every numeric component shown. Use them to pattern-match when designing or debugging — never as a runtime authority.

---

## Example A — Personal Assistant parity cycle

### Scenario

- **Profile:** personal-assistant (`strictMode: false`, `durability.enabled: false`)
- **Time:** 2026-04-18 22:30 local (Asia/Manila)
- **Active modes:** core, rem, deep
- **Last cycle:** 2026-04-18 04:30 (core ran), 2026-04-18 16:30 (rem ran), 2026-04-17 22:30 (deep ran)
- **Existing index:** 87 active entries, last health 74

### Step 0 — Mode dispatch

`now = 2026-04-18T22:30:00+08:00`

| Mode | lastRun | elapsed | interval | due? |
|------|---------|---------|----------|------|
| core | 2026-04-18 04:30 | 18h | 24h | no (only 18h elapsed) |
| rem | 2026-04-18 16:30 | 6h | 6h | yes |
| deep | 2026-04-17 22:30 | 24h | 12h | yes |

`due_modes = ["rem", "deep"]`
`MODE_CSV = "rem,deep"`
`STRICT_MODE = false`
`DURABILITY_ON = false`
`SCAN_DAYS = 7`

### Step 0-A — Scan

Scans `memory/2026-04-12.md` through `memory/2026-04-18.md`. Finds 2 unconsolidated logs:
- `memory/2026-04-17.md` (no `<!-- consolidated -->` marker)
- `memory/2026-04-18.md` (no marker)

`unconsolidated = ["memory/2026-04-17.md", "memory/2026-04-18.md"]`
`has_work = true`

Decision: due_modes non-empty AND has_work → continue.

### Step 0.5 — Snapshot BEFORE

```
RTMEMORY_LINES = 287
RTMEMORY_SECTIONS = 10
PROCEDURES_LINES = 84
EPISODES = 3
INDEX_ENTRIES_ACTIVE = 87
INDEX_ENTRIES_ARCHIVED = 12
```

### Step 1 — Collect

Reads both logs. Extracts 4 candidates:

| id | source | summary | category | refCount | uniqueDay | marker | target_section |
|----|--------|---------|----------|----------|-----------|--------|----------------|
| c1 | 2026-04-17.md | "Brother Miguel started at BPI" | relationship | 1 | 1 | null | People and Relationships |
| c2 | 2026-04-17.md | "User prefers afternoon meetings" | preference | 1 | 1 | PREFERENCE | Strategy and Priorities |
| c3 | 2026-04-18.md | "Decided: morning routine includes meditation" | decision | 1 | 1 | ROUTINE | Key Decisions and Rationale |
| c4 | 2026-04-18.md | "Cat napped on the keyboard again" | observation | 1 | 1 | null | (no clear section) |

### Step 1.5 — Branch

`STRICT_MODE = false` → skip Steps 1.2/1.3/1.6/1.7/1.8 → go to Step 2.

### Step 2 — Stage consolidations (parity flow)

Every candidate stages via semantic LLM routing. **In memory only** — no disk writes yet.

| id | Destination | Staged action |
|----|-------------|---------------|
| c1 | RTMEMORY.md > People and Relationships | Add to `STAGED_INDEX.entries` as `mem_088`. Add surface op: append `- [mem_088] (2026-04-17) Brother Miguel started at BPI as a teller`. |
| c2 | RTMEMORY.md > Strategy and Priorities | Add as `mem_089`. Add surface op for the line. |
| c3 | RTMEMORY.md > Key Decisions and Rationale | Add as `mem_090`. Add surface op for the line. |
| c4 | (skip) | Small-talk observation, semantic routing decides not to promote — no staging. |

`STAGED_INDEX` now has 90 active entries (87 + 3); `STAGED_SURFACE_OPS` has 3 RTMEMORY appends queued.

### Step 2.5 — Snapshot AFTER (from staged state)

```
AFTER_SNAPSHOT (computed from BEFORE + staged ops, not disk):
  RTMEMORY_LINES = 290 (+3 from staged appends)
  INDEX_ENTRIES_ACTIVE = 90 (+3 from STAGED_INDEX)
DELTA_NEW = 3, DELTA_UPDATED = 0
```

### Step 2.7 — Stage log markings

Both daily logs have all candidates handled (c1–c3 staged-promote; c4 deliberately skipped) → both added to `LOGS_TO_MARK`. Actual `<!-- consolidated -->` write happens in Step 3.10.

### Step 2.8 — Stale detection

Top 3 stale (>14 days unreferenced):
1. (45d) `[mem_022]` Q2 priority alignment
2. (32d) Stripe webhook retry policy
3. (28d) `[mem_055]` Dev server noon-restart

### Step 3 — Stage health + archival

Health computation (against `STAGED_INDEX`):

```
freshness:    32 of 90 active referenced in last 30 days  → 0.356
coverage:     all 10 canonical sections have content       → 1.000
coherence:    18 of 90 entries have >=1 relation           → 0.200
efficiency:   max(0, 1 - 290/500) = 1 - 0.58               → 0.420
reachability: components [12, 8, 5, 3, 1×62] → ...
              weighted_sum = 144 + 64 + 25 + 9 + 62 = 304
              total² = 90² = 8100
              reachability = 304 / 8100 ≈ 0.038            → 0.038

health_raw = 0.356×0.25 + 1.000×0.25 + 0.200×0.20 + 0.420×0.15 + 0.038×0.15
           = 0.089 + 0.250 + 0.040 + 0.063 + 0.006
           = 0.448
health_score = 45
rating = "fair"
```

Archival pass (staged): scores all active entries in `STAGED_INDEX`. Two qualify (`importance < 0.3`, `days > 90`, no PERMANENT/PIN, not in episode):

- `mem_005` (created 2025-09, last ref 2026-01, importance 0.18) → mark archived in `STAGED_INDEX`; queue surface ops: append archive line + remove from RTMEMORY.md.
- `mem_011` (created 2025-10, last ref 2026-01, importance 0.22) → same.

`STAGED_INDEX` and `STAGED_SURFACE_OPS` updated. Disk untouched. `DELTA.archived = 2`.

### Step 3.5 — Insights

LLM generates:
1. "Health dropped to 45 (fair) — reachability is the dominant drag at 4%. Consider linking related entries to consolidate the 62 isolated nodes."
2. "Three new relationships/preferences this cycle suggests the operator is settling into routines. Watch for compounding fast-path candidates."

### Step 3.6 — Stage stats payload (in memory)

```json
STAGED_INDEX = {
  "lastDream": "2026-04-18T14:30:14Z",
  "stats": {
    "totalEntries": 88,           // 90 - 2 archived
    "avgImportance": 0.412,
    "healthScore": 45,
    "healthMetrics": {
      "freshness": 0.356, "coverage": 1.000, "coherence": 0.200,
      "efficiency": 0.420, "reachability": 0.038
    },
    "insights": ["Health dropped to 45...", "Three new relationships..."],
    "healthHistory": [
      ..., {"date": "2026-04-18", "score": 45}
    ],
    "gateStats": {
      "lastCycleQualified": 0,
      "lastCycleDeferred": 0,
      "lastCycleBreakdown": {"rem": 0, "deep": 0, "core": 0}
    }
  }
}
```

(In parity flow, `gateStats` counters are 0 — no gate ran.) Still in memory; nothing on disk yet.

### Step 3.7 — Pre-commit checks

All target files writable. `.bak` destinations OK. `STAGED_INDEX` coherent (`lastDream` set, `healthHistory[-1].date == today`, all entry ids unique). No runtime/reference conflict detected. → Continue.

### Step 3.8 — Persist index (FIRST DURABLE WRITE)

Copy `runtime/reflections-metadata.json` → `.bak`. Write `STAGED_INDEX` to disk.

### Step 3.9 — Telemetry (SECOND DURABLE WRITE)

```json
{
  "timestamp": "2026-04-18T22:30:14+08:00",
  "timestamp_utc": "2026-04-18T14:30:14Z",
  "domain": "memory",
  "component": "reflections.consolidator",
  "event": "run_completed",
  "run_id": "refl-2026-04-18T22-30-14-...",
  "status": "ok",
  "agent": "main",
  "profile": "personal-assistant",
  "mode": "scheduled",
  "details": {
    "logs_scanned": 2,
    "entries_extracted": 4,
    "entries_consolidated": 3,
    "logs_marked_consolidated": 2
  }
}
```

Appended to `$TELEMETRY_ROOT/memory-log-<today>.jsonl`.

### Step 3.10 — Atomic surface commit

`.bak` each surface, then apply staged ops:
- RTMEMORY.md: append c1, c2, c3 lines; remove archived `mem_005`, `mem_011`.
- `memory/.reflections-archive.md`: append two archive lines.
- `memory/.reflections-log.md`: append cycle entry per `runtime-templates.md` §3.
- Mark `memory/2026-04-17.md` and `memory/2026-04-18.md` with `<!-- consolidated -->`. Re-read each to confirm marker present.

All succeed → cycle fully committed.

### Step 4.1 — Notify gate

`reflections.reporting.sendReport == true` → emit notification.

### Step 4.2 — Notification

Field computation:

```
reflection_count = 28 (length of healthHistory)
streak = 5 (5 consecutive daily cycles)
modes_fired = "rem+deep"
total_before = 87
total_after = 88   (90 promoted - 2 archived)
entries_delta = 1  (88 - 87)
percent_growth = "+1.1%"
new = 3
updated = 0
archived = 2
logs_count = 2
health_score = 45
rating = "fair"
next_due.mode = "core"
next_due.in = "6h"  (core lastRun was 18h ago, 24h interval)
milestones = []  (no thresholds crossed)
```

Output:

```
💭 Reflection #28 complete (rem+deep cycle)

📥 Consolidated: 1 entries from 2 logs
   new: 3 · updated: 0 · archived: 2
📈 Total: 87 → 88 entries (+1.1% growth)
🧠 Health: 45/100 — fair

✨ Highlights:
   • Recorded that Miguel started at BPI — relationship update.
   • New routine logged: morning meditation.

💡 Insight: Health dropped to 45 (fair) — reachability is the dominant drag at 4%. Consider linking related entries to consolidate the 62 isolated nodes.

⏳ Stale: (45d) [mem_022] Q2 priority alignment with operator

💬 Flag anything that didn't land.
```

### Step 5 — Update lastRun

```json
"lastRun": {
  "core": "2026-04-18T04:30:14+08:00",      // unchanged
  "rem": "2026-04-18T22:30:14+08:00",       // updated
  "deep": "2026-04-18T22:30:14+08:00"       // updated
}
```

Pre-write `.bak` of `reflections.json`. Save.

---

## Example B — Business Employee strict+durability cycle

### Scenario

- **Profile:** business-employee (`strictMode: true`, `durability.enabled: true`)
- **Time:** 2026-04-18 12:30 local (Asia/Manila)
- **Active modes:** core, rem, deep
- **Last cycle:** rem 6h ago, deep 13h ago, core 25h ago → all due
- **Existing index:** 142 active entries, last health 78
- **Existing trend node:** `mem_099` with `trendKey = "dev-server-noon-restart"`, `trendSupportCount = 4`, `uniqueDayCount = 3`
- **Existing durable:** `mem_042` (Stripe billing decision)

### Step 0 — Dispatch

`due_modes = ["core", "rem", "deep"]`. All three overdue.
`MODE_CSV = "core,rem,deep"`, `STRICT_MODE = true`, `DURABILITY_ON = true`, `SCAN_DAYS = 3`.

### Step 0-A — Scan

3 unconsolidated logs in last 3 days: `2026-04-16.md`, `2026-04-17.md`, `2026-04-18.md`.

### Step 0.5 — Snapshot BEFORE

```
INDEX_ENTRIES_ACTIVE = 142
RTMEMORY_LINES = 412
```

### Step 1 — Collect

Extracts 7 candidates:

| id | source | summary | refCount | uniqueDay | marker | target_section |
|----|--------|---------|----------|-----------|--------|----------------|
| c1 | 2026-04-18.md | "Decided: dev-server health checks at 11:55 daily to catch noon restart" | 1 | 1 | HIGH | Key Decisions and Rationale |
| c2 | 2026-04-18.md | "Customer flagged Tuesday slowness again — staffing thinner Tuesdays" | 2 | 2 | null | Lessons and Patterns |
| c3 | 2026-04-18.md | "Stripe webhook signature failed once — investigated, was clock skew" | 1 | 1 | null | Open Threads |
| c4 | 2026-04-17.md | "Server returned 200 OK at 14:22:01" | 1 | 1 | null | (none) |
| c5 | 2026-04-17.md | "Dev server restarted around noon (again)" | 1 | 1 | null | (none) |
| c6 | 2026-04-17.md | "Patient Maria prefers afternoon appointments" | 3 | 3 | PREFERENCE | People and Relationships |
| c7 | 2026-04-16.md | "Pattern: morning meetings feel rushed (3rd time this week)" | 3 | 1 | null | Lessons and Patterns |

### Step 1.2 — Load deferred state

`DEFERRED_RECORDS` = 18 records (accumulated across prior cycles).

### Step 1.3 — Annotate deferred

For each candidate, compose `identityKey` and check against deferred store.

| id | identityKey (truncated) | deferred_status |
|----|--------------------------|------------------|
| c1 | `2026-04-18::decisions key rationale::checks daily dev health noon restart server...::-` | fresh |
| c2 | `2026-04-18::lessons patterns::again customer flagged staffing slowness thinner tuesday::-` | fresh |
| c3 | `2026-04-18::open threads::clock investigated once signature skew stripe was webhook failed::-` | fresh |
| c4 | `2026-04-17::-::200 14 22 01 ok returned server::-` | **persisted** (matched fingerprint from 3 prior cycles) |
| c5 | `2026-04-17::-::around again dev noon restarted server::-` | fresh |
| c6 | `2026-04-18::people relationships::afternoon appointments maria patient prefers::-` | fresh |
| c7 | `2026-04-16::lessons patterns::3rd feel meetings morning pattern rushed week::-` | fresh |

c4 already persisted → won't be re-evaluated.

### Step 1.5 — Branch

`STRICT_MODE = true` → continue to Step 1.6.

### Step 1.6 — Score + Gate

For each non-persisted candidate, compute importance per `scoring.md`:

| id | base | recency (days=0) | boost (refCount) | raw | importance |
|----|------|------------------|------------------|-----|------------|
| c1 | 2.0 (HIGH) | 1.000 | 1.000 (refCount=1) | 2.000 | 0.250 |
| c2 | 1.0 | 1.000 | 1.585 (refCount=2) | 1.585 | 0.198 |
| c3 | 1.0 | 1.000 | 1.000 | 1.000 | 0.125 |
| c5 | 1.0 | 1.000 | 1.000 | 1.000 | 0.125 |
| c6 | 1.0 | 1.000 | 2.000 (refCount=3) | 2.000 | 0.250 |
| c7 | 1.0 | 1.000 | 2.000 | 2.000 | 0.250 |

Apply BE gates strictest first (rem→deep→core):

| id | rem (0.85/3/2) | deep (0.80/2/2) | core (0.72/2/1) | fast-path? | gate_status |
|----|----------------|-----------------|------------------|------------|-------------|
| c1 | fail (0.25 < 0.85) | fail | fail | HIGH in rem fastPathMarkers? No (BE rem fastPath = HIGH/PIN/PROCEDURE). YES — HIGH matches | **qualified** by rem (FAST_PATH bypass) |
| c2 | fail | fail | fail | no marker, no fast-path | **deferred** |
| c3 | fail | fail | fail | no | **deferred** |
| c5 | fail | fail | fail | no | **deferred** |
| c6 | fail (0.25<0.85) | fail | fail | PREFERENCE not in BE fastPathMarkers | **deferred** |
| c7 | fail | fail | fail | no | **deferred** |

c1: gate_status = "qualified", gate_bypass = "FAST_PATH"
c2, c3, c5, c6, c7: gate_status = "deferred"

Append c2, c3, c5, c6, c7 to deferred store. (5 new lines.)

### Step 1.7 — Durability annotation

Scope: c1 (qualified) plus rescue-eligible deferred candidates.

Rescue eligibility check:

| id | gate_bypass set? | marker in HIGH/PERMANENT/PIN? | high-meaning class? | rescue? |
|----|-----------------|-------------------------------|----------------------|---------|
| c2 | no | no | "lesson" → yes | **yes** |
| c3 | no | no | "observation" → no | no |
| c5 | no | no | "observation" → no | no |
| c6 | no | no | "preference" → no (not in high-meaning list — preference is borderline; LLM judges no rescue here) | no |
| c7 | no | no | "lesson"-ish but pattern → yes | **yes** |

Annotate c1, c2, c7:

**c1** (decision: dev-server health checks at 11:55):
- memory_type: `decision`
- changed_future_decision: `true`
- cross_day_relevance: `true`
- trend_key: `dev-server-noon-restart` (matches existing trend)
- (other flags false)

**c2** (lesson: Tuesday slowness ~ staffing):
- memory_type: `lesson`
- changed_future_decision: `false`
- cross_day_relevance: `true`
- (other flags false)

**c7** (pattern: morning meetings rushed):
- memory_type: `observation`
- pattern_only: `true`
- cross_day_relevance: `false` (3 mentions all this week, single-day overall)
- (other flags false)

### Step 1.8 — Durability routing

Per `durability.md` precedence:

**c1:**
- Hard-suppress? no
- Hard-promote? `memory_type=decision` AND `changed_future_decision=true` → YES
- trendKey set + matches existing `mem_099`. Check trend promotion criteria: `trendSupportCount=4 >= 5? NO`. So *not* trend-to-durable promote. Just regular hard-promote.
- route = `promote`, destination = `RTMEMORY`, promotionReason = `hard-trigger:decision-with-consequence`

**c2:**
- Hard-suppress? no
- Hard-promote? memory_type=lesson, changed_future_decision=false, changed_behavior_or_policy=false → no
- duplicate_of_existing? no
- trendKey? no
- Net score: structural=2 (refCount=2 +1, uniqueDay=2 +1), meaning=0, futureConsequence=1 (cross_day=1), noise=0 → net=3
- BE thresholds: promote=6, defer=3 → net=3 ≥ 3 → **defer**
- route = `defer`, destination = `NONE`, promotionReason = `net=3>=3`

**c7:**
- Hard-suppress? `pattern_only=true AND cross_day_relevance=false` → YES → route = `reject`
- promotionReason = `hard-suppress:pattern_only_same_day`

Append c2 to deferred store (1 more line).

### Step 2 — Stage consolidations

Per route. **In memory only.**

| id | route | Staged action |
|----|-------|---------------|
| c1 | promote | Add to `STAGED_INDEX.entries` as `mem_143` (with durability fields). Add surface op: append to RTMEMORY.md > Key Decisions and Rationale. |
| c2 | defer | No staging (already in deferred store). |
| c3 | (gate-deferred) | No staging (already in deferred store). |
| c4 | (deferred persisted) | No staging. |
| c5 | (gate-deferred) | No staging. |
| c6 | (gate-deferred) | No staging. |
| c7 | reject | No staging — discarded. |

### Step 2.5 — Snapshot AFTER (from staged state)

```
AFTER_SNAPSHOT.INDEX_ENTRIES_ACTIVE = 143 (+1 from STAGED_INDEX)
AFTER_SNAPSHOT.RTMEMORY_LINES = 413 (+1 from staged append)
DELTA_NEW = 1
```

### Step 2.7 — Stage log markings

All candidates from each log reached terminal state — added to `LOGS_TO_MARK`:
- `2026-04-16.md`: c7 → reject (terminal).
- `2026-04-17.md`: c4 deferred-persisted, c5 deferred-fresh (now in store), c6 deferred-fresh (now in store). All terminal.
- `2026-04-18.md`: c1 promoted (staged), c2 deferred (in store), c3 deferred (in store). All terminal.

3 logs queued for marking. Actual `<!-- consolidated -->` write happens in Step 3.10.

### Step 2.8 — Stale

(Same shape as Example A — top 3 by days_stale.)

### Step 3 — Stage health + archival

(Health metric computation similar to Example A; this trace skips numeric detail. Suppose `healthScore = 78`, `rating = good`, no archival eligible this cycle.)

### Step 3.5 — Insights

LLM:
1. "One promotion this cycle (Stripe webhook decision); reject and defer pressure suggests gating thresholds match the noise topology well."
2. "Trend `dev-server-noon-restart` is one cycle away from durable-promotion eligibility (5/3 supports threshold)."

### Step 3.6 — Stage stats payload (in memory)

```json
STAGED_INDEX = {
  "stats": {
    "totalEntries": 143,
    "avgImportance": 0.51,
    "healthScore": 78,
    "healthMetrics": { ... },
    "insights": [ ... ],
    "healthHistory": [..., {"date": "2026-04-18", "score": 78}],
    "gateStats": {
      "lastCycleQualified": 1,
      "lastCycleDeferred": 5,
      "lastCycleBreakdown": {"rem": 1, "deep": 0, "core": 0},
      "lastCycleDurable": {
        "promoted": 1,
        "merged": 0,
        "compressed": 0,
        "deferred": 1,
        "rejected": 1
      }
    }
  }
}
```

### Step 3.7 — Pre-commit checks

Targets writable, `.bak` paths OK, `STAGED_INDEX` coherent. → Continue.

### Step 3.8 — Persist index (FIRST DURABLE WRITE)

`.bak` runtime/reflections-metadata.json. Write `STAGED_INDEX` to disk.

### Step 3.9 — Telemetry (SECOND DURABLE WRITE)

```json
{
  "details": {
    "logs_scanned": 3,
    "entries_extracted": 7,
    "entries_qualified": 1,
    "entries_deferred": 5,
    "entries_durable_promoted": 1,
    "entries_durable_merged": 0,
    "entries_durable_compressed": 0,
    "entries_durable_deferred": 1,
    "entries_durable_rejected": 1,
    "logs_marked_consolidated": 3
  }
}
```

### Step 3.10 — Atomic surface commit

`.bak` each surface, then apply staged ops:
- RTMEMORY.md > Key Decisions and Rationale: append `mem_143` line.
- `memory/.reflections-log.md`: append cycle entry.
- Mark `2026-04-16.md`, `2026-04-17.md`, `2026-04-18.md` with `<!-- consolidated -->`. Re-read each to confirm.

All succeed → cycle fully committed.

### Step 4.2 — Notification

```
💭 Reflection #143 complete (core+rem+deep cycle)

📥 Consolidated: 1 entries from 3 logs
   new: 1 · updated: 0 · archived: 0
📈 Total: 142 → 143 entries (+0.7% growth)
🧠 Health: 78/100 — good

✨ Highlights:
   • Decided: 11:55 health checks to preempt the noon dev-server restart.

💡 Insight: One promotion this cycle (Stripe webhook decision); reject and defer pressure suggests gating thresholds match the noise topology well.

⏳ Stale: (45d) [mem_022] Q2 priority alignment with operator

💬 Tell me if I missed a beat.
```

### Step 5 — Update lastRun

All three modes' `lastRun` updated to now.

---

## What these examples demonstrate

| Pattern | Example A (PA parity) | Example B (BE strict+durability) |
|---------|------------------------|-----------------------------------|
| Routing | Semantic LLM choice (per surface) | Deterministic 5-route dispatch |
| Gate behavior | None — all consolidate | AND gate, fast-path bypass for HIGH-marked candidate |
| Durability annotation | Skipped | Annotated qualified + rescue subset; 3 routes fired |
| Trend handling | N/A | Existing trend `mem_099` was within reach of promotion but not yet eligible |
| Identity-key suppression | Not used | c4 suppressed by deferred-store match; not re-evaluated |
| Telemetry shape | Parity payload (4 fields) | Full durability payload (10 fields) |
| Notification | Same template, different counts | Same template, different counts |

When in doubt about how to apply a rubric, find the analogous pattern in these traces and follow it. But remember: **these examples are illustrative only**. The runtime and references are authoritative.
