# Defs, XML patches, and content systems

## Prefer additive content

Create new Defs and use parent inheritance when possible. Keep stable `defName` values after release. Treat renames as data migrations and supply aliases or migration logic where the game/framework supports them.

Use XML patches when changing upstream content:

- guard optional targets with mod/DLC-aware operations;
- make selectors as specific as necessary but no more brittle than required;
- avoid index-based XPath when semantic selectors exist;
- consider whether the operation remains correct if another mod already added, removed, or reordered the node;
- use `Replace` only when the whole upstream contract is intentionally owned;
- check patch-operation failures in the startup log.

## Model content reachability

For every player-facing item, follow the graph:

```text
research -> designation/recipe -> work giver/job -> ingredients -> product
        -> trader/loot/pawn kind/faction -> UI text/texture/sound
```

For abilities, apparel, races, and buildings also check comps, verbs, hediffs, thoughts, stat workers, gizmos, power/fuel, placement, destruction, and save state.

Content that loads without errors can still be unreachable, impossible to craft, invalid for a race, or orphaned by missing research.

## Conditional content

Use `LoadFolders.xml` to keep optional integration XML and assemblies out of the load graph unless their dependencies are active. A runtime `ModsConfig.IsActive` check cannot rescue a type-load failure caused by an absent referenced assembly.

Prefer one integration folder/module per external contract. Do not mix required and optional types in an assembly that must always load.

## Race and faction content

Validate:

- body types, body parts, apparel layers, draw sizes, and textures;
- pawn generation at every age/stage and all pawn kinds;
- faction goodwill, raids, settlements, traders, incidents, quests, and world pawns;
- DLC-specific genes, xenotypes, ideologies, titles, entities, and vehicles;
- death, resurrection, faction changes, kidnapping, caravan/world transitions, and removal.

## Balance and settings

Keep authorial balance policy separate from compatibility mechanics. If settings mutate Defs globally, record original values once, apply changes at a defined lifecycle point, prevent compounding on reapply, and state whether a restart is required. Validate bounds and combinations, not only individual sliders.

## Asset and localization checks

- Verify case-sensitive paths even when developing on Windows.
- Confirm texture dimensions, masks, shaders, material pools, and atlas behavior.
- Avoid creating materials/textures repeatedly at draw time.
- Provide translation keys for labels, descriptions, messages, commands, settings, and keyed strings.
- Scan for missing and unused keys, but review dynamic keys manually.
