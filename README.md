# reflections v1.1.0

A reflective memory consolidator. Behaviorally equivalent to [Hybrid Reflections](https://github.com/catx0rr/reflections-hybrid) within the reference rubrics, profile thresholds, and verification suite — implemented entirely in prompt logic, with no Python scripts. Adds Zero's durability filter and 5-route system on top of the original Auto-Dream scoring family.

LLM judgment introduces per-cycle variance the verification suite scopes; this fork is contract-equivalent to Reflections, not numerically identical at the per-cycle level.

## Precedence Rule

When the runtime prompt and a reference doc disagree, **stop and treat as a blocker** — do not silently reconcile. Authority order:

1. **Runtime prompt** owns execution order (which step runs when, what fires, what stops).
2. **Reference docs** own formulas, rubrics, decision tables, and routing law.
3. **Worked examples** are illustrative only.

This rule is enforced at the top of `runtime/reflections-prompt.md` and the runtime emits a blocker telemetry event on conflict.

## The Problem

AI agents forget. Session ends, context gone. Files pile up. Daily logs accumulate but remain unconsolidated and disconnected. The agent has data but can't reason about it across time.

reflections runs periodic consolidation cycles that scan, extract, score, route, and archive the agent's knowledge — automatically and safely. Same role as Reflections within the reference rubrics, profile thresholds, and verification suite, but the entire control plane is human-readable prompt logic instead of Python scripts.

## What This Package Is

reflections is a **scheduled cron consolidator** — a host-side structured memory maintenance layer with no script dependencies. It reads daily logs on a timer, consolidates extracted entries into its long-horizon surfaces, and archives low-importance entries via a forgetting curve.

It is **not** a plugin. It is **not** the active memory system. It does **not** intercept, modify, or compete with the host's native memory pipeline.

## What It Owns

- `RTMEMORY.md` — reflective long-horizon continuity (consolidated from daily logs)
- `PROCEDURES.md` — reusable workflows and stable operating patterns
- `episodes/*.md` — bounded event/project narratives
- `TRENDS.md` — recurring patterns / weak ops material
- `runtime/reflections-metadata.json` — index, stats, health history, route counters
- `runtime/reflections-deferred.jsonl` — persisted deferred-candidate store (strict mode)
- `memory/.reflections-log.md` — human-readable cycle reports
- `memory/.reflections-archive.md` — compressed older entries
- `reflections.json` — its own configuration (resolved dynamically at runtime)

## Why It Exists

Daily logs accumulate but stay disconnected. reflections runs periodic consolidation cycles that produce structured long-horizon memory — complementing the host's active memory system. The control plane is prompt-driven, so the entire pipeline is auditable from the prompts alone, with no opaque Python helpers.

## Works With memory-core

reflections is designed to run alongside OpenClaw's native memory-core dreaming system, not replace it.

**memory-core** handles active memory: recall during conversations, promotion of high-signal facts into `MEMORY.md`, native dreaming cycles that maintain the live durable surface. It owns what the agent knows right now.

**reflections** handles long-horizon maintenance: it reads the same daily logs that memory-core reads, but writes to different surfaces (`RTMEMORY.md`, `PROCEDURES.md`, `episodes/`, `TRENDS.md`). It captures decision rationale, project arcs, relationship history, and operational patterns that matter across weeks and months — material too structural or slow-moving for memory-core's active promotion.

| System | Time Horizon | What It Captures | Primary Surface |
|--------|-------------|-------------------|-----------------|
| memory-core | Active/recent | Durable facts, preferences, decisions | `MEMORY.md` |
| reflections | Long-horizon | Rationale, patterns, arcs, procedures | `RTMEMORY.md` |

Both systems share daily logs as input but never write to each other's surfaces. memory-core does not write to `RTMEMORY.md`. reflections does not write to `MEMORY.md`. There is no conflict, no race condition, and no ownership overlap.

If memory-wiki is also installed, it provides a third layer (compiled provenance-rich wiki). reflections does not interact with memory-wiki directly.

## Non-Goals

reflections does **NOT**:
- Replace memory-core's active memory — `MEMORY.md` remains owned by memory-core
- Intercept or modify daily logs — they are read-only input (only the `<!-- consolidated -->` marker is added)
- Delete any entries — only archives old low-importance ones
- Own agent identity or user profile surfaces — those remain in `IDENTITY.md` / `USER.md`
- Run without elapsed-time gating — mode dispatch self-regulates

## Memory Layers

| Layer | Storage | What Goes Here |
|-------|---------|----------------|
| **Working** | LCM plugin (optional) | Real-time context compression and recall |
| **Episodic** | `episodes/*.md` | Bounded event/project narratives |
| **Long-horizon** | `RTMEMORY.md` | Reflective continuity, decisions, lessons, identity/relationship shifts, architecture conclusions |
| **Procedural** | `PROCEDURES.md` | Reusable workflows, routines, operating patterns |
| **Trend** | `TRENDS.md` | Recurring patterns without stable method (weak ops material) |
| **Index** | `runtime/reflections-metadata.json` | Consolidation metadata, route counters, health stats |

## Features

- **Multi-mode consolidation** — rem (6h), deep (12h), core (daily) dispatch cadence
- **Importance scoring** — Auto-Dream family: marker × recency × log-reference boost
- **Profile-driven scored admission** (`strictMode`) — when enabled, `minScore`/`minRecallCount`/`minUnique` thresholds gate candidates before promotion; non-qualified items go into a persistent deferred store and are deterministically suppressed on future cycles. Default: `false` for personal-assistant (parity flow), `true` for business-employee (strict flow). Profile-opt-in.
- **Durability filter** — second-stage semantic admission after the structural gate. Routes each candidate into one of five lanes:
  - `promote` (new durable node)
  - `merge` (reinforce existing node, no new surface entry)
  - `compress` (upsert trend node in `TRENDS.md`)
  - `defer` (re-evaluate next cycle)
  - `reject` (discard)

  Hard-promote triggers rescue rare one-off high-consequence items that structural scoring underweights. Hard-suppress triggers reject telemetry noise regardless of reinforcement. Trend-to-durable promotion: an accumulated trend becomes eligible for RTMEMORY only when a fresh cycle adds a hard-promote trigger AND the trend meets support thresholds. Default on for business-employee; off for personal-assistant.
- **Fast-path markers** — `PERMANENT`/`HIGH`/`PIN` recognized for archival immunity and strict-mode routing
- **Intelligent forgetting** — old low-importance entries archived, never deleted
- **Knowledge graph** — semantic relation linking with reachability metrics
- **Health monitoring** — 5-metric health score (freshness, coverage, coherence, efficiency, reachability)
- **Push notifications** — silent or full consolidation reports per `sendReport` toggle
- **Plain-text identity keys** — deferred-store identity is composed from source date + target section + normalized summary stem + existingId. No SHA256 — identity stability comes from deterministic normalization, not cryptographic digest.
- **Token-usage visibility** (v1.1.0) — every telemetry event carries a `token_usage` block (`prompt_tokens` / `completion_tokens` / `total_tokens` / `source ∈ {exact, approximate, unavailable}`). When host metadata is unavailable, a char-count approximation may be used and is clearly labeled. The cycle log, final notification, and weekly block surface a `🪙 Tokens:` line when data exists; never fabricated when missing. Visibility-only — does not affect scoring, gating, deferring, routing, or archival behavior.

## Manual Triggers

- "Reflect now" (alias: "Dream now") — run all due modes immediately
- "Reflect core" / "Reflect rem" / "Reflect deep" — run a specific mode
- "Show reflection config" — display current `reflections.json`
- "Set consolidation mode to core only" — update active modes

## Safety

| Rule | Why |
|------|-----|
| Never delete daily logs | Immutable source of truth |
| Never remove `⚠️ PERMANENT` entries | User protection is absolute |
| Episodes are append-only | Narrative history preserved forever |
| Backup before any mutation | `runtime/reflections-metadata.json.bak` and `reflections.json.bak` precede every write |
| Daily logs marked `<!-- consolidated -->` per-log | Immediately after each log's consolidation succeeds; unmarked logs retry next cycle |
| Runtime/reference disagreement → blocker | Never silently reconcile rubric drift |

## Configuration

Key fields in `reflections.json` (full schema in `references/memory-template.md`):

- `strictMode` (boolean) — when `true`, recurring flow inserts a pre-consolidation gate and deterministic deferred-suppression. Profile-opt-in.
- `scanWindowDays` (integer) — days of recent daily logs the recurring prompt scans. Personal-assistant default `7`; business-employee default `3`. First-reflection bypasses this.
- `dispatchCadence` (cron expr) — when the host cron fires the recurring prompt.
- `durability.enabled` (boolean) — activates the second-stage semantic filter. Personal-assistant default `false`; business-employee default `true`. Only runs when `strictMode == true`.
- `durability.netPromoteThreshold` / `netDeferThreshold` — net-score bands (business 6/3; personal 5/2).
- `durability.trendPromoteSupportCount` / `trendPromoteUniqueDayCount` — trend-to-durable structural thresholds (business 5/3; personal 4/2).
- Per-mode thresholds (`minScore`, `minRecallCount`, `minUnique`) — used by strict-mode gate when `strictMode: true`.

**Dispatch semantics** — two separate layers:
- **Host scheduling** uses top-level `dispatchCadence`. That is the only cron schedule.
- **Mode-due check** inside the prompt uses hardcoded intervals — rem=6h, deep=12h, core=24h — compared against each mode's `lastRun` timestamp.

## Install

### Option 1: Quick Install (operator)

```bash
curl -fsSL https://raw.githubusercontent.com/catx0rr/reflections/main/install.sh | bash
```

Override defaults if needed:

```bash
CONFIG_ROOT="$HOME/.openclaw" \
WORKSPACE="$HOME/.openclaw/workspace" \
SKILLS_PATH="$HOME/.openclaw/workspace/skills" \
curl -fsSL https://raw.githubusercontent.com/catx0rr/reflections/main/install.sh | bash
```

### Option 2: Agent Setup

Tell your agent to read `INSTALL.md`:

> Install reflections, read the `INSTALL.md` follow every step and provide summary of changes after the install.

## Reference Documentation

| Document | Audience | Content |
|----------|----------|---------|
| `INSTALL.md` | Agent | Setup, configuration, profile selection, cron wiring, first-run bootstrap |
| `SKILL.md` | Agent | Manual-use skill — operator-triggered reflections |
| `runtime/reflections-prompt.md` | Agent | Recurring 9-step cycle execution contract |
| `runtime/first-reflections-prompt.md` | Agent | One-time bootstrap execution contract |
| `references/scoring.md` | Agent/operator | Importance formula, gates, fast-path, markers, decay table |
| `references/durability.md` | Agent/operator | 5-route filter, classification taxonomy, destinations, 15 worked examples |
| `references/health.md` | Agent/operator | 5-metric formula, reachability BFS, stale rubric, forgetting curve |
| `references/memory-template.md` | Agent/operator | Config presets, metadata.json schema, surface templates |
| `references/runtime-templates.md` | Agent/operator | Chat templates, log entry, weekly math, milestones, streak |
| `references/telemetry-schema.md` | Agent/operator | JSONL event shape per mode |
| `references/deferred-store.md` | Agent/operator | Deferred JSONL schema, plain-text identity key composition, normalization |
| `references/index-operations.md` | Agent/operator | Metadata CRUD with BEFORE/AFTER diffs |
| `references/worked-examples.md` | Agent/operator | End-to-end PA + BE cycle traces (illustrative only) |

## Credits

> **Origin:** This is a fork of [Hybrid Reflections](https://github.com/catx0rr/reflections-hybrid), itself a fork of [LeoYeAI/openclaw-auto-dream](https://github.com/LeoYeAI/openclaw-auto-dream). The control plane is implemented in prompt logic — no Python scripts.
>
> Reflections preserves the same long-horizon role as Auto-Dream and uses memory-core-safe surfaces (RTMEMORY.md, not MEMORY.md). reflections is behaviorally equivalent to Reflections within the reference rubrics, profile thresholds, and verification suite — but moves the entire control plane from Python scripts to prompt logic.
>
> Both forks coexist with OpenClaw's native memory-core. They read the same daily logs but write to separate primary targets — complementary but distinct roles.

## License

[MIT](LICENSE)
