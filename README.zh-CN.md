# RimWorld Mod 工程 Skill

[English](README.md)

一套采用渐进式披露的 Codex Skill，用于开发、审查、调试和规划 RimWorld Mod。内容覆盖 XML 与 Def、C# 运行时系统、Harmony 集成、生命周期与存档兼容、性能、可观测性、可复现构建、打包发布以及框架型 Mod。

仓库提供两个结构和工程准则一致、可以独立使用的语言版本：

- [英文版 Skill](rimworld-mod-engineering-en/SKILL.md) — `rimworld-mod-engineering-en`
- [简体中文版 Skill](rimworld-mod-engineering-cn/SKILL.md) — `rimworld-mod-engineering-cn`

每个版本都包含一个精简的 `SKILL.md` 入口，以及位于 `references/` 下的专题参考文档。入口会根据当前任务，只引导 Codex 读取真正需要的参考内容。

## 安装

将需要的语言文件夹复制到 Codex Skills 目录。请保留原文件夹名称，使其与 `SKILL.md` frontmatter 中的 Skill 名称一致。

```text
rimworld-mod-engineering-en/
rimworld-mod-engineering-cn/
```

如果希望两个语言版本都可用，可以同时安装两个文件夹；它们会作为两个独立 Skill 出现。

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
