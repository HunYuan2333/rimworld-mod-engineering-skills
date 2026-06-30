---
name: rimworld-mod-engineering
description: Use when writing, reviewing, or debugging RimWorld mod code (C#, XML Defs, Harmony patches). Use when mods have performance problems (UI stutter, memory leak, lag), mod compatibility issues, or Harmony patches behave unexpectedly. Use when the user is working on a RimWorld mod project, mentions RimWorld/RW mod development, or asks about mod performance, mod conflicts, Def design, or Harmony patching.
---

# RimWorld Mod 开发指南

## 这是什么

一份给 RimWorld Mod 作者的软件工程生存指南。你不需要计算机学位——只需要知道**什么会出问题、为什么、怎么避免**。

每条规则都配有环世界的类比。先讲"会发生什么灾难"，再讲"怎么预防"。

**核心理念：Mod 不是一次性的脚本。你要维护它、别人要兼容它、玩家要在几十个 Mod 一起跑的环境里用它。**

---

## 何时使用

**自动触发场景：**
- 你在写新的 Mod 代码（C#/XML/Harmony）
- 你在审查 Mod 代码
- Mod 出现性能问题（UI 卡顿、内存一直涨）
- 多个 Mod 一起跑就出问题
- Harmony 补丁行为异常

**你应该主动调用的场景：**
- 新开一个 Mod 项目，不知道从哪里开始
- 某个 Bug 修了三次还没好——这时候该反思架构了
- "我的代码能跑了"但不确定"写得对不对"

---

## 六条铁律

违反任意一条，Mod 会在某个时刻以奇怪的方式爆炸。可能不是现在——可能在你装了另一个 Mod、开了第 10 个存档、或者玩家玩了 3 小时后。

---

### 铁律 #1：Draw 路径上别 new 东西

RimWorld 用 IMGUI（立即模式 GUI）。你的 `DoWindowContents` / `Draw` 方法**每帧执行一次**（60fps = 每秒 60 次）。

每帧 new 对象 → GC 积累 → 帧率间歇性暴跌。玩家感受："你的 Mod 好卡。"

```csharp
// ❌ 每帧都在分配内存——灾难
public void Draw(Rect inRect)
{
    var regex = new Regex(@"<[^>]+>");          // 每帧 new
    var content = new GUIContent("hello");       // 每帧 new
    var filtered = messages.Where(m => m.IsNew)  // LINQ 分配枚举器
                           .ToList();            // 又分配
}

// ✅ 正确：缓存 + 脏标记
private static readonly Regex TagRegex =
    new Regex(@"<[^>]+>", RegexOptions.Compiled); // 只编译一次
private GUIContent _cachedContent;
private List<Message> _cachedMessages = new();
private bool _dirty = true;                       // 脏标记

public void Draw(Rect inRect)
{
    if (_dirty) { UpdateCache(); _dirty = false; }
    // 用缓存的数据，零分配
}
```

> **详细指南**：[imgui-performance.md](imgui-performance.md) — 什么会分配内存、怎么查、怎么修。

---

### 铁律 #2：别硬引用其他 Mod 的类型

你的 Mod 直接 `new OtherMod.SomeClass()` — 别人的 Mod 更新/移除/改名 → 你的 Mod 直接崩溃。

```csharp
// ❌ 硬依赖——对方 Mod 不在或者更新了，你的就炸
var trader = new AwesomeTradingMod.TradingPost();

// ❌ 同样糟糕——写死 Mod 名字做判断
if (otherMod.Name == "AwesomeTrading") { ... }
```

```csharp
// ✅ 通过反射检测 Mod 是否存在
private bool _hasAwesomeTrading;
private Type _tradingPostType;

public void Init()
{
    var mod = LoadedModManager.RunningMods
        .FirstOrDefault(m => m.PackageId == "author.awesomeTrading");
    if (mod != null)
    {
        _hasAwesomeTrading = true;
        _tradingPostType = mod.assemblies
            .SelectMany(a => a.GetTypes())
            .FirstOrDefault(t => t.Name == "TradingPost");
    }
}

// 使用时检查
if (_hasAwesomeTrading)
{
    var post = Activator.CreateInstance(_tradingPostType);
    // ...
}
// else → 优雅降级，你的 Mod 继续正常工作
```

**原则**：你的 Mod 在别人不在时应该**失去附加功能**，而不是**直接崩溃**。

---

### 铁律 #3：+= 必须配 -=

C# 的事件订阅（`+=`）会阻止 GC 回收你的对象。每次进出存档/服务器，泄漏一点。累积到几百 MB → 游戏卡死。

```csharp
// ❌ 只订阅不取消——内存泄漏
public void OnGameStart()
{
    Find.TickManager.Tick += OnTick;  // 订阅
}
// 没有地方 -= ! 永远不会被回收

// ❌ 同样糟糕——用 lambda 订阅，永远没法取消
someEvent += (s, e) => DoStuff();

// ✅ 正确：缓存委托引用，配对联消
private TickHandler _tickHandler;

public void OnGameStart()
{
    _tickHandler = (sender, args) => DoTickWork();  // 缓存引用
    Find.TickManager.Tick += _tickHandler;
}

public void OnGameEnd()
{
    if (_tickHandler != null)
        Find.TickManager.Tick -= _tickHandler;  // 配对取消
}
```

**规则**：你在 `OnGameStart`/`Activate`/构造函数里每写一个 `+=`，就必须在对应的 `OnGameEnd`/`Shutdown`/`Dispose` 里写对应的 `-=`。

有 `Timer`、`FileStream`、`Thread` 的类？必须实现 `IDisposable`。

---

### 铁律 #4：别空吞异常

空 `catch {}` 是你在家里埋地雷——出问题了没人知道，直到某天整个房子塌了。

```csharp
// ❌ 最危险的行为——把问题完全藏起来
try { DoSomethingRisky(); }
catch { }  // 什么都没留下

// ❌ 仍然糟糕——只记 Message，扔掉堆栈
try { DoSomethingRisky(); }
catch (Exception ex) { Log.Message(ex.Message); }  // 堆栈丢了！

// ✅ 正确：完整记录 + 不让一条消息拖垮全部
try { ProcessOneMessage(msg); }
catch (Exception ex)
{
    Log.Error($"处理消息失败: {ex}");  // 完整堆栈
    // 继续处理下一条——不要让一个坏消息阻塞整个管线
}
```

**一口气处理一批东西时**：一条失败就跳过，继续处理其他的。100 条消息坏 1 条 → 跳过那 1 条，不是丢掉 99 条。

**网络断开时**：立刻清理相关状态。别等定时器兜底——它可能永远不会触发。

---

### 铁律 #5：网络线程别碰 UI

RimWorld 是单线程游戏——所有 Unity API 必须在主线程调用。网络回调跑在别的线程上。

```csharp
// ❌ 网络线程上直接操作 UI ——随机崩溃，极难复现
public void OnNetworkMessage(byte[] data)
{
    _myWindow.title = ParseTitle(data);  // 不是主线程！竞态崩溃！
    Find.WindowStack.Add(newDialog);      // 同样致命
}

// ✅ 正确：封送到主线程
public void OnNetworkMessage(byte[] data)
{
    var title = ParseTitle(data);  // 数据解析可以在网络线程
    // 所有 UI 操作必须封送到主线程
    LongEventHandler.ExecuteWhenFinished(() =>
    {
        _myWindow.title = title;
    });
}
```

**适用场景**：任何 `[HarmonyPatch]` 的补丁方法、网络回调、后台线程——只要不是 Unity 主循环触发的，操作 UI 前都要检查。

> 多线程（`Thread`、`Task`）读写共享集合（`List`、`Dictionary`）必须加 `lock` 或改用 `ConcurrentDictionary`。

---

### 铁律 #6：新增功能不改旧代码

好的架构像 USB 接口：新增一个设备不需要拆开电脑重焊主板。

```csharp
// ❌ 每次新增功能都要修改核心方法
public void ProcessMessage(Packet pkt)
{
    if (pkt.Type == "chat") HandleChat(pkt);
    else if (pkt.Type == "trade") HandleTrade(pkt);
    else if (pkt.Type == "mail") HandleMail(pkt);      // 你新加的
    else if (pkt.Type == "auction") HandleAuction(pkt); // 又要改这里
    // 每次新增功能都要改这里 → 改多了会出错 → 你的代码是别人的崩溃源
}

// ✅ 正确：用接口定义挂载点
public interface IMessageHandler
{
    bool CanHandle(string type);
    void Handle(Packet pkt);
}

// 新增功能 = 新增类，不改核心
public class MailHandler : IMessageHandler { ... }
public class AuctionHandler : IMessageHandler { ... }
```

**判断标准**：新增一个完整功能（比如给 Mod 加个"邮件系统"），需要改动已有代码的行数是多少？理想答案是 **0**。

---

## 快速自查表

出问题时对照这张表：

| 症状 | 最可能的原因 | 深入阅读 |
|------|-------------|----------|
| UI 间歇性卡顿 | Draw 路径上分配了对象 | [imgui-performance.md](imgui-performance.md) |
| 进出几次就卡死 | 事件泄漏 / IDisposable 没实现 | [csharp-pitfalls.md](csharp-pitfalls.md) |
| 装了某个 Mod 后崩溃 | 硬引用了对方类型 | 铁律 #2（上方） |
| 偶发崩溃，很难复现 | 网络线程操作 UI / 集合没锁 | 铁律 #5（上方） |
| 修了三次还没好 | 架构问题，不是代码问题 | [se-principles.md](se-principles.md) |
| Mod 之间互相干扰 | 补丁优先级冲突 / Def 名冲突 | [harmony-patching.md](harmony-patching.md) |
| foreach 后内存暴涨 | 老 Mono 上 foreach 分配迭代器 | [csharp-pitfalls.md](csharp-pitfalls.md) |
| 存档越来越大 | ExposeData 写了太多东西 | [csharp-pitfalls.md](csharp-pitfalls.md) |

---

## 什么时候该反思架构

以下信号出现 **3 次以上**，说明你的软件结构有问题，不单是代码写错了：

- 修一个 Bug 引出了另一个 Bug
- 每次新增小功能都要改好几个文件
- 一个类超过了 300 行
- "我记不清这段逻辑是干什么的了"——你自己写的，三个月后
- 两个功能明明独立，但代码互相缠绕
- "重构来不及了，先加个 if 进去凑合"

凑合累积 = 技术债务。详见 [se-principles.md](se-principles.md)。

---

## 反模式速查

这是"一眼就知道有问题"的代码模式：

| 反模式 | 为什么危险 | 修复方向 |
|--------|-----------|----------|
| `catch { }` | 问题被完全隐藏 | 记录完整异常日志 |
| `new Regex(...)` 不在 static 里 | 每帧编译正则 | `static readonly` + `RegexOptions.Compiled` |
| `messages.Where().ToList()` 在 Draw 里 | 每帧分配枚举器 + 新列表 | 缓存 + 脏标记 |
| `otherMod.SomeConcreteClass.DoSomething()` | 硬依赖别人的实现 | 反射 + 接口抽象 |
| `someEvent += (s, e) => ...` | lambda 无法取消订阅 | 缓存为命名委托 |
| `Thread.Sleep(1000)` 在网络回调里 | 卡住网络线程 | 异步 + 条件等待 |
| `Dictionary<T, U>` 被多线程读写 | 数据损坏崩溃 | `ConcurrentDictionary` 或 `lock` |
| 类的字段超过 15 个 | 职责太多 | 拆成多个类 |

---

## 文件导航

本 Skill 包含以下重型参考文件，按需加载：

| 文件 | 内容 | 何时读 |
|------|------|--------|
| [imgui-performance.md](imgui-performance.md) | IMGUI 每帧零分配技术细节 | UI 卡顿 / 写新窗口 |
| [csharp-pitfalls.md](csharp-pitfalls.md) | C# + 老 Mono 的陷阱集合 | 奇怪行为 / 内存问题 |
| [harmony-patching.md](harmony-patching.md) | Harmony 补丁最佳实践 | 写补丁 / 补丁冲突 |
| [se-principles.md](se-principles.md) | 软件工程原则（环世界类比版） | 设计新功能 / 重构 |
| [design-patterns.md](design-patterns.md) | 常用设计模式 + Mod 场景 | 架构设计 / 代码组织 |
| [project-structure.md](project-structure.md) | 项目目录结构 + 分层模型 + 何时拆分 | 新项目 / 代码失控 / 多 DLL 组织 |

---

## 写给完全没学过 C# 的作者

如果你是因为想给 RimWorld 做 Mod 才开始学代码的：

1. **C# 的 `new` 是有代价的**。在普通程序里几乎不计，但在 IMGUI 的 Draw 方法里——它每帧跑 60 次——就成了灾难。这是本指南最频繁提到的坑，没有之一。

2. **值类型 vs 引用类型**：`int`、`float`、`struct` 是值类型——直接放在栈上，用完就扔。`class`、`string`、`List` 是引用类型——分配在堆上，GC 要专门回收。**IMGUI 热路径上尽量用值类型**。

3. **事件 (`event`) = 广播系统**。你订阅 (`+=`) 就是告诉广播站"有消息通知我"。你不取消 (`-=`) 就是搬家了还让广播站给你寄报纸——报纸堆在老房子门口，越堆越多。这就是内存泄漏。

4. **接口 (`interface`) = 合同**。`interface IWeapon { void Attack(); }` 的意思是"我不管你怎么实现，你保证有 Attack 这个方法"。依赖接口而不是具体类 = 你的 Mod 不关心对方是谁，只关心对方有没有你需要的方法。

> [csharp-pitfalls.md](csharp-pitfalls.md) 里有更完整的 C# 陷阱清单，每条都有通俗解释。
