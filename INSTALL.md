# reflections — Installation Guide

This guide is **agent-facing**. The agent should read it end to end and execute every step. No Python is required — the entire package is prompt-driven.

---

## Path Terminology

| Variable | Default | Notes |
|----------|---------|-------|
| `CONFIG_ROOT` | `$HOME/.openclaw` | Where `reflections.json` lives |
| `WORKSPACE` | `$HOME/.openclaw/workspace` | Where memory surfaces, runtime state, daily logs live |
| `SKILLS_PATH` | `$HOME/.openclaw/workspace/skills` | Where the `reflections` skill is installed |
| `SKILL_ROOT` | `$SKILLS_PATH/reflections` | The skill directory (this package) |
| `TELEMETRY_ROOT` | `$HOME/.openclaw/telemetry` | Where consolidation telemetry JSONL lives |

Override at install time with environment variables (see Step 0).

---

## Prerequisites

- **Git** — for cloning the repo
- **A standard shell** (bash/zsh) — for running `install.sh`
- **NO Python required.** The control plane is prompt-based — no scripts to invoke.

---

## Step 0: Run install.sh (or manual clone)

### Option A — Quick install

```bash
curl -fsSL https://raw.githubusercontent.com/catx0rr/reflections/main/install.sh | bash
```

To override defaults:

```bash
CONFIG_ROOT="$HOME/.openclaw" \
WORKSPACE="$HOME/.openclaw/workspace" \
SKILLS_PATH="$HOME/.openclaw/workspace/skills" \
curl -fsSL https://raw.githubusercontent.com/catx0rr/reflections/main/install.sh | bash
```

### Option B — Manual clone

```bash
mkdir -p "$SKILLS_PATH"
git clone https://github.com/catx0rr/reflections.git "$SKILL_ROOT"
bash "$SKILL_ROOT/install.sh"
```

The installer will:
- Clone or update the repo at `$SKILL_ROOT`
- Create the workspace topology (`RTMEMORY.md`, `PROCEDURES.md`, `runtime/`, `episodes/`, `memory/`, `TRENDS.md`, `memory/.reflections-log.md`, `memory/.reflections-archive.md`, `runtime/reflections-metadata.json`)
- Initialize `runtime/memory-state.json` with the `reflections` namespace (preserving any existing namespaces from other packages)

After this completes, continue with the steps below.

---

## Step 1: Register the skill (host-specific)

Tell the host where the skill lives. The exact mechanism depends on the host (OpenClaw, custom CLI, etc.). For an OpenClaw-style install:

Add `$SKILLS_PATH` to the host's `extraDirs` if not already present. The host should auto-discover the skill from its `SKILL.md` file.

Verify:

```bash
ls "$SKILL_ROOT/SKILL.md"
```

Expected: file exists.

---

## Step 2: Select profile

Choose one of:

- **personal-assistant** — owner-centric assistants, family/butler/concierge agents. Defaults: parity flow (`strictMode: false`), 7-day scan, broader fast-path markers, durability disabled.
- **business-employee** — bounded work agents, supervisor DM, small team GCs. Defaults: strict flow (`strictMode: true`), 3-day scan, narrower fast-path, durability enabled.

Set the `REFLECTIONS_PROFILE` environment variable, OR ask the operator interactively.

```bash
export REFLECTIONS_PROFILE=personal-assistant   # or business-employee
```

---

## Step 3: Create reflections.json

Compose `$CONFIG_ROOT/reflections/reflections.json` from the selected profile preset in `references/memory-template.md` §1.1 (personal-assistant) or §1.2 (business-employee).

Set:

- `agent` — the agent's identifier (default `"main"`)
- `timezone` — IANA timezone name (e.g. `"Asia/Manila"`)
- All other fields from the preset

Example for personal-assistant in Asia/Manila:

