# reflections — Recurring Runtime Prompt

Read USER.md first. All output in user's language.

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

## Surfaces

| Surface | Path |
|---------|------|
| Reflective memory | `RTMEMORY.md` |
| Procedures | `PROCEDURES.md` |
| Episodes | `episodes/*.md` |
| Trends | `TRENDS.md` |
| Index | `runtime/reflections-metadata.json` |
| Cycle log | `memory/.reflections-log.md` |
| Archive | `memory/.reflections-archive.md` |
| Deferred store | `runtime/reflections-deferred.jsonl` |
| Daily input | `memory/YYYY-MM-DD.md` |
| Shared state | `runtime/memory-state.json` `reflections.*` |

## Guardrails

- Execute all steps in the current agent. No sub-agent.
- No chat narration. Internal details → telemetry / log files only.
- Chat emits exactly one of: Step 4.2 notification, blocker, or nothing (when `sendReport == false`).
- `.bak` `runtime/reflections-metadata.json` and `reflections.json` before any mutation.
- `.bak` `RTMEMORY.md` only when this cycle changes it by > 30%.
- Never delete daily logs. Never remove `⚠️ PERMANENT` entries.
- **Stage/commit discipline:** Steps 2–3.6 mutate only in-memory artifacts. Disk writes happen in strict order at Step 3.8 (surfaces), 3.9 (index), 3.10 (markers), 3.11 (telemetry). Atomic rollback if 3.8 or 3.9 fails. See Blocker Handling.

---

## Step 0 — Mode dispatch

Read `$CONFIG_PATH`. For each mode in `activeModes`, mode is due iff `lastRun.<mode>` is null OR `(now - lastRun.<mode>) >= interval`:

| Mode | Interval |
|------|----------|
| rem | 6h |
| deep | 12h |
| core | 24h |

Derive in-memory:

```
MODE_CSV       = comma-joined due_modes
STRICT_MODE    = config.strictMode (default false)
DURABILITY_ON  = STRICT_MODE AND config.durability.enabled (default false)
SCAN_DAYS      = config.scanWindowDays (default 7)
PROFILE        = config.profile
AGENT          = config.agent | "main"
```

## Step 0-A — Scan

List `memory/YYYY-MM-DD.md` within last `SCAN_DAYS` days. For each, read; check for `<!-- consolidated -->`.

```
unconsolidated = files without marker
has_work       = len(unconsolidated) > 0
```

## Step 0-B — Skip with recall

| `due_modes` non-empty | `has_work` | Next |
|-----------------------|-----------|------|
| no | — | this step |
| yes | no | this step |
| yes | yes | Step 0.5 |

If skipping:

1. Append telemetry per `references/telemetry-schema.md` §4.1 (always, regardless of sendReport).
2. Read `runtime/memory-state.json`. If `reflections.reporting.sendReport != true` → stop the cycle.
3. Compute skip fields:
   - top-1 stale entry per `recurring-rules.md` §4b → `{N days_stale}`, `{date from lastReferenced}`, `{stale_summary}`
   - `{total_after}` = active entry count in index
   - `{health_score}` = `index.stats.healthScore`
   - `{streak}` per `recurring-rules.md` §6a
   - `{active_modes_csv}` = comma-joined `config.activeModes`
   - `{next_due}` = mode with smallest remaining interval per §1 of this prompt
4. Emit (omit any line whose source is null; if no stale, omit the entire `✨ From your memory:` block):

```
💭 No modes due — skipped reflection
✨ From your memory:
   {N} days ago ({date}), {stale_summary}.
📈 Memory: {total_after} entries · Health {health_score}/100 · Streak: {streak}
⚙️ Active modes: {active_modes_csv} · Next due: {next_due.mode} in {next_due.in}
```

5. Stop the cycle.

## Step 0.5 — Snapshot BEFORE

```
BEFORE_SNAPSHOT = {
  rtmemory_lines, rtmemory_sections, procedures_lines,
  episodes_count, index_active, index_archived
}
```

In memory only.

---

## Step 1 — Collect

Read each unconsolidated daily log. Skip small talk and unchanged content. Build per-candidate record:

