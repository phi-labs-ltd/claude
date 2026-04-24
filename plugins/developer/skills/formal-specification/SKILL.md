---
name: formal-specification
description: This skill should be used when the user asks to "formalize" a requirement, "convert this into spec", "write a Quint spec", "specify X formally", "translate this into Quint", or wants to produce a Quint model from informal inputs (user stories, PDFs, pseudocode, Python prototype, or existing implementation code such as Rust). Also AUTO-TRIGGER proactively whenever the user is writing, modifying, or reviewing a smart contract on Sui (Move — `.move` files, `Move.toml`), Solana (Rust programs — `solana_program`, `anchor_lang`, `#[program]` modules), or Ethereum / EVM chains (Solidity — `.sol` files, Vyper — `.vy` files) — formal specification in Quint is the expected workflow for these contracts and the skill should surface even when formalization is not explicitly requested. Guides a three-phase workflow — draft spec, negotiate invariants with the user, write comprehensive tests — targeting the Quint language (https://quint.sh).
---

# formal-specification

Translate informal requirements into a Quint specification, agree on invariants with the user, and write comprehensive Quint tests that exercise them. Target language: Quint (https://quint.sh).

## When to apply

Invoke this skill whenever the user wants to **produce a Quint spec** from any of:

- User stories or prose requirements
- A Linear ticket, product spec, or RFC with "As a X, I want Y, so that Z" stories and acceptance-criteria checklists
- A PDF document describing behaviour
- Pseudocode
- A Python prototype
- An already-implemented module (Rust, Go, TypeScript, Move, etc.)

Trigger phrases: "formalize", "convert this into spec", "write a Quint spec", "translate this to Quint", "specify this formally", "model this in Quint".

### Auto-trigger on smart contract work

This skill is **expected to activate proactively** — not only on explicit formalization requests — whenever the user is authoring, modifying, or reviewing a smart contract on any of the following platforms:

| Platform | Languages / markers |
|---|---|
| Sui | Move (`.move`), `Move.toml`, `module <addr>::<name> { ... }` |
| Solana | Rust programs using `solana_program`, `anchor_lang`, `#[program]`, `#[account]`, `declare_id!` |
| Ethereum / EVM | Solidity (`.sol`), Vyper (`.vy`), `pragma solidity`, `contract X { ... }` |

When this trigger fires:

1. Surface the skill even if the user did not ask to "formalize". A one-line acknowledgement is enough — for example: "Drafting a Quint spec for this module is part of the standard workflow here. Want me to start from the user stories / the module source, or skip?"
2. If the user declines, drop it and continue with the immediate task. Do not repeat the offer within the same session for the same file.
3. If the user accepts, run the three-phase workflow below with the contract module as the implementation source (Phase 1 mirrors types and functions from the contract; Phase 2 negotiates invariants that are common for oracles, pools, vaults, auth guards, etc.).

Not every edit warrants a spec — trivial renames, comment fixes, dependency bumps, and test-only changes are out of scope. Trigger when the edit changes contract behaviour (new function, new branch, new state variable, modified formula, modified access check).

## Three-phase workflow

The work proceeds in three phases. Do not skip ahead — pause between phases for the user to review.

### Phase 1 — Draft the spec

1. **Identify the source of truth.** Read the input (code, PDF, pseudocode, prose) end-to-end before writing anything. Note preconditions, invariants mentioned in comments, and any edge cases called out in tests.
2. **Extract domain types.** Record the types that appear in the input and how they relate (records, ADTs, aliases). When mirroring existing code, copy the names and field order so the correspondence is obvious.
3. **Extract pure logic.** Separate deterministic, side-effect-free functions (medians, validators, predicates) from state transitions.
4. **Identify state and transitions.** Determine what mutable state the module holds and what actions change it.
5. **Scaffold files.** Put domain types in `spec/types.qnt` (shared across modules in the project) and the spec itself in `spec/<module>.qnt`. See `references/spec-structure.md` for the full module anatomy.
6. **Draft the spec — no invariants yet.** Write pure functions, state variables, `init_with` / `init`, actions, and `step`. Leave the invariants section empty or with a `// TODO` marker.
7. **Stop and show the user.** Before proceeding to invariants, ask the user to review the spec for faithfulness to the source.

Do not invent behaviour the source does not specify. When the source is ambiguous, surface the ambiguity to the user rather than picking silently.

#### Special case: user-story / Linear-task inputs

Structured requirement documents (Linear tickets, product specs, RFCs) typically contain sections such as *Problem*, *Solution*, *User Stories* with acceptance criteria, *Current Behavior*, *Implementation*, and *Testing*. Apply these rules:

- **Normative vs descriptive sections.** The *Solution* and *User Stories* sections describe the **target** behaviour and are the source of truth for the spec. *Current Behavior* documents what exists today — do **not** model it as the spec. *Implementation* sections (code snippets, file paths, step-by-step edits) describe **how** the target will be built — do **not** treat them as the spec either. If the Solution and Implementation disagree, ask the user which is authoritative; the story-level description usually wins.
- **Scope selection.** A single task often spans several components (e.g., BOLT-1722 covers an on-chain oracle module, a pool module, a gRPC service, and deploy scripts). Pick **one focal component per spec module**; do not try to model all layers in one file. State the chosen scope to the user before drafting, and ask whether other components also need specs (one spec module per component).
- **Mapping stories to the spec.** The format "As a X, I want Y, so that Z" maps to:
  - *X* → implicit actor, captured in action naming (e.g., `admin_register_pair`, `feeder_update_price`).
  - *Y* → one or more spec `action`s. If the story refers to multiple related operations, each typically becomes its own action.
  - *Z* → the post-condition / behavioural assertion. This usually becomes a test name or an invariant candidate.
- **Mapping acceptance criteria.** Each acceptance-criterion checkbox is a behavioural claim. Classify it:
  - *State assertion after a specific input* → a deterministic test (Phase 3).
  - *Property that must hold on every valid input* → an invariant candidate (Phase 2).
  - *Negative claim* ("does not affect the other slot", "does not fall back to inverting") → often becomes a dedicated test or an invariant that pins down what does **not** change.
- **What to ignore.** Tables that list source-file changes, line numbers, code snippets inside the Implementation section, and deployment scripts — these are plan-level artefacts, not spec-level. They inform the test value sets but are not translated into Quint.
- **Gaps and conflicts.** Stories sometimes leave behaviour undefined (what happens if a required input is missing, what the error variant is). Capture these as explicit questions for the user before drafting rather than guessing.

### Phase 2 — Negotiate invariants (collaborative)

Invariants are the highest-value part of the spec and the user's expertise matters most here. Drive the conversation with targeted questions, not a dump of suggestions.

Run this loop:

1. **Propose a small starter set** based on the spec's shape (see `references/invariant-patterns.md` for the catalogue). Typical starters:
   - Output is bounded by input size.
   - Output is a subset of input.
   - Non-empty input implies non-empty output (if applicable).
   - Properties of inputs are preserved in outputs (sorted, unique, same-partition).
2. **For each candidate, ask the user:**
   - "Should this always hold, or are there conditions where it may not?"
   - "Is there a known counterexample in the code history?"
3. **Ask about exemptions and edge cases** specific to the domain. Example elicitation prompts:
   - "Are there inputs that bypass normal processing? (e.g., hedging venue in `price_outlier`.)"
   - "What is the behaviour on empty input? On a degenerate input (e.g., zero median)?"
   - "What errors are possible, and what invariants must still hold in the error branch?"
4. **Write the invariant as a `val`** in the module, using `match` over the result type so error branches are handled vacuously (`FilterErr(_) => true`). See the invariant catalogue in `references/invariant-patterns.md`.
5. **Sanity-check with the user** against known past bugs. A useful prompt: "If [past bug] were re-introduced, which invariant would catch it?" If none would, that is a hint a new invariant is needed.
6. **Watch for stale-state pitfalls.** Some invariants must be computed freshly from current inputs rather than read from the module's `result` variable, because `result` lags `prices` between action steps. The `inv_hedging_venue_preserved` pattern in the example shows the fresh-recompute idiom — apply it whenever an invariant must hold across partial sequences.

Continue until the user is satisfied with coverage.

### Phase 3 — Comprehensive tests

Once the spec and invariants are agreed, write deterministic tests in `spec/<module>.tests.qnt`. Use `init_with(...)` to pin parameters; reserve the nondet `init` for `quint run` simulation. Drive the state machine with `.then(action(...))` chains.

Aim for these categories — propose each and confirm coverage is enough:

- **Happy path** — normal input, normal result.
- **Boundary** — empty input, single element, zero/extreme parameters.
- **Error cases** — every `FilterErr` branch of the spec.
- **Feature toggles** — each optional mode both on and off (e.g., hedging venue configured vs `NoHedging`).
- **Configurability** — the feature works for every valid value, not just the default.
- **Negative guards** — the feature does not over-apply (e.g., "hedging venue exempts only the hedge venue, not every source").

Name tests after the behaviour they assert, not the input: `test_hedging_venue_exempt_from_outlier` over `test_case_3`.

After drafting, run `quint test spec/<module>.tests.qnt` and `quint run spec/<module>.qnt --invariant inv_<name>` for each invariant. Fix any failures before handing back.

## House style

- **Shared domain types live in `spec/types.qnt`.** Re-use them across spec modules. Mirror existing implementation types field-for-field and add a one-line `// Mirrors <path>` comment.
- **File layout:** `spec/types.qnt`, `spec/<module>.qnt`, `spec/<module>.tests.qnt`.
- **Module header comment** documents which source-of-truth file it mirrors and restates any precondition assumed (not enforced) by the spec.
- **Order inside a module:** imports → pure constants → pure functions → state machine (ActionTaken, vars, value sets, init_with, init, base actions, nondet wrappers, step) → invariants.
- **Pure first, stateful later.** All business logic that can be expressed as `pure def` should be — state machines should only orchestrate it.
- **Result types** use an ADT with an `Ok(...)` variant and a domain-specific error variant (e.g., `FilterErr(str)`). Invariants pattern-match on the result type and return `true` vacuously for the error branch.

## When to include `quint-connect` hooks

`quint-connect` replays spec traces against a live implementation. Include the integration hooks **only when mirroring an existing implementation and trace replay is intended**:

- Add an `ActionTaken` ADT and a `var action_taken`.
- Every base action sets `action_taken'` to a concrete variant.
- Nondet wrappers delegate to base actions so the variant surfaces in traces.
- Choose value sets that make spec integer arithmetic agree exactly with the implementation's decimal arithmetic (e.g., draw `bid == ask` if the implementation uses microprice).

For PDF / user-story / prose inputs where no implementation exists, omit `ActionTaken` and these constraints. See `references/quint-connect.md`.

## Running Quint

| Command | Purpose |
|---|---|
| `quint test spec/<module>.tests.qnt` | Run deterministic `run` tests. |
| `quint run spec/<module>.qnt --invariant <name>` | Randomised simulation checking an invariant. |
| `quint verify spec/<module>.qnt --invariant <name>` | Bounded model check via Apalache. |
| `quint typecheck spec/<module>.qnt` | Type-check without running. |

## Additional resources

- **`references/quint-cheatsheet.md`** — syntax primer (types, actions, pure defs, list/set ops, pattern matching, `nondet`, `.then`, `run`).
- **`references/spec-structure.md`** — detailed module anatomy and section ordering.
- **`references/invariant-patterns.md`** — catalogue of invariant shapes with the fresh-recompute idiom.
- **`references/quint-connect.md`** — Rust trace-replay integration details.