```bash
mkdir -p "$CONFIG_ROOT/reflections"
cat > "$CONFIG_ROOT/reflections/reflections.json" <<'CFG'
{
  "version": "1.0.0",
  "profile": "personal-assistant",
  "agent": "main",
  "timezone": "Asia/Manila",
  "strictMode": false,
  "scanWindowDays": 7,
  "dispatchCadence": "30 4,10,16,22 * * *",
  "activeModes": ["core", "rem", "deep"],
  "lastRun": {"core": null, "rem": null, "deep": null},
  "durability": {
    "enabled": false,
    "netPromoteThreshold": 5,
    "netDeferThreshold": 2,
    "trendPromoteSupportCount": 4,
    "trendPromoteUniqueDayCount": 2
  },
  "modes": {
    "core": {
      "enabled": true, "minScore": 0.72, "minRecallCount": 2, "minUnique": 1,
      "uniqueMode": "day_or_session",
      "fastPathMinScore": 0.90, "fastPathMinRecallCount": 2,
      "fastPathMarkers": ["HIGH", "PIN", "PREFERENCE", "ROUTINE"]
    },
    "rem": {
      "enabled": true, "minScore": 0.85, "minRecallCount": 2, "minUnique": 2,
      "uniqueMode": "day_or_session",
      "fastPathMinScore": 0.88, "fastPathMinRecallCount": 2,
      "fastPathMarkers": ["HIGH", "PIN", "PREFERENCE", "ROUTINE", "PROCEDURE"]
    },
    "deep": {
      "enabled": true, "minScore": 0.80, "minRecallCount": 2, "minUnique": 2,
      "uniqueMode": "day_or_session",
      "fastPathMinScore": 0.86, "fastPathMinRecallCount": 2,
      "fastPathMarkers": ["HIGH", "PIN", "PREFERENCE", "ROUTINE"]
    }
  }
}
CFG
```

For business-employee, use the §1.2 preset.

---

## Step 4: Verify workspace topology

The installer should have created these. If anything is missing, create it.

```bash
test -f "$WORKSPACE/RTMEMORY.md"                       # must exist
test -f "$WORKSPACE/PROCEDURES.md"                     # must exist
test -f "$WORKSPACE/memory/.reflections-log.md"        # must exist
test -f "$WORKSPACE/memory/.reflections-archive.md"    # must exist
test -f "$WORKSPACE/TRENDS.md"                  # must exist
test -f "$WORKSPACE/runtime/reflections-metadata.json" # must exist
test -f "$WORKSPACE/runtime/memory-state.json"         # must exist (with reflections namespace)
test -d "$WORKSPACE/episodes"                          # must exist
test -d "$WORKSPACE/memory"                            # must exist
test -d "$WORKSPACE/runtime"                           # must exist
```

---

## Step 5: Create the cron job

Wire the host cron to fire `runtime/reflections-prompt.md` at the cadence specified in `reflections.json`'s `dispatchCadence`.

For an OpenClaw-style install:

```bash
openclaw cron add \
  --name "reflections" \
  --cron "30 4,10,16,22 * * *" \
  --tz "Asia/Manila" \
  --session isolated \
  --no-deliver \
  --message "Read $SKILL_ROOT/runtime/reflections-prompt.md and follow every step."
```

Adjust `--cron` to match the profile's `dispatchCadence`. Adjust `--tz` to the operator's IANA timezone. Use `--no-deliver` initially (telemetry-only); switch to `--announce <route>` later if the operator wants chat reports.

**Idempotency:** check first whether a `reflections` job already exists:

```bash
openclaw cron list --json | grep -q '"name":"reflections"' && echo "already exists" || echo "creating"
```

If it exists, skip the `cron add` step.

---

## Step 6: Configure reporting (optional)

Edit `$WORKSPACE/runtime/memory-state.json`. The `reflections.reporting` namespace controls chat eligibility:

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

