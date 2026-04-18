# Telemetry Schema — JSONL Event Shape per Mode

This document is the **rubric authority** for Step 0-B (skip telemetry) and Step 3.9 (cycle telemetry — second durable write, after index commit). The runtime cites it; this file owns the event shape, the mode-aware details payload, and the path resolution.

Telemetry is **always written** regardless of `reflections.reporting.sendReport`. The reporting toggle controls chat delivery only — never logging.

---

## 1. Output target

```
$TELEMETRY_ROOT/memory-log-YYYY-MM-DD.jsonl
```

Daily-sharded, append-only, one JSON line per event.

### 1.1 Path resolution (priority order)

1. Explicit `--telemetry-dir` style override (operator-provided)
2. `REFLECTIONS_TELEMETRY_ROOT` environment variable
3. `MEMORY_TELEMETRY_ROOT` environment variable
4. `~/.openclaw/telemetry` (default)

The runtime resolves `TELEMETRY_ROOT` once at the top of the cycle and uses it for every append.

### 1.2 File creation

If `$TELEMETRY_ROOT` doesn't exist, create it (with parents) before the first append. The file `memory-log-<today>.jsonl` is created if missing on first append of the day.

---

## 2. Event envelope (common to every event)

Every event has this base shape:

```json
{
  "timestamp": "2026-04-18T22:30:14+08:00",
  "timestamp_utc": "2026-04-18T14:30:14Z",
  "domain": "memory",
  "component": "reflections.consolidator",
  "event": "<event name>",
  "run_id": "refl-2026-04-18T22-30-14-08-00-a3f9c1",
  "status": "ok | error | skipped",
  "agent": "main",
  "profile": "personal-assistant | business-employee",
  "mode": "scheduled | manual | first-reflection",
  "token_usage": {
    "prompt_tokens":     <int> | null,
    "completion_tokens": <int> | null,
    "total_tokens":      <int> | null,
    "source":            "exact | approximate | unavailable"
  }
}
```

### 2.1 Field rules

| Field | Type | Notes |
|-------|------|-------|
| `timestamp` | string | Local-aware ISO 8601 with timezone offset (primary) |
| `timestamp_utc` | string | UTC ISO 8601 with `Z` suffix (companion for correlation) |
| `domain` | string | Always `"memory"` |
| `component` | string | Always `"reflections.consolidator"` |
| `event` | string | One of `run_completed`, `run_skipped`, `run_failed` |
| `run_id` | string | Generated per §3 |
| `status` | enum | `"ok"` (success) / `"error"` (failure) / `"skipped"` (no work) |
| `agent` | string | Agent identifier (default `"main"`) |
| `profile` | string | Active profile name |
| `mode` | string | `"scheduled"` (cron), `"manual"` (operator-triggered), `"first-reflection"` (bootstrap) |
| `token_usage` | object | Visibility-only field. Resolve per `references/recurring-rules.md` §7 (exact / approximate / unavailable ladder). Never fabricate `"exact"`. |

### 2.2 Timestamp triple discipline

The runtime always produces both `timestamp` and `timestamp_utc`. The local timestamp is primary; the UTC companion is for cross-host correlation.

Helper pattern:

```
now_local = current local time, timezone-aware
now_utc   = now_local in UTC

timestamp     = now_local.isoformat()                       (e.g. "2026-04-18T22:30:14+08:00")
timestamp_utc = now_utc.isoformat() with "+00:00" → "Z"     (e.g. "2026-04-18T14:30:14Z")
```

Never write naive timestamps. Never write `+00:00Z` (invalid).

---

## 3. run_id generation

```
ts_clean = timestamp string with ':' → '-', '+' → '-', '.' → '-', truncated to 19 chars
suffix   = first 6 hex chars of any short stable identifier derived from timestamp
           (a simple deterministic derivation — e.g. take the last 6 alphanumeric chars
           of the timestamp string after dropping non-alphanumerics)

run_id   = "refl-" + ts_clean + "-" + suffix
```

