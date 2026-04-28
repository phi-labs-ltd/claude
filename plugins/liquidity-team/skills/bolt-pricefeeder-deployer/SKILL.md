---
name: bolt-pricefeeder-deployer
description: Validate bolt-pricefeeder calibration / parameter-update knobs against the latest released config schema BEFORE the request becomes a Linear ticket the dev applies to the deployment YAML. Catches (1) knobs that do not exist in the schema, (2) name divergence (e.g. `clamp_delta` vs `clamp_price_delta`, `spread_sens` vs `spread_sensitivity`), and (3) unit drift — calibration emits decimals while several fields are typed `Bps` (integer basis points), so copy-paste lands off by 10000×. Trigger automatically whenever bolt-pricefeeder config or parameter changes are discussed — knob names in chat (`avo`, `base_alpha`, `spread_sens`, `vol_sens`, `clamp_delta`, `base_spread`, `jump_threshold`, `crisis_sigma`, `min_venues`, etc.), a calibration winner identifier, a `Knob | Value` markdown table, casual proposals like "can we bump X to Y?", a draft Linear ticket. Do not wait for the words "validate" or "review". Surface findings as questions; never auto-correct.
---

# bolt-pricefeeder-deployer

Confirms that knob names being discussed for a bolt-pricefeeder calibration / parameter update actually exist in the latest published config schema, and flags the typical issues that get caught in dev review.

## Audience and intent

The user is usually the **liquidity team**, talking through a calibration winner or proposing parameter changes that a developer will eventually apply to the deployment YAML. The user does **not** have the repo checked out and does **not** run CLI commands. The conversation may be informal — chat messages, pasted snippets, half-tables, single-knob mentions in passing.

This skill catches the three failure modes that have repeatedly slipped past ticket review:

1. **Nonexistent knobs.** Some calibration knobs are model-internal and have no corresponding deployed config field. Catch them before the request is filed.
2. **Typo'd or shorthand names.** Calibration vocabulary often diverges from the schema's exact field name — e.g. `clamp_delta` while the schema field is `clamp_price_delta`, or `spread_sens` while the schema field is `spread_sensitivity`. Surface a candidate schema name as a question, never silently rewrite.
3. **Unit drift.** Calibration emits decimals (e.g. `base_spread: 0.0005`); the schema field may be typed `Bps` (integer basis points). Whenever a `Bps` field is targeted with a fractional value, surface the candidate `× 10000` conversion as a question.

This is a **review skill, not an apply skill**. Surface the report; the dev applies the change later.

## Core principle: ask, never assume

Every discrepancy between the proposed knobs and the schema must be **surfaced as a question to the user**, not silently corrected. The agent's job is to *detect and disclose*, not to *decide*. This holds even when a candidate match looks confident, even when a `× 10000` unit fix looks obvious, and even when a near-miss looks like an unambiguous typo.

Concretely:

- A schema property that fuzzy-matches a calibration knob is a **candidate suggestion**, not ground truth. New schema releases may introduce names that look the same but mean something different. Always present the match as "I think this maps to X — please confirm" rather than rewriting silently.
- A decimal value on a `Bps`-typed field is **probably** a unit drift, but the calibration tool may already have converted to bps and the value may have been edited by hand. Always present the proposed `× 10000` conversion as a question, not a rewrite.
- A knob that does not appear anywhere in the schema may be a calibration-model-only parameter to drop, **or** a brand-new schema field, **or** a typo. The agent must not pick — it must ask.
- Even an unambiguous-looking typo (e.g. proposed `clamp_dleta` when the schema has `clamp_price_delta`) gets phrased as "did you mean…?". The user resolves it, not the agent.

After the report, do **not** emit a corrected ticket-ready table until the user has explicitly resolved every flagged item. If the user pushes back on any suggestion ("no, leave that as-is", "the value is correct, the unit is what we want"), respect it.

## Workflow

### 1. Notice that pricefeeder changes are being discussed

The input is **whatever the user has shared in the conversation so far**. There is no required format. Treat any of these as input and proceed without waiting for a formal request:

- A markdown table of `Knob | Value` rows pasted from a draft ticket.
- A bare list of `name: value` or `name = value` lines.
- A YAML / JSON snippet from a calibration tool.
- A chat message naming a single knob with a proposed value ("can we bump `base_alpha` to 0.012?").
- Any casual mention of a pricefeeder parameter name in a context that implies a change is being proposed.

If the input is partial or ambiguous (a knob is named without a value, a value is given without a name, a table column is unclear), **ask the user to fill in the gap before classifying**. Do not guess.

### 2. Find the latest schema version

The schema is published per release on GitHub Pages. Fetch this URL to discover the current release tag:

```
https://api.github.com/repos/phi-labs-ltd/bolt-pricefeeder/releases/latest
```

The response is JSON; the `tag_name` field is the latest tag (e.g. `v1.11.0`). Use whatever web-fetching capability is available in the current session to retrieve it.

### 3. Fetch the schema for that release

Once the tag is known, fetch the schema document from GitHub Pages:

```
https://phi-labs-ltd.github.io/bolt-pricefeeder/<tag>/bolt-pricefeeder-config-schema.json
```

For example, for `v1.11.0` the URL resolves to `https://phi-labs-ltd.github.io/bolt-pricefeeder/v1.11.0/bolt-pricefeeder-config-schema.json`. The response is the JSON Schema document the deployed binary expects. Read it in full — it is a single small JSON document and is the **single source of truth** for what fields exist and what types they have.

Do not rely on field names listed in this skill document, in chat history, or in past tickets. Always re-derive the field set from the freshly fetched schema.

### 4. Build a property index from the schema

Walk the fetched schema and assemble, in-memory for this session:

- **Field names** — every `properties.<name>` key under the root and under each entry of `$defs`. Pay particular attention to the variants (`oneOf`) and their nested properties.
- **Field paths** — record the dotted path from the relevant root (e.g. `assets.<pair>.fetch_pipeline.spread_sensitivity`, or `circuit_breaker.stages[oracle_jump].threshold_bps`) so the report can name where the dev will edit.
- **`Bps`-typed fields** — any property whose type definition is `{ "$ref": "#/$defs/Bps" }` (or whatever the schema's basis-points definition is named) is integer basis points. Field names ending in `_bps` are also bps by convention. Record this set so unit drift can be detected.
- **Required-fields lists** — under `required` on each object schema. These explain "missing field" findings.
- **`oneOf` discriminators** — note which variant is selected by which discriminator field (e.g. a fetch-pipeline variant selected by a `strategy` field, a halt-stage variant selected by a `type` field). These explain "additional properties not allowed" findings.

The point of this step is that **you re-derive the index every session**, so this skill document does not need to be updated when the schema changes. Whatever fields the schema currently defines are the fields the dev can apply.

### 5. For each proposed knob, classify it

For each `name: value` extracted from the conversation:

1. **Verbatim match in the property index → DIRECT MATCH.** No prompt needed for the name. Still check the value type against the schema (integer vs number, in-range, matches enum if any, fits the `oneOf` variant) and demote to a question if the value looks suspicious for the field.
2. **No verbatim match → search for candidate schema fields.** Use the property index built in step 4. Heuristics:
    - Substring inclusion in either direction (`spread_sens` ⊂ `spread_sensitivity`, `clamp_delta` ⊂ `clamp_price_delta`).
    - Common abbreviation expansions to *try*, not to assume: `sens → sensitivity`, `mult → multiplier`, `vol → volatility` *or* `volume`, `min/max` kept as-is.
    - Drop or add suffix patterns like `_bps`, `_half_spread_bps` (e.g. `base_spread` near `base_half_spread_bps`).
    - Token overlap on `_`-split parts.
3. **Exactly one plausible candidate → NAME TRANSLATION (ask).** Surface as: "I think `<knob>` maps to `<schema_path>` — please confirm." Do not rewrite.
4. **Multiple plausible candidates → NAME TRANSLATION (ask, with options).** List every candidate path and ask the user which (or none).
5. **No plausible candidate → UNKNOWN.** Surface as: "this knob is not in the schema. Is it (a) a calibration-only parameter to drop, (b) a new schema field I have not yet seen, or (c) a typo for something else? please advise."
6. **Unit drift check (independent of the name bucket).** If the matched or candidate schema field is in the `Bps` set and the proposed value is fractional or `< 1`, raise a UNIT DRIFT question regardless of which name bucket the knob landed in. Surface the literal `× 10000` conversion as a candidate, not a decision.

### 6. Report — as a list of open questions

Show the user the schema version used, then list every non-DIRECT-MATCH item as an **open question** that needs their answer before the request is filed. Phrase each as a question, not a decision:

```
Schema: <tag>
URL:    https://phi-labs-ltd.github.io/bolt-pricefeeder/<tag>/bolt-pricefeeder-config-schema.json

UNKNOWN — please clarify each:
  - <calibration_knob> = <value>
      not found in the schema. closest schema fields: <list or "none">.
      is this (a) a calibration-only parameter to drop, (b) a new schema field, or (c) a typo? please advise.

UNIT DRIFT — please confirm intended value:
  - <calibration_knob> = <decimal_value>
      candidate schema field: <path> (typed `Bps`, integer basis points).
      literal value × 10000 = <integer_bps>. which value did you intend — the decimal as written, or <integer_bps>?

NAME TRANSLATION — please confirm each mapping:
  - <calibration_knob> = <value>
      candidate schema field: <path>. confirm this is the intended target (yes / different / dropped)?

DIRECT MATCH — passing without prompt:
  - <knob_name> = <value>
  ...
```

If anything is in UNKNOWN, UNIT DRIFT, or NAME TRANSLATION, **stop and wait for the user's answers**. Do not proceed to a corrected table on assumption. Resolving these questions here is the whole point of running the skill — the same questions will surface in dev review otherwise.

### 7. After the user resolves every question, emit the corrected table

Only once every flagged item has an explicit user resolution, offer to produce a corrected, ticket-ready markdown table the liquidity team can paste into the Linear ticket. The table reflects exactly what the user told you — not what a heuristic match or `× 10000` rule would have done on its own. If the user disagreed with the skill on any item, the table follows the user.

### 8. Hand off

The skill ends at the corrected table. The dev applies the change to the deployment YAML after the ticket is filed.

## What to expect when reading the schema

A few structural notes about the bolt-pricefeeder schema that explain the most common findings — these describe the **shape** of the schema rather than specific field names, so they remain valid as fields are added or renamed:

- The schema is JSON Schema draft 2020-12 with a `$defs` section that holds the reusable type definitions. Most asset-level properties are defined inside one of those `$defs` entries (e.g. an asset-config def, a fetch-pipeline def, a dynamic-spread def, a circuit-breaker def, a halt-condition def). When walking, descend into `$defs` and into every `oneOf` branch — fields can live several levels down.
- A reusable `Bps` definition (typically `{ "type": "integer", "format": "uint64", "minimum": 0 }`) is referenced by every basis-points field. Identifying which fields use this `$ref` is how you build the unit-drift check; do not rely on the `_bps` suffix alone.
- Pipeline and circuit-breaker variants are encoded as `oneOf` arrays with a discriminator field (commonly `strategy` for the fetch pipeline and `type` for halt conditions). "Additional properties not allowed" findings almost always mean the proposed knobs belong to a variant other than the one selected.
- `required` arrays on object schemas explain "missing field" findings. A field can be present in `properties` but still be required — surface that distinction when reporting.

If the schema's actual structure diverges from these notes (because of a future release), trust the fetched schema, not these notes.

## When this skill should NOT trigger

- The user is editing pricefeeder **source code** (the Rust crates of the project) — that is a code change, not a config review.
- The user has already had a knob list validated and is just asking the dev to apply it. They are past the validation step.
- A different oracle / feeder / pricefeeder repo is involved. This skill is `phi-labs-ltd/bolt-pricefeeder`-specific. If unsure which feeder is meant, confirm before triggering.