| Field | Meaning |
|-------|---------|
| `sendReport: true` | Step 4.2 of the recurring cycle emits a chat notification |
| `sendReport: false` | Telemetry-only mode (always logs to JSONL but no chat) |
| `delivery.channel` | `"last"` (reuse last route) or explicit channel name |
| `delivery.to` | Explicit target identifier (takes precedence over `channel`) |

The installer ships with `sendReport: true` and `channel: "last"` by default. The cron `--no-deliver` flag from Step 5 overrides this if you want to silence the cron specifically.

---

## Step 7: Run the first reflection

Tell the agent to execute the bootstrap prompt:

> Read `$SKILL_ROOT/runtime/first-reflections-prompt.md` and follow every phase.

The first reflection:
- Bypasses all quality gates by design — every extracted entry consolidates
- Scans **all** unconsolidated daily logs (not limited by `scanWindowDays`)
- Produces a before/after snapshot report
- Marks each processed log with `<!-- consolidated -->`
- If the workspace is fresh (no daily logs, minimal RTMEMORY.md), produces a "fresh-instance" report instead

---

## Step 8: Verify

Manual verification checklist:

```bash
# Topology
test -f "$WORKSPACE/RTMEMORY.md"
test -f "$WORKSPACE/PROCEDURES.md"
test -f "$WORKSPACE/TRENDS.md"
test -f "$WORKSPACE/runtime/reflections-metadata.json"
test -f "$CONFIG_ROOT/reflections/reflections.json"

# No script artifacts
test ! -d "$SKILL_ROOT/scripts"

# Skill registered
ls "$SKILL_ROOT/SKILL.md"

# After at least one cycle, telemetry should exist
ls "$TELEMETRY_ROOT/memory-log-$(date +%Y-%m-%d).jsonl" 2>/dev/null || echo "no telemetry yet — wait for first cycle"
```

If everything passes, the install is complete.

---

## Boundary statement

reflections is a host-side memory consolidator. It:
- Reads daily logs (read-only except for the `<!-- consolidated -->` marker)
- Writes to its own surfaces (RTMEMORY.md, PROCEDURES.md, episodes/, TRENDS.md, runtime/reflections-metadata.json, memory/.reflections-log.md, memory/.reflections-archive.md, runtime/reflections-deferred.jsonl)
- Updates one shared namespace (`runtime/memory-state.json` `reflections.*`) with merge-not-overwrite discipline
- Writes telemetry to `$TELEMETRY_ROOT`

It does **not**:
- Modify daily logs beyond marking
- Touch `MEMORY.md` (memory-core's surface)
- Touch `IDENTITY.md` / `USER.md`
- Auto-install plugins
- Modify host configuration

**No Python is invoked at any point.** The entire control plane is prompt-driven.

---

## Cleanup (after a fresh install)

The repo includes documentation and the install script. After a successful install, you can optionally remove these from the deployed `$SKILL_ROOT` (keeping only what the runtime needs):

- `.git/` (only if you don't intend to `git pull` for updates)
- `LICENSE`, `README.md`, `install.sh`, `INSTALL.md` (informational; not consumed by runtime)

The runtime needs only `runtime/`, `references/`, `profiles/`, and `SKILL.md`. There is no `scripts/` directory to clean up.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Cycle runs but nothing consolidates | All daily logs already marked, or no work in `scanWindowDays` window | Check log markers; expand window; or run first-reflection again |
| Notification missing fields | Field source was null and was correctly omitted | This is the omission rule — never invented |
| Blocker on cycle | Runtime/reference rubric drift, or workspace unwritable, or config missing | Read telemetry `details.step` and `details.blocker_type` |
| Telemetry not appearing | `$TELEMETRY_ROOT` not writable | Check directory exists and permissions |
| Chat output missing despite `sendReport: true` | Cron set with `--no-deliver` | Recreate cron with `--announce <route>` |

For runtime/reference disagreement specifically: the runtime emits a `run_failed` blocker telemetry event with `details.blocker_type: "rubric_conflict"`. Read both the runtime step and the cited reference to identify which one needs updating.
