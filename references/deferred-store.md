# Deferred Store — JSONL Schema, Identity Composition, Normalization

This document is the **rubric authority** for Steps 1.2 (load deferred state), 1.3 (annotate deferred), and the deferred-append paths in Steps 1.6 / 1.8.

The deferred store is a persistent record of candidates that failed structural gates (Step 1.6) or routed to `defer` (Step 1.8). It exists so future cycles can deterministically suppress candidates that have already been judged not-yet-promotable — without the LLM having to remember decisions across runs.

**Strict-mode only.** In parity flow (`strictMode == false`), the deferred store does not exist and these steps are skipped.

---

## 1. File path and format

- **Path:** `runtime/reflections-deferred.jsonl` (relative to `WORKSPACE_ROOT`)
- **Format:** JSONL — one JSON object per line, newline-terminated
- **Discipline:** append-only. Lines are never edited or deleted by the runtime.
- **Encoding:** UTF-8. Each line is compact JSON (no indentation) with `\n` terminator.

The append-per-line discipline means concurrent cron fires are safe at the line level — the OS append guarantees no interleaving inside a single line.

---

## 2. Per-record schema

Each line is one record describing a deferred candidate event:

```json
{
  "summary": "One-line candidate summary",
  "source": "memory/2026-04-18.md",
  "target_section": "Key Decisions and Rationale",
  "identityKey": "2026-04-18::key-decisions-and-rationale::stripe paymongo gateway switch::-",
  "existingId": null,
  "fail_reasons": {"rem": ["minScore: 0.62 < 0.85"], "deep": ["minRecallCount: 1 < 2"]},
  "modes_evaluated": ["rem", "deep", "core"],
  "referenceCount": 1,
  "timestamp": "2026-04-18T22:30:14+08:00",
  "timestamp_utc": "2026-04-18T14:30:14Z"
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `summary` | string | yes | Candidate summary at time of deferral |
| `source` | string | yes | Daily log file path (`memory/YYYY-MM-DD.md`) |
| `target_section` | string | yes | RTMEMORY section the candidate was destined for |
| `identityKey` | string | yes | Plain-text composed key (see §3) |
| `existingId` | string \| null | yes | When the candidate matched an existing index entry, that entry's `mem_NNN`; else `null` |
| `fail_reasons` | object | yes | Object keyed by mode showing which condition(s) failed gate |
| `modes_evaluated` | array | yes | Modes that evaluated this candidate (e.g. `["rem", "deep", "core"]`) |
| `referenceCount` | number | yes | Reference count at time of deferral |
| `timestamp` | string | yes | Local-aware ISO 8601 with timezone offset |
| `timestamp_utc` | string | yes | UTC ISO 8601 with `Z` suffix |

When the deferral comes from durability routing (Step 1.8), `fail_reasons` is replaced/augmented with a `durability` block:

```json
{
  "fail_reasons": {
    "durability": {
      "route": "defer",
      "reason": "net=3>=3",
      "durabilityScore": 3,
      "noisePenalty": 1,
      "mergeKey": null,
      "trendKey": null,
      "duplicateOfExisting": null
    }
  }
}
```

---

## 3. Identity composition (plain text, deterministic)

The `identityKey` is a plain-text composition that the LLM writes directly. **No hashing.** Composition is fully deterministic — same inputs always produce the same key.

### 3.1 Composition formula

```
identityKey = "<source_date>::<normalized_target_section>::<normalized_summary>::<existingId_or_dash>"
```

Where:

- `source_date` — the YYYY-MM-DD date extracted from `source` (the substring `2026-04-18` from `memory/2026-04-18.md`). If `source` doesn't match the date pattern, use `"unknown-date"`.
- `normalized_target_section` — `target_section` after the normalization rule in §3.2.
- `normalized_summary` — `summary` after the normalization rule in §3.2.
- `existingId_or_dash` — the `existingId` value if non-null, else literal `-`.

The `::` separator (two colons) is the delimiter. Components themselves cannot contain `::` because normalization strips punctuation.

### 3.2 Normalization rule

Apply these steps in order to `target_section` and to `summary`:

1. **Lowercase the entire string.**
2. **Replace any non-word character with a single space.** (Word characters are letters, digits, and underscore. Punctuation, emoji, brackets all become spaces.)
3. **Collapse runs of whitespace into a single space.**
4. **Strip leading/trailing whitespace.**
5. **Split into tokens on space boundaries.**
6. **Drop tokens in the stopword list (§3.3).**
7. **Sort the remaining tokens alphabetically.**
8. **Deduplicate adjacent identical tokens.**
9. **Re-join with single spaces.**

This is a token-bag normalization: same content words in any order produce the same normalized string. Light rewording (case, punctuation, word order, filler words) survives. Heavy rewording (different content words) does not.

### 3.3 Stopword list

These 25 English filler words are dropped during normalization. The list is intentionally small — it removes trivial filler without stripping semantic content.

```
a an the and or but of to in on at for with by from
is are was were be been being that this these those it its
```

Non-English text passes through unchanged (no per-language stopword handling).

### 3.4 Worked normalization examples

| Input string | After normalization |
|--------------|---------------------|
| `"Decided to switch from Paymongo to Stripe!"` | `"decided from paymongo stripe switch"` |
| `"DECIDED to SWITCH from paymongo to stripe."` | `"decided from paymongo stripe switch"` (same) |
| `"From Paymongo: switching to Stripe — decided."` | `"decided from paymongo stripe switching"` (similar but `switching` ≠ `switch`) |
| `"We're rebuilding the entire payment stack with a new vendor."` | `"entire new payment rebuilding re stack vendor we"` (different content words) |
| `"Key Decisions and Rationale"` | `"decisions key rationale"` |

The first two normalize to the same string — light rewording survives. The third differs by one token (`switching` vs `switch`) — substantive enough to differ. The fourth is entirely different content words — completely different.

### 3.5 Worked identityKey examples

| source | target_section | summary | existingId | identityKey |
|--------|----------------|---------|------------|-------------|
| `memory/2026-04-18.md` | `Key Decisions and Rationale` | `Decided to switch from Paymongo to Stripe!` | `null` | `2026-04-18::decisions key rationale::decided from paymongo stripe switch::-` |
| `memory/2026-04-18.md` | `Key Decisions and Rationale` | `DECIDED to SWITCH from paymongo to stripe.` | `null` | `2026-04-18::decisions key rationale::decided from paymongo stripe switch::-` (same — light rewording) |
| `memory/2026-04-18.md` | `Key Decisions and Rationale` | `Decided to switch from Paymongo to Stripe!` | `mem_042` | `2026-04-18::decisions key rationale::decided from paymongo stripe switch::mem_042` (different — existingId now set) |
| `memory/2026-04-19.md` | `Key Decisions and Rationale` | `Decided to switch from Paymongo to Stripe!` | `null` | `2026-04-19::decisions key rationale::decided from paymongo stripe switch::-` (different — different day) |

### 3.6 Stability properties

| Variation in input | Same identityKey? | Reason |
|--------------------|-------------------|--------|
| Capitalization differences | Yes | Step 1 lowercases |
| Punctuation differences | Yes | Step 2 strips |
| Whitespace differences | Yes | Steps 3–4 collapse |
| Reordered words | Yes | Step 7 sorts |
| Filler word added/removed | Yes (if filler is in stopword list) | Step 6 drops |
| Different source date | No | Date is part of the key |
| Different target section (substantively) | No | Section tokens differ |
| Substantively reworded summary | No | Different content tokens |
| Same content, different existingId | No | existingId is part of the key |

**Collision surface is bounded** by the source date + target section pair. Two genuinely-distinct candidates from the same daily log + same section that happen to share their content tokens after normalization would collide — but at the daily-log level the collision risk is small, and a false collision just means one deferred record absorbs a near-duplicate (the intended behavior for light-rewording suppression).

---

## 4. Step 1.3 — Annotate deferred (lookup logic)

For each candidate from Step 1, compute its `identityKey` and look it up against the loaded `DEFERRED_RECORDS`.

### 4.1 Lookup algorithm

```
For each candidate in candidates:
  key = compose_identity_key(candidate.source, candidate.target_section,
                             candidate.summary, candidate.existingId)

  match = None
  match_layer = None

  # Layer 1: existingId match (strongest — survives any rewording)
  if candidate.existingId is not None:
    for record in DEFERRED_RECORDS:
      if record.existingId == candidate.existingId:
        match = record
        match_layer = "existingId"
        break

  # Layer 2: full identityKey match (rewording-stable)
  if match is None:
    for record in DEFERRED_RECORDS:
      if record.identityKey == key:
        match = record
        match_layer = "identityKey"
        break

  # Annotate the candidate
  candidate.identityKey = key
  candidate.deferred_status = "persisted" if match else "fresh"
  candidate.deferred_matched_by = match_layer  # or None
