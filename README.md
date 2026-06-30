# RimWorld Mod Engineering

用环世界的语言解释怎么写不崩、不卡、不跟别人打架的 Mod。

不需要计算机学位。每条原则配了游戏内类比和好坏代码对照。

同时是一个 Claude Code Skill——在你写 Mod 时 AI 自动遵守这些规则。

## 这是什么

你在给 RimWorld 写 Mod。一开始一个 `Mod.cs` 几百行，挺好。后来加了交易、加了聊天——某天装了个别人的 Mod，你的炸了。或者玩家说"你的 Mod 好卡"。你打开自己三个月前写的代码，不知道它在干什么。

这份指南就是为了防止这些事情。它不是 C# 教程，不是 RimWorld Mod 入门——它假设你已经能写出能跑的 Mod，然后告诉你怎么写不会在日后爆炸。

## 内容

| 模块 | 解决什么 |
|------|---------|
| 项目结构 | 代码全堆一个文件，改不动 |
| 工程原则 | 修一个 Bug 引出新 Bug、加功能要改旧代码 |
| 设计模式 | 每次遇到类似问题不知道标准解法 |
| IMGUI 性能 | UI 间歇性卡顿 |
| C# 坑 | 莫名其妙的崩溃、内存泄漏、老 Mono 特有 Bug |
| Harmony | 补丁跟别人的 Mod 冲突 |

每份文件独立，格式统一：**环世界类比 → 会遇到的问题 → 好代码 vs 坏代码 → 判断标准**。

## 用法

作为 Claude Code Skill，扔到 skills 目录：

```
~/.claude/skills/rimworld-mod-engineering/
```

重启 Claude Code 后自动生效。写 Mod、审查代码、排查性能问题时 AI 会按这些规则工作。

手动调用：`/rimworld-mod-engineering`

也可以直接读文件——从 `SKILL.md` 开始，按需跳。

## 文件结构

```
SKILL.md                总览，六条铁律，自查表，反模式速查
project-structure.md    三层项目模板，分层模型，什么时候拆文件
se-principles.md        17 条工程原则
design-patterns.md      10 个设计模式，带 Mod 场景代码
imgui-performance.md    IMGUI 每帧零分配，脏标记，Profiler
csharp-pitfalls.md      C# 和 Mono 的坑，存档注意项
harmony-patching.md     Harmony 补丁怎么写不冲突
```

## 几个基本想法

1. Mod 不是一次性的。你要维护它、别人要兼容它、玩家在 50 个 Mod 一起跑的环境里用它。

2. 依赖接口不依赖具体实现。别人的 Mod 不在时你的应该失去附加功能，不是直接崩溃。

3. 每步保持可运行。重构不改行为。增量做。每次 commit 是能跑的状态。

4. 临时方案没问题——记下来，定个时间还。永远不还的临时方案才是问题。

## 谁需要

- 写完第一个能跑的 Mod，想知道然后干嘛
- Mod 越来越复杂，一个文件好几百行
- 玩家说卡，但不知道从哪查
- Mod 跟别人的冲突，不知道怎么隔离开
- 想给 Mod 加插件系统让第三方扩展

## 许可

MIT
