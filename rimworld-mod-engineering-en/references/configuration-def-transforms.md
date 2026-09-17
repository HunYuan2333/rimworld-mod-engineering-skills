# Configuration and global Def transformations

Use this reference when settings change runtime mechanics, when code rewrites loaded Defs, or when a mod performs automatic compatibility conversion.

## Configuration effect contract

Classify every setting by when it can safely take effect:

| Effect | Meaning |
|---|---|
| Immediate | Existing runtime state and UI can be updated atomically now. |
| Next operation | Only newly created jobs, projectiles, pawns, or commands use it. |
| Next map/game load | Caches or persistent systems must be rebuilt at a lifecycle boundary. |
| New game | Existing saves cannot acquire equivalent semantics safely. |
| Restart required | Patch set, assembly loading, Def transformation, or static initialization is fixed for the process. |

Expose the classification to the user and code. A checkbox is not immediate merely because the UI writes its value immediately.

For each setting define validation, default, migration, authority, change notification, rollback behavior, and whether it changes saved meaning. Avoid applying global mutations from the draw path; enqueue or mark dirty and apply at a defined phase.

## Transformation pipeline

For global Def conversion, define ordered stages such as:

```text
select candidates -> capture baseline -> validate -> plan -> apply -> rebuild caches -> diagnose
```

Require each stage to state:

- its input set and ownership;
- whether it is one-shot or repeatable;
- ordering relative to XML patching, Def resolution, other converters, and cache creation;
- failure semantics: atomic, per-Def best effort, or abort-before-commit;
- diagnostics sufficient to identify the transformed Def and rule.

If per-Def best effort is intentional, isolate exceptions per Def and report a bounded summary. Do not leave a single Def half-transformed.

## Baselines and provenance

“Original value” is ambiguous in a shared mod process. It may mean vanilla, XML-resolved, state after earlier mods, or state at first application. Name the baseline explicitly.

- Capture immutable baselines once at the declared phase.
- Do not overwrite the baseline during reapplication.
- Track which rule changed which field when conflicts matter.
- Decide whether reset restores the captured baseline or recomputes the current mod stack.
- Never assume restoring a captured value will preserve later modifications from another mod.

When several transforms can touch one field, use deterministic ordering and diagnostics. A full provenance graph is unnecessary for small mods, but a global converter should be able to explain its final result.

## Idempotence and cache coherence

An idempotent transform applied twice produces the same result as once. Achieve this through markers, immutable baselines, replace-not-append behavior, or rebuilding from authority.

After mutation, identify every affected cache or derived database. Rebuild them in a documented phase. Mutating a Def without invalidating consumers can be worse than failing the transform.

## Private access and version drift

When a required field is private:

- resolve it once using an exact type/name/signature expectation;
- validate it before applying any transformation;
- fail the feature visibly if it is absent or incompatible;
- keep access in one version-anchored adapter;
- include a startup diagnostic or targeted test for supported game versions.

Do not scatter reflection across settings UI and runtime code.

## Review questions

- What is the authoritative setting value and when does it become effective?
- What is the declared baseline for each rewritten field?
- Can apply/reset run twice safely?
- What happens if one candidate fails?
- Which caches must be invalidated?
- How are conflicts with later transforms detected or explained?
- Can the setting change while a save is running, and what happens to existing objects?
- Does disabling/removing the mod leave persisted objects or globally altered state with undefined meaning?