```

### 4.2 Lookup priority

1. **`existingId` first** — if both the candidate and a deferred record share an `existingId`, that match wins regardless of summary changes. This handles the "we keep mentioning mem_042 in different ways" case.
2. **Full `identityKey`** — exact match on the composed key. Catches reworded duplicates from the same source+section.

If no record matches, `deferred_status = "fresh"` and the candidate proceeds through the cycle normally.

---

## 5. Append path (Steps 1.6 + 1.8)

When Step 1.6 (gate) defers candidates, OR Step 1.8 (durability) routes candidates to `defer`, append one record per candidate to `runtime/reflections-deferred.jsonl`:

1. Read the current file (skip if missing — the directory will be created on first append).
2. For each new deferred candidate:
   - Build the per-record schema from §2 (compute `identityKey` if not already set).
   - Serialize as compact single-line JSON (no indentation, separators `,` and `:`).
   - Append the line plus `\n`.
3. Save the file.

Always include the timestamp triple per the project convention. The `target_section` field should match the candidate's target section so identity composition is stable.

---

## 6. Loading the store (Step 1.2)

Read `runtime/reflections-deferred.jsonl` line by line. Skip:

- Empty lines (whitespace only)
- Lines that fail JSON parsing (log internally; do not block the cycle)

The result is `DEFERRED_RECORDS` — an in-memory list of records used by Step 1.3.

If the file does not exist, `DEFERRED_RECORDS = []` (empty list). This is normal on first run.

---

## 7. Backstop verification (Step 2.7)

When marking a daily log `<!-- consolidated -->`, the runtime checks each candidate from that log reached a terminal state. For candidates with `route == "defer"`, the runtime verifies the candidate's `identityKey` (or `existingId`) is now present in the deferred store before allowing the log to be marked.

Verification is a simple membership check — the candidate's identity matches some record in the store.

---

## 8. Authority and conflict handling

If this rubric appears to disagree with the runtime's Step 1.2 / 1.3 / 1.6 / 1.8 instruction, **stop and emit a blocker** per the runtime's precedence rule. Do not silently reconcile.

This doc owns the `identityKey` composition rule and the lookup priority. The runtime owns the *when* and *whether*.

The plain-text composition is intentional — no hashing, no sha256. Identity stability is achieved via deterministic normalization, not cryptographic digest. This is one of the architectural differences from the script-based parent project.