```json
{
  "id": "c1",
  "summary": "...",
  "source": "memory/YYYY-MM-DD.md",
  "category": "decision|lesson|preference|procedure|obligation|relationship|identity|architecture|observation|status|trend",
  "referenceCount": 1,
  "uniqueSessionCount": 1,
  "uniqueDayCount": 1,
  "marker": null|"HIGH"|"PERMANENT"|"PIN"|"PREFERENCE"|"ROUTINE"|"PROCEDURE",
  "target_section": "<RTMEMORY section name>",
  "existingId": null|"mem_NNN",
  "lastReferenced": "YYYY-MM-DDTHH:MM:SS+TZ",
  "created": "YYYY-MM-DD",
  "tags": []
}
```

Required: `id`, `summary`, `source`, `category`, `referenceCount`, `uniqueSessionCount`, `uniqueDayCount`, `marker`, `target_section`. Required when matched: `existingId`, `lastReferenced`, `created`.

Counter math when candidate matches existing index entry:
- `referenceCount += 1`
- `lastReferenced = today`
- if `source` not in `sessionSources`: `uniqueSessionCount += 1`; append `source` (cap 30)
- if `day` (YYYY-MM-DD from source) not in `uniqueDaySources`: `uniqueDayCount += 1`; append `day` (cap 30)

## Step 1.2 — Load deferred state (STRICT_MODE only)

Read `runtime/reflections-deferred.jsonl` → `DEFERRED_RECORDS` per `references/deferred-store.md` §6.

## Step 1.3 — Annotate deferred (STRICT_MODE only)

For each candidate: compose `identityKey` per `deferred-store.md` §3, look up against `DEFERRED_RECORDS` per §4. Set `identityKey`, `deferred_status ∈ {persisted, fresh}`, `deferred_matched_by`.

## Step 1.5 — Branch

| STRICT_MODE | Next |
|-------------|------|
| false | Step 2 |
| true | Step 1.6 |

## Step 1.6 — Score + Gate (STRICT_MODE only)

For each candidate with `deferred_status != "persisted"`:
1. Compute `importance` per `references/recurring-rules.md` §1.
2. Apply per-mode AND gate per §2 (strictest first, with PERMANENT hard / fast-path soft bypass).
3. Write back `importance`, `gate_status`, `gate_promoted_by`, `gate_bypass`, `gate_fail_reasons`.

For each `gate_status == "deferred"`: append record to `runtime/reflections-deferred.jsonl` per `references/deferred-store.md` §5.

If `DURABILITY_ON == false` → Step 2.

## Step 1.7 — Durability annotation (DURABILITY_ON only)

Scope per `references/recurring-rules.md` §3a (rescue eligibility for deferred candidates).

Emit one annotation record per in-scope candidate:

```json
{
  "candidate_id": "<id from candidate>",
  "memory_type": "decision|lesson|preference|procedure|obligation|relationship|identity|architecture|observation|status|trend",
  "durability_class": "durable|semi-durable|volatile|noise",
  "changed_future_decision": true|false,
  "changed_behavior_or_policy": true|false,
  "created_stable_preference": true|false,
  "created_obligation_or_boundary": true|false,
  "relationship_or_identity_shift": true|false,
  "cross_day_relevance": true|false,
  "rare_high_consequence": true|false,
  "actionable_procedure": true|false,
  "pattern_only": true|false,
  "pure_status": true|false,
  "telemetry_noise": true|false,
  "duplicate_of_existing": "mem_NNN"|null,
  "merge_key": "stable-slug"|null,
  "trend_key": "stable-slug"|null,
  "explanation": "short rationale (telemetry only)"
}
```

All booleans must be explicit `true` or `false`. Missing → router treats as `false`.

## Step 1.8 — Durability routing (DURABILITY_ON only)

For each annotated candidate: apply routing precedence per `references/recurring-rules.md` §3b–§3h (net-score components, hard triggers, routing precedence, destination map, profile thresholds, fields written).

For each `route == "defer"`: append to deferred store per `references/deferred-store.md` §5.

---

## Step 2 — Stage consolidations

Per route, mutate **in-memory only** (`STAGED_INDEX` + `STAGED_SURFACE_OPS`).

