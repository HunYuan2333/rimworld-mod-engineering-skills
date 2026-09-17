# Harmony and compatibility engineering

## Patch selection

- Postfix: observe or adjust a result while preserving the original call.
- Prefix: validate, prepare state, or intentionally skip/replace behavior. Returning `false` expands responsibility to the skipped contract.
- Finalizer: contain or transform exceptions only when failure semantics are explicit.
- Transpiler: last-resort structural edit when no stable seam exists; anchor semantically and fail visibly if the expected pattern is absent.

Avoid copying an entire vanilla method to change one branch. A copied body silently freezes upstream behavior and conflicts with other patches.

## Patch contract

For every patch record:

- target type/method/signature and game version;
- purpose and why a shallower extension point is insufficient;
- whether the target is hot, reentrant, or called during loading;
- original behavior that must remain true;
- known patch peers and ordering requirements;
- safe behavior if the target or expected IL shape changes.

Use a unique Harmony ID. Set priority or before/after constraints only for a documented dependency. Do not rely on incidental patch order.

## Cross-mod dependencies

Use direct references for required dependencies and declare them in `About.xml`. For optional integrations choose one of:

- conditional XML/load folder;
- conditional compatibility assembly;
- adapter over a stable public API;
- narrowly cached reflection when no stable contract exists.

Resolve reflection once, validate signatures, cache delegates when useful, and degrade the feature—not the whole mod—when optional integration fails. Identify mods by package ID, not display name.

## Intrusion containment

Deep integration is sometimes the feature. Contain it with:

- one module per external mod or upstream subsystem;
- centralized target discovery and diagnostics;
- no optional external types in always-loaded signatures;
- version/capability checks;
- circuit breakers for repeatedly failing optional hooks;
- tests with dependency absent, present, and changed where feasible.

Patch count alone is not a quality measure. One patch to a global pawn or combat method may have a larger blast radius than many patches to isolated custom types.

## Compatibility compromise levels

Use the shallowest level that can preserve the required behavior. The levels express cost and required controls, not moral quality.

| Level | Technique | Typical controls |
|---|---|---|
| 0 | Public Def/API/extension point | Contract and ordinary tests |
| 1 | Conditional XML, adapter, optional assembly | Presence/absence matrix and declared ownership |
| 2 | Narrow postfix/prefix or cached public reflection | Patch inventory, ordering diagnostics, graceful degradation |
| 3 | Private access, global method interception, Def-wide rewrite | Version anchor, startup validation, provenance, stack tests |
| 4 | Transpiler, method replacement, global rules overhaul | Semantic anchors, fail-visible matching, upstream diff review, explicit incompatibilities |

Moving deeper is justified when replacement scope or an upstream limitation requires it. Record why the preceding level cannot satisfy the invariant. Keep the exception local and define when it should be reassessed.

## Extension discovery and installation

For a registry or compatibility-module loader define more than `CanInstall` and `Install` once the participant count becomes material:

- stable ID, version/capability range, and owner;
- required and optional dependencies;
- deterministic order and cycle behavior;
- duplicate policy and idempotence;
- separate discovery, validation, registration, activation, and shutdown phases where useful;
- per-participant exception isolation;
- downstream behavior when a dependency fails;
- visible state and reason for skipped/degraded/failed modules;
- deregistration or an explicit process-lifetime guarantee.

Do not depend on loaded-assembly order or reflection type order. Catching `ReflectionTypeLoadException` during scanning does not isolate constructor, capability-check, or installation failures. One optional compatibility module should not prevent unrelated modules from installing.

Public callback collections are extension APIs even if they are only fields. Prefer controlled registration that can enforce identity, duplicate policy, ordering, exception isolation, and cleanup. If a raw list is retained for compatibility, wrap new use behind those controls.

## Diagnostic sequence

When debugging a conflict:

1. reproduce with the smallest relevant list;
2. capture the full startup log and patch list;
3. confirm the target method and actual patch order;
4. isolate whether the failure is Def-time, type-load, initialization, runtime, save/load, or UI;
5. compare behavior with each integration removed;
6. fix the contract violation or add a contained adapter—do not blindly increase priority.
