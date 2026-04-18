# Durability Rubric — Filter, Classification, 5-Route System

> **Library-only.** The recurring runtime does not load this doc. Hot-path rules live in `recurring-rules.md` §3 (annotation scope, net-score components, hard triggers, routing precedence, destination map, profile thresholds, written fields). This long-form doc carries the full memory-type taxonomy, flag-by-flag definitions, and 15 worked classification examples for humans, debugging, and first-reflection bootstrap. If `recurring-rules.md` and this doc disagree, **the card wins** for the cron path.

Activation: durability runs only when `strictMode == true` AND `durability.enabled == true`. Parity flow (`strictMode == false`) and strict-without-durability flow skip this stage entirely.

---

## 1. Core question

Structural scoring (`scoring.md`) answers: *"is this candidate reinforced enough to deserve durable memory?"*

Durability answers a harder question: *"is this candidate actually meaningful long-horizon memory, or just reinforced noise?"*

A heartbeat repeated across sessions may pass the structural gate; durability rejects it. A single statement establishing a boundary may never pass the structural gate; durability can **rescue-promote** it.

**Core law:** A memory should promote when losing it would weaken future judgment. Promote durable guidance, not durable presence.

---

## 2. Step 1.7 — Semantic annotation

For each in-scope candidate, the LLM produces a JSON record with semantic flags. The router (Step 1.8) consumes them — never the runtime narration.

### 2.1 Scope (which candidates get annotated)

Annotate every candidate with `gate_status == "qualified"`, **plus** the **rescue subset** of candidates with `gate_status == "deferred"` that meet any of:

- `gate_bypass` is set (`"PERMANENT"` or `"FAST_PATH"`)
- `marker in {"HIGH", "PERMANENT", "PIN"}`
- belongs to a high-meaning class (decision, lesson, obligation, relationship, identity, architecture) — judged from the candidate's `summary` and `category` before writing the annotation