| `route` | Staged action |
|---------|---------------|
| `promote` | Add to `STAGED_INDEX.entries` per `references/index-operations.md` §3.1. Queue surface op: `(file, section, "- [mem_NNN] (YYYY-MM-DD) <summary>")`. |
| `merge` | Apply `reinforce_entry` per §3.3 to target inside `STAGED_INDEX`. No surface op. |
| `compress` | Apply `compress_trend` per §3.4 inside `STAGED_INDEX`. Queue surface op: upsert `### <trendKey>` in `TRENDS.md`. |
| `defer` | No staging (already persisted Step 1.6 / 1.8). |
| `reject` | No staging. |

**Eligibility per flow:**

| Flow | Eligible candidates |
|---|---|
| `STRICT_MODE == false` (parity) | every extracted candidate |
| `STRICT_MODE == true` AND `DURABILITY_ON == false` | `deferred_status != "persisted"` AND `gate_status == "qualified"` |
| `DURABILITY_ON == true` | dispatch by `route` field set in Step 1.8 (table above) |

**Dedup rule (parity + strict-without-durability):** before staging an eligible candidate as `promote`, check `existingId`:

- `existingId` set AND target entry is active in `STAGED_INDEX` → apply `update_session` per `references/index-operations.md` §3.2 (counters bump in place); **count this as `updated` in DELTA**, not new. No surface append.
- `existingId` null OR target archived → stage as new `promote` via semantic surface routing per `references/recurring-rules.md` §3f: reflective conclusions / durable preferences / decisions / lessons / obligations / relationships / identity / architecture → `RTMEMORY.md`; repeatable actionable know-how → `PROCEDURES.md`; bounded multi-event narratives → `episodes/<name>.md`. Fallback → `RTMEMORY.md`.

This prevents obvious duplication when the same content is re-extracted across cycles. Strict+durability already handles this via the explicit `merge` route.

Pass durability fields per `references/memory-template.md` §2.3 when present.

## Step 2.5 — Snapshot AFTER (from staged state)

Recompute the same metrics from `BEFORE_SNAPSHOT + STAGED_SURFACE_OPS` and `STAGED_INDEX`. Compute:

```
DELTA = {
  new:      count of route == "promote" (durability) OR new-promote stages (other flows),
  updated:  count of route == "merge" (durability) OR existingId-matched stages (other flows),
  archived: 0  (filled by Step 3),
  index_delta: AFTER.index_active - BEFORE.index_active
}
```

## Step 2.7 — Stage log markings

Build `LOGS_TO_MARK` per `references/recurring-rules.md` §5 (terminal-state rule). No marking happens here — Step 3.10 commits the markers.

## Step 2.8 — Stale

Per `references/recurring-rules.md` §4b. Top-3 (threshold 14d) → `STALE`. Read-only.

---

## Step 3 — Stage health + archival

1. Compute `HEALTH = {score, rating, metrics}` per `references/recurring-rules.md` §4a against `STAGED_INDEX`.
2. Recompute `importance` for each active entry per `recurring-rules.md` §1.
3. For each archival-eligible entry per `recurring-rules.md` §4c: mark archived in `STAGED_INDEX`; queue surface ops; `DELTA.archived += 1`.

In memory only.

## Step 3.5 — Insights

Generate 1–2 insights from `HEALTH`, route counters, `STALE` → `INSIGHTS`.

## Step 3.6 — Stage stats payload

Apply `update_stats` semantics per `references/index-operations.md` §3.6 to `STAGED_INDEX` (sets `lastDream`, recomputes `totalEntries` / `avgImportance`, appends `healthHistory` cap-90, sets `gateStats`).

`gateStats` payload:

```json
{
  "lastCycleQualified": N, "lastCycleDeferred": N,
  "lastCycleBreakdown": {"rem": N, "deep": N, "core": N},
  "lastCycleDurable":   {"promoted": N, "merged": N, "compressed": N, "deferred": N, "rejected": N}
}
```

In parity / strict-without-durability mode, omit `lastCycleDurable` (or zero its counters).

---

## Step 3.7 — Pre-commit checks

