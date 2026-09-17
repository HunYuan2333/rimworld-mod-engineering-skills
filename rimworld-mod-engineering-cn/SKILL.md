---
name: rimworld-mod-engineering-cn
description: 开发、审查、调试或规划 RimWorld Mod，覆盖 XML Def、C#、Harmony、内容系统、存档、兼容性、性能、打包与发布。适用于小型 XML Mod、UI/QoL Mod、种族或派系 Mod、系统 Mod、战斗大改、框架 Mod 及框架扩展，也适用于评估补丁侵入性、跨 Mod 耦合、升级风险和可维护性。
---

# RimWorld Mod 工程

把 RimWorld Mod 视为运行在共享、持久化且通常高度模组化进程中的软件。工程流程应与变更规模相称：两个 Def 的内容调整不需要企业级架构，而战斗大改或扩展框架需要明确的边界、兼容性所有权、迁移和分阶段验证。

## 从证据开始

提出方案或修改代码前：

1. 阅读仓库指令、设计哲学和开发者文档。
2. 检查 `About/About.xml`、`LoadFolders.xml`、版本目录、工程文件、程序集引用、构建脚本和打包布局。
3. 确认目标 RimWorld 构建、Unity 版本、目标框架、DLC 矩阵、必需依赖、可选集成及支持的存档生命周期。
4. 区分已验证事实与推断。无源码时检查 XML、元数据、包结构、日志和程序集，并说明未知项。
5. 只有当未决选择会改变公开行为、兼容性、存档格式、破坏性迁移或架构时才询问用户。

建立环境或项目基线时阅读 [references/fundamentals.md](references/fundamentals.md)。

## 判断任务类型

选择最小适用类型，只读取关联参考：

- 架构、重构、边界、耦合或设计审查：[references/engineering-philosophy.md](references/engineering-philosophy.md)
- XML、内容、配方、研究、武器、服装、建筑：[references/content-xml.md](references/content-xml.md)
- 运行时组件、工作、需求、地图、任务、世界状态、存档：[references/runtime-save.md](references/runtime-save.md)
- Harmony 或跨 Mod 集成：[references/harmony-compat.md](references/harmony-compat.md)
- 会改变运行行为的设置或全局 Def 重写：[references/configuration-def-transforms.md](references/configuration-def-transforms.md)
- UI、Tick 成本、分配、线程、网络：[references/performance-ui-threading.md](references/performance-ui-threading.md)
- 日志、诊断、支持包或开发工具：[references/observability-developer-tools.md](references/observability-developer-tools.md)
- 架构或项目规划：[references/mod-archetypes.md](references/mod-archetypes.md)
- 审查或高侵入性评估：[references/engineering-review.md](references/engineering-review.md)
- 测试、打包、发布或升级支持：[references/testing-release.md](references/testing-release.md)
- 框架/基础设施核心或其扩展：[references/framework-profile.md](references/framework-profile.md)

## 工作流程

### 1. 定义变更契约

写明：

- 玩家可见结果和明确的非目标；
- 受影响的 Def、类型、方法、存档、地图、派系和集成；
- 必需依赖和可选依赖；
- 可选依赖缺失、过期或失败时的行为；
- 现有存档是否支持添加、升级、禁用或移除该 Mod。

同时确定权威状态、派生状态、所有路径都必须保持的不变量，以及变更从哪个生命周期阶段开始有效。先做这些，再选择类或设计模式。

### 2. 选择耦合最小的扩展点

在能表达相同行为时，优先顺序为：

1. 新 Def 与继承；
2. 条件 XML Patch；
3. `ModExtension`、`ThingComp`、`HediffComp`、`GameComponent`、`MapComponent` 或公开框架 API；
4. 保留原契约的窄 Harmony Postfix 或 Prefix；
5. 仅在说明原因并具备兼容测试时使用私有成员反射、Reverse Patch 或 Transpiler。

这是决策顺序，不是禁令。必要的深层补丁可以接受，但必须明确影响范围和失败行为。

### 3. 先设计边界，再实现机制

- 让 Def/schema、领域逻辑、游戏适配器、集成、UI 和持久化保持可区分。
- 将可选集成隔离在适配器、条件加载目录或独立程序集后面。
- 必需依赖使用直接引用；不要用反射隐藏真实契约。
- 持久化数据使用稳定标识；Def、类型、字段和包 ID 改名均视为迁移。
- 明确事件、缓存、订阅、后台工作和原生资源的所有权与寿命。
- 显式描述初始化依赖；不要把程序集、反射、补丁或回调的偶然顺序当架构。
- 每项可变事实只有一个权威来源；缓存、索引、UI 模型和重写后的 Def 视图都必须有失效或重建规则。
- 只有存在多个实现、易变边界或有价值测试缝时才引入抽象。

### 4. 按垂直切片实现

每个切片一起完成 Def/schema、行为、持久化、兼容、UI、本地化和验证。不要把所有兼容工作留到最后。

### 5. 扩大验证范围

从最窄的有效检查开始：

1. 编译与静态/XML 检查；
2. 干净启动和 Def 解析；
3. 聚焦的开发者模式场景；
4. 存档读写与迁移；
5. 支持的 DLC/依赖组合；
6. 代表性 Mod 组合；
7. 热路径性能测量；
8. 最终打包产物，而不只是源码目录。

未实际运行的测试不得声称通过；说明跳过项及原因。

## 工程规则

- 在成功、拒绝、取消、异常、存读档和兼容路径中保持已命名的不变量。
- 除非功能明确替换原行为，否则保留原版和上游契约。
- 补丁尽量幂等，防止重载或重新初始化时重复注册。
- 获取与释放、注册与注销必须配对；特别检查地图移除、返回主菜单、依赖失败、工作取消、世界对象过期和部分激活。
- 在边界验证空游戏状态、已销毁/未生成对象、无地图、未解析 Def 和不完整存档。
- 日志一次性提供可操作上下文；避免静默捕获和逐 Tick 刷屏。
- 后台线程不得修改 Unity 或 Verse 对象；通过项目已验证的主线程边界应用结果。
- 测量热路径。不要机械禁止分配或 LINQ；只优化已确认的每帧/每 Tick 压力。
- 批处理明确选择原子、可重试或尽力而为语义。
- 尽量将平衡策略与基础设施、兼容机制分离。
- 记录刻意的边界突破：范围、原因、负责人、测试以及移除或重新评估条件。兼容性例外是受控债务，不是隐形先例。
- 不根据平衡选择、补丁数量或题材推断代码质量、动机或作者伦理。
- 除非仓库要求其他语言，交付物使用用户语言。

## 交付内容

规划或审查时给出：

1. 已验证环境与假设；
2. 选定的 Mod 类型和受影响表面；
3. 设计与依赖方向；
4. 存档、兼容和性能风险；
5. 实施切片；
6. 验证矩阵；
7. 需要用户决定的未决项。

修改代码时还要说明变更文件、实际运行的检查和剩余限制。
