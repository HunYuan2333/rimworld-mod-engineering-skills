# Framework and infrastructure project profile

Use this only for a framework/infrastructure core or mods intentionally integrating with one. The generic guidance remains authoritative where this profile is silent.

## Project intent

Sample F is primarily framework/infrastructure rather than a gameplay-content overhaul. Preserve that distinction: client systems, contracts, extension loading, service boundaries, UI, networking, and observability should not silently absorb balance or content policy.

Read the repository’s current agent instructions, design philosophy, developer guide, and compatibility-boundary document before changing code. Repository documents override this snapshot when they conflict.

## Baseline boundaries

- Client and server/tooling targets may differ; do not leak server-only APIs into the game client.
- Depend inward on contracts; keep loaders, network transports, game adapters, persistence, and UI outside domain policy.
- Prefer extension/plugin registration over modifying core feature switches for each add-on.
- Make lifecycle states explicit: discovery, validation, load, initialization, running, stopping, failed.
- Isolate extension failure so one optional plugin cannot disable unrelated features.
- Keep the host neutral: feature-specific policy belongs to peer extensions or their contracts, not special host branches.
- Allow peer extensions to collaborate through their own small contracts; do not make the host a mediator for every business interaction.

## Extension contract

For an extension define:

- identity, version, compatible API/capability range, and dependencies;
- registration and initialization order;
- owned resources and cleanup;
- configuration schema/defaults/migration;
- thread-affinity requirements;
- user-visible failure/diagnostic behavior;
- behavior when a server, dependency, or capability is unavailable.

Use a dependency graph when order is semantic. Detect cycles, block dependents of failed required extensions, and shut down in reverse dependency order. Do not use DLL filename order as the only semantic ordering mechanism, even if physical loading still requires a compatible file order.

Separate phases:

- discovery locates candidates and metadata;
- registration contributes APIs/handlers without starting work;
- activation obtains live services and subscribes;
- shutdown reverses activation and releases resources.

Track states such as discovered, disabled, dependency-disabled, registered, active, failed, and shutdown when the framework exposes runtime management. On registration or activation failure, remove contributions owned by that extension before continuing unrelated participants.

Use compile-time contracts for required framework APIs. Use capability checks/adapters for optional or version-varying features. Do not use reflection merely to avoid declaring a real dependency.

## Reliability

Bound queues, payload sizes, retries, timeouts, and logs. Keep external I/O off the game thread. Return immutable/pure results, then apply them through a verified main-thread handoff after revalidating game state.

For batch operations explicitly select atomic, retryable, or best-effort behavior. Do not silently skip failures when partial completion would violate state consistency.

Adapters translate protocol/capability differences but must not weaken domain invariants. A transport send, handler return value, optimistic UI update, and authoritative remote confirmation are distinct events. Unknown results require reconciliation; they must not be silently converted into success or safe rollback.

Registration, event subscription, timers, queues, and background work require symmetric shutdown. Repeated activate/shutdown or load/menu/load cycles should not duplicate handlers or retain old game objects.

## Validation

At minimum test:

- framework without extensions;
- one valid extension;
- invalid metadata/dependency/API range;
- extension initialization and runtime failure isolation;
- enable/disable/reload lifecycle if supported;
- configuration migration;
- network timeout/disconnect/backpressure;
- save/load when extensions persist game state;
- packaged client assembly under the actual RimWorld build.

Keep framework-specific tests and examples separate from generic RimWorld recommendations so the skill remains useful to conventional content mods.
