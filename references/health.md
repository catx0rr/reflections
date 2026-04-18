# Health Rubric — 5-Metric Score, Reachability, Stale Detection, Forgetting Curve

> **Library-only.** The recurring runtime does not load this doc. Hot-path rules live in `recurring-rules.md` §4 (forgetting curve / archival). The 5-metric health formula and reachability BFS algorithm are computed by the runtime each cycle but their derivations / worked examples / suggestion triggers stay here for humans and debugging. If `recurring-rules.md` and this doc disagree on archival, **the card wins** for the cron path.

---

## 1. Health score formula

```
health = (freshness × 0.25
       + coverage × 0.25
       + coherence × 0.20
       + efficiency × 0.15
       + reachability × 0.15) × 100
```

The result is rounded to the nearest integer and rated:

| Score | Rating |
|-------|--------|
| 80–100 | excellent |
| 60–79 | good |
| 40–59 | fair |
| 20–39 | poor |
| 0–19 | critical |

---

## 2. The five metrics

### 2.1 Freshness (weight 0.25)

What proportion of active entries have been referenced in the last 30 days?

```
active_entries = entries where archived != true
recent_entries = active where lastReferenced >= today - 30 days
freshness = recent_entries / active_entries
```

If `active_entries == 0`, freshness = 0.

### 2.2 Coverage (weight 0.25)

How many of the canonical RTMEMORY.md sections currently have non-empty, non-comment content?

The 10 canonical sections:

```
1. Scope Notes
2. Active Initiatives
3. Business Context and Metrics
4. People and Relationships
5. Strategy and Priorities
6. Key Decisions and Rationale
7. Lessons and Patterns
8. Episodes and Timelines
9. Environment Notes
10. Open Threads
```

```
sections_with_content = count of canonical sections in RTMEMORY.md
                        that contain at least one line which is:
                          - non-empty
                          - not starting with "<!--"
                          - not starting with "_" (e.g. "_Last updated:_")
coverage = sections_with_content / 10
```

### 2.3 Coherence (weight 0.20)

What fraction of active entries have at least one relation link?

```
with_relations = count of active entries where related is non-empty
coherence = with_relations / active_entries
```

### 2.4 Efficiency (weight 0.15)

How concise is RTMEMORY.md? Inversely proportional to line count, capped at 500 lines.

```
efficiency = max(0.0, 1.0 - (memory_md_line_count / 500))
```

| Lines | Efficiency |
|-------|-----------|
| 0 | 1.000 |
| 100 | 0.800 |
| 250 | 0.500 |
| 400 | 0.200 |
| 500+ | 0.000 |

### 2.5 Reachability (weight 0.15)

What fraction of the active memory graph is mutually reachable via relation links?

The graph: nodes are active entry ids, edges are bidirectional `related` links between two active ids (links to archived entries are dropped).

```
reachability = sum_over_components(component_size²) / total_active²
clamp to [0.0, 1.0]
```

Worked example — graph with 10 nodes in 2 connected components of size 7 and 3:

```
weighted_sum = 7² + 3² = 49 + 9 = 58
reachability = 58 / 10² = 58 / 100 = 0.58
```

Worked example — fully connected graph (one component of size 10):

```
weighted_sum = 10² = 100
reachability = 100 / 100 = 1.00
```

Worked example — fully fragmented graph (10 components of size 1 each):

```
weighted_sum = 1² × 10 = 10
reachability = 10 / 100 = 0.10
```

#### Plain-language algorithm

The LLM walks the graph as follows. For graphs with > 50 active entries, do this section by section to keep the working set small.

```
1. Build active_ids = {e.id for e in entries if not e.archived}
2. Build adjacency map adj = {} (id → set of neighbor ids)
   For each active entry:
     for each related_id in entry.related:
       if related_id in active_ids:
         adj[entry.id].add(related_id)
         adj[related_id].add(entry.id)
3. components = []
   visited = empty set
   For each id in active_ids:
     if id in visited: skip
     component = empty set
     queue = [id]
     while queue not empty:
       current = pop first from queue
       if current in visited: skip
       add current to visited
       add current to component
       for each neighbor in adj[current]:
         if neighbor not in visited: append to queue
     components.append(size of component)
4. weighted_sum = sum of (size² for each size in components)
5. reachability = weighted_sum / (len(active_ids))²
6. clamp to [0.0, 1.0]
```

#### Reachability interpretation

