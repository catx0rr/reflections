# Memory Template — Config Presets, Index Schema, Surface Templates

This document is the **rubric authority** for the on-disk shapes of `reflections.json`, `runtime/reflections-metadata.json`, and the canonical surface files. The runtime cites it during install and when authoring new entries.

---

## 1. reflections.json — Per-profile presets

The agent's installed config. One copy per workspace, located at `$CONFIG_PATH` (default `~/.openclaw/reflections/reflections.json`).

### 1.1 Personal-assistant preset

```json
{
  "version": "1.0.0",
  "profile": "personal-assistant",
  "agent": "main",
  "timezone": "Asia/Manila",
  "strictMode": false,
  "scanWindowDays": 7,
  "dispatchCadence": "30 4,10,16,22 * * *",
  "activeModes": ["core", "rem", "deep"],
  "lastRun": {
    "core": null,
    "rem": null,
    "deep": null
  },
  "durability": {
    "enabled": false,
    "netPromoteThreshold": 5,
    "netDeferThreshold": 2,
    "trendPromoteSupportCount": 4,
    "trendPromoteUniqueDayCount": 2
  },
  "modes": {
    "core": {
      "enabled": true,
      "minScore": 0.72,
      "minRecallCount": 2,
      "minUnique": 1,
      "uniqueMode": "day_or_session",
      "fastPathMinScore": 0.90,
      "fastPathMinRecallCount": 2,
      "fastPathMarkers": ["HIGH", "PIN", "PREFERENCE", "ROUTINE"]
    },
    "rem": {
      "enabled": true,
      "minScore": 0.85,
      "minRecallCount": 2,
      "minUnique": 2,
      "uniqueMode": "day_or_session",
      "fastPathMinScore": 0.88,
      "fastPathMinRecallCount": 2,
      "fastPathMarkers": ["HIGH", "PIN", "PREFERENCE", "ROUTINE", "PROCEDURE"]
    },
    "deep": {
      "enabled": true,
      "minScore": 0.80,
      "minRecallCount": 2,
      "minUnique": 2,
      "uniqueMode": "day_or_session",
      "fastPathMinScore": 0.86,
      "fastPathMinRecallCount": 2,
      "fastPathMarkers": ["HIGH", "PIN", "PREFERENCE", "ROUTINE"]
    }
  }
}
```

### 1.2 Business-employee preset

```json
{
  "version": "1.0.0",
  "profile": "business-employee",
  "agent": "main",
  "timezone": "Asia/Manila",
  "strictMode": true,
  "scanWindowDays": 3,
  "dispatchCadence": "30 5,12,18,22 * * *",
  "activeModes": ["core", "rem", "deep"],
  "lastRun": {
    "core": null,
    "rem": null,
    "deep": null
  },
  "durability": {
    "enabled": true,
    "netPromoteThreshold": 6,
    "netDeferThreshold": 3,
    "trendPromoteSupportCount": 5,
    "trendPromoteUniqueDayCount": 3
  },
  "modes": {
    "core": {
      "enabled": true,
      "minScore": 0.72,
      "minRecallCount": 2,
      "minUnique": 1,
      "uniqueMode": "day_or_session",
      "fastPathMinScore": 0.92,
      "fastPathMinRecallCount": 2,
      "fastPathMarkers": ["HIGH", "PIN"]
    },
    "rem": {
      "enabled": true,
      "minScore": 0.85,
      "minRecallCount": 3,
      "minUnique": 2,
      "uniqueMode": "day_or_session",
      "fastPathMinScore": 0.90,
      "fastPathMinRecallCount": 2,
      "fastPathMarkers": ["HIGH", "PIN", "PROCEDURE"]
    },
    "deep": {
      "enabled": true,
      "minScore": 0.80,
      "minRecallCount": 2,
      "minUnique": 2,
      "uniqueMode": "day_or_session",
      "fastPathMinScore": 0.88,
      "fastPathMinRecallCount": 2,
      "fastPathMarkers": ["HIGH", "PIN", "PROCEDURE"]
    }
  }
}
```

### 1.3 Field reference

