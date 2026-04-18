# Scoring Rubric — Importance Score, Quality Gates, Markers

> **Library-only.** The recurring runtime does not load this doc. Hot-path rules live in `recurring-rules.md` §1–§2. This long-form doc carries derivations, decision tables, worked examples, and uniqueness-tracking detail for humans, debugging, and first-reflection bootstrap. If `recurring-rules.md` and this doc disagree, **the card wins** for the cron path.

The companion archival rule (forgetting curve) is in `health.md` so it sits alongside the health metrics that govern memory upkeep.

---

## 1. Importance score formula

```
importance = clamp(base_weight × recency_factor × reference_boost / 8.0, 0.0, 1.0)
```

Components are deterministic. The LLM applies them entry-by-entry per the tables below.

### 1.1 base_weight (from marker)

| Marker | base_weight | Notes |
|--------|-------------|-------|
| (none) | 1.0 | Default |
| `🔥 HIGH` | 2.0 | Doubles importance |
| `📌 PIN` | 1.0 | Normal weight but exempt from archival |
| `⚠️ PERMANENT` | — | **Final score is always 1.0**. Skip the formula entirely. PERMANENT also bypasses quality gates. |

### 1.2 recency_factor (decay table)

```
days_elapsed = today_date - lastReferenced_date
recency_factor = max(0.1, 1.0 - (days_elapsed / 180))
```

Pre-computed lookup table — the LLM uses this table by default and only computes from the formula when `days_elapsed > 180`:

| days_elapsed | recency_factor |
|--------------|----------------|
| 0 | 1.000 |
| 7 | 0.961 |
| 14 | 0.922 |
| 30 | 0.833 |
| 60 | 0.667 |
| 90 | 0.500 |
| 120 | 0.333 |
| 150 | 0.167 |
| 180 | 0.100 (floor) |
| > 180 | 0.100 (floor — clamped) |

Linear interpolation between rows is acceptable. The 0.10 floor at day 180 is hard.

### 1.3 reference_boost (logarithmic)

```
reference_boost = max(1.0, log2(referenceCount + 1))
```

Pre-computed lookup table — use these values directly:

| referenceCount | reference_boost |
|----------------|-----------------|
| 0 | 1.000 |
| 1 | 1.000 |
| 2 | 1.585 |
| 3 | 2.000 |
| 5 | 2.585 |
| 7 | 3.000 |
| 10 | 3.459 |
| 15 | 4.000 |
| 20 | 4.392 |
| 31 | 5.000 |

Always floored at 1.0 (so 0 or 1 references gives no boost).

### 1.4 Normalization

```
raw = base_weight × recency_factor × reference_boost
importance = min(1.0, max(0.0, raw / 8.0))
```

Maximum raw value = `2.0 × 1.0 × 4.0 = 8.0` (HIGH marker, today, 15 references). The `/8.0` divisor places that at importance 1.0.

### 1.5 Worked examples

Six representative cases — the LLM mimics this pattern when scoring fresh candidates.

| # | Marker | days_elapsed | referenceCount | base | recency | boost | raw | importance |
|---|--------|--------------|----------------|------|---------|-------|-----|------------|
| 1 | (none) | 0 | 1 | 1.0 | 1.000 | 1.000 | 1.000 | 0.125 |
| 2 | (none) | 30 | 3 | 1.0 | 0.833 | 2.000 | 1.667 | 0.208 |
| 3 | HIGH | 0 | 7 | 2.0 | 1.000 | 3.000 | 6.000 | 0.750 |
| 4 | HIGH | 90 | 15 | 2.0 | 0.500 | 4.000 | 4.000 | 0.500 |
| 5 | (none) | 180 | 5 | 1.0 | 0.100 | 2.585 | 0.259 | 0.032 |
| 6 | PERMANENT | (any) | (any) | — | — | — | — | **1.0 (always)** |

---

## 2. Quality Gates (strictMode profile-driven)

