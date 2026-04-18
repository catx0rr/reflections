# Index Operations — Metadata CRUD with BEFORE/AFTER Diffs

This document is the **rubric authority** for any operation that mutates `runtime/reflections-metadata.json`. Steps 2 (consolidate), 3 (archival), and 3.6 (persist stats) cite it.

The runtime always rewrites the whole file (full-file rewrite, not patch). The file is small (typically < 500 entries × small object). Before any rewrite, copy the file to `runtime/reflections-metadata.json.bak`.

---

## 1. File schema

```json
{
  "version": "1.0.0",
  "lastDream": "2026-04-18T14:30:14Z",
  "entries": [
    { ... per-entry record (see §2) ... }
  ],
  "stats": {
    "totalEntries": 142,
    "avgImportance": 0.4863,
    "lastPruned": "2026-04-15",
    "healthScore": 78,
    "healthMetrics": {
      "freshness": 0.82, "coverage": 0.70, "coherence": 0.45,
      "efficiency": 0.62, "reachability": 0.58
    },
    "insights": ["Insight 1", "Insight 2"],
    "healthHistory": [
      {"date": "2026-04-17", "score": 76},
      {"date": "2026-04-18", "score": 78}
    ],
    "gateStats": {
      "lastCycleQualified": 6,
      "lastCycleDeferred": 3,
      "lastCycleBreakdown": {"rem": 2, "deep": 2, "core": 2},
      "lastCycleDurable": {
        "promoted": 3, "merged": 1, "compressed": 1, "deferred": 1, "rejected": 0
      }
    }
  }
}
```

`gateStats.lastCycleDurable` is only populated when `durability.enabled == true`. Parity / strict-without-durability cycles omit it (or set all counters to 0).

---

## 2. Per-entry schema

A regular durable entry:

```json
{
  "id": "mem_042",
  "summary": "One-line summary of the memory",
  "source": "memory/2026-04-01.md",
  "target": "RTMEMORY.md#key-decisions-and-rationale",
  "created": "2026-04-01",
  "lastReferenced": "2026-04-18",
  "referenceCount": 7,
  "uniqueSessionCount": 4,
  "sessionSources": ["memory/2026-04-01.md", "memory/2026-04-02.md", "memory/2026-04-04.md", "memory/2026-04-18.md"],
  "uniqueDayCount": 4,
  "uniqueDaySources": ["2026-04-01", "2026-04-02", "2026-04-04", "2026-04-18"],
  "importance": 0.82,
  "tags": ["decision", "billing"],
  "related": ["mem_018", "mem_039"],
  "archived": false
}
```

Optional durability fields (set when the entry came from strict+durability flow):

```json
{
  "memoryType": "decision",
  "durabilityClass": "durable",
  "route": "promote",
  "destination": "RTMEMORY",
  "durabilityScore": 8,
  "noisePenalty": 0,
  "promotionReason": "hard-trigger:decision-with-consequence",
  "supportCount": 2,
  "mergeKey": null,
  "trendKey": null,
  "duplicateOfExisting": null,
  "promotedFromTrend": null,
  "compressionReason": null,
  "mergeKeys": [],
  "reinforcedBy": []
}
```

Trend-specific fields (set when the entry is a trend node, `memoryType == "trend"`):

```json
{
  "trendKey": "dev-server-noon-restart",
  "trendFirstObserved": "2026-04-12",
  "trendLastUpdated": "2026-04-18",
  "trendSupportCount": 6,
  "trendSources": ["memory/2026-04-12.md", "memory/2026-04-13.md", ...],
  "sourceCount": 6
}
```

Archived entries:

```json
{
  "archived": true,
  "archived_at": "2026-04-18"
}
```

---

## 3. Operations

For each operation: BEFORE state, action, AFTER state, invariants. The runtime always copies the file to `.bak` before writing.

### 3.1 add_entry (route = promote, new node)

When Step 2 promotes a new candidate and the candidate isn't a duplicate.

**BEFORE:** index has N entries. The next available id is `mem_NNN+1`.

**Action:**

1. Compute `next_id` = largest `mem_NNN` integer + 1, formatted as `mem_NNN` (zero-padded to 3 digits at minimum). If no `mem_*` entries exist, `next_id = mem_001`.
2. Build the new entry:
   - Set `id = next_id` if not provided
   - Set `created = today` (YYYY-MM-DD) if not provided
   - Set `lastReferenced = today` if not provided
   - Set `referenceCount = 1` if not provided
   - Set `uniqueSessionCount = 1` if not provided
   - Set `sessionSources = [source]` if not provided
   - Extract day from `source` (YYYY-MM-DD substring); set `uniqueDayCount = 1`, `uniqueDaySources = [day]` (or 0 / [] if no day extractable)
   - Set `importance = 0.5` if not provided (will be recomputed in Step 3 archival pass)
   - Set `tags = []`, `related = []`, `archived = false` if not provided
   - Pass through any durability fields (`memoryType`, `route`, `destination`, `durabilityScore`, etc.) unchanged.