| Check | Pass condition |
|-------|---------------|
| Writability | All target files writable (surfaces, index, daily logs, log file, telemetry file) |
| `.bak` paths | All `.bak` destinations writable |
| Index coherence | `STAGED_INDEX.lastDream` set; `healthHistory[-1].date == today`; unique entry ids; counts match |
| Reference conflict | None detected this cycle |

Any fail → blocker. No writes performed.

Take `.bak` of every file about to be written: each surface in `STAGED_SURFACE_OPS`, `runtime/reflections-metadata.json`, `memory/.reflections-archive.md`, `memory/.reflections-log.md`. (Daily-log markers are append-only; no `.bak` taken.)

## Step 3.8 — Surface commit (FIRST DURABLE WRITE)

Surfaces commit before the index so what the operator sees is the durable artifact; the index is then derived from surfaces. Order:

1. For each unique file in `STAGED_SURFACE_OPS`: apply staged ops (appends, removals, trend upserts).
2. Append staged archive lines to `memory/.reflections-archive.md`.
3. Append cycle entry to `memory/.reflections-log.md` per `references/runtime-templates.md` §3.

On failure mid-sequence: restore touched `.bak`'s in reverse order. Skip Steps 3.9–3.11. Blocker.

## Step 3.9 — Persist index

Write `STAGED_INDEX` to `runtime/reflections-metadata.json` (indented, trailing newline).

On failure: restore `runtime/reflections-metadata.json.bak` AND restore all surface `.bak`'s from Step 3.8 (atomic rollback — both surfaces and index revert to pre-cycle state). Skip Steps 3.10–3.11. Blocker.

## Step 3.10 — Mark processed daily logs

For each in `LOGS_TO_MARK`: append `<!-- consolidated -->`; re-read to confirm marker present.

On failure: blocker; markers may be partial. Surfaces + index remain durable. Re-run is safe — Step 2 dedup rule (`existingId` matched → reinforce, not new) prevents surface duplication; strict-mode `gate_status` re-evaluation produces the same outcome.

## Step 3.11 — Telemetry

Compose mode-aware payload per `references/telemetry-schema.md` §4. Resolve `token_usage` per `references/recurring-rules.md` §7b (exact → approximate → unavailable). Append one compact JSON line + `\n` to `$TELEMETRY_ROOT/memory-log-<today>.jsonl`. Always written regardless of `sendReport`; the envelope always carries `token_usage` (nulls when unavailable).

On failure: fallback to `$TELEMETRY_ROOT/memory-log-<today>.jsonl.fallback` or stderr. Cycle still considered committed.

---

## Step 4.1 — Notify gate

Read `runtime/memory-state.json`. If `reflections.reporting.sendReport != true` → skip Step 4.2 → Step 5.

## Step 4.2 — Send notification

Pre-compute these in-memory values:

```
REFLECTION_COUNT = len(STAGED_INDEX.stats.healthHistory)   # Step 3.6 already appended this cycle
TOTAL_BEFORE     = BEFORE_SNAPSHOT.index_active
TOTAL_AFTER      = AFTER_SNAPSHOT.index_active
PCT_GROWTH       = per references/recurring-rules.md §6a
STREAK           = per references/recurring-rules.md §6a
```

### Check for milestones

| Condition | Banner |
|-----------|--------|
| `REFLECTION_COUNT == 1` | 🎉 First reflection complete! |
| `REFLECTION_COUNT == 7` OR `STREAK == 7` | 🏅 One week streak! |
| `REFLECTION_COUNT == 30` OR `STREAK == 30` | 🏆 One month streak! |
| `TOTAL_BEFORE < 100 <= TOTAL_AFTER` | 📊 Memory milestone! 100 entries. |
| `TOTAL_BEFORE < 200 <= TOTAL_AFTER` | 📊 Memory milestone! 200 entries. |
| `TOTAL_BEFORE < 500 <= TOTAL_AFTER` | 📊 Memory milestone! 500 entries. |
| `TOTAL_BEFORE < 1000 <= TOTAL_AFTER` | 📊 Memory milestone! 1000 entries. |

### Is today Sunday? → Add weekly summary

Trigger if **either**:
- Local time is Sunday AND hour:minute ≥ 18:30
- OR `REFLECTION_COUNT % 7 == 0` (fallback for missed Sunday runs)

