# Runtime state, lifecycle, and saves

## Choose state ownership deliberately

- `ThingComp` / `HediffComp`: state belongs to one spawned or held object.
- `MapComponent`: state belongs to one map and should disappear with that map.
- `GameComponent`: state belongs to the current game/save.
- `WorldComponent`: state spans maps and belongs to the world.
- static cache: derived, reconstructible process state only; clear/rebuild across game transitions.

Do not put save-owned mutable state in global statics. Do not serialize a cache that can be rebuilt unless rebuilding would lose meaning or be prohibitively expensive.

Classify static state explicitly:

| Scope | Examples | Required reset |
|---|---|---|
| Process | immutable metadata, reflection delegates | normally none; must not retain game objects |
| Def-load cycle | indexes over resolved Defs | after Def reload/reinitialization if supported |
| Game/save | throttles, lookup tables, derived services | new/load game and return to menu |
| Map | spatial grids, network lookup | map removal and game teardown |
| Operation | recursion guards, transaction context | `finally`, including exception and early return |

Static state without an explicit scope is a review finding even if it currently appears harmless.

## Object ownership transfers

Pawn, Thing, WorldPawn, Lord, Quest, Map, and spawned/despawned state participate in game-owned lifecycles. Treat transitions as ownership transfers, not collection edits.

For every transfer record the source owner, destination owner, legal intermediate state, persistence mode, and cleanup path. Common pairs include:

- spawn / despawn or destroy;
- register / deregister;
- add to world pawns / remove or return to discard decision;
- retain indefinitely / release for game cleanup;
- subscribe / unsubscribe;
- acquire native or managed resource / dispose.

Removing a Pawn from a mod-owned list does not release a `KeepForever` world-pawn decision. Likewise, destroying an object does not automatically remove every cache, dictionary key, event subscription, or saved reference that points to it.

Audit uncommon exits: expiry, rejection, failed generation, canceled quest/job, map removal, faction change, death during callbacks, partial load, and mod shutdown. Acquisition and release should be symmetric even when implemented in different components.

## Persistence rules

For every persisted field define:

- owner and lifetime;
- Scribe mode (`Value`, `Def`, `Reference`, `Deep`, or collection look mode);
- default when missing from an older save;
- validity after references resolve;
- behavior when a Def/mod/DLC is missing;
- migration from previous names/shapes;
- cleanup when referenced things are destroyed, despawned, or removed.

Use `PostLoadInit` or a later lifecycle stage for repairs that require resolved cross-references. Keep migrations versioned, idempotent, and observable in logs. Never silently reinterpret old state when that changes player-visible meaning.

## State machines

Jobs, quests, long operations, network requests, and multi-stage combat actions should expose explicit states and legal transitions. Define cancellation, timeout, save/load, retry, and cleanup behavior. Avoid scattered booleans whose combinations can represent impossible states.

For reentrant or recursive game calls, keep contextual state scoped to the call chain. Prefer an explicit context parameter when available. Otherwise use a stack/token or thread-local context with push/pop in `try/finally`; a single static boolean cannot distinguish nesting and can leak after exceptions.

## Tick and collection work

Estimate cost as `entities × frequency × work`. Prefer event-driven invalidation, staggered ticks, spatial indexes, or bounded queues where semantics permit. Never change correctness-sensitive atomic work into best-effort skipping just for performance; decide semantics first.

When iterating mutable game collections, account for removal/despawn during callbacks. Copy only when necessary and measured; otherwise use stable iteration protocols or deferred changes.

## Save compatibility statement

Document separately:

- add to an existing save;
- update from each supported prior version;
- disable temporarily;
- remove permanently;
- downgrade.

“Loads successfully once” is not a migration test. Exercise pre-change save -> update -> play -> save -> reload, and test missing optional content where supported.