3. Append the entry to `index.entries`.

**AFTER:** index has N+1 entries; the new entry is the last in the array.

**Invariants:**
- `id` is unique across all entries
- `archived` is `false`
- `created` and `lastReferenced` are valid ISO dates

**Pre-write:** copy `runtime/reflections-metadata.json` to `runtime/reflections-metadata.json.bak`.

### 3.2 update_session (any cycle that re-references an entry)

When Step 1 collects a candidate that matches an existing entry (`existingId` is set), the runtime calls update_session to bump counters.

**BEFORE:** entry has `referenceCount = R`, `uniqueSessionCount = US`, `uniqueDayCount = UD`, `lastReferenced = old_date`.

**Action:**

1. Find entry by id.
2. `referenceCount += 1`
3. `lastReferenced = today` (YYYY-MM-DD)
4. If `source_log` not in `sessionSources`:
   - `uniqueSessionCount += 1`
   - Append `source_log` to `sessionSources`
   - Cap `sessionSources` at 30 most-recent entries (drop from the front)
5. Extract `day` from `source_log`.
6. If `day` is non-empty AND not in `uniqueDaySources`:
   - `uniqueDayCount += 1`
   - Append `day` to `uniqueDaySources`
   - Cap `uniqueDaySources` at 30 most-recent entries

**AFTER:** entry has updated counters and `lastReferenced`.

**Invariants:**
- `len(sessionSources) <= 30` and `len(uniqueDaySources) <= 30`
- Counters only increase, never decrease

**Pre-write:** copy `runtime/reflections-metadata.json` to `.bak` before saving the file at end of cycle.

### 3.3 reinforce_entry (route = merge)

When Step 1.8 routes a candidate to `merge`, Step 2 calls reinforce_entry on the target (`mergedInto`).

**BEFORE:** target entry has `referenceCount = R`, `mergeKeys = [...]`, `reinforcedBy = [...]`.

**Action:**

1. Find target entry by id.
2. Delegate counter math to update_session (3.2) — bumps `referenceCount`, `lastReferenced`, sessionSources, uniqueDayCount.
3. If `mergeKey` is provided AND not in `mergeKeys`:
   - Append `mergeKey` to `mergeKeys`
4. Append to `reinforcedBy`:
   ```json
   {
     "source": "<source log path>",
     "mergeKey": "<the mergeKey>",
     "mergeReason": "<the promotionReason from durability, e.g. merge-into:mem_042>",
     "timestamp": "<ISO 8601 with timezone>"
   }
   ```
5. Cap `reinforcedBy` at 50 most-recent events (drop from the front).
6. If a refined `summary` is provided, replace the target's `summary`.

**AFTER:** target has bumped counters, mergeKey appended (deduped), reinforcedBy appended.

**Invariants:**
- `len(reinforcedBy) <= 50`
- `mergeKeys` has no duplicate strings
- No new entry is created

**Pre-write:** `.bak`.

### 3.4 compress_trend (route = compress)

When Step 1.8 routes a candidate to `compress`, Step 2 calls compress_trend with the `trendKey`.

**BEFORE:** index may or may not contain an active trend node with `memoryType == "trend"` AND matching `trendKey`.

**Action — case 1: existing trend found:**

1. Reinforce via update_session (3.2) on the trend's id.
2. `trendSupportCount += 1`
3. `trendLastUpdated = today` (YYYY-MM-DD)
4. If `source` provided AND not in `trendSources`:
   - Append `source` to `trendSources`
   - Cap `trendSources` at 200 most-recent entries
5. Recompute `sourceCount = len(trendSources)`.
6. If a refined `summary` is provided, replace the trend's `summary`.
7. Also upsert the `### <trendKey>` section in `TRENDS.md` (separate file edit; see §4 for surface format).

**Action — case 2: no existing trend:**

1. Get `next_id` per add_entry (3.1).
2. Build new trend entry:
   ```json
   {
     "id": "<next_id>",
     "summary": "<from payload>",
     "source": "<source>",
     "target": "TRENDS.md",
     "memoryType": "trend",
     "durabilityClass": "semi-durable",
     "route": "compress",
     "destination": "TREND",
     "trendKey": "<trendKey>",
     "trendFirstObserved": "<today>",
     "trendLastUpdated": "<today>",
     "trendSupportCount": 1,
     "trendSources": ["<source>"],
     "sourceCount": 1,
     "compressionReason": "<from payload, e.g. new-trend:<key>>",
     "tags": [],
     "related": [],
     "archived": false
   }
   ```
