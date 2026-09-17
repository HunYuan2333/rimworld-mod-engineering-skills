# RimWorld Mod Engineering Skill

[简体中文](README.zh-CN.md)

A progressive-disclosure Agent Skill for engineering, reviewing, debugging, and planning RimWorld mods. It works with Codex, Claude Code, OpenCode, and DeepSeek Harness. It covers XML and Def content, C# runtime systems, Harmony integration, lifecycle and save compatibility, performance, observability, reproducible builds, packaging, and framework-style mods.

The repository provides two standalone variants with the same structure and engineering guidance:

- [English skill](rimworld-mod-engineering-en/SKILL.md) — `rimworld-mod-engineering-en`
- [Simplified Chinese skill](rimworld-mod-engineering-cn/SKILL.md) — `rimworld-mod-engineering-cn`

Each variant contains a concise `SKILL.md` entry point and topic-specific files under `references/`. The entry point routes the agent to only the references needed for the current task.

## Installation

Clone or download this repository, then copy the complete language folder you want into one of the skill directories below. Keep the folder name unchanged and copy `references/` together with `SKILL.md`.

```text
rimworld-mod-engineering-en/
rimworld-mod-engineering-cn/
```

Install both folders if you want both language variants available as separate skills. Because their scopes overlap, use explicit invocation when both are installed to choose the response language reliably.

`~` means your home directory (`%USERPROFILE%` on Windows). Choose a personal location to use the skill in every repository, or a project location to commit it with one repository.

| Agent | Personal installation | Project installation |
| --- | --- | --- |
| Codex | `~/.agents/skills/<skill-name>/` | `<repo>/.agents/skills/<skill-name>/` |
| Claude Code | `~/.claude/skills/<skill-name>/` | `<repo>/.claude/skills/<skill-name>/` |
| OpenCode | `~/.config/opencode/skills/<skill-name>/` | `<repo>/.opencode/skills/<skill-name>/` |
| DeepSeek Harness | `~/.dsh/skills/<skill-name>/` | `<repo>/.dsh/skills/<skill-name>/` |

Codex, OpenCode, and DeepSeek Harness also scan `~/.agents/skills/` and `<repo>/.agents/skills/`. Put the skill there if you want those three agents to share one installation. Claude Code uses `.claude/skills/`, so copy the folder there separately.

For example, a personal Chinese installation should contain `~/.agents/skills/rimworld-mod-engineering-cn/SKILL.md` for the shared location, or `~/.claude/skills/rimworld-mod-engineering-cn/SKILL.md` for Claude Code.

After installing, start a new agent session if the skill does not appear immediately. See the official documentation for [Codex](https://developers.openai.com/codex/skills), [Claude Code](https://code.claude.com/docs/en/skills), [OpenCode](https://opencode.ai/docs/skills/), and [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/skills.md) for discovery and configuration details.

## Usage

Open your RimWorld mod repository in the agent, then invoke the installed language variant. Explicit invocation is the most predictable way to use it:

- Codex: type `$rimworld-mod-engineering-en` or `$rimworld-mod-engineering-cn`; `/skills` lists installed skills.
- Claude Code: type `/rimworld-mod-engineering-en` or `/rimworld-mod-engineering-cn`.
- OpenCode: ask it to “use the `rimworld-mod-engineering-en` skill” (or the Chinese variant). OpenCode loads it through its built-in `skill` tool.
- DeepSeek Harness: type `/rimworld-mod-engineering-en` or `/rimworld-mod-engineering-cn`.

For example:

```text
$rimworld-mod-engineering-en review this mod's Harmony patches, save compatibility, and release packaging
```

```text
/rimworld-mod-engineering-cn 检查这个 Mod 的 Harmony Patch、存档兼容性和发布打包
```

The agent may also activate the skill automatically when your request matches its description. Once loaded, the skill first inspects the repository and task constraints, then reads only the relevant files under `references/`. You do not need to open those files manually.

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