| Value | Meaning |
|-------|---------|
| 1.0 | All entries in one connected component — perfect graph |
| 0.7–0.9 | Most entries connected, a few isolated clusters |
| 0.4–0.6 | Significant fragmentation — many topics not linked |
| 0.1–0.3 | Heavily fragmented — knowledge silos |
| 0.0–0.1 | Almost no connections — a flat list, not a graph |

---

## 3. Suggestions (auto-generated when metrics are weak)

Append these to the cycle insights when the corresponding condition holds:

| Condition | Suggestion |
|-----------|-----------|
| `freshness < 0.5` | "Many entries are stale — review for relevance or increase cross-referencing" |
| `coverage < 0.5` | "Several RTMEMORY.md sections haven't been updated — check for knowledge gaps" |
| `coherence < 0.3` | "Low entry connectivity — consider linking related memories manually" |
| `efficiency < 0.3` | "RTMEMORY.md is large (N lines) — review for pruning or archival opportunities" |
| `reachability < 0.4` | "Memory graph is fragmented (N components, M isolated entries) — add cross-references" |
| `gateStats.lastCycleDeferred > 10` | "Many entries deferred — consider lowering gate thresholds or running in core mode" |
| no entries pass `rem` gates for 3+ cycles | "rem mode is too strict — no entries qualifying. Review minScore threshold" |
| `healthHistory` declining 3+ cycles | "Health trending down — investigate which metric is deteriorating" |

---

## 4. Forgetting curve (archival rule)

An active entry is eligible for archival when **all five** conditions hold:

```
1. days_since_last_referenced > 90
2. importance < 0.3
3. NOT marked ⚠️ PERMANENT
4. NOT marked 📌 PIN
5. NOT in an episode file (episodes are append-only)
```

Days are computed against the entry's `lastReferenced` field.

### 4.1 Archival process

For each archival-eligible entry:

```
1. Compress the entry to a one-line summary (preserve essential meaning)
2. Append to memory/.reflections-archive.md:
   - [mem_NNN] (created: YYYY-MM-DD → archived: YYYY-MM-DD) One-line summary
3. Remove the full entry from its source surface (RTMEMORY.md or PROCEDURES.md)
4. In runtime/reflections-metadata.json:
   set entry.archived = true
   set entry.archived_at = today (YYYY-MM-DD)
   keep the index entry (for relation tracking and reachability graph)
```

The archive file is append-only. Never edit existing archive lines.

### 4.2 Decay visualization

```
Importance
1.0 │ ████
    │ ████████
    │ ████████████
0.5 │ ████████████████
    │ ████████████████████
0.3 │─────────────────────────── archival threshold
    │ ████████████████████████████
0.1 │ ████████████████████████████████
0.0 └──────────────────────────────────→ Days
    0    30    60    90    120   150   180
```

---

## 5. Stale thread detection (Step 2.8 + 0-B)

Stale items are entries the agent should surface to the operator — open threads or memories that haven't been touched in a while but might still matter.

### 5.1 Thresholds

- `threshold = 14` days (default — entries not referenced in 14 days are candidates)
- `top = 3` for the cycle notification, `top = 1` for the skip-with-recall message

### 5.2 Source priority

Scan in this order:

1. **RTMEMORY.md → "Open Threads" section** — `- [ ]` items with optional inline date or `mem_NNN` reference. These are explicit open threads; surface oldest first.
2. **runtime/reflections-metadata.json → entries with `lastReferenced > 14 days ago` AND not archived** — closed-loop fallback when Open Threads is empty.

### 5.3 Days-stale computation

For each candidate:

- If the candidate has an inline date (`(2026-03-15)` or similar) → use that date.
- Else if the candidate references `mem_NNN` → look up that entry's `lastReferenced` in the index.
- Else if the candidate is from the index → use its `lastReferenced`.

```
days_stale = today_date - candidate_date
```

Candidates with `days_stale < threshold` are excluded. The remainder is sorted descending by `days_stale` and the top N are returned.

### 5.4 Returned record shape

```json
{
  "id": "mem_NNN or null",
  "summary": "One-line summary (truncate at 100 chars)",
  "days_stale": 42,
  "source": "RTMEMORY.md#open-threads or runtime/reflections-metadata.json"
}
```

---

## 6. Authority and conflict handling

If this rubric appears to disagree with the runtime prompt's Step 2.8 or Step 3 instruction, **stop and emit a blocker** per the runtime's precedence rule. Do not silently reconcile.

The formulas in §1–§2 are authoritative for math; the archival rule in §4 is authoritative for which entries are eligible; the stale rubric in §5 is authoritative for which entries are surfaced. The runtime is authoritative for *when* and *whether* to apply them.
