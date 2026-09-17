# 环境与项目基线

## 版本事实与验证

不要把版本号当成永久事实。核实目标安装并记录结果。

在 2026-09-17 检查的本地 Windows 安装为：

- RimWorld：`1.6.4871 rev590`
- Unity Player：`2022.3.35f1`

从 `RimWorld/Version.txt` 读取游戏构建；从 `Player.log` 开头或 `RimWorldWin64_Data/globalgamemanagers` 读取 Unity 版本。制作 AssetBundle 时尽量使用与游戏一致的 Unity 编辑器版本，并在实际游戏构建中测试。普通 XML/C# Mod 不需要打开 Unity 编辑器。

## .NET 与 C#

对于 RimWorld 1.6，实际生产 Mod 中常见 `.NET Framework 4.7.2` 和 `.NET Framework 4.8`。根据游戏运行时与依赖生态选择目标，不要照搬流行 SDK 版本。

区分：

- 目标框架：可用的运行时和库表面；
- C# 语言版本：编译器语法和降级方式；
- Unity 版本：引擎与托管运行时行为；
- 引用的游戏 DLL 版本：编译期 API 形状。

新语言版本可以生成兼容 IL，但编译成功的 Mod 仍可能调用游戏运行时不存在的 API。不要随 Mod 发布 `Assembly-CSharp.dll`、Unity DLL 或框架引用程序集。

常见引用包括 `Assembly-CSharp.dll`、必要的 Unity 模块和已声明的库 Mod 程序集。通过仓库相对属性、环境属性或有文档的本地覆盖解析路径；不要提交某位开发者的绝对 Steam 路径。

## 包结构

至少检查：

```text
About/About.xml
LoadFolders.xml                 # 版本、DLC 或集成不同时
Common/ 或版本目录
Defs/
Patches/
Assemblies/
Languages/
Textures/、Sounds/、AssetBundles/（按需）
```

有意识地使用 `supportedVersions`、`packageId`、依赖、`loadAfter`、`loadBefore` 和 `incompatibleWith`。`loadAfter` 只排序，不声明必需依赖。不要列出所有仅仅“可能相关”的 Mod。

## 调查清单

- 真正支持哪些游戏版本？
- 共用一个程序集还是按版本构建？
- 哪些 DLC 必需或可选？
- 哪些依赖分别属于编译期、加载期和功能期？
- Harmony 和框架库由依赖提供还是自行打包？
- `LoadFolders.xml` 是否保证依赖缺失时加载器看不到可选类型？
- 干净检出能否使用有文档的本地路径构建？
- 包内是否有仅源码、陈旧、调试或重复二进制文件？

证据优先级：精确本地事实、官方游戏/Mod 文档、仓库源码、打包产物。社区报告用于寻找复现线索，不作为因果证明。