3. Apply add_entry (3.1) defaults (referenceCount, uniqueDayCount, etc.).
4. Append to `index.entries`.
5. Upsert the new `### <trendKey>` section in `TRENDS.md`.

**AFTER:** trend node exists with bumped or initialized counters; `TRENDS.md` reflects the upserted section.

**Invariants:**
- Only one active trend node per `trendKey`
- `len(trendSources) <= 200`
- `trendFirstObserved` is set on creation and never modified after

**Pre-write:** `.bak`.

### 3.5 archive_entry (Step 3 archival)

When Step 3 identifies an archival-eligible entry (per `health.md` §4 forgetting curve).

**BEFORE:** entry is active (`archived: false`).

**Action:**

1. Find entry by id.
2. Set `archived = true`.
3. Set `archived_at = today` (YYYY-MM-DD).
4. Optionally replace `summary` with a more compressed one-liner (the runtime usually does this when transferring to the archive surface).
5. Append the corresponding archive line to `memory/.reflections-archive.md`:
   ```
   - [mem_NNN] (YYYY-MM-DD → YYYY-MM-DD) One-line summary
   ```
6. Remove the full entry from its source surface (RTMEMORY.md or PROCEDURES.md).

**AFTER:** entry is archived; remains in the index (for relation/reachability); is gone from the surface; appears in the archive file.

**Invariants:**
- `archived == true`
- `archived_at` is a valid ISO date
- Entry is NOT removed from `index.entries`
- Source-surface entry is removed
- Archive file has one new line

**Pre-write:** `.bak`.

### 3.6 update_stats (Step 3.6 stages it in memory; Step 3.8 persists)

The end-of-cycle write that records this cycle's health, insights, and counters.

**BEFORE:** `index.lastDream` is from the previous cycle. `index.stats` may be stale.

**Action:**

1. Set `index.lastDream = now` (full ISO timestamp with timezone).
2. Compute `active = entries where archived != true`.
3. `index.stats.totalEntries = len(active)`.
4. If active is non-empty: `index.stats.avgImportance = round(sum(e.importance for e in active) / len(active), 4)`. Else 0.
5. Merge from the provided stats payload — set if present:
   - `index.stats.healthScore`
   - `index.stats.healthMetrics`
   - `index.stats.insights`
   - `index.stats.gateStats` (replace whole object)
6. If `healthScore` was provided this cycle:
   - Append `{"date": "today YYYY-MM-DD", "score": <score>}` to `index.stats.healthHistory`
   - Cap history at 90 most-recent entries (drop from the front)
7. Save the file.

**AFTER:** index has fresh stats, lastDream timestamp, and a new healthHistory entry.

**Invariants:**
- `lastDream` is a valid ISO 8601 with timezone
- `len(healthHistory) <= 90`
- `totalEntries` matches the count of non-archived entries
- `healthHistory[-1].date == today` (this cycle's date)

**Staging vs persistence (per the runtime):** Step 3.6 stages the new stats into the in-memory `STAGED_INDEX`. Step 3.7 verifies `STAGED_INDEX.lastDream` and `healthHistory[-1].date == today`. Step 3.8 is the actual durable write of `runtime/reflections-metadata.json`. Step 3.9 then writes telemetry. Surfaces commit only after both succeed (Step 3.10).

**Pre-write:** `.bak`.

---

## 4. Surface upserts (TRENDS.md)

When `compress_trend` runs, also upsert the `### <trendKey>` section in `TRENDS.md`. The format:

```markdown
### dev-server-noon-restart

_First observed: 2026-04-12. Last updated: 2026-04-18. Support: 6 across 4 days._

Dev server restarts around noon each day — no clear trigger identified yet.

**Sources:** memory/2026-04-12.md, memory/2026-04-13.md, memory/2026-04-15.md, memory/2026-04-18.md
```

If the section already exists, replace it in place. Otherwise append a new section.

The file header (created on first install) is:

```markdown
# Trends

_Observed patterns without stable method._

---
```

---

## 5. Backup discipline

Every operation that mutates `runtime/reflections-metadata.json` must:

1. **Before write:** copy current file to `runtime/reflections-metadata.json.bak` (overwriting any previous .bak — only the most-recent state is kept).
2. **Write:** save the new content with `indent=2` and trailing newline.
3. **On write failure:** the .bak is the recovery target; emit a blocker telemetry event per the runtime's blocker handling.

The same discipline applies to `reflections.json` when Step 5 updates `lastRun` timestamps.

`RTMEMORY.md` is backed up only when a cycle changes it by >30% (line count delta / before line count > 0.30).

---

## 6. Authority and conflict handling

If this rubric appears to disagree with the runtime's Step 2 / 3 / 3.6 instruction, **stop and emit a blocker** per the runtime's precedence rule. Do not silently reconcile.

This doc owns the BEFORE/AFTER contract for each operation. The runtime owns the *when*, *whether*, and *with what payload*.