Example: `timestamp = "2026-04-18T22:30:14+08:00"` → `ts_clean = "2026-04-18T22-30-14"` → suffix = (some 6-char tail) → `run_id = "refl-2026-04-18T22-30-14-a3f9c1"` (suffix shape may vary).

The suffix exists to disambiguate concurrent fires within the same second. Any 6-char tail derivation is acceptable as long as it's deterministic for the same timestamp.

---

## 4. Per-mode `details` payload

The `details` field is appended to the envelope and varies by which flow ran.

### 4.1 Skip event (`event = "run_skipped"`, `status = "skipped"`)

```json
{
  "details": {
    "reason": "no_modes_due | no_work | both",
    "due_modes": ["core"],
    "unconsolidated_count": 0
  }
}
```

### 4.2 Parity flow (`strictMode = false`)

`event = "run_completed"`, `status = "ok"`:

```json
{
  "details": {
    "logs_scanned": 3,
    "entries_extracted": 7,
    "entries_consolidated": 7,
    "logs_marked_consolidated": 3
  }
}
```

### 4.3 Strict flow without durability (`strictMode = true`, `durability.enabled = false`)

```json
{
  "details": {
    "logs_scanned": 3,
    "entries_extracted": 12,
    "entries_qualified": 6,
    "entries_deferred": 6,
    "entries_promoted": 6,
    "logs_marked_consolidated": 2
  }
}
```

### 4.4 Strict flow with durability (`strictMode = true`, `durability.enabled = true`)

```json
{
  "details": {
    "logs_scanned": 3,
    "entries_extracted": 12,
    "entries_qualified": 6,
    "entries_deferred": 4,
    "entries_durable_promoted": 3,
    "entries_durable_merged": 1,
    "entries_durable_compressed": 1,
    "entries_durable_deferred": 1,
    "entries_durable_rejected": 0,
    "logs_marked_consolidated": 2
  }
}
```

### 4.5 First-reflection flow (`mode = "first-reflection"`)

First-reflection bypasses all gates. There is no `entries_qualified` to report — use `entries_consolidated` instead.

```json
{
  "details": {
    "logs_scanned": 14,
    "entries_extracted": 47,
    "entries_consolidated": 47
  }
}
```

### 4.6 Failure event (`event = "run_failed"`, `status = "error"`)

```json
{
  "error": "Config file missing at /home/user/.openclaw/reflections/reflections.json",
  "details": {
    "step": "Step 0 dispatch",
    "blocker_type": "config_missing | workspace_unwritable | malformed_index | other"
  }
}
```

The `error` field is at the envelope level (not inside `details`).

---

## 5. Append discipline

Each event is one line. Use compact JSON (no indentation):

1. Build the event object.
2. Serialize to single-line JSON with separators `,` and `:` (no spaces).
3. Append the line plus `\n` to `memory-log-<today>.jsonl`.

Append-per-line guarantees concurrent cron fires don't interleave inside a record.

If the append fails (disk full, permission), the runtime emits a blocker chat message (if reporting is on) and stops — but still attempts to record the failure event somewhere (e.g., stderr, fallback file). Telemetry failure does not silently drop.

---

## 6. Always-on principle

| Phase | Telemetry written? |
|-------|-------------------|
| Skip cycle (Step 0-B) | Yes — `run_skipped` event |
| Successful cycle (Step 3.9) | Yes — `run_completed` event |
| Failure / blocker | Yes — `run_failed` event |
| First reflection (post-install) | Yes — `run_completed` with `mode: "first-reflection"` |

`reflections.reporting.sendReport == false` means **no chat output**. Telemetry is unaffected.

Every event (skip, completed, failed, first-reflection) carries the `token_usage` envelope field. When nothing is known, `source: "unavailable"` with nulls — still written, never omitted from the envelope.

---

## 7. Authority and conflict handling

If this rubric appears to disagree with the runtime's Step 0-B / 4.2 instruction, **stop and emit a blocker** per the runtime's precedence rule. Do not silently reconcile.

This doc owns the event envelope, run_id derivation, and per-mode details payload. The runtime owns the *when* and *with what counter values*.
