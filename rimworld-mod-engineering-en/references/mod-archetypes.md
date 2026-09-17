# Mod archetypes and proportional architecture

Classify by behavior and risk, not by download count or perceived ambition. A mod can occupy several profiles. The samples below are deliberately anonymous and describe inspected source or decompiled release assemblies.

## XML-only or small content mod

Examples: a few weapons, recipes, apparel, research nodes, or balance patches.

Priorities: stable Def names, inheritance, localization, recipe/research reachability, texture paths, conditional DLC patches, and load error checks. Avoid adding C# merely to make the layout look engineered.

## UI or QoL patch mod

Examples: tabs, gizmos, overlays, filters, alerts, or workflow changes.

Priorities: preserve vanilla semantics, patch narrow public seams, cache only measured expensive work, handle resolution/scale/input, and test with other UI patches.

## Race or faction mod

Sample D spans race/body/rendering/apparel, pawn kinds, factions, incidents, traders, backstories, DLC-gated genes, world pawns, and persistent components.

Priorities: race-framework contracts, apparel/body compatibility, generation invariants, faction/world lifecycle, DLC gates, pawn-reference persistence, and safe behavior when content packs are absent.

## System or simulation mod

Decompiled Sample A implements a cross-cutting loop: needs feed AI/jobs, buildings register into map-owned utility networks, networks drive thoughts/hediffs/incidents/settings, and map state is persisted.

Its useful structural pattern is `ThingComp -> register/deregister -> MapComponent -> dirty grid/network rebuild -> bounded tick`. It also demonstrates explicit back-compatibility mappings and clearing static per-map caches on map/game teardown. Treat these as sample patterns, not mandatory class names.

## Combat or rules overhaul

Sample E replaces combat algorithms, converts Defs, auto-patches compatibility, uses many Harmony hooks, and separates compatibility assemblies.

Priorities: explicit incompatibilities, deterministic conversion, compatibility ownership, patch conflict diagnostics, new-save policy, representative stack tests, and profiling. Private access and broad patches may be consequences of replacement scope; they are not defaults for unrelated mods.

## Framework or library mod

Sample F has little direct game content and concentrates on extension loading, contracts, networking, UI, lifecycle, and observability.

Priorities: small stable contracts, dependency direction toward contracts, capability/version negotiation, lifecycle ordering, failure isolation, diagnostics, examples, and deprecation/migration policy. Framework convenience never overrides host-game thread or lifecycle constraints.

## High-integration content suite

Decompiled Samples B and C sit between content pack and overhaul. They combine large XML/asset surfaces with persistent mechanics, global runtime patches, settings, and conditionally loaded integrations.

Use [engineering-review.md](engineering-review.md) to quantify this profile. Do not equate balance escalation or competitive design with poor engineering. Ask whether global behavior, coupling, persistence, and failure modes are controlled.

## Scaling guidance

- Small: keep a simple layout and direct code until volatility or reuse appears.
- Medium: organize by feature, with shared infrastructure kept small.
- Large: establish a composition root, contracts/adapters, compatibility modules, state ownership, migrations, and a validation matrix.

Line counts are signals, not architecture thresholds. Split when a file has multiple reasons to change, hides ownership, prevents focused tests, or creates merge/compatibility hotspots.
