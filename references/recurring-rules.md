# Recurring Rules — Compact Card for Cron Execution

This is the **only rules doc the recurring runtime reads**. Long-form theory and worked examples (`scoring.md`, `durability.md`, `health.md`, `worked-examples.md`) are the **reference library** — for humans, debugging, and first-reflection bootstrap; not loaded on the cron path.

Authority: this card is the single source of truth for the recurring cycle. If it disagrees with a long-form doc, this card wins (long-form docs are derivations / examples that may drift).

---

## §1 Scoring formula (Step 1.6)

```
importance = clamp(base × recency × ref_boost / 8.0, 0.0, 1.0)
```

| Component | Rule |
|---|---|
| `base` | `PERMANENT` → final score 1.0 (skip the formula); `HIGH` → 2.0; `PIN` / `none` → 1.0 |
| `recency` | `max(0.1, 1.0 − days_elapsed / 180)` — table: 0d→1.00, 30d→0.83, 90d→0.50, 180d+→0.10 |
| `ref_boost` | `max(1.0, log2(referenceCount + 1))` — table: 0/1→1.00, 3→2.00, 7→3.00, 15→4.00 |

`PERMANENT` always returns 1.0 final regardless of components.

---

## §2 Gate rules (Step 1.6, STRICT_MODE only)

AND-gate per mode, evaluated **strictest first**: rem → deep → core. Once any mode qualifies a candidate, skip remaining modes.

A candidate **qualifies** when ALL three hold:

```
importance       >= mode.minScore
referenceCount   >= mode.minRecallCount
effective_unique >= mode.minUnique
```

`effective_unique` resolves from `mode.uniqueMode`:

| uniqueMode | Field |
|---|---|
| `day_or_session` (default) | `uniqueDayCount` if > 0, else `uniqueSessionCount` |
| `day` | `uniqueDayCount` |
| `session` | `uniqueSessionCount` |
| `channel` | `uniqueChannelCount` |
| `max` | max of day / session / channel |

Bypasses (set `gate_bypass`):

| Bypass | Trigger | gate_bypass value |
|---|---|---|
| Hard | marker == `PERMANENT` | `"PERMANENT"` |
| Soft | `(importance >= fastPathMinScore AND referenceCount >= fastPathMinRecallCount)` OR `marker in fastPathMarkers` | `"FAST_PATH"` |

Write back per candidate: `importance`, `gate_status ∈ {qualified, deferred}`, `gate_promoted_by` (mode name or null), `gate_bypass`, `gate_fail_reasons` (object keyed by mode when deferred).

---

## §3 Durability precedence (Steps 1.7 + 1.8, DURABILITY_ON only)

### 3a. Annotation scope (Step 1.7)

Annotate every candidate where `deferred_status != "persisted"` AND one of:
- `gate_status == "qualified"`, OR
- **rescue-eligible deferred** — `gate_status == "deferred"` AND any of: `gate_bypass` set; `marker in {HIGH, PERMANENT, PIN}`; semantic class is high-meaning (`decision`, `lesson`, `obligation`, `relationship`, `identity`, `architecture`).

### 3b. Net-score components (Step 1.8)

Each clamped 0..4:

```
structuralEvidence:
  gate_bypass in {PERMANENT, FAST_PATH} → 4
  else: +1 if refCount>=1, +1 if refCount>=3, +1 if uniqueDayCount>=2, +1 if uniqueDayCount>=4

meaningWeight:
  count of TRUE among {changed_future_decision, changed_behavior_or_policy,
                       created_stable_preference, created_obligation_or_boundary,
                       relationship_or_identity_shift}

futureConsequence:
  +1 cross_day_relevance, +2 rare_high_consequence, +1 actionable_procedure

noisePenalty:
  +1 pattern_only, +1 pure_status, +2 telemetry_noise

net = structuralEvidence + meaningWeight + futureConsequence − noisePenalty
```

### 3c. Hard-promote triggers (force `route = "promote"`)

Any one fires:
1. `memory_type in {decision, lesson}` AND (`changed_future_decision` OR `changed_behavior_or_policy`)
2. `created_stable_preference == true`
3. `created_obligation_or_boundary == true`
4. `relationship_or_identity_shift == true`
5. `actionable_procedure == true` AND `structuralEvidence >= 2`
6. `memory_type == "architecture"` AND `rare_high_consequence == true`

`promotionReason = "hard-trigger:<name>"`.

### 3d. Hard-suppress triggers (force `route = "reject"`)

Any one fires:
1. `telemetry_noise == true`
2. `pure_status == true` AND NOT `rare_high_consequence`
3. `pattern_only == true` AND NOT `cross_day_relevance`