**Always skip:** candidates with `deferred_status == "persisted"` (they're already suppressed by the deferred store; respect that decision).

### 2.2 Annotation record schema

One record per in-scope candidate:

```json
{
  "candidate_id": "<id from the candidate>",
  "memory_type": "decision | lesson | preference | procedure | obligation | relationship | identity | architecture | observation | status | trend",
  "durability_class": "durable | semi-durable | volatile | noise",
  "changed_future_decision": true | false,
  "changed_behavior_or_policy": true | false,
  "created_stable_preference": true | false,
  "created_obligation_or_boundary": true | false,
  "relationship_or_identity_shift": true | false,
  "cross_day_relevance": true | false,
  "rare_high_consequence": true | false,
  "actionable_procedure": true | false,
  "pattern_only": true | false,
  "pure_status": true | false,
  "telemetry_noise": true | false,
  "duplicate_of_existing": "mem_042 or null",
  "merge_key": "stable-slug or null",
  "trend_key": "stable-slug or null",
  "explanation": "short rationale (telemetry only — router ignores this)"
}
```

All booleans must be explicit `true` or `false` — never `null`. The router treats missing as `false`.

### 2.3 Memory type taxonomy

| memory_type | Use when… |
|-------------|-----------|
| `decision` | A choice was made — a direction picked, a tool chosen, a path taken. |
| `lesson` | A failure or success that teaches something for next time. |
| `preference` | A stated taste, default, or "I prefer X over Y". |
| `procedure` | A repeatable how-to — steps, commands, recipes. |
| `obligation` | A commitment or boundary the agent or user must honor. |
| `relationship` | New fact about a person — role, identity, contact info, dynamic. |
| `identity` | Shift in how the agent or user self-describes. |
| `architecture` | A structural conclusion about a system, codebase, or workflow. |
| `observation` | A noted-but-not-resolved phenomenon (one-off, low-action). |
| `status` | A current-state report ("server is up", "deployment in progress"). |
| `trend` | A pattern across multiple events without a stable method. |

### 2.4 Flag definitions

**Meaning flags** (set when the candidate establishes meaning, not just presence):

| Flag | Set when |
|------|----------|
| `changed_future_decision` | The candidate will alter how the agent decides next time (e.g., "we're switching from X to Y"). |
| `changed_behavior_or_policy` | A standing rule, default, or behavior changed. |
| `created_stable_preference` | A new durable preference is now in force. |
| `created_obligation_or_boundary` | A commitment, deadline, or limit was created. |
| `relationship_or_identity_shift` | Someone's role, identity, or relationship changed. |

**Consequence flags** (set when the candidate matters across time):

| Flag | Set when |
|------|----------|
| `cross_day_relevance` | The candidate is relevant beyond the day it occurred (not "what time is it now"). |
| `rare_high_consequence` | One-off but high-consequence — losing it would meaningfully weaken future judgment. |
| `actionable_procedure` | The candidate describes a repeatable workflow the agent could re-apply. |

**Noise flags** (set when the candidate is presence-without-meaning):

| Flag | Set when |
|------|----------|
| `pattern_only` | The candidate is a pattern detection ("X happens repeatedly") with no decision attached. |
| `pure_status` | A current-state ping with no decision or guidance ("server is healthy now"). |
| `telemetry_noise` | Heartbeats, repeated symptom reports, log echoes — operational chatter. |

**Identity flags** (set when the candidate relates to existing memory):

| Flag | Set when |
|------|----------|
| `duplicate_of_existing` | The candidate semantically restates an existing index entry — set to that entry's `id` (e.g. `"mem_042"`), else `null`. |
| `merge_key` | A stable slug describing the merged-into concept (e.g. `"dental-clinic-pricing-2026Q2"`). Used by `merge` route. Else `null`. |
| `trend_key` | A stable slug describing the recurring pattern (e.g. `"dev-server-noon-restart"`). Used by `compress` route. Else `null`. |

---

## 3. Step 1.8 — Deterministic routing

Every annotated candidate gets one of five routes. The router applies rules in **strict precedence order** — the first matching rule wins. No ambiguity, no LLM judgment at this layer.

### 3.1 Net-score computation

Computed from structural evidence (Step 1.6 outputs) and semantic flags (Step 1.7 outputs). Each component clamps to `0..4`.

```
structuralEvidence  = max 4 if gate_bypass in {"PERMANENT", "FAST_PATH"}
                      else
                      sum of:
                        +1 if referenceCount >= 1
                        +1 if referenceCount >= 3
                        +1 if uniqueDayCount >= 2
                        +1 if uniqueDayCount >= 4
                      (capped at 4)

meaningWeight       = count of TRUE among:
                        changed_future_decision
                        changed_behavior_or_policy
                        created_stable_preference
                        created_obligation_or_boundary
                        relationship_or_identity_shift
                      (capped at 4)

futureConsequence   = sum of:
                        +1 if cross_day_relevance
                        +2 if rare_high_consequence  (heavier weight)
                        +1 if actionable_procedure
                      (capped at 4)

noisePenalty        = sum of:
                        +1 if pattern_only
                        +1 if pure_status
                        +2 if telemetry_noise         (heavier weight)
                      (capped at 4)

net = structuralEvidence + meaningWeight + futureConsequence − noisePenalty
```

### 3.2 Hard-promote triggers (force `route = "promote"`)

Any one of these short-circuits the net-score banding:

1. `memory_type in {"decision", "lesson"}` AND (`changed_future_decision` OR `changed_behavior_or_policy`)
2. `created_stable_preference` is true
3. `created_obligation_or_boundary` is true
4. `relationship_or_identity_shift` is true
5. `actionable_procedure` is true AND `structuralEvidence >= 2` (validated repeatable — not one-shot)
6. `memory_type == "architecture"` AND `rare_high_consequence` is true

The matching trigger name becomes `promotionReason`, e.g. `"hard-trigger:decision-with-consequence"`, `"hard-trigger:created_obligation_or_boundary"`.

### 3.3 Hard-suppress triggers (force `route = "reject"`)

Any one of these short-circuits everything:

1. `telemetry_noise` is true
2. `pure_status` is true AND `rare_high_consequence` is not true
3. `pattern_only` is true AND `cross_day_relevance` is not true (single-day repetition)

The matching reason becomes `promotionReason`, e.g. `"hard-suppress:telemetry_noise"`.

### 3.4 Routing precedence (strict order, first match wins)

```
1. Hard-suppress trigger fires → route = "reject"; destination = "NONE"
2. Hard-promote trigger fires  → route = "promote"; destination from §4
                                  (also: if trendKey matches an existing trend that
                                  meets promotion thresholds, set promotedFromTrend)
3. duplicate_of_existing is set AND resolves in the index → route = "merge";
                                  destination = "NONE"; mergedInto = duplicate_of_existing
4. duplicate_of_existing is set but does NOT resolve in the index → route = "defer";
                                  destination = "NONE" (re-eval next cycle)
5. trendKey is set
   AND memory_type in {observation, status, trend}
   AND no hard-trigger fired
   AND actionable_procedure is not true
   → route = "compress"; destination = "TREND"
   (mergedInto = existing trend id when reinforcing; else new trend created)
6. Net-score banding:
     net >= netPromoteThreshold → route = "promote"; destination from §4
     net >= netDeferThreshold   → route = "defer"; destination = "NONE"
     else                        → route = "reject"; destination = "NONE"
```

### 3.5 Threshold tables (per profile)

| Profile | netPromoteThreshold | netDeferThreshold | trendPromoteSupportCount | trendPromoteUniqueDayCount |
|---------|---------------------|-------------------|--------------------------|----------------------------|
| business-employee | 6 | 3 | 5 | 3 |
| personal-assistant (only if operator opts into strictMode) | 5 | 2 | 4 | 2 |

Values come from the top-level `durability` block in `reflections.json`.

---

## 4. Destination map (only for promote / compress)

Destinations: `RTMEMORY` / `PROCEDURES` / `EPISODE` / `TREND` / `NONE`.

| memory_type | Destination | Route + condition |
|-------------|-------------|-------------------|
| decision, lesson, obligation, relationship, identity, architecture, preference | `RTMEMORY` | `promote` |
| procedure | `PROCEDURES` | `promote` (via `actionable_procedure` hard-trigger AND `structuralEvidence >= 2`) |
| observation | `EPISODE` | `promote` only when a hard-promote trigger fires alongside `cross_day_relevance`. Generic observations (no hard-trigger) route to `compress` → TREND. |
| trend, status | `TREND` | `compress` |
| trend (with promote route) | `RTMEMORY` | `promote` — accumulated trend has crossed support thresholds AND a hard-trigger fired this cycle. The new RTMEMORY node carries `promotedFromTrend = <existing trend id>`. |
| (any) duplicate_of_existing matched, no hard-trigger | `NONE` | `merge` — reinforces target via index reinforce |
| any other / NONE | `RTMEMORY` (fallback) | `promote` only |

**Repeated ops material rule:**
- Repetition yields a validated actionable workflow → PROCEDURES (`promote`)
- Repetition reveals a durable condition/pattern but no stable method → TREND (`compress`)
- Repeated presence with no guidance and no durable pattern → `reject` or `defer`
- Repeated symptom chatter is never promoted as a new RTMEMORY node

---

## 5. Trend-to-durable promotion

A trend node (`memoryType == "trend"`) accumulates `trendSupportCount` via `compress` route updates. It becomes eligible to promote into RTMEMORY only when **all three** conditions hold:

1. `trendSupportCount >= trendPromoteSupportCount` (structural — accumulated)
2. `uniqueDayCount >= trendPromoteUniqueDayCount` (structural — spread across days)
3. A fresh candidate this cycle annotates a matching `trend_key` with a hard-promote trigger (semantic — operator tied a decision/consequence to it)

Conditions 1+2 are checked structurally against the existing trend node. Condition 3 is the LLM's annotation in Step 1.7. **All three** must hold — accumulation alone does not promote.

When all three hold, the route is `promote` (not `compress`), the destination is `RTMEMORY`, and the new entry carries `promotedFromTrend = <existing trend id>`. The original trend node stays in place as historical context.

---

## 6. Fields written back to each candidate

Step 1.8 writes these fields onto every annotated candidate:

| Field | Value |
|-------|-------|
| `route` | `"promote"` / `"merge"` / `"compress"` / `"defer"` / `"reject"` |
| `destination` | `"RTMEMORY"` / `"PROCEDURES"` / `"EPISODE"` / `"TREND"` / `"NONE"` |
| `durabilityScore` | `net` from §3.1 |
| `noisePenalty` | the penalty component in isolation |
| `promotionReason` | hard-trigger name, `"net=N>=threshold"`, `"merge-into:<id>"`, `"reinforce-trend:<id>"`, `"new-trend:<key>"`, or `"hard-suppress:<reason>"` |
| `memoryType` | echoed from annotation |
| `durabilityClass` | echoed from annotation |
| `mergeKey` | echoed from annotation |
| `trendKey` | echoed from annotation |
| `duplicateOfExisting` | echoed from annotation |
| `mergedInto` | target entry id when route=merge; existing trend id when route=compress reinforces existing |
| `promotedFromTrend` | existing trend id when trend-to-durable promotion fires; else `null` |
| `compressionReason` | `"reinforce-trend:<id>"` or `"new-trend:<key>"` when route=compress; else `null` |
| `supportCount` | `referenceCount` snapshot at routing time |
| `durabilityComponents` | `{structuralEvidence, meaningWeight, futureConsequence, noisePenalty}` |

Defer-routed candidates are also persisted to `runtime/reflections-deferred.jsonl` per `deferred-store.md`.

---

## 7. Worked classification examples

These are pattern-match references. The LLM mimics them when classifying fresh candidates.

### 7.1 Hard-promote (decision + consequence)

> "Decided to move gateway billing from Paymongo to Stripe."

| Field | Value |
|-------|-------|
| memory_type | `decision` |
| durability_class | `durable` |
| changed_future_decision | `true` |
| cross_day_relevance | `true` |
| route | `promote` |
| destination | `RTMEMORY` |
| promotionReason | `hard-trigger:decision-with-consequence` |

### 7.2 Hard-promote (stable preference)

> "User prefers Filipino over English in replies."

| Field | Value |
|-------|-------|
| memory_type | `preference` |
| created_stable_preference | `true` |
| route | `promote` |
| destination | `RTMEMORY` |
| promotionReason | `hard-trigger:created_stable_preference` |

### 7.3 Hard-promote (obligation/boundary)

> "Email landlord by Friday — rent due."

| Field | Value |
|-------|-------|
| memory_type | `obligation` |
| created_obligation_or_boundary | `true` |
| route | `promote` |
| destination | `RTMEMORY` |
| promotionReason | `hard-trigger:created_obligation_or_boundary` |

### 7.4 Hard-promote (relationship shift)

> "Brother Miguel just started at BPI as a teller."

| Field | Value |
|-------|-------|
| memory_type | `relationship` |
| relationship_or_identity_shift | `true` |
| route | `promote` |
| destination | `RTMEMORY` |
| promotionReason | `hard-trigger:relationship_or_identity_shift` |

### 7.5 Hard-promote (validated procedure)

> "Deploy command: pnpm build, then rsync ./dist user@host:/srv, then ssh user@host 'sudo systemctl restart api'." (seen 3+ cycles, structuralEvidence=3)

| Field | Value |
|-------|-------|
| memory_type | `procedure` |
| actionable_procedure | `true` |
| structuralEvidence | `3` |
| route | `promote` |
| destination | `PROCEDURES` |
| promotionReason | `hard-trigger:validated_actionable_procedure` |

### 7.6 Hard-promote (architecture, rare high consequence)

> "Switched from SQLite to Postgres across three services — affected migrations, connection pooling, all backups."

| Field | Value |
|-------|-------|
| memory_type | `architecture` |
| rare_high_consequence | `true` |
| route | `promote` |
| destination | `RTMEMORY` |
| promotionReason | `hard-trigger:architecture_rare_high_consequence` |

### 7.7 Hard-suppress (telemetry noise)

> "GET /api/health returned 200 OK at 10:42:13."

| Field | Value |
|-------|-------|
| memory_type | `status` |
| telemetry_noise | `true` |
| route | `reject` |
| destination | `NONE` |
| promotionReason | `hard-suppress:telemetry_noise` |

### 7.8 Hard-suppress (pure status, no consequence)

> "Server is healthy."

| Field | Value |
|-------|-------|
| memory_type | `status` |
| pure_status | `true` |
| rare_high_consequence | `false` |
| route | `reject` |
| destination | `NONE` |
| promotionReason | `hard-suppress:pure_status_no_consequence` |

### 7.9 Hard-suppress (pattern only, single day)

> "Mentioned the cat three times this morning."

| Field | Value |
|-------|-------|
| memory_type | `observation` |
| pattern_only | `true` |
| cross_day_relevance | `false` |
| route | `reject` |
| destination | `NONE` |
| promotionReason | `hard-suppress:pattern_only_same_day` |

### 7.10 Compress to TREND

> "Dev server restarted around noon again." (seen 3 prior days, no clear cause)

| Field | Value |
|-------|-------|
| memory_type | `observation` |
| trend_key | `dev-server-noon-restart` |
| cross_day_relevance | `true` |
| pattern_only | `true` |
| route | `compress` |
| destination | `TREND` |
| compressionReason | `reinforce-trend:mem_099` (when existing trend node found) |

### 7.11 Merge into existing

> "Patient prefers afternoon appointments." (already promoted as `mem_042`)

| Field | Value |
|-------|-------|
| memory_type | `preference` |
| duplicate_of_existing | `mem_042` |
| created_stable_preference | `false` (no NEW preference; reinforces existing) |
| (no hard-trigger fires) | |
| route | `merge` |
| destination | `NONE` |
| mergedInto | `mem_042` |
| promotionReason | `merge-into:mem_042` |

### 7.12 Defer (unresolved duplicate)

> "Refers to mem_999 about the old API." (mem_999 doesn't exist in current index)

| Field | Value |
|-------|-------|
| duplicate_of_existing | `mem_999` |
| route | `defer` |
| destination | `NONE` |
| promotionReason | `duplicate-of-unknown:mem_999` |

### 7.13 Net-score promote (no hard-trigger)

> "Customer noted that Tuesday afternoons are slower than Mondays — should staff lighter."
> structuralEvidence=2, meaningWeight=2, futureConsequence=2, noisePenalty=0 → net=6 (≥6 BE threshold)

| Field | Value |
|-------|-------|
| memory_type | `lesson` |
| route | `promote` |
| destination | `RTMEMORY` |
| promotionReason | `net=6>=6` |

### 7.14 Net-score defer (borderline)

> "Sometimes the morning meeting feels rushed."
> structuralEvidence=1, meaningWeight=0, futureConsequence=2, noisePenalty=0 → net=3 (≥3 BE defer threshold, <6 promote)

| Field | Value |
|-------|-------|
| memory_type | `observation` |
| route | `defer` |
| destination | `NONE` |
| promotionReason | `net=3>=3` |

### 7.15 Trend-to-durable promote

> "Decided we will run dev-server health checks at 11:55 every day to catch the noon restart proactively."
> trendKey matches existing trend `mem_099` (trendSupportCount=6, uniqueDayCount=4), AND `changed_future_decision=true` fires hard-trigger.

| Field | Value |
|-------|-------|
| memory_type | `decision` |
| trend_key | `dev-server-noon-restart` |
| changed_future_decision | `true` |
| route | `promote` |
| destination | `RTMEMORY` |
| promotedFromTrend | `mem_099` |
| promotionReason | `hard-trigger:decision-with-consequence;promoted-from-trend:mem_099` |

(The original `mem_099` trend node stays in place as historical context.)

---

## 8. Authority and conflict handling

If this rubric appears to disagree with the runtime prompt's Step 1.7 or 1.8 instruction, **stop and emit a blocker** per the runtime's precedence rule. Do not silently reconcile. The runtime owns execution order; this doc owns the rubric.

When in doubt, the routing precedence in §3.4 is authoritative for the *order* of decisions; the runtime is authoritative for *whether* the durability stage runs at all.