| Field | Type | Notes |
|-------|------|-------|
| `version` | string | Schema version (`"1.0.0"` for this baseline) |
| `profile` | string | `"personal-assistant"` or `"business-employee"` |
| `agent` | string | Agent identifier (default `"main"`) |
| `timezone` | string | IANA timezone name (e.g. `"Asia/Manila"`) |
| `strictMode` | boolean | Profile-driven; controls whether scoring/gates run |
| `scanWindowDays` | integer | How many days back to scan for unconsolidated logs |
| `dispatchCadence` | string | Cron expression for host scheduling |
| `activeModes` | array | Which modes are enabled (e.g. `["core", "rem", "deep"]`) |
| `lastRun` | object | Per-mode ISO timestamps of last fire (`null` until first run) |
| `durability.enabled` | boolean | Whether the second-stage durability filter runs |
| `durability.netPromoteThreshold` | integer | Net-score threshold for `route = "promote"` |
| `durability.netDeferThreshold` | integer | Net-score threshold for `route = "defer"` |
| `durability.trendPromoteSupportCount` | integer | Trend-to-durable structural threshold |
| `durability.trendPromoteUniqueDayCount` | integer | Trend-to-durable distinct-day threshold |
| `modes.<mode>.enabled` | boolean | Per-mode enable |
| `modes.<mode>.minScore` | number | Importance threshold for AND gate |
| `modes.<mode>.minRecallCount` | integer | Reference count threshold for AND gate |
| `modes.<mode>.minUnique` | integer | Uniqueness threshold for AND gate |
| `modes.<mode>.uniqueMode` | string | How `effective_unique` is resolved (see `scoring.md` §2.1) |
| `modes.<mode>.fastPathMinScore` | number | Importance threshold for fast-path soft bypass |
| `modes.<mode>.fastPathMinRecallCount` | integer | Reference count threshold for fast-path |
| `modes.<mode>.fastPathMarkers` | array | Markers that trigger fast-path bypass |

---

## 2. runtime/reflections-metadata.json — Index schema

The persistent index. Lives at `$WORKSPACE_ROOT/runtime/reflections-metadata.json`. Full file rewrite per cycle (with `.bak` backup).

### 2.1 Top-level shape

```json
{
  "version": "1.0.0",
  "lastDream": "2026-04-18T14:30:14Z",
  "entries": [ /* per-entry records */ ],
  "stats": {
    "totalEntries": 142,
    "avgImportance": 0.4863,
    "lastPruned": "2026-04-15",
    "healthScore": 78,
    "healthMetrics": {
      "freshness": 0.82,
      "coverage": 0.70,
      "coherence": 0.45,
      "efficiency": 0.62,
      "reachability": 0.58
    },
    "insights": ["..."],
    "healthHistory": [
      {"date": "2026-04-17", "score": 76},
      {"date": "2026-04-18", "score": 78}
    ],
    "gateStats": {
      "lastCycleQualified": 6,
      "lastCycleDeferred": 3,
      "lastCycleBreakdown": {"rem": 2, "deep": 2, "core": 2},
      "lastCycleDurable": {
        "promoted": 3, "merged": 1, "compressed": 1,
        "deferred": 1, "rejected": 0
      }
    }
  }
}
```

### 2.2 Per-entry shape (durable node)

```json
{
  "id": "mem_042",
  "summary": "Decided to switch from Paymongo to Stripe for gateway billing",
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

### 2.3 Per-entry shape (durable node with durability fields)

When the entry comes from strict+durability flow, additional fields:

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

### 2.4 Per-entry shape (trend node)

```json
{
  "id": "mem_099",
  "summary": "Dev server restarts around noon each day — no clear trigger identified",
  "source": "memory/2026-04-12.md",
  "target": "TRENDS.md",
  "memoryType": "trend",
  "durabilityClass": "semi-durable",
  "route": "compress",
  "destination": "TREND",
  "trendKey": "dev-server-noon-restart",
  "trendFirstObserved": "2026-04-12",
  "trendLastUpdated": "2026-04-18",
  "trendSupportCount": 6,
  "trendSources": ["memory/2026-04-12.md", "memory/2026-04-13.md", "memory/2026-04-15.md", "memory/2026-04-16.md", "memory/2026-04-17.md", "memory/2026-04-18.md"],
  "sourceCount": 6,
  "compressionReason": "new-trend:dev-server-noon-restart",
  "tags": [],
  "related": [],
  "archived": false,
  "created": "2026-04-12",
  "lastReferenced": "2026-04-18",
  "referenceCount": 6,
  "uniqueSessionCount": 6,
  "uniqueDayCount": 4,
  "uniqueDaySources": ["2026-04-12", "2026-04-13", "2026-04-15", "2026-04-18"],
  "importance": 0.45
}
```

### 2.5 Per-entry shape (archived)

```json
{
  "id": "mem_007",
  "summary": "Old API endpoint — deprecated 2026-01",
  "archived": true,
  "archived_at": "2026-04-15",
  "...other fields preserved as-is..."
}
```

---

## 3. RTMEMORY.md — Reflective long-horizon memory

The 10 canonical sections (used by `health.md` coverage metric). Operators may add custom sections, but these 10 are what coverage scores against.

### 3.1 Section template

```markdown
# RTMEMORY.md — Reflective Memory

_Last updated: 2026-04-18_

## Scope Notes

<!-- One-line statements about what this memory file is for. -->

## Active Initiatives

- [mem_001] (2026-04-01) Move billing to Stripe
- [mem_004] (2026-04-12) Refactor session router

## Business Context and Metrics

- [mem_010] (2026-03-15) Average revenue $X/month, target $Y by Q3

## People and Relationships

- [mem_018] (2026-02-08) Brother Miguel — works at BPI as a teller

## Strategy and Priorities

- [mem_022] (2026-04-05) Q2 priority: stabilize billing pipeline before scaling

## Key Decisions and Rationale

- [mem_042] (2026-04-01) Switched gateway from Paymongo to Stripe — better webhook reliability