`promotionReason = "hard-suppress:<reason>"`.

### 3e. Routing precedence (strict order, first match wins)

```
1. Hard-suppress fires                            → route = reject;   destination = NONE
2. Hard-promote fires                             → route = promote;  destination per §3f
   Trend-to-durable: if trendKey matches an existing trend node AND that trend has
   trendSupportCount >= trendPromoteSupportCount AND uniqueDayCount >= trendPromoteUniqueDayCount
   → set promotedFromTrend = <existing trend id>
3. duplicate_of_existing set AND resolves in index → route = merge;    mergedInto = id
4. duplicate_of_existing set AND NOT in index      → route = defer
5. trendKey set AND memory_type in {observation, status, trend}
   AND NO hard-trigger AND NOT actionable_procedure → route = compress; destination = TREND
6. Net-score banding:
     net >= netPromoteThreshold                    → route = promote
     net >= netDeferThreshold                      → route = defer
     else                                          → route = reject
```

### 3f. Destination map (only on promote / compress)

| memory_type | Destination on promote |
|---|---|
| decision, lesson, obligation, relationship, identity, architecture, preference | `RTMEMORY` |
| procedure (with hard-trigger validated_actionable_procedure AND structuralEvidence ≥ 2) | `PROCEDURES` |
| observation (with hard-trigger AND cross_day_relevance) | `EPISODE` |
| trend / status (when promoted via hard-trigger / trend-to-durable) | `RTMEMORY` |
| any (compress route) | `TREND` |
| any (merge / defer / reject) | `NONE` |
| fallback | `RTMEMORY` |

### 3g. Profile thresholds

| Profile | netPromoteThreshold | netDeferThreshold | trendPromoteSupportCount | trendPromoteUniqueDayCount |
|---|---|---|---|---|
| `business-employee` | 6 | 3 | 5 | 3 |
| `personal-assistant` (only if operator opts into strictMode) | 5 | 2 | 4 | 2 |

### 3h. Fields written per candidate (Step 1.8)

`route`, `destination`, `durabilityScore` (= net), `noisePenalty`, `promotionReason`, `memoryType`, `durabilityClass`, `mergeKey`, `trendKey`, `duplicateOfExisting`, `mergedInto`, `promotedFromTrend`, `compressionReason`, `supportCount`, `durabilityComponents`.

---

## §4 Health + Archival (Steps 2.8 + 3)

### 4a. 5-metric health score (Step 3)

```
health = (freshness × 0.25
       + coverage  × 0.25
       + coherence × 0.20
       + efficiency × 0.15
       + reachability × 0.15) × 100
```

Round to integer. Rating bands: `>=80 excellent`, `>=60 good`, `>=40 fair`, `>=20 poor`, else `critical`.

| Metric | Formula |
|---|---|
| freshness | `(active entries with lastReferenced >= today − 30d) / active total` |
| coverage | `(canonical RTMEMORY sections with non-empty non-comment content) / 10` |
| coherence | `(active entries with len(related) >= 1) / active total` |
| efficiency | `max(0, 1 − rtmemory_line_count / 500)` |
| reachability | `Σ(component_size²) / total_active²`, clamped to `[0, 1]` |

10 canonical sections: Scope Notes, Active Initiatives, Business Context and Metrics, People and Relationships, Strategy and Priorities, Key Decisions and Rationale, Lessons and Patterns, Episodes and Timelines, Environment Notes, Open Threads.

**Reachability BFS** (active entries only, undirected edges from `related`):
```
1. active_ids = {e.id for e in entries if not e.archived}
2. adj = {}; for each active e, for each rid in e.related where rid in active_ids:
   adj[e.id].add(rid); adj[rid].add(e.id)
3. components = []; visited = {}
   for id in active_ids:
     if id in visited: skip
     run BFS from id; record component size; mark visited
4. reachability = sum(size² for size in components) / (len(active_ids))²
```

Worked: 10 nodes in 2 components (sizes 7 and 3) → `(49+9)/100 = 0.58`.

### 4b. Stale detection (Step 2.8 + 0-B)

Threshold: 14 days. Top-N: 3 (cycle), 1 (skip-with-recall).

Sources, in order:
1. RTMEMORY.md "Open Threads" `- [ ]` items (use inline date; else `mem_NNN` lookup against index `lastReferenced`)
2. Index entries with `lastReferenced > today − 14d`, not archived

`days_stale = today − candidate_date`. Drop if `< 14`. Sort descending by `days_stale`. Return top-N.

### 4c. Archival — forgetting curve (Step 3)

A candidate entry in `STAGED_INDEX` is archival-eligible iff **all five** hold:

```
1. days_since_lastReferenced > 90
2. importance < 0.3
3. marker NOT IN {PERMANENT, PIN}
4. NOT in an episode file (episodes are append-only)
5. NOT already archived
```

Process per eligible entry (staged in memory; commit at Step 3.10):
1. Mark entry: `archived = true`, `archived_at = today` (YYYY-MM-DD); optionally compress `summary`.
2. Queue surface op: append `- [mem_NNN] (created → archived) <summary>` to `memory/.reflections-archive.md`.
3. Queue surface op: remove entry from source surface (RTMEMORY.md or PROCEDURES.md).
4. `DELTA.archived += 1`.

The index entry is **kept** (only marked archived) — needed for relation/reachability graph.

---

## §5 Processed-log rule (Step 2.7)

A daily log enters `LOGS_TO_MARK` iff **every** candidate it produced satisfies one of:

| Condition | Applies to |
|---|---|
| `route ∈ {promote, merge, compress, reject}` | strict + durability |
| `route == "defer"` AND `identityKey` was just appended to deferred store this cycle | strict + durability |
| staged this cycle | parity / strict-without-durability |
| recorded in deferred store this cycle | strict-without-durability gate-deferred |

Step 2.7 only **stages** the marking. The actual `<!-- consolidated -->` write happens at Step 3.10 after surfaces commit. Backstop: re-read each marked log to confirm marker present.

---

## §6 Report field provenance (Step 4.2)

### 6a. Pre-computed values (at the top of Step 4.2)

