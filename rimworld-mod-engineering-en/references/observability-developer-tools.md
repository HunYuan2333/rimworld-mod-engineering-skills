# Observability and developer tools

Use this reference for logging, diagnostics, support workflows, debug actions, or in-game inspection tools.

## Observability contract

Diagnostics should answer:

- which version, game build, dependency set, and feature configuration ran;
- which lifecycle phase succeeded or failed;
- which optional integrations were detected, activated, skipped, or degraded;
- which patch/transform/migration rule affected the failing object;
- whether the current state is authoritative, pending, degraded, or recovered.

Log at ownership boundaries and state transitions, not every tick. Preserve the exception and relevant stable identifiers; avoid secrets, full save payloads, or uncontrolled object dumps.

## Throttling without blindness

Repeated failures need bounded output, but throttling has a lifecycle:

- select the key: failure kind plus meaningful owner/Def/patch identity;
- select the scope: burst window, map, game, save-load cycle, or process;
- emit a final suppression summary;
- retain counters for inspection when useful;
- reset counters at the declared lifecycle boundary.

A process-static “first N ever” limiter can hide failures in later saves. Rate limiting is not a substitute for a circuit breaker when the failing work itself is expensive.

## Developer tools

Release builds may contain developer-mode actions when they are gated and support real diagnosis. Define:

- availability gate and required game state;
- preconditions and confirmation for destructive actions;
- deterministic setup or recorded seed where reproduction matters;
- cleanup of spawned objects and temporary state;
- output that states what was changed and what failed;
- separation between smoke scenarios and assertions.

Useful tools create representative states, inspect ownership/caches/patches, exercise migrations, or export a bounded diagnostic summary. A button that merely invokes a method without checking outcomes is not an automated test.

## Failure handling

- Isolate optional participants so one logger, observer, handler, or debug provider cannot stop peers.
- Do not silently catch. If a failure is intentionally ignored, record the reason and enough context to distinguish it from success.
- Avoid logging from per-frame/per-tick paths without deduplication.
- Do not let diagnostic formatting allocate heavily on a hot path when the log level is disabled.
- Prefer one startup summary over hundreds of repetitive discovery messages.

## Support bundle

For complex mods, make it possible to collect a bounded report containing version/build, enabled DLCs and dependencies, relevant settings, extension states, migration versions, patch ownership, and recent errors. Redact credentials and user content. The report should describe observed state; it must not mutate the game to diagnose it.

