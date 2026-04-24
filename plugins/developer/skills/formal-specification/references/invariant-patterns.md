# Invariant patterns

A catalogue of invariant shapes for Quint specs. Use as a starting menu in Phase 2 of the workflow — propose candidates, then discuss with the user which apply.

Every pattern below uses an `Ok / ErrVariant` result ADT. Match on the result and return `true` vacuously for the error branch.

## 1. Bounded output

The operation never produces more output than input.

```quint
val inv_output_bounded = match result {
  | Ok(r)        => r.length() <= prices.length()
  | FilterErr(_) => true
}
```

Applies whenever the operation is a filter, selection, or projection.

## 2. Subset preservation

Every element in the output existed in the input. The operation never fabricates.

```quint
val inv_subset = match result {
  | Ok(r) => r.indices().forall(i =>
      prices.indices().exists(j => r[i] == prices[j])
    )
  | FilterErr(_) => true
}
```

Applies to filters, deduplicators, sorters.

## 3. Non-emptiness implications

Non-empty output requires non-empty input; or, more strongly, non-empty input yields non-empty output (with a fallback).

```quint
// Weaker direction:
val inv_nonempty_preserved = match result {
  | Ok(r)        => r.length() > 0 implies prices.length() > 0
  | FilterErr(_) => true
}

// Stronger direction (when the op has an all-filtered fallback):
val inv_fallback_keeps_all = match result {
  | Ok(r)        => prices.length() > 0 implies r.length() > 0
  | FilterErr(_) => true
}
```

The stronger direction captures fallback behaviour (e.g., "if every input was an outlier, keep them all").

## 4. Property preservation across the operation

A property that holds on the input continues to hold on the output.

```quint
val inv_same_pair_preserved = match result {
  | Ok(r) =>
    r.length() <= 1 or
    r.indices().forall(i => r[i].pair == r[0].pair)
  | FilterErr(_) => true
}

val inv_no_duplicate_sources = match result {
  | Ok(r)        => all_unique_sources(r)
  | FilterErr(_) => true
}
```

Common preserved properties: same partition, uniqueness, sortedness, size parity, non-negative values.

## 5. Exemption preserved (fresh-recompute idiom)

Some elements are exempt from processing and must never be dropped.

```quint
val inv_hedging_venue_preserved = match outlier_filter(prices, threshold_bps, hedging_venue) {
  | Ok(r) =>
    prices.indices().forall(i =>
      not(is_hedging_venue(hedging_venue, prices[i].source)) or
      r.indices().exists(j => r[j] == prices[i])
    )
  | FilterErr(_) => true
}
```

**Critical detail:** this invariant matches on `outlier_filter(prices, ...)` — a **fresh call** — not on the module variable `result`. Reason: between `add_price` steps and the next `run_filter` step, `prices` changes but `result` does not. If the invariant read `result` it would be comparing yesterday's output against today's input and produce spurious counterexamples.

Use the fresh-recompute form whenever an invariant must hold after every state change, not just after the operation fires.

## 6. Idempotence

Running the operation twice gives the same result as once.

```quint
val inv_idempotent = match result {
  | Ok(r)        => outlier_filter(r, threshold_bps, hedging_venue) == Ok(r)
  | FilterErr(_) => true
}
```

Applies to filters, normalisers, canonicalisers. Watch for fallback behaviour — an operation with an "if all filtered, keep all" fallback may not be idempotent in the strict sense.

## 7. Monotonicity

Adding more input does not remove anything already in the output.

```quint
// Sketch — usually checked via a relational spec rather than a state invariant.
// For a state-machine spec, encode as: after a new add, result_after ⊇ projection(result_before).
```

Useful for accumulators, caches.

## 8. Error-branch consistency

The error variant is only returned on inputs that violate a precondition, and never on valid inputs.

```quint
val inv_err_only_on_invalid_input = match result {
  | Ok(_)        => true
  | FilterErr(_) => not(all_same_pair(prices)) or not(all_unique_sources(prices))
}
```

Pairs with positive tests that valid-input actions produce `Ok(...)`.

## 9. Invariants involving configuration

If the spec has a mode switch (e.g., `NoHedging | Hedging(v)`), write paired invariants that cover both modes:

- A vacuous-in-one-mode invariant: "when `NoHedging`, exemption invariant is vacuously true."
- A strict-in-the-other invariant: "when `Hedging(v)`, only source `v` is exempt."

## Picking a starter set

For a typical filter/selector component, propose these four first and iterate:

1. Bounded output (Pattern 1)
2. Subset (Pattern 2)
3. Non-emptiness in whichever direction the operation guarantees (Pattern 3)
4. One property-preservation invariant that names a domain concern (Pattern 4)

Add Pattern 5 (exemption preserved) when the component has a mode that bypasses processing. Add Pattern 8 when the component can return errors.

## Elicitation prompts for Phase 2

Questions that tend to surface invariants the user knows but has not written down:

- "What is the worst-case output size in terms of input size?"
- "Could the output contain elements that were not in the input?"
- "Is there any input shape for which the output should be guaranteed non-empty?"
- "Is there any input property (ordering, uniqueness, partitioning) that must survive the operation?"
- "Are any inputs exempt from normal processing?"
- "What is the behaviour on empty input? On a single-element input?"
- "What makes the operation return an error rather than a value?"
- "If you ran the operation twice, would the second run change anything?"
- "Has a bug ever shipped in this component? What would an invariant that catches it look like?"