When triggered, compute weekly stats per `references/recurring-rules.md` §6c from `STAGED_INDEX`. If `weekly_snapshot_available == false` (fresh install) → skip the weekly block. Do not emit placeholders.

Prepend weekly block to the notification:

```
📊 Weekly Report ({date_range})

🧠 This week: +{weekly_new} new · {weekly_updated} updated · {weekly_archived} archived
   {total_before_week} → {TOTAL_AFTER} entries ({percent}% growth)

🌟 Biggest memories this week:
   1. {biggest_memories[0].summary}
   2. {biggest_memories[1].summary}
   3. {biggest_memories[2].summary}
```

If fewer than 3 biggest memories returned, render only entries present.

### Notification format

```
💭 Reflection #{REFLECTION_COUNT} complete ({MODE_CSV joined with +} cycle)

📥 Consolidated: {DELTA.new + DELTA.updated} entries from {len(unconsolidated)} logs
   new: {DELTA.new} · updated: {DELTA.updated} · archived: {DELTA.archived}
📈 Total: {TOTAL_BEFORE} → {TOTAL_AFTER} entries ({PCT_GROWTH} growth)
🧠 Health: {HEALTH.score}/100 — {HEALTH.rating}

✨ Highlights:
   • {change_1}
   • {change_2}

💡 Insight: {top insight}

⏳ Stale: {stale items if any}

🪙 Token Usage: {token line per recurring-rules §7d}

{milestone if any}
💬 Let me know if anything was missed
```

LLM-authored fields (the four `{...}` not in the pre-compute table):

| Placeholder | Source |
|-------------|--------|
| `{change_1}` / `{change_2}` (✨ Highlights) | 1–2 short bullets from this cycle's `route == "promote"` candidates |
| `{top insight}` | top line from `INSIGHTS` |
| `{stale items if any}` | one line from `STALE[0]` — omit the entire `⏳ Stale:` line if `STALE` is empty |
| `🪙 Token Usage line` | per `references/recurring-rules.md` §7d — `Token Usage:` (exact) or `Token Usage (approx):` (approximate); **omit the entire line** when `source == "unavailable"` |
| `{milestone if any}` | banners from milestone table above; emit each on its own line; omit the line if no milestone fires |
| `💬 Let me know if anything was missed` | replace with a varied warm closing each cycle (e.g. "Flag anything that didn't land.", "Tell me if I missed a beat."). Do not include numbers. |

**Omission rule:** if a placeholder source is null, omit the line. Never invent a number.

This reply is your ONLY output. Concise and high-value. No narration. No meta-commentary.

## Step 5 — Update lastRun

For each mode in `MODE_CSV`: set `config.lastRun.<mode> = now` (full ISO with timezone). `.bak reflections.json` → save. On failure: see Blocker Handling.

---

## Blocker Handling

On any failure, missing config, unwritable workspace, or runtime/reference rubric conflict:

1. Append `run_failed` telemetry per `references/telemetry-schema.md` §4.6 with `status: error`, `details.step`, `details.blocker_type ∈ {config_missing, workspace_unwritable, rubric_conflict, surface_rollback, index_rollback, other}`.
   - If telemetry write itself fails: fallback to `$TELEMETRY_ROOT/memory-log-<today>.jsonl.fallback` or stderr. Do not silently drop.
2. If `sendReport == true`: emit one short blocker message naming the failed step and recovery action.
3. Recovery by phase:

| Phase failed | Rollback action | Logs marked | Re-run safe? |
|---|---|---|---|
| Stage (0 → 3.7) | none — no writes attempted | no | yes |
| Step 3.8 (surfaces) | restore touched surface `.bak`'s in reverse | no | yes |
| Step 3.9 (index) | restore index `.bak` AND all surface `.bak`'s from 3.8 | no | yes — atomic rollback to pre-cycle |
| Step 3.10 (markers) | none — markers may be partial | partial | yes — Step 2 dedup rule prevents surface dup; matched candidates reinforce instead of new-promote |
| Step 3.11 (telemetry) | none — fallback append to stderr/`.fallback` | yes | cycle still considered committed |
