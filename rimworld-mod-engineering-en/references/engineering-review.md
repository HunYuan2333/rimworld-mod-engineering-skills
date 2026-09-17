# Engineering review and intrusion assessment

## Evidence discipline

Review facts in this order: source, packaged assemblies/decompilation, XML and load layout, logs and reproducible behavior, author documentation, then third-party reports. Decompiled C# can reveal control flow, patch targets, state ownership, and persistence, but it is not canonical source: names may be synthesized, some IL may not decode, and comments/build configuration are generally unavailable.

Never infer motives, ethics, or code quality from balance, theme, popularity, nationality, or “arms race” claims. Translate concerns into observable technical properties.

For architectural reviews, also read [engineering-philosophy.md](engineering-philosophy.md). Separate a demonstrated failure from a latent risk and from a contained compromise:

- **defect**: a reachable path violates a stated invariant;
- **uncontrolled risk**: the design lacks ordering, ownership, isolation, migration, or diagnostics needed for its blast radius;
- **controlled compromise**: a deep technique is necessary and has proportionate containment;
- **style difference**: no meaningful invariant, support promise, or maintenance property is affected.

## Intrusion dimensions

Rate each dimension `0 none`, `1 local`, `2 subsystem`, or `3 global/replacement` and cite evidence:

| Dimension | Question |
|---|---|
| Patch breadth | How much upstream game/mod surface is changed? |
| Patch depth | Additive public seam, method interception, private access, or IL replacement? |
| Dependency coupling | How many required/optional external contracts can break it? |
| Global mutation | Are base Defs, all humanlikes, combat rules, factions, or shared registries changed? |
| Persistent state | How much state survives in saves, maps, world objects, quests, or references? |
| Hot-path exposure | Does code run per frame/tick/pawn/projectile or on global generation/combat paths? |
| Reversibility | Can it be updated/disabled/removed without corrupting or semantically breaking saves? |
| Compatibility surface | How many version/DLC/mod combinations are promised? |

Do not sum scores mechanically into “good/bad.” Use the profile to require proportionate controls.

## Proportionate controls

- Broad patches: explicit target inventory, patch-order diagnostics, representative stack tests.
- Deep patches: version anchors, fail-visible matching, upstream-diff review.
- Many integrations: conditional folders/assemblies, adapters, compatibility ownership matrix.
- Global Def mutation: original-value snapshot, idempotent application, restart policy.
- Persistent systems: schema version, migrations, reference repair, removal policy.
- Hot paths: profiler evidence and bounded work.
- Low reversibility: clear metadata/documentation and migration tooling where feasible.

## Anonymous counterexamples

Use these as failure mechanisms, not as accusations about an entire project.

### Extension scanning without an installation protocol

A large overhaul discovers compatibility classes by scanning all loaded assemblies, constructs each implementation, then calls capability and install methods in enumeration order. Type-load exceptions are handled during scanning, but constructor and installation failures are not isolated per participant. The protocol has no stable identity, dependency, ordering, duplicate, or status contract.

Risk: one optional integration can prevent later integrations from installing, and correctness may depend on unspecified assembly/type order. The lesson is not “avoid reflection”; it is to make discovery and activation a deterministic, diagnosable, failure-isolated lifecycle.

### Business removal without lifecycle release

A persistent race/faction system retains selected world pawns indefinitely so they can return later. One reset path returns them to the game's discard decision, while an expiry path removes them only from mod-owned collections.

Risk: business state says the pawn is gone while the game still retains it forever. The lesson is to model ownership transitions explicitly and audit every exit, not merely clear lists.

### Logging throttle with undeclared scope

A static per-key limiter prevents repeated log floods after a fixed count but has no reset boundary.

Risk: failures in a later save or reload can be permanently hidden for the remainder of the process. The useful mechanism should be retained while declaring whether its scope is burst, map, game, load cycle, or process.

### Global callbacks without ownership

A compatibility surface exposes public mutable static callback lists. Participants append delegates, but registration has no identity, deduplication, priority, exception isolation, or removal.

Risk: repeated initialization can duplicate effects, one callback can break the chain, and delegates can outlive their intended owner. The lesson is to treat callback registration as an API with lifecycle, even if the initial implementation was only a list.

### Def reset with an ambiguous “original”

