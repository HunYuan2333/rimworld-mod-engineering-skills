# RimWorld Mod 工程 Skill

[English](README.md)

一套采用渐进式披露的 Agent Skill，可在 Codex、Claude Code、OpenCode 和 DeepSeek Harness 中用于开发、审查、调试和规划 RimWorld Mod。内容覆盖 XML 与 Def、C# 运行时系统、Harmony 集成、生命周期与存档兼容、性能、可观测性、可复现构建、打包发布以及框架型 Mod。

仓库提供两个结构和工程准则一致、可以独立使用的语言版本：

- [英文版 Skill](rimworld-mod-engineering-en/SKILL.md) — `rimworld-mod-engineering-en`
- [简体中文版 Skill](rimworld-mod-engineering-cn/SKILL.md) — `rimworld-mod-engineering-cn`

每个版本都包含一个精简的 `SKILL.md` 入口，以及位于 `references/` 下的专题参考文档。入口会根据当前任务，只引导 Agent 读取真正需要的参考内容。

## 安装

克隆或下载本仓库，然后将需要的完整语言文件夹复制到下表中的 Skill 目录。请保留原文件夹名称，并将 `references/` 和 `SKILL.md` 一起复制。

```text
rimworld-mod-engineering-en/
rimworld-mod-engineering-cn/
```

如果希望两个语言版本都可用，可以同时安装两个文件夹；它们会作为两个独立 Skill 出现。由于两者的适用范围高度重合，同时安装时建议显式调用，以便稳定选择回复语言。

`~` 表示用户主目录，在 Windows 上即 `%USERPROFILE%`。放在用户目录中可供所有仓库使用；放在项目目录中则可随该仓库一起提交和共享。

| Agent | 用户级安装位置 | 项目级安装位置 |
| --- | --- | --- |
| Codex | `~/.agents/skills/<skill-name>/` | `<repo>/.agents/skills/<skill-name>/` |
| Claude Code | `~/.claude/skills/<skill-name>/` | `<repo>/.claude/skills/<skill-name>/` |
| OpenCode | `~/.config/opencode/skills/<skill-name>/` | `<repo>/.opencode/skills/<skill-name>/` |
| DeepSeek Harness | `~/.dsh/skills/<skill-name>/` | `<repo>/.dsh/skills/<skill-name>/` |

Codex、OpenCode 和 DeepSeek Harness 也都会扫描 `~/.agents/skills/` 与 `<repo>/.agents/skills/`。如果希望这三个 Agent 共用同一份 Skill，推荐安装到这里。Claude Code 使用 `.claude/skills/`，需要另外复制一份。

例如，个人全局安装中文版时，共享目录中的入口应为 `~/.agents/skills/rimworld-mod-engineering-cn/SKILL.md`；Claude Code 中的入口应为 `~/.claude/skills/rimworld-mod-engineering-cn/SKILL.md`。

安装后如果没有立即出现，请新建一次 Agent 会话。各平台的发现与配置细节可查阅官方文档：[Codex](https://developers.openai.com/codex/skills)、[Claude Code](https://code.claude.com/docs/en/skills)、[OpenCode](https://opencode.ai/docs/skills/) 和 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/skills.md)。

## 使用方法

用 Agent 打开你的 RimWorld Mod 仓库，然后显式调用已安装的语言版本。这是最稳定的使用方式：

- Codex：输入 `$rimworld-mod-engineering-cn` 或 `$rimworld-mod-engineering-en`；可用 `/skills` 查看已安装 Skill。
- Claude Code：输入 `/rimworld-mod-engineering-cn` 或 `/rimworld-mod-engineering-en`。
- OpenCode：在任务中明确要求“使用 `rimworld-mod-engineering-cn` skill”（或英文版）；OpenCode 会通过内置 `skill` 工具加载。
- DeepSeek Harness：输入 `/rimworld-mod-engineering-cn` 或 `/rimworld-mod-engineering-en`。

例如：

```text
$rimworld-mod-engineering-cn 检查这个 Mod 的 Harmony Patch、存档兼容性和发布打包
```

```text
/rimworld-mod-engineering-en review this mod's Harmony patches, save compatibility, and release packaging
```

当请求与 Skill 的描述明显匹配时，Agent 也可能自动加载它。加载后，Skill 会先检查仓库和任务约束，再按需读取 `references/` 中相关的专题文档；无需手动逐个打开这些文件。

## 能力范围

这套 Skill 可以帮助处理：

- 游戏环境、版本和依赖基线；
- XML、Def、内容关系图与条件加载；
- 架构、内聚、耦合与设计决策；
- 运行时状态、所有权、生命周期和存档迁移；
- Harmony Patch 与兼容性妥协；
- 设置与全局 Def 转换；
- UI、性能、线程和网络；
- 日志、诊断和开发工具；
- 测试、可复现构建、打包与发布审查；
- 框架与扩展宿主工程。

## 设计原则

Skill 不会机械禁止反射、静态缓存、Harmony、Transpiler 或全局 Def 修改，而是要求这些技术具有与风险相称的边界控制：

- 明确使用原因和需要保护的不变量；
- 明确状态权威、所有权和生命周期；
- 明确初始化、失效、清理和失败行为；
- 明确兼容范围、迁移政策和验证方式；
- 必要的高侵入性修改应当局部化、可诊断并有版本锚点。

真实项目中的实现模式和反例均已去命名化。评估依据是可以观察的软件工程属性，而不是 Mod 的题材、平衡选择、流行度或 Patch 数量。
