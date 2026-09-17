# Validation, packaging, and release

## Test pyramid for mods

Use the layers the project can support:

1. pure C# tests for parsers, policies, state transitions, calculations, migrations;
2. schema/XML checks for duplicate Defs, missing parents, bad paths, and patch failures;
3. build checks against supported game/dependency references;
4. developer-mode scenarios for jobs, incidents, raids, quests, abilities, UI, and destruction;
5. save/load fixtures and update migrations;
6. compatibility smoke matrices;
7. packaged-install smoke tests.

Do not mock all of Verse to test a trivial wrapper. Extract deterministic policy from game adapters instead.

Choose the test form by failure risk:

- pure policy/invariant: ordinary unit or property-style tests;
- Def selection/transformation: fixture database or packaged startup validation;
- lifecycle/ownership: repeated load-unload, map removal, expiry, cancellation, and cleanup scenarios;
- Harmony/private API: target/signature checks plus in-game behavior;
- rendering/UI: resolution, scale, event-type, and long-content scenarios with profiler evidence;
- compatibility: dependency absent/present/outdated and known interacting stacks;
- packaging: launch the exact staged artifact.

Coverage percentage is a weak proxy for mod risk. A single test of an ownership transfer or packaging collision may be more valuable than many wrapper tests.

## Compatibility matrix

Select combinations by risk rather than testing every permutation:

- core only;
- each required dependency baseline;
- each supported DLC gate;
- each optional integration alone;
- known interacting integrations together;
- dependency absent/outdated when graceful degradation is promised;
- a representative large player stack.

Record unsupported combinations in metadata or documentation instead of relying on tribal knowledge.

## Release artifact audit

Inspect the folder that will ship:

- exactly the intended assemblies per version/load condition;
- no game DLLs, reference assemblies, PDBs, test binaries, or stale duplicate DLLs unless intentional;
- valid `About.xml`, preview, version folders, and `LoadFolders.xml`;
- complete languages and assets with correct case;
- source/license notices required by dependencies;
- version/changelog and migration notes;
- clean startup from the packaged artifact.

Also compare the artifact against an explicit manifest or staging rule. Building directly into a playable mod folder is convenient but can mix current XML/assets with an old DLL; detect or remove stale outputs deliberately rather than trusting the last build location.

## Reproducible build contract

A supported build should state:

- game and dependency assembly versions;
- how local reference paths are supplied without committing one machine's Steam path;
- target framework and language version;
- clean/build/package commands;
- deterministic staging layout and expected assemblies;
- how stale or duplicate binaries are detected;
- which generated/vendored inputs are required;
- whether CI verifies the same artifact shape used for release.

Absolute local `HintPath` values, mutable workshop references, and direct output into the release folder create hidden environmental dependencies. They may be tolerated locally, but a release workflow should resolve them through configurable properties or a documented dependency cache and then audit the staged result.

Build success is not artifact success. Verify that XML, DLLs, load folders, and optional compatibility assemblies all come from the same revision and configuration.

## Release policy

State whether a new game is required and whether removal from an existing save is supported. For breaking changes, explain migration or provide a clean failure—not silent corruption.

Keep a rollback artifact for released versions when practical. A source tag is not sufficient if builds depend on mutable local game DLLs or undocumented tooling.

## Completion report

List commands and scenarios actually run, their results, skipped validation, tested versions/configurations, migration impact, and remaining known risks. Avoid “should work” phrasing where verification is possible.
