# Performance, UI, and threading

## Measure the correct budget

Classify the path before optimizing:

- per GUI event/frame;
- every tick or periodic tick;
- per pawn/projectile/building/map;
- load-time only;
- rare player command;
- background I/O or compute.

Allocation is one cost among CPU, draw calls, material creation, pathfinding, Def lookup, logging, reflection, and collection churn. Optimize measured hotspots and preserve clarity elsewhere.

## IMGUI

`DoWindowContents`, gizmos, overlays, and inspectors can execute frequently and for multiple event types. In hot UI paths:

- precompute filtered/sorted data when inputs change;
- cache expensive text measurement, regex, textures, and derived models with explicit invalidation;
- virtualize or clip large lists;
- avoid repeated Def scans and material creation;
- keep event handling deterministic and restore GUI state.

`new` is not automatically a heap allocation: value types and compiler behavior matter. LINQ is not forbidden; remove it from a hot path only when profiling or inspection shows meaningful cost.

## Tick work

Use `TickRare`/`TickLong`, staggering, dirty queues, or event-driven updates only if latency semantics permit. Do not perform a full-map or all-pawn scan for each entity. Cache Def queries after Def loading, not before the database is ready.

## Threading

Assume Verse and Unity objects are main-thread-bound unless the project proves otherwise. Background work may operate on immutable snapshots and pure data. It must not touch maps, pawns, Def databases, textures, GUI, logging facilities with unknown thread safety, or Scribe state.

Return results through the project’s verified dispatcher/lifecycle hook, revalidate referenced game state on arrival, and support cancellation during game unload or mod shutdown. `LongEventHandler.ExecuteWhenFinished` is for the long-event lifecycle; do not treat it as a universal runtime dispatcher without verifying context.

Harmony patches run on the caller’s thread. A patched method is not made main-thread-safe by being patched.

## Networking or external processes

Define timeouts, cancellation, retry/backoff, maximum queue sizes, message limits, authentication boundaries, and failure isolation. Never block the game thread on remote I/O. Bound logs and user notifications during repeated failure.

## Performance evidence

Report scenario, map/pawn/projectile counts, mod list, duration, profiler, baseline, result, and variance. A microbenchmark outside the game does not prove an in-game improvement.
