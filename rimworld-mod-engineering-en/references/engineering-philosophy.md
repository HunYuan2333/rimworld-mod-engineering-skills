# Engineering philosophy and decision rules

Use this reference for architecture, refactoring, design review, or any change whose main risk is coupling rather than a single mechanic. It is a decision framework, not a demand to turn every mod into a framework.

## Governing ideas

1. Preserve behavior and data before improving shape.
2. Make important state, dependencies, phases, and failure modes explicit.
3. Put policy with the feature that owns it; keep shared infrastructure neutral.
4. Prefer reversible, observable changes. When a change is intentionally irreversible, say so.
5. Scale ceremony to blast radius. Small content changes need stable identifiers and validation; replacement-level systems need contracts, migrations, diagnostics, and compatibility ownership.
6. Judge a technique by controlled consequences, not by label. Reflection, statics, Harmony, managers, and Def mutation can be appropriate when their scope and lifecycle are explicit.

## Begin with invariants

Write the facts that must remain true regardless of implementation. Useful forms include:

- a request is committed at most once;
- an object has exactly one lifecycle owner at a time;
- a failed batch changes nothing, or its documented successful subset is durable;
- derived state can always be rebuilt from its declared authority;
- disabling an optional integration does not disable unrelated features;
- a save either migrates to the new schema or fails visibly without partial reinterpretation.

Exercise each invariant across success, rejection, cancellation, exception, timeout, save/load, reload, missing dependency, and duplicate callback paths. Tests should target these properties rather than merely reproducing method bodies.

## Authority and derived state

For each mutable fact, name one authority. Other representations are views, caches, indexes, projections, or transport snapshots.

| State kind | Requirement |
|---|---|
| Authoritative | One owner controls mutation and defines transaction semantics. |
| Persisted | Schema, defaults, migration, identity, and missing-content behavior are explicit. |
| Derived | Source, rebuild phase, invalidation trigger, and stale-read policy are explicit. |
| External | Trust boundary, validation, timeout, retry, and reconciliation are explicit. |
| UI | Reflects domain state; it does not manufacture confirmations or bypass state transitions. |

Duplicated mutable state is acceptable only when synchronization is part of the design. “Both are usually updated together” is not a consistency model.

## Cohesion

A component is cohesive when its members change for the same domain reason and share the same owner and lifetime. Prefer feature cohesion over folders that merely group technical nouns.

Split a component when it:

- mixes game rules with UI, persistence, transport, or compatibility policy;
- owns state with different lifetimes, such as process cache and save data;
- changes for unrelated features;
- cannot be tested or replaced without constructing unrelated systems;
- has become a coordinator that also performs every participant's work.

Do not split merely to shorten files. A large, cohesive state machine can be safer than many services with hidden call order.

## Coupling

Review coupling by form rather than counting references:

| Coupling | Typical symptom | Preferred control |
|---|---|---|
| Compile-time | Required external types leak through core signatures | Contract assembly or adapter; keep required dependencies explicit |
| Temporal | Calls must happen in an undocumented order | Lifecycle phase, state machine, or dependency graph |
| State | Several modules mutate the same collection or flag | Single authority and commands/events |
| Behavioral | A patch depends on undocumented upstream side effects | Record preserved contract and version anchor |
| Data-shape | Saves/configs depend on type or field names | Stable IDs, schema version, migration |
| Global | Static registries, Def databases, shared random streams | Scoped ownership, deterministic ordering, cleanup |
| Failure | One optional component can abort unrelated initialization | Per-component isolation and dependency-aware blocking |

The goal is not zero coupling. Required behavior needs coupling; architecture should place it at deliberate, inspectable seams.

## Dependency direction and layers

A useful default is:

```text
feature policy -> contracts <- adapters/integrations
                         ^
              composition and lifecycle
```

- Domain policy should not know Harmony targets, XML patch mechanics, transport packets, windows, or a specific optional mod unless that is the domain.
- Adapters translate external semantics; they must not silently weaken domain invariants.
- A composition root discovers concrete parts, orders them, and supplies dependencies. It should coordinate work, not absorb feature logic.
- Shared/Common code must be neutral and stable enough to justify its dependency gravity. Do not move code there merely because two callers exist.
- Peer features may share a small contract directly. Do not force the host to become a business mediator.

## Patterns by pressure

Choose patterns because a pressure exists:

| Pressure | Candidate pattern | Warning |
|---|---|---|
| Optional external API | Adapter / anti-corruption layer | Do not hide a truly required dependency behind reflection |
| Multiple contributors | Registry / pipeline | Define identity, order, duplicate policy, isolation, and ownership |
| Multi-stage behavior | Explicit state machine | Avoid boolean combinations that permit impossible states |
| Expensive derived view | Cache + invalidation | State its scope and rebuild authority |
| Atomic resource change | Transaction / unit of work | Define rollback and unknown-result handling |
| Variant construction | Factory | Keep it close to the varying construction policy |
| Cross-feature notification | Event stream / observer | Subscription lifetime and exception isolation are part of the contract |
| Algorithm family | Strategy | Do not create interfaces for a single stable implementation |

Patterns are vocabulary, not compliance badges. Reject abstractions that add indirection without isolating volatility, clarifying ownership, or enabling meaningful tests.

## Initialization topology

Represent initialization as phases with declared inputs and outputs. A typical large mod may need:

```text
discover -> validate -> register -> transform Defs -> build caches -> activate -> run
                                                              -> diagnose
run -> deactivate -> unregister -> release
```

The exact phases may differ. Require:

- stable participant identity;
- dependency and ordering rules;
- duplicate-registration policy;
- idempotence or an explicit one-shot guarantee;
- per-participant failure isolation;
- downstream blocking when a required dependency fails;
- reverse-order cleanup where dependencies require it;
- visible phase and failure status.

Do not rely on filesystem order, `AppDomain.GetAssemblies()` order, reflection type order, Harmony coincidence, or unspecified long-event ordering when correctness depends on sequence.

## Engineering exceptions

When compatibility or replacement scope requires violating a preferred boundary, record a short decision note containing:

- the invariant or capability being protected;
- the boundary crossed and why a shallower seam is insufficient;
- affected versions and integrations;
- containment and failure behavior;
- tests and diagnostics;
- owner and reassessment/removal trigger.

Exceptions should be local and searchable. Do not generalize a one-off workaround into a shared abstraction until a repeated pressure is demonstrated.

## Definition of done

A change is done when, proportionate to risk:

- behavior, non-goals, support matrix, and invariants are stated;
- dependency direction and state ownership are inspectable;
- initialization, configuration, and cleanup behavior are defined;
- save/config/API compatibility is handled or explicitly unsupported;
- failures are isolated and diagnosable;
- hot paths and global effects have evidence-based checks;
- the packaged artifact, not only the source tree, has been exercised;
- deliberate compromises are documented with their boundary.

