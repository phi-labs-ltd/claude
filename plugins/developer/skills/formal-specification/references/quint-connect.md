# Quint-connect integration

`quint-connect` replays a Quint trace against a real implementation and checks that implementation state matches spec state after every step. Include the hooks in a spec **only when mirroring an existing implementation and trace replay is intended**. For specs built from PDFs, user stories, or greenfield pseudocode, skip this file.

## What trace replay requires from the spec

1. An `ActionTaken` ADT with one variant per base action, carrying the action's arguments.
2. A `var action_taken: ActionTaken` that every action assigns.
3. Base actions set `action_taken'` to a concrete variant.
4. Nondet wrappers delegate to base actions so the variant surfaces in traces (not the wrapper's name, which has no arguments).
5. Integer arithmetic in the spec agrees with the implementation's arithmetic on the chosen value sets.

## `ActionTaken` pattern

```quint
type ActionTaken =
  | Init({ t: int, hv: HedgingVenue })
  | AddSpecificPrice({
      pair: AssetPair, src: str,
      bid: int, ask: int, bid_sz: int, ask_sz: int,
    })
  | RunFilter
```

One variant per base action. The payload carries every argument needed to replay that step against the implementation driver.

## Base action + nondet wrapper pattern

Base action — explicit args, sets `action_taken'`:

```quint
action add_specific_price(pair, src, bid, ask, bid_sz, ask_sz) = all {
  prices.length() < 5,
  prices' = prices.append({
    pair: pair, source: src,
    data: Micro({ bid: bid, ask: ask, bid_size: bid_sz, ask_size: ask_sz }),
  }),
  threshold_bps' = threshold_bps,
  hedging_venue' = hedging_venue,
  result'        = result,
  action_taken'  = AddSpecificPrice({
    pair: pair, src: src,
    bid: bid, ask: ask, bid_sz: bid_sz, ask_sz: ask_sz,
  }),
}
```

Nondet wrapper — delegates so the concrete variant still surfaces:

```quint
action add_price = {
  nondet pair   = PAIRS.oneOf()
  nondet src    = SOURCES.oneOf()
  nondet mid    = PRICE_VALUES.oneOf()
  nondet bid_sz = SIZE_VALUES.oneOf()
  nondet ask_sz = SIZE_VALUES.oneOf()
  add_specific_price(pair, src, mid, mid, bid_sz, ask_sz)
}
```

Never duplicate assignment logic in the wrapper — always delegate. The wrapper's job is value-set sampling, not state transition.

## Integer-vs-decimal arithmetic discipline

Quint integers are unbounded but exact; most implementations use floating point or decimal libraries. The spec's arithmetic must agree with the implementation's on every trace the replay exercises — otherwise `quint-connect` will report spurious mismatches.

Typical fix: restrict value sets so the two arithmetics give the same answer. In the price_outlier example, microprice is `(ask * bid_size + bid * ask_size) / (bid_size + ask_size)`. With `bid != ask` and uneven sizes this can round differently in Quint (integer division) than in `rust_decimal` (banker's rounding). The fix: `add_price` draws `bid == ask` from a single `mid`, which makes microprice equal to `mid` in both systems. Spread-sensitive behaviour is covered by deterministic tests in `.tests.qnt` instead, where exact inputs are chosen to match.

Document the constraint in a comment on the nondet wrapper so future readers understand why the axis was collapsed.

## Implementation-side driver

The other half of `quint-connect` is an implementation driver that:

1. Exposes one function per base action, matching the arguments in the spec.
2. Maintains the implementation's state.
3. Exposes a snapshot function (commonly `SpecState::from_driver`) that returns the implementation's state in the same shape as the spec's `var`s (for price_outlier: prices, threshold_bps, hedging_venue, result).

For each replayed step, `quint-connect` reads `action_taken` from the trace, dispatches to the matching driver function with the variant's arguments, then compares the driver snapshot to the spec state. A divergence is a bug in either the spec or the implementation.

The implementation of the driver lives in the implementation's test code, not in the spec — it is out of scope for this skill. Touch-point for future specs: the spec's `ActionTaken` variants and the `var`s must match the driver's API one-to-one.

## Checklist when adding `quint-connect` to a spec

- [ ] `ActionTaken` ADT with one variant per base action.
- [ ] `var action_taken: ActionTaken` declared.
- [ ] Every base action sets `action_taken'` to the matching variant.
- [ ] Every nondet wrapper delegates to a base action (no duplicate logic).
- [ ] Every `var` is assigned in every branch of every action.
- [ ] Value sets chosen so spec arithmetic agrees with implementation arithmetic.
- [ ] Header comment names the implementation file the spec mirrors.
