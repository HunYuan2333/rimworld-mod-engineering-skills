# Environment and project baseline

## Version facts and verification

Do not treat version numbers as timeless. Verify the target installation and record the result.

For the locally inspected Windows installation on 2026-09-17:

- RimWorld: `1.6.4871 rev590`
- Unity player: `2022.3.35f1`

Read the game build from `RimWorld/Version.txt`. Read the Unity version from the first lines of `Player.log` or the version string in `RimWorldWin64_Data/globalgamemanagers`. If creating AssetBundles, use the game’s Unity editor version as closely as practical and test the produced bundle in the actual game build. Ordinary XML/C# mods do not require opening the Unity editor.

## .NET and C#

For RimWorld 1.6, `.NET Framework 4.7.2` is a conservative client baseline found in inspected production mods. Other inspected 1.6 assemblies target `.NET Framework 4.8`. Select the target from the game/runtime and dependency ecosystem rather than copying a fashionable SDK target.

Distinguish:

- target framework: available runtime/library surface;
- C# language version: compiler syntax and lowering;
- Unity version: engine and managed runtime behavior;
- referenced game DLL version: compile-time API shape.

A newer language version can emit compatible IL, but a successfully compiled mod can still call APIs unavailable in the game runtime. Do not ship `Assembly-CSharp.dll`, Unity DLLs, or framework reference assemblies with the mod.

Typical references include `Assembly-CSharp.dll`, required Unity modules, and declared library-mod assemblies. Resolve them through repository-relative properties, environment properties, or a documented local override; avoid committing one developer’s absolute Steam path.

## Package layout

Check at minimum:

```text
About/About.xml
LoadFolders.xml                 # when versions/DLC/integrations differ
Common/ or version folders
Defs/
Patches/
Assemblies/
Languages/
Textures/, Sounds/, AssetBundles/ as needed
```

Use `supportedVersions`, `packageId`, dependencies, `loadAfter`, `loadBefore`, and `incompatibleWith` deliberately. `loadAfter` orders mods; it does not declare a required dependency. Do not list every vaguely related mod.

## Discovery checklist

Before work, answer:

- Which game versions are truly supported?
- Is one assembly shared or built per game version?
- Which DLCs are required or optional?
- Which dependencies are compile-time, load-time, and feature-time?
- Are Harmony and framework libraries supplied by a dependency or bundled?
- Does `LoadFolders.xml` isolate optional types so the loader never sees them when dependencies are absent?
- Can the project build from a clean checkout with documented local paths?
- Does the package contain source-only, stale, debug, or duplicate binaries?

Prefer exact local evidence, official game/mod documentation, repository source, and packaged artifacts in that order. Community reports are useful reproduction leads, not proof of causation.
