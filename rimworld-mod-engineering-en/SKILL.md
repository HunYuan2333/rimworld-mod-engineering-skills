---
name: rimworld-mod-engineering-en
description: Engineer, review, debug, or plan RimWorld mods across XML Defs, C#, Harmony, content systems, saves, compatibility, performance, packaging, and releases. Use for small XML mods, UI/QoL mods, race or faction mods, system mods, combat overhauls, framework mods, or framework-based extensions. Also use when assessing patch intrusiveness, cross-mod coupling, update risk, or maintainability.
---

# RimWorld Mod Engineering

Treat a RimWorld mod as software running inside a shared, stateful, heavily modded process. Scale the process to the change: a two-Def content tweak does not need an enterprise architecture, while a combat overhaul or framework needs explicit boundaries, compatibility ownership, migrations, and staged validation.

## Start with evidence

Before proposing or changing code:

1. Read repository instructions and the project’s design/developer documents.
2. Inspect `About/About.xml`, `LoadFolders.xml`, version folders, project files, assembly references, build scripts, and packaging layout.
3. Identify the target RimWorld build, Unity player version, target framework, DLC matrix, required dependencies, optional integrations, and supported save lifecycle.
4. Separate verified facts from inference. If source is absent, inspect XML, metadata, package layout, and logs, then state what remains unknown.
5. Ask the user only when an unresolved choice changes public behavior, compatibility, save format, destructive migration, or architecture.

Read [references/fundamentals.md](references/fundamentals.md) when establishing the environment or project baseline.

## Classify the task

Choose the smallest applicable profile, then read only the linked references.

- Architecture, refactoring, boundaries, coupling, or design review: [references/engineering-philosophy.md](references/engineering-philosophy.md)
- XML/content, recipes, research, weapons, apparel, buildings: [references/content-xml.md](references/content-xml.md)
- Runtime components, jobs, needs, maps, quests, world state, saves: [references/runtime-save.md](references/runtime-save.md)
- Harmony or cross-mod integration: [references/harmony-compat.md](references/harmony-compat.md)
- Settings that change runtime behavior or global Def rewriting: [references/configuration-def-transforms.md](references/configuration-def-transforms.md)
- UI, tick cost, allocation, threading, networking: [references/performance-ui-threading.md](references/performance-ui-threading.md)
- Logging, diagnostics, support bundles, or developer tools: [references/observability-developer-tools.md](references/observability-developer-tools.md)
- Architecture or project planning: [references/mod-archetypes.md](references/mod-archetypes.md)
- Review or high-intrusion assessment: [references/engineering-review.md](references/engineering-review.md)
- Tests, packaging, release, or update support: [references/testing-release.md](references/testing-release.md)
- Framework/infrastructure core or an extension targeting it: [references/framework-profile.md](references/framework-profile.md)

## Use this workflow

### 1. Define the change contract

Write down:

- player-visible outcome and explicit non-goals;
- affected Defs, types, methods, saves, maps, factions, and integrations;
- required versus optional dependencies;
- behavior when an optional dependency is absent, outdated, or fails;
- whether adding, updating, disabling, or removing the mod is supported on an existing save.

Also identify the authoritative state, derived state, invariants that must survive every path, and the lifecycle phase in which the change becomes valid. Do this before selecting classes or patterns.

### 2. Choose the least-coupled extension point

Prefer, in order when they can express the same behavior:

1. new Defs and inheritance;
2. conditional XML patches;
3. `ModExtension`, `ThingComp`, `HediffComp`, `GameComponent`, `MapComponent`, or documented framework API;
4. a narrow Harmony postfix or prefix that preserves the original contract;
5. private-member reflection, reverse patches, or transpilers only with a stated reason and compatibility tests.

This is a decision order, not a ban. A necessary deep patch is acceptable when its blast radius and failure behavior are explicit.

### 3. Design boundaries before mechanics

- Keep Def/schema, domain logic, game adapters, integrations, UI, and persistence distinguishable.
- Put optional integrations behind adapters and conditional load folders or assemblies.
- Use direct references for declared required dependencies; do not hide required contracts behind reflection.
- Use stable identifiers in persisted data. Treat renamed Defs, types, fields, and package IDs as migrations.
- Make ownership and lifetime explicit for events, caches, subscriptions, background work, and native resources.
- Make initialization dependencies explicit; do not use incidental assembly, reflection, patch, or callback order as architecture.
- Give each mutable fact one authority. Treat caches, indexes, UI models, and rewritten Def views as derived state with an invalidation or rebuild rule.
- Do not introduce an abstraction until there are multiple implementations, a volatile boundary, or a test seam that justifies it.

### 4. Implement in vertical slices

For each slice, complete Def/schema, behavior, persistence, compatibility, UI, localization, and validation together. Do not accumulate a large untested compatibility phase at the end.

### 5. Validate in widening rings

Run the narrowest useful checks first:

1. compile and static/XML checks;
2. clean startup and Def-resolution checks;
3. focused developer-mode scenario;
4. save/load and migration checks;
5. supported DLC/dependency combinations;
6. representative mod-stack compatibility;
7. performance measurement for hot paths;
8. packaged artifact test, not only source-tree test.

Do not claim a test passed unless it ran. Report skipped checks and why.

## Engineering rules

- Preserve named invariants across success, rejection, cancellation, exception, save/load, and compatibility paths.
- Preserve vanilla and upstream contracts unless the feature explicitly replaces them.
- Keep patches idempotent where practical and avoid duplicate registration on reload or reinitialization.
- Pair acquisition with release and registration with deregistration. Verify the less-visible exits: map removal, return to menu, dependency failure, canceled jobs, expired world objects, and partial activation.
- Validate nullable game state, destroyed/despawned objects, absent maps, unresolved Defs, and partial saves at boundaries.
- Log actionable context once; avoid silent catches and per-tick log floods.
- Do not mutate Unity or Verse objects from background threads. Marshal results through a project-proven main-thread boundary.
- Measure hot paths. Avoid blanket rules such as “never allocate” or “never use LINQ”; optimize confirmed per-frame/per-tick pressure.
- For batch operations, decide explicitly whether semantics are atomic, retryable, or best-effort.
- Keep balancing policy separate from infrastructure and compatibility mechanics where feasible.
- Record deliberate boundary violations with scope, reason, owner, tests, and a removal or reassessment trigger. A compatibility exception is controlled debt, not an invisible precedent.
- Never infer code quality, intent, or authorship ethics from balance choices or patch count alone.
- Write the deliverable in the user’s language unless the repository requires another language.

## Deliverables

When planning or reviewing, return:

1. verified environment and assumptions;
2. selected mod profile and affected surfaces;
3. design and dependency direction;
4. save/compatibility/performance risks;
5. implementation slices;
6. validation matrix;
7. unresolved decisions requiring the user.

When editing, also summarize changed files, checks actually run, and remaining limitations.
