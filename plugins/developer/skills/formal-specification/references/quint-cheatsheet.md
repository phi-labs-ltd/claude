# Quint syntax cheatsheet

A concise primer for writing Quint specs. Full documentation at https://quint.sh.

## Modules and imports

```quint
module my_module {
  import types.* from "./types"   // import everything from a sibling file
  import other as o from "./other" // import qualified

  // ... body ...
}
```

A file may contain one or more modules. Imports resolve relative to the file.

## Types

### Record types

```quint
type AssetPair = {
  base:  str,
  quote: str,
}
```

Construct: `{ base: "BTC", quote: "USDC" }`. Access: `p.base`.

### Algebraic data types (sum types / tagged unions)

```quint
type FilterResult =
  | Ok(List[FetchedPrice])
  | FilterErr(str)
```

Construct: `Ok([])`, `FilterErr("mixed pairs")`. Destruct with `match`.

### Type aliases and built-ins

Primitives: `int`, `bool`, `str`. Collections: `List[T]`, `Set[T]`, `(K -> V)` (maps).

## Values and functions

- `pure val NAME: T = expr` — compile-time-pure named constant.
- `pure def f(x: T): U = expr` — pure function (no state, no nondet).
- `val NAME: T = expr` — may read state (useful for invariants).
- `def f(x: T): U = expr` — may read state.

Prefer `pure` wherever possible; it keeps reasoning local.

## Pattern matching

```quint
match result {
  | Ok(xs)       => xs.length()
  | FilterErr(_) => 0
}
```

Every variant must be handled. Use `_` to ignore a payload.

## Lists

```quint
val xs: List[int] = [1, 2, 3]

xs.length()              // 3
xs[0]                    // 1 — zero-indexed
xs.indices()             // Set of valid indices: Set(0, 1, 2)
xs.append(4)             // [1, 2, 3, 4]
xs.concat([4, 5])        // [1, 2, 3, 4, 5]
xs.select(x => x > 1)    // filter: [2, 3]
xs.foldl(0, (acc, x) => acc + x)  // 6
xs.indices().forall(i => xs[i] > 0)
xs.indices().exists(i => xs[i] == 2)
```

Note: Quint lists do not have a direct `filter`. Use `select`.

## Sets

```quint
val s = Set(1, 2, 3)

s.size()                 // 3
s.contains(2)            // true
s.map(x => x * 2)        // Set(2, 4, 6)
s.filter(x => x > 1)     // Set(2, 3)
s.forall(x => x > 0)
s.exists(x => x == 2)
s.oneOf()                // nondet element (only inside nondet)
```

## State and actions

```quint
var prices: List[int]
var result: int

action init = all {
  prices' = [],
  result' = 0,
}

action add_price(p: int) = all {
  prices' = prices.append(p),
  result' = result,
}

action run = all {
  prices.length() > 0,          // precondition (guard)
  prices' = prices,
  result' = prices.foldl(0, (a, x) => a + x),
}

action step = any { init, add_price(1), run }
```

- `x'` denotes the next-state value of `x`. Every `var` must be assigned in every branch of `all { ... }`.
- `all { c1, c2, ... }` — conjunction. All conditions must hold and all assignments fire together.
- `any { a1, a2, ... }` — disjunction. Nondeterministically pick one enabled action.
- A non-assignment expression inside `all { ... }` is a guard — the action is only enabled when the guard is true.

## Nondeterminism

```quint
action init = {
  nondet t = THRESHOLD_VALUES.oneOf()
  nondet hv = HEDGING_VENUE_VALUES.oneOf()
  init_with(t, hv)
}
```

`nondet x = s.oneOf()` draws `x` from set `s`. Used to pick values for simulation / model checking.

## Tests: `run`

```quint
run test_name =
  init_with(30, Hedging("gateio"))
    .then(add_specific_price(...))
    .then(add_specific_price(...))
    .then(run_filter)
```

`A.then(B)` sequences actions. Each step must be enabled at the current state. Use `init_with` (deterministic) in tests, not `init` (nondet).

Run with `quint test <file>.tests.qnt`.

## Invariants

Define as `val`:

```quint
val inv_output_bounded = match result {
  | Ok(r)        => r.length() <= prices.length()
  | FilterErr(_) => true
}
```

Check with `quint run <spec>.qnt --invariant inv_output_bounded` or `quint verify ... --invariant inv_output_bounded`.

## Common gotchas

- **No floats.** Quint integers are unbounded. Mirror fractional semantics with rescaled integer arithmetic (e.g., `bps_scale = 10000`).
- **`'` (prime) is a postfix operator on variables.** `prices'` is the next-state value, not a new variable.
- **Every branch of an `all { ... }` must assign every `var`.** To leave a var unchanged, write `x' = x`.
- **`List.select` not `.filter`.** Sets use `.filter`.
- **Indices, not iteration.** Loops are expressed with `foldl`, `forall`, `exists` over `indices()` or a set.

## Further reading

- Language reference: https://quint.sh/docs/lang
- Tutorial: https://quint.sh/docs/getting-started
- Standard library: https://quint.sh/docs/builtin