| Variable | Source |
|---|---|
| `REFLECTION_COUNT` | `len(STAGED_INDEX.stats.healthHistory)` (Step 3.6 already appended this cycle's entry) |
| `TOTAL_BEFORE` | `BEFORE_SNAPSHOT.index_active` |
| `TOTAL_AFTER` | `AFTER_SNAPSHOT.index_active` |
| `MODE_CSV` | comma-joined `due_modes` (display joined with `+`) |
| `STREAK` | count consecutive distinct calendar days backward from latest `healthHistory` date; gap ≥ 2 days breaks |
| `PCT_GROWTH` | `round((TOTAL_AFTER − TOTAL_BEFORE) / TOTAL_BEFORE × 100, 1)` with sign; `"+∞%"` if TOTAL_BEFORE = 0 and TOTAL_AFTER > 0; `"0.0%"` if both 0 |

### 6b. Milestones

| Condition | Banner |
|---|---|
| `REFLECTION_COUNT == 1` | `🎉 First reflection complete!` |
| `REFLECTION_COUNT == 7` OR `STREAK == 7` | `🏅 One week streak!` |
| `REFLECTION_COUNT == 30` OR `STREAK == 30` | `🏆 One month streak!` |
| `TOTAL_BEFORE < 100 <= TOTAL_AFTER` | `📊 Memory milestone! 100 entries.` |
| `TOTAL_BEFORE < 200 <= TOTAL_AFTER` | `📊 Memory milestone! 200 entries.` |
| `TOTAL_BEFORE < 500 <= TOTAL_AFTER` | `📊 Memory milestone! 500 entries.` |
| `TOTAL_BEFORE < 1000 <= TOTAL_AFTER` | `📊 Memory milestone! 1000 entries.` |

Multiple banners may fire in one cycle; emit each on its own line.

### 6c. Weekly trigger + math

Trigger: today is **Sunday AND local time ≥ 18:30**, OR `REFLECTION_COUNT % 7 == 0`.

When triggered, compute against `STAGED_INDEX`:

```
end_date           = today (YYYY-MM-DD)
start_date         = end_date − 7 days
date_range         = "<start_date> – <end_date>"

active             = entries where archived != true
weekly_new         = count of active where created >= start_date
weekly_updated     = count of active where lastReferenced >= start_date AND created < start_date
weekly_archived    = count where archived_at >= start_date
total_before_week  = TOTAL_AFTER − weekly_new + weekly_archived
percent (weekly)   = round((TOTAL_AFTER − total_before_week) / total_before_week × 100, 1)

biggest_memories   = top-3 by importance among (created >= start_date OR lastReferenced >= start_date)
                     each: {id, summary, importance}
```

`weekly_snapshot_available` = `len(active) > 0 AND (weekly_new + weekly_updated + weekly_archived) > 0`. If false, omit the weekly block (do not emit placeholders).

### 6d. Notification field provenance

| Placeholder | Source |
|---|---|
| `Reflection #N` | `REFLECTION_COUNT` |
| `({modes_fired} cycle)` | `due_modes` joined with `+` |
| `{E} entries` (Consolidated count) | `DELTA.new + DELTA.updated` |
| `{L} logs` | `len(unconsolidated)` |
| `new` / `updated` / `archived` | `DELTA.new` / `DELTA.updated` / `DELTA.archived` |
| `Total: {BEFORE} → {AFTER}` | `TOTAL_BEFORE` / `TOTAL_AFTER` |
| `({pct}% growth)` | `PCT_GROWTH` |
| `Health: {score}/100 — {rating}` | `HEALTH.score` / `HEALTH.rating` |
| `{milestone if any}` | per §6b (multi-line OK) |

### 6e. LLM-authored fields (semantic only)

| Placeholder | Source |
|---|---|
| `✨ Highlights` (1–2 bullets) | this cycle's `route == "promote"` candidates — distill into short bullets |
| `💡 Insight` | top line from `INSIGHTS` (Step 3.5) |
| `⏳ Stale` | one line from `STALE[0]` (Step 2.8); omit the entire `⏳ Stale:` line if `STALE` empty |
| `💬 closing` | warm closing — vary wording each cycle, never include numbers |

### 6f. Omission rule

If a placeholder source is null, **omit the entire line** containing it. Never invent a number. Never emit placeholder text like `{...}` literally.

---

## §7 Token usage (visibility only — no behavior change)

Token usage is a **visibility field** — it appears in telemetry, the cycle log, and (when available) the final notification. It never affects scoring, gating, deferring, durability routing, merge/compress/trend, log marking, or archive behavior.

### 7a. Shape

```json
"token_usage": {
  "prompt_tokens":     <int> | null,
  "completion_tokens": <int> | null,
  "total_tokens":      <int> | null,
  "source":            "exact" | "approximate" | "unavailable"
}
```

### 7b. Source resolution (strict ladder)

| Ladder | Condition | Fields | `source` |
|---|---|---|---|
| 1 | Host/runtime exposes per-turn token metadata for this cycle | fill `prompt_tokens`, `completion_tokens`, `total_tokens` from that metadata | `"exact"` |
| 2 | Only `total_tokens` available from metadata | fill `total_tokens`; leave `prompt_tokens` / `completion_tokens` null | `"exact"` |
| 3 | Metadata unavailable but character counts of prompt input + model output are known | apply §7c approximation | `"approximate"` |
| 4 | Nothing known | leave all three numbers null | `"unavailable"` |

**Never fabricate `"exact"`.** If the value didn't come from host-provided metadata, it is not exact.

### 7c. Approximation rule (single fallback — use consistently)

```
estimated_tokens = ceil(char_count / 4)
```

Where `char_count` is the total characters counted over the same text the token count should cover (e.g. prompt input chars for `prompt_tokens`, model output chars for `completion_tokens`). Use the same divisor (4) everywhere. Label every approximation with `source: "approximate"`.

### 7d. Final notification line (Step 4.2)

When `source in {exact, approximate}`, emit one compact line using the 🪙 emoji:

- All three values present:
  - `🪙 Tokens: {prompt_tokens} in / {completion_tokens} out / {total_tokens} total`
- Only total present:
  - `🪙 Tokens: {total_tokens}`
- Approximate:
  - Replace `Tokens:` with `Tokens (approx):` — applies to either variant above.

When `source == "unavailable"` → **omit the line entirely**. Do not emit placeholders.

### 7e. Cycle log entry

Append a matching line to the cycle log entry per `runtime-templates.md` §3:

- `🪙 Tokens:` / `🪙 Tokens (approx):` same formatting as §7d.
- Omit when unavailable.

### 7f. Weekly rollup

Only when historical cycles already carry `token_usage`. Sum `total_tokens` across the week's cycles where either `"exact"` or `"approximate"` is set; emit a single line in the weekly block:

```
🪙 Weekly tokens: {sum_total_tokens} total
```

If **any** cycle in the window had `"approximate"`, label the line `🪙 Weekly tokens (approx): ...`. If no cycle carries token data, omit the line. Never backfill historical counts.

### 7g. Provenance rule

- Token numbers **must** come from runtime-provided usage metadata when available (ladder step 1 or 2).
- Otherwise use the §7c approximation, clearly labeled.
- Otherwise **omit**. Never invent.

---

## Conflict handling

If this card disagrees with a long-form doc, **this card wins** for the recurring path. Long-form docs (scoring.md / durability.md / health.md / worked-examples.md) are derivations or examples that exist for humans and may drift; the card is the load-bearing authority for cron execution.

If this card disagrees with the runtime prompt's execution order, **stop and emit a blocker** per the runtime's precedence rule. Card defines rules; runtime defines order.
