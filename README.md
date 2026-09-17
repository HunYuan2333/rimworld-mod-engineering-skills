# RimWorld Mod Engineering Skill

[简体中文](README.zh-CN.md)

A progressive-disclosure Codex skill for engineering, reviewing, debugging, and planning RimWorld mods. It covers XML and Def content, C# runtime systems, Harmony integration, lifecycle and save compatibility, performance, observability, reproducible builds, packaging, and framework-style mods.

The repository provides two standalone variants with the same structure and engineering guidance:

- [English skill](rimworld-mod-engineering-en/SKILL.md) — `rimworld-mod-engineering-en`
- [Simplified Chinese skill](rimworld-mod-engineering-cn/SKILL.md) — `rimworld-mod-engineering-cn`

Each variant contains a concise `SKILL.md` entry point and topic-specific files under `references/`. The entry point routes Codex to only the references needed for the current task.

## Installation

Copy the language folder you want into your Codex skills directory. Keep the folder name unchanged so it remains aligned with the skill name in its frontmatter.

```text
rimworld-mod-engineering-en/
rimworld-mod-engineering-cn/
```

Install both folders if you want both language variants available as separate skills.

## Scope

The skill helps with:

- environment and dependency baselines;
- XML, Defs, content graphs, and conditional loading;
- architecture, cohesion, coupling, and design decisions;
- runtime state, ownership, lifecycle, and save migrations;
- Harmony patching and compatibility compromises;
- settings and global Def transformations;
- UI, performance, threading, and networking;
- diagnostics and developer tools;
- testing, reproducible builds, packaging, and release review;
- framework and extension-host engineering.

Real-world implementation patterns and counterexamples are anonymized. The guidance evaluates observable engineering properties rather than judging a mod by its theme, balance choices, popularity, or patch count.
