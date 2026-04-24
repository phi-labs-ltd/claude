# Spec structure and house style

Detailed layout for Quint spec files. Pairs with the workflow in `SKILL.md`.

## File layout

```
spec/
├── types.qnt                   — shared domain types, ADT result, math utilities
├── <module>.qnt                — one module per component under test
├── <module>.tests.qnt          — deterministic `run` tests for that module
├── <other_module>.qnt
└── <other_module>.tests.qnt
```

One `types.qnt` for the whole project. One `.qnt` + one `.tests.qnt` per component.

## `types.qnt` contents

Organise in this order:

1. **Domain types** — records that mirror implementation types. Add `// Mirrors <source-path>` comments.
2. **Result ADT** — a single sum type with an `Ok(...)` variant and one or more error variants. Provide `is_ok`, `is_err`, `unwrap` helpers.
3. **Math utilities** — Quint-integer mirrors of any arithmetic the implementation relies on (median, abs, sort). Include a comment noting which implementation function they mirror.
4. **Pure functions** common across modules (e.g., extracting a scalar price from a union payload).
5. **Named constants** for well-known values (asset pairs, common configs).
6. **Configuration enums** (e.g., `HedgingVenue = NoHedging | Hedging(str)`).

Example header:

```quint
// Shared domain types for <project> Quint specs.
// Mirrors Rust types from <path/to/source>.
module types {
  // ...
}
```

## Module anatomy (`<module>.qnt`)

Sections in this fixed order so future readers scan predictably:

```quint
// 1. Header
// Quint specification for <ComponentName>.
// Mirrors: <path/to/source>
//
// Precondition: <any assumption the spec does not enforce>.
// <Other one-paragraph context>.

module <module> {
  // 2. Imports
  import types.* from "./types"

  // 3. Pure constants
  pure val SOME_SCALE = 10000

  // 4. Pure logic
  // --------------------------------------------------------------------------
  // Core filter logic
  // --------------------------------------------------------------------------
  pure def helper(...) : ... = ...
  pure def main_op(...) : Result = ...

  // 5. State machine
  // --------------------------------------------------------------------------
  // State machine
  // --------------------------------------------------------------------------

  // 5a. ActionTaken ADT (only if quint-connect integration is intended)
  type ActionTaken =
    | Init({ ... })
    | Add(...)
    | Run

  // 5b. State variables
  var state_field: T
  var result: Result
  var action_taken: ActionTaken

  // 5c. Value sets — finite domains for model checking
  val PAIRS = Set(...)
  val VALUES = Set(95, 100, 105)

  // 5d. Parameterised init (for deterministic tests)
  action init_with(...) = all { ... }

  // 5e. Nondet init (for quint run simulation)
  action init = {
    nondet x = VALUES.oneOf()
    init_with(x)
  }

  // 5f. Base actions — explicit arguments, set action_taken' to a concrete variant
  action add_specific(...) = all { ... }
  action run_op = all { ... }

  // 5g. Nondet wrappers — delegate to base actions
  action add = {
    nondet x = VALUES.oneOf()
    add_specific(x)
  }

  // 5h. Step
  action step = any { add, run_op }

  // 6. Invariants
  // --------------------------------------------------------------------------
  // Invariants
  // --------------------------------------------------------------------------
  val inv_foo = match result {
    | Ok(r)        => ...
    | ErrVariant(_) => true
  }
}
```

### Why base actions + nondet wrappers?

- **Base actions** take explicit arguments. They set `action_taken'` to a concrete variant. Used by `quint-connect` (explicit args → reproducible replay) and by `.tests.qnt` files (deterministic driving).
- **Nondet wrappers** draw arguments from value sets using `nondet x = VALUES.oneOf()`, then delegate to the base action. Used by `init` and `step` during `quint run` / `quint verify`.

This split keeps model-checking exploration and deterministic replay working from the same action definitions.

### Why `init_with` + `init`?

- `init_with(params...)` lets tests pin exact configuration: `price_outlier::init_with(30, Hedging("gateio"))`.
- `init` draws configuration nondeterministically from value sets — used during simulation and model checking.

Configuration set at init time should not change during a run. In the price_outlier example, threshold and hedging venue are fixed at `init` and every subsequent action preserves them (`threshold_bps' = threshold_bps`).

### Value sets

Value sets should be:

- **Small** — model checking is exponential in state size. 3–5 values per axis is usually enough.
- **Discriminating** — pick values that make each behaviour branch reachable. For a threshold check, include values at, below, and above the threshold. For optional modes, include `None` and at least one concrete configuration.
- **Arithmetically compatible with the implementation.** If the implementation uses decimal arithmetic and the spec uses integer arithmetic, choose values that give the same answer in both. The price_outlier spec forces `bid == ask` in the `add_price` wrapper for exactly this reason.

## Tests file anatomy (`<module>.tests.qnt`)

```quint
// Tests for <ComponentName>.
// Run with `quint test spec/<module>.tests.qnt`.
//
// <One-paragraph summary of what the tests assert and how.>

module <module>_tests {
  import types.* from "./types"
  import <module> from "./<module>"

  /// <One-line behavioural description>
  run test_<behaviour> =
    <module>::init_with(...)
      .then(<module>::add_specific(...))
      .then(<module>::run_op)

  // ... more tests ...
}
```

Each test's name describes the behaviour, not the shape of the input. Read the test name aloud — it should be a claim.

## Naming conventions

| Thing | Convention | Example |
|---|---|---|
| Module name | `snake_case` matching the source file | `price_outlier` |
| Pure function | `snake_case` verb or predicate | `outlier_filter`, `is_within_threshold` |
| Action | `snake_case` verb | `add_specific_price`, `run_filter` |
| Base vs nondet action | Base: explicit noun; Nondet: shorter | `add_specific_price` / `add_price` |
| Invariant | `inv_<property>` | `inv_output_bounded` |
| Test | `test_<behaviour>` | `test_outlier_excluded` |
| ADT variant | `PascalCase` | `NoHedging`, `Hedging(str)` |
| Constant | `SCREAMING_SNAKE_CASE` | `BPS_SCALE`, `PAIRS` |