A settings system captures weapon fields, mutates loaded Defs, and later restores the captured values. In a shared process, the captured baseline may already include earlier modifications while restore may overwrite later modifications.

Risk: correct local reset semantics can still violate cross-mod composition. Declare the baseline phase, deterministic order, provenance needed for diagnosis, and restart/reload policy. See [configuration-def-transforms.md](configuration-def-transforms.md).

### Local build repair that masks environmental coupling

A project references one developer's absolute game/workshop paths and builds directly into a playable assemblies folder. A post-build step deletes a known stale DLL to prevent mixed versions.

The deletion is a useful containment, but the underlying build remains non-reproducible and other stale artifacts can survive. The lesson is to separate configurable dependency resolution, deterministic staging, and final artifact auditing.

## Decompiled samples

These samples were inspected from RimWorld 1.6 release assemblies. Counts are evidence of surface area, not quality scores.

### Sample A — map-scale utility simulation

The main assembly targets `net472`; decompilation produced roughly 257 C# files and 24k lines. It contains two map components, many object components, 46 files with persistence methods, 38 tick-related files, 18 attribute-declared Harmony targets plus manual conditional patches, and one transpiler.

Observed design:

- objects register and deregister with a map-owned network service;
- per-pipe-type dirty flags defer full grid reconstruction;
- the map component owns cell grids, network instances, and per-map lifecycle;
- static dictionaries accelerate map-component lookup and are cleared on map removal/game teardown;
- old type and Def names are mapped explicitly for save compatibility;
- optional features install their patches conditionally during startup.

Reusable lesson: map-scale simulations benefit from map ownership, explicit registration, dirty invalidation, and teardown. Review per-tick network work, full-grid rebuild triggers, static-cache cleanup, multiplayer determinism, and startup-only feature toggles.

### Sample B — faction, narrative, and combat suite

The main assembly targets `net48`; decompilation produced roughly 418 C# files and 40k lines. It contains about 50 attribute-declared Harmony targets, two transpilers, 69 files with persistence methods, 42 tick-related files, multiple game/world managers, and several separately loaded integration assemblies.

Observed design:

- a game component coordinates campaign, narrative, title progression, special-pawn, support, and end-state managers;
- legacy component migration is represented explicitly;
- transaction logic captures resource snapshots and separates validation from consumption;
- conditional integrations are physically separated from the always-loaded assembly;
- thread-local context brackets recursive/global raid-selection state rather than using one process-global flag;
- patches touch pawn generation, damage, death, jobs, faction changes, raids, mental state, rendering, trade, titles, and pregnancy.

Reusable lesson: broad gameplay suites need manager boundaries, migration ownership, transactional resource changes, and isolated integrations. Review shared-method conflicts, manager ordering, reference repair, recursive context cleanup, hot-path patch cost, and the complete save/update matrix.

### Sample C — arsenal and custom-mechanics suite

The main assembly targets `net48`; decompilation produced roughly 366 C# files and 47k lines. It contains about 25 attribute-declared Harmony targets, no observed transpiler in the main assembly, 53 files with persistence methods, 55 tick-related files, and separate combat/animation integration assemblies.

Observed design:

- a versioned game component persists aircraft, traders, discounts, slots, cooldowns, and other suite-wide state, then runs explicit migrations;
- settings scan selected weapon Defs, snapshot original damage/cooldown/tag data, and reapply configured values;
- private projectile fields are reached through reflection for global Def mutation;
- rendering code checks Unity main-thread access before constructing visual caches or loading shaders;
- patches touch health, hediffs, mental states, path cost, projectile launch/interception, explosions, power, raids, map removal, range, and weapon rendering.

Reusable lesson: runtime Def rewriting requires immutable baselines, idempotent apply/reset behavior, a defined application phase, and compatibility ownership. Review private-field drift, repeated application, settings UI side effects, main-thread asset creation, global-method conflicts, and removal/migration of persisted custom objects.

## Review output

Conclude with:

- verified facts and decompilation/source limitations;
- intrusion profile by dimension;
- concrete failure modes rather than labels;
- existing containment strengths;
- missing controls or tests;
- questions whose answers would change the architecture.

When reviewing a proposed fix, also state which invariant it restores, whether it changes authority or lifecycle, and which compromise level remains afterward. Do not recommend a broad rewrite when a local ownership, ordering, or cleanup correction is sufficient.
