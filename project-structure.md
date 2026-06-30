# 项目结构：从单文件到分层架构

> 每个 Mod 都从 `Mod.cs` 开始。问题不是"什么时候要做架构"——是"代码什么时候开始咬你"。

---

## 目录
- [单文件陷阱](#单文件陷阱)
- [三种规模三种结构](#三种规模三种结构)
- [分层模型](#分层模型)
- [Contracts 模式：把接口拆出来](#contracts-模式把接口拆出来)
- [何时拆分](#何时拆分)
- [命名空间约定](#命名空间约定)
- [多 DLL 组织（大型 Mod）](#多-dll-组织大型-mod)
- [从混沌到有序的重构路线](#从混沌到有序的重构路线)

---

## 单文件陷阱

最开始的 Mod 长这样：

```
MyMod/
  Source/
    Mod.cs         ← 所有逻辑在这儿
    HarmonyPatch.cs ← 所有补丁在这儿
  Defs/
    MyThings.xml    ← 所有 Def 在这儿
```

300 行时还行。800 行时开始吃力。1500 行时你害怕改任何东西。

**信号**：你在 `Mod.cs` 里用 `#region` 分段——说明你已经在心理上拆分了，只是物理上还没拆。

---

## 三种规模三种结构

### 小型 Mod（<500 行代码，单一功能）

```
MyMod/
  About/
  Defs/                    ← XML Def 全在这
  Languages/               ← 翻译
  Source/
    Mod.cs                 ← 入口 + 设置
    HarmonyPatches.cs      ← 所有补丁
    MyFeature.cs           ← 核心逻辑
  Textures/
```

**规则**：逻辑全在一个 DLL 里，但至少把"入口""补丁""核心逻辑"三个文件分开。

---

### 中型 Mod（500-2000 行，多个功能模块）

```
MyMod/
  About/
  Defs/
    Things/                ← 按类型分目录
    Recipes/
    ResearchProjects/
  Languages/
  Source/
    Mod.cs                 ← 入口：只做初始化，不写业务逻辑
    Settings.cs            ← 设置管理
    Harmony/
      ThingPatches.cs      ← 按补丁目标分类
      PawnPatches.cs
      GUIPatches.cs
    UI/
      MainWindow.cs        ← 每个窗口一个文件
      SettingsWindow.cs
    Core/
      MyFeatureManager.cs  ← 业务逻辑
      DataStore.cs         ← 数据/存档
    Utils/
      Helpers.cs           ← 工具方法
  Textures/
```

**核心规则**：`Mod.cs` 不超过 100 行——只做初始化，不写业务逻辑。

---

### 大型 Mod（2000+ 行，多子系统，可能有插件系统）

```
MyMod/
  About/
  Defs/
  Languages/
  Contracts/                  ← 接口层（公共契约）
    IMyFeatureApi.cs
    IModEventHandler.cs
    DataContracts/
      MyMessageTypes.cs        ← 协议常量
  Source/
    Mod.cs                     ← 入口：初始化 + 组装
    Settings.cs
    Framework/
      ExtensionRegistry.cs     ← 扩展发现（如果有插件系统）
      ServiceLocator.cs        ← 服务定位
    Modules/                    ← 每个业务模块一个子目录
      Chat/
        ChatUI.cs
        ChatMessageHandler.cs
        ChatSettingsPanel.cs
      Trade/
        TradeUI.cs
        TradeProcessor.cs
        TradeNetworkHandler.cs
    Shared/
      NetworkManager.cs
      SaveManager.cs
      UI/
        ThemeManager.cs
        WidgetHelpers.cs
    Utils/
  Textures/
```

**关键特征**：`Contracts/` 目录物理隔离了接口和实现。

---

## 分层模型

你的代码有天然的依赖方向。让这个方向**单向且明确**：

```
┌────────────────────────────────────┐
│  Modules (Chat, Trade, 你的业务)     │  ← 业务层
├────────────────────────────────────┤
│  Contracts (接口定义, 协议常量)      │  ← 契约层
├────────────────────────────────────┤
│  Framework (Mod.cs, 服务管理, 网络)  │  ← 宿主层
├────────────────────────────────────┤
│  Utils (工具类, 扩展方法, 常量)      │  ← 基础设施层
└────────────────────────────────────┘
```

**铁律**：上层可以依赖下层，下层**绝不**反向依赖上层。

```
Modules → Contracts → Framework → Utils   ✅
Utils → Framework → Modules               ❌ 永远别这样
```

**具体到 C# 的引用规则**：

| 你在哪一层 | 可以引用 | 禁止引用 |
|-----------|----------|----------|
| Modules | Contracts, Framework, Utils, 其他模块的 Contracts | 其他模块的 Source/ 内部实现 |
| Contracts | 无（纯接口） | 任何实现代码 |
| Framework | Utils | Modules（宿主不知道插件存在） |
| Utils | 无（纯工具） | Framework, Modules |

**环世界类比**：地基（Utils）不知道墙壁（Framework）怎么建的，墙壁不知道房间里的人在干什么（Modules）。

---

## Contracts 模式：把接口拆出来

这是最重要的架构决策之一。灵感来自 [Phinix 的 ClientExtensionAbstractions 层](https://github.com/hunyuan2333/Phinix-Rework)。

**问题**：你的聊天模块暴露了 `IChatApi`，交易模块想调用它。如果交易模块直接引用聊天模块的 DLL → 硬耦合。哪天聊天模块内部改了，交易模块编译失败。

**方案**：把接口定义单独放一个目录/程序集。

```
MyMod/
  Contracts/
    IChatApi.cs           ← 只有接口定义，没有实现
    ITradeApi.cs
    ProtocolConstants.cs   ← 消息类型常量
  Source/
    Modules/
      Chat/
        ChatModule.cs      ← 实现 IChatApi
      Trade/
        TradeModule.cs     ← 引用 Contracts/IChatApi
```

```csharp
// Contracts/IChatApi.cs —— 纯接口
namespace MyMod.Contracts
{
    public interface IChatApi
    {
        void SendMessage(string target, string message);
        event Action<ChatMessage> MessageReceived;
    }
}

// Source/Modules/Chat/ChatModule.cs —— 实现
namespace MyMod.Chat
{
    public class ChatModule : IChatApi
    {
        public void SendMessage(string target, string message) { ... }
        public event Action<ChatMessage> MessageReceived;
    }
}

// Source/Modules/Trade/TradeModule.cs —— 消费接口
// TradeModule 只引用 Contracts/，不引用 Chat 的实现
namespace MyMod.Trade
{
    public class TradeModule
    {
        private readonly IChatApi _chatApi;
        public TradeModule(IChatApi chatApi)  // 依赖注入
        {
            _chatApi = chatApi;
        }
    }
}
```

**好处**：
- Chat 模块的内部重构不影响 Trade 的编译
- 第三方 Submod 可以安全引用你的 Contracts 而不依赖你的实现
- 你的模块可以在不破坏下游的前提下任意演化

---

## 何时拆分

| 信号 | 行动 |
|------|------|
| 文件超过 300 行 | 拆成 2-3 个文件 |
| 一个类有 15+ 个字段 | 找出内聚的子集，拆出新类 |
| 一个方法超过 50 行 | 提取子方法 |
| 你用 `#region` 把文件分成几块 | 每块拆成独立文件 |
| 两个类总是同时被改动 | 它们可能应该合并 |
| 两个类"独立"但代码互相 import | 它们实际上不独立——理清依赖方向 |
| `Mod.cs` 里有业务逻辑 | 移到对应模块 |
| 多个模块用同样的数据结构 | 提取到 Shared/ 或 Contracts/ |
| `Utils.cs` 超过 200 行 | 按职责分组（StringUtils, MathUtils, ReflectionUtils） |

**什么时候不急**：200 行以内的类、职责清晰的单文件——不需要拆。拆分本身有代价，别为了"看起来整齐"而过度拆分。

---

## 命名空间约定

让命名空间反映目录结构——这样任何人（包括 AI）都能从文件路径推测出命名空间。

```
Source/UI/MainWindow.cs          → namespace MyMod.UI
Source/Modules/Chat/ChatUI.cs    → namespace MyMod.Chat
Contracts/IChatApi.cs            → namespace MyMod.Contracts
```

```csharp
// ❌ 所有东西同一个命名空间——失去了组织力
namespace MyMod { ... }  // 50 个类全挤在一起

// ✅ 命名空间跟随功能模块
namespace MyMod.Chat { ... }
namespace MyMod.Trade { ... }
namespace MyMod.UI { ... }
namespace MyMod.Contracts { ... }
namespace MyMod.Utils { ... }
```

---

## 多 DLL 组织（大型 Mod）

当 Mod 包含多个 DLL（框架 DLL + 插件 DLL）时：

```
MyMod/
  Common/
    Assemblies/
      01-MyModUtils.dll             ← 基础设施
      02-MyModContracts.dll          ← 契约层
      03-MyModFramework.dll          ← 宿主层
    Extensions/
      10-MyModChat.dll               ← 插件（依赖 01-03）
      11-MyModTrade.dll              ← 插件（依赖 01-03）
      12-ThirdPartySubmod.dll        ← 第三方插件
```

**编号规则**：RimWorld 按文件名字符串序加载 DLL。数字前缀必须确保**依赖项先于被依赖项加载**：

```
01-Utils.dll           ← 无依赖
02-Contracts.dll       ← 可能依赖 01
03-Framework.dll       ← 依赖 01, 02
10-Chat.dll            ← 依赖 01, 02, 03
11-Trade.dll           ← 依赖 01, 02, 03
12-ThirdParty.dll      ← 依赖 01, 02, 03（可能还依赖 10）
```

**数字前缀规则**：你的 DLL 前缀必须大于它所依赖的所有 DLL 的前缀。第三方 Submod 从 12 开始分配。

---

## 物理 vs 逻辑：代码放在哪不代表它属于哪

一个容易踩的坑：物理位置 ≠ 编译归属。

```
❌ 把所有 .cs 文件放在 Source/ 下 — 但在 .csproj 里用 Compile Remove/Include 控制归属
   → 后来没人知道哪个文件属于哪个 DLL
   → 添加新文件时不知道该放哪、该被谁编译

✅ 物理目录直接对应编译归属
   Source/Framework/  → 被 Framework.csproj 编译
   Source/Chat/       → 被 ChatModule.csproj 编译
   Contracts/         → 被 Contracts.csproj 编译
```

**原则**：一个人 clone 了你的仓库，只看目录结构就应该能画出依赖图。不需要去读 `.csproj`。

---

## 从混沌到有序的重构路线

如果你已经有一个 3000 行的 `Mod.cs`，不要一次拆完。分步走：

```
第 1 步：把 Utils 方法抽到 Utils/ —— 风险最低，不改变任何行为
         → 编译通过 → 测试通过 → commit

第 2 步：把 UI 代码抽到 UI/ —— 每个窗口一个文件
         → 编译通过 → 测试通过 → commit

第 3 步：把 Harmony 补丁按目标类拆分
         → 编译通过 → 测试通过 → commit

第 4 步：识别业务模块边界（聊天/交易/等），抽到 Modules/
         → 编译通过 → 测试通过 → commit

第 5 步：如果模块之间有互相引用，提取 Contracts/
         → 编译通过 → 测试通过 → commit
```

每步做完都能跑。不要跳到第 5 步才第一次编译。增量重构——和增量开发一样的纪律。

---

## 快速清单

检查你的 Mod 项目结构：

- [ ] `Mod.cs` 是否不超过 100 行？（只做初始化）
- [ ] 有没有单个文件超过 300 行？
- [ ] 有没有 `#region` 分段的文件？（该拆了）
- [ ] UI 代码和业务逻辑是否在同一个文件里？
- [ ] 多个模块是否在同一个命名空间里混着？
- [ ] 有没有模块直接引用其他模块的实现类（而非接口）？
- [ ] 如果有多个 DLL，编号是否保证加载顺序？
- [ ] 第三方能不能不看你的实现代码就知道怎么调用你？
- [ ] 物理目录结构是否反映了逻辑架构？
- [ ] 能不能一句话说清楚"这个目录里放的是什么"？