Gates only run when `strictMode == true` (business-employee default). In parity flow (`strictMode == false`, personal-assistant default), all extracted candidates consolidate without gating.

### 2.1 Three-part AND gate

A candidate qualifies for promotion when **all** three conditions hold for the active mode:

```
importance >= mode.minScore
AND
referenceCount >= mode.minRecallCount
AND
effective_unique >= mode.minUnique
```

`effective_unique` is resolved from `mode.uniqueMode`:

| uniqueMode | Field used | Behavior |
|------------|-----------|----------|
| `day_or_session` (default) | `uniqueDayCount` if > 0, else `uniqueSessionCount` | Prefer day-based reinforcement |
| `day` | `uniqueDayCount` | Day count only |
| `session` | `uniqueSessionCount` | Session count only |
| `channel` | `uniqueChannelCount` | Channel count only |
| `max` | max of the three | Highest available signal |

### 2.2 Mode threshold tables

**Personal-assistant defaults** (only relevant if operator opts into `strictMode: true`):

| Mode | minScore | minRecallCount | minUnique |
|------|----------|----------------|-----------|
| core | 0.72 | 2 | 1 |
| rem | 0.85 | 2 | 2 |
| deep | 0.80 | 2 | 2 |

**Business-employee defaults** (default `strictMode: true`):

| Mode | minScore | minRecallCount | minUnique |
|------|----------|----------------|-----------|
| core | 0.72 | 2 | 1 |
| rem | 0.85 | 3 | 2 |
| deep | 0.80 | 2 | 2 |

(Full presets live in `profiles/personal-assistant.md` and `profiles/business-employee.md`.)

### 2.3 Gate evaluation order

Evaluate **strictest mode first**: rem → deep → core. Once a candidate is qualified by any mode, it is not re-evaluated by subsequent (looser) modes.

```
for mode in ['rem', 'deep', 'core']:           # strictest first
    if mode not in due_modes: continue
    for candidate in candidates:
        if candidate already qualified: continue

        # Hard bypass — PERMANENT always passes
        if candidate.marker == 'PERMANENT':
            mark qualified by 'PERMANENT'; gate_bypass = 'PERMANENT'; continue

        # Soft bypass — fast-path
        if passes_fast_path(candidate, mode):
            mark qualified by mode; gate_bypass = 'FAST_PATH'; continue

        # Regular AND gate
        eu = effective_unique(candidate, mode.uniqueMode)
        if (candidate.importance >= mode.minScore
            AND candidate.referenceCount >= mode.minRecallCount
            AND eu >= mode.minUnique):
            mark qualified by mode; continue

deferred = candidates not marked qualified
```

### 2.4 Fast-path bypass (soft)

A candidate passes the fast-path when **either** condition holds:

1. **Score + recall fast-path** — `importance >= mode.fastPathMinScore AND referenceCount >= mode.fastPathMinRecallCount`
2. **Marker fast-path** — `candidate.marker in mode.fastPathMarkers`

When fast-path fires, set `gate_bypass = "FAST_PATH"` so the durability filter (Step 1.7) recognizes it as rescue-eligible and forces `structuralEvidence = 4`.

### 2.5 Bypass summary table

| Condition | Bypass? | Rationale |
|-----------|---------|-----------|
| `⚠️ PERMANENT` marker | Yes (hard) | User-protected entries always consolidate |
| Fast-path (marker match or score+recall match) | Yes (soft) | High-salience entries skip the regular AND gate |
| `🔥 HIGH` marker alone | No (unless in `fastPathMarkers`) | HIGH doubles base weight but doesn't bypass on its own |
| `📌 PIN` marker alone | No (unless in `fastPathMarkers`) | PIN prevents archival but doesn't bypass intake |
| First Reflection (post-install) | Yes | Bootstrap consolidates everything to seed memory |

### 2.6 Fields written back per candidate

After Step 1.6, every candidate carries:

- `importance` — number from 1.4
- `gate_status` — `"qualified"` or `"deferred"`
- `gate_promoted_by` — mode name (`"rem"`, `"deep"`, `"core"`) when qualified, `null` when deferred
- `gate_bypass` — `"PERMANENT"`, `"FAST_PATH"`, or `null`
- `gate_fail_reasons` — when deferred, an object keyed by mode showing which condition(s) failed (e.g. `{"rem": ["minScore: 0.62 < 0.85"]}`)

---

## 3. Markers

Markers are user-set or LLM-detected tokens on the candidate `summary` or `tags` that influence scoring/gating.

### 3.1 Detection rule

The LLM detects a marker by scanning the candidate's `summary` and `tags`:

| Pattern in summary or tag | Marker assigned |
|---------------------------|------------------|
| `⚠️ PERMANENT` literal in summary, or tag `PERMANENT` | `PERMANENT` |
| `🔥 HIGH` literal in summary, or tag `HIGH` | `HIGH` |
| `📌 PIN` literal in summary, or tag `PIN` | `PIN` |
| `PREFERENCE` tag (or per-profile fast-path marker) | `PREFERENCE` (PA fast-path) |
| `ROUTINE` tag | `ROUTINE` (PA fast-path) |
| `PROCEDURE` tag | `PROCEDURE` (BE/PA fast-path) |
| (none) | `null` |

If multiple markers match, the precedence is `PERMANENT > HIGH > PIN > PROCEDURE > PREFERENCE > ROUTINE > null`.

### 3.2 Marker effects summary

| Marker | base_weight | Archival immunity | Default fast-path | Notes |
|--------|-------------|--------------------|--------------------|-------|
| (none) | 1.0 | No | No | |
| HIGH | 2.0 | No | No (unless in profile fast-path list) | Doubles base weight |
| PERMANENT | (1.0 final) | Yes | Yes (hard bypass) | Final score always 1.0 |
| PIN | 1.0 | Yes | No (unless in profile fast-path list) | User-pinned, archival-immune |
| PREFERENCE / ROUTINE / PROCEDURE | 1.0 | No | Yes if in profile fast-path list | Profile-driven |

---

## 4. Uniqueness tracking

`referenceCount` is the raw count of times an entry has been referenced. It can be inflated by a single long conversation. `uniqueSessionCount` and `uniqueDayCount` are the cross-context corrections.

### 4.1 Two uniqueness signals

| Field | Increments when |
|-------|------------------|
| `uniqueSessionCount` | A new daily-log file (`memory/YYYY-MM-DD.md`) references the entry |
| `uniqueDayCount` | A new YYYY-MM-DD date references the entry |

### 4.2 Update algorithm (during Step 1 collect)

For each new mention of an existing entry:

```
existing.referenceCount += 1
existing.lastReferenced = today
day = extract YYYY-MM-DD from source log filename
if source_log not in existing.sessionSources:
    existing.uniqueSessionCount += 1
    existing.sessionSources.append(source_log)   # cap at 30 most recent
if day not in existing.uniqueDaySources:
    existing.uniqueDayCount += 1
    existing.uniqueDaySources.append(day)        # cap at 30 most recent
```

### 4.3 Edge cases

| Case | Behavior |
|------|----------|
| Same entry mentioned 5 times in one daily log | `referenceCount += 5`, `uniqueSessionCount += 1` |
| Entry referenced in 3 different daily logs | `uniqueSessionCount = 3` |
| Entry exists in index but never extracted before | Initialize `uniqueSessionCount = 1`, `sessionSources = [current_log]` |
| Manual "Reflect now" trigger (no daily log source) | Use `"manual-YYYY-MM-DD"` as synthetic session ID |

---

## 5. Authority and conflict handling

If this rubric appears to disagree with the runtime prompt's Step 1.6 instruction, **stop and emit a blocker** per the runtime's precedence rule. Do not silently reconcile. The runtime owns execution order; this doc owns the formulas. They must agree.

When in doubt, the formulas in this doc are authoritative for the math; the runtime is authoritative for *when* and *whether* to apply them.