## Lessons and Patterns

- [mem_055] (2026-03-20) Dev server restart pattern correlates with cron pressure at noon

## Episodes and Timelines

(Brief timeline pointers; full narratives live in episodes/*.md)

## Environment Notes

- [mem_063] (2026-04-10) Workspace runs on WSL2; postgres on host:5432

## Open Threads

- [ ] (2026-03-15) [mem_022] Confirm Q2 priority alignment with operator
- [ ] (2026-04-08) Decide on Stripe webhook retry policy
```

### 3.2 Entry format inside sections

```
- [mem_NNN] (YYYY-MM-DD) One-line summary
```

The `[mem_NNN]` reference is optional but recommended — it lets the runtime cross-reference index entries to surface lines.

### 3.3 Open Threads format

Open Threads use markdown checkbox syntax:

```
- [ ] (YYYY-MM-DD) [mem_NNN] One-line description of the open thread
```

The date is when the thread was opened. The `mem_NNN` reference is optional. Stale detection (per `health.md` §5) uses these dates.

---

## 4. PROCEDURES.md — Reusable workflows

Free-form structure. The runtime appends new procedures as `### Procedure: <name>` sections.

### 4.1 Template

```markdown
# Procedures — How I Do Things

_Last updated: 2026-04-18_

### Procedure: Deploy API to production

[mem_073] (2026-04-12)

1. `pnpm build`
2. `rsync ./dist user@host:/srv`
3. `ssh user@host 'sudo systemctl restart api'`

**Validated:** 4 successful deploys 2026-04-12 to 2026-04-18.

---

### Procedure: Reset stale dev session

[mem_080] (2026-04-15)

1. Find session: `sessions list --recent`
2. Force-close: `sessions close <id> --force`
3. Restart with `--fresh` flag

**Validated:** 2 invocations.
```

---

## 5. TRENDS.md — Recurring patterns

```markdown
# Trends

_Observed patterns without stable method._

---

### dev-server-noon-restart

_First observed: 2026-04-12. Last updated: 2026-04-18. Support: 6 across 4 days._

Dev server restarts around noon each day — no clear trigger identified yet.

**Sources:** memory/2026-04-12.md, memory/2026-04-13.md, memory/2026-04-15.md, memory/2026-04-18.md

---

### customer-tuesday-slowness

_First observed: 2026-04-09. Last updated: 2026-04-18. Support: 3 across 2 days._

Customer-facing endpoints slower on Tuesdays. Cause unclear — possibly batch jobs.

**Sources:** memory/2026-04-09.md, memory/2026-04-16.md, memory/2026-04-18.md
```

Sections are keyed by `trendKey`. Each section is a standalone block delimited by `---`. The runtime upserts (replace if `### <key>` exists, append otherwise).

---

## 6. episodes/<name>.md — Bounded narratives

Append-only. One file per episode (e.g. `episodes/auth-refactor-2026Q2.md`).

```markdown
# Auth refactor — 2026 Q2

_Started: 2026-04-01. Status: in progress._

## 2026-04-01

[mem_120] Decided to extract session middleware into separate package — currently bundled with API surface.

## 2026-04-08

[mem_127] Discovered race condition between session creation and JWT signing. Adding mutex.

## 2026-04-15

[mem_134] Mutex resolved race; throughput dropped 10% but acceptable for Q2.
```

Episodes are append-only — never edit prior dated entries. New entries go at the bottom under a new `## YYYY-MM-DD` header.

---

## 7. memory/.reflections-log.md — Cycle log

The human-readable consolidation report log. Append-only. Format documented in `runtime-templates.md` §3.

---

## 8. memory/.reflections-archive.md — Archival log

Append-only. One line per archived entry:

```markdown
# Memory Archive

_Compressed entries that fell below importance threshold._

---

<!-- Format: [id] (created → archived) One-line summary -->

- [mem_007] (2025-09-12 → 2026-04-15) Old API endpoint — deprecated
- [mem_011] (2025-10-03 → 2026-04-15) Old vendor list — replaced by Stripe-only
```

---

## 9. runtime/memory-state.json — Shared state

Shared across packages (memory-core, memory-wiki, etc.). The `reflections` namespace is owned by reflections.

```json
{
  "reflections": {
    "reporting": {
      "sendReport": true,
      "delivery": {
        "channel": "last",
        "to": null
      }
    }
  }
}
```

| Field | Notes |
|-------|-------|
| `reporting.sendReport` | When `true`, Step 4.2 emits chat notification. When `false`, telemetry-only run. |
| `reporting.delivery.channel` | `"last"` (reuse last route) or explicit channel name |
| `reporting.delivery.to` | Explicit target (e.g. user ID); takes precedence over `channel` |

The install must use **merge-not-overwrite** semantics to preserve other namespaces.

---

## 10. Authority and conflict handling

If this rubric appears to disagree with the runtime instruction, **stop and emit a blocker**. Do not silently reconcile.

This doc owns the on-disk shapes; the runtime owns *when* and *how* to mutate them.
