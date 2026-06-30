# 软件工程原则（环世界类比版）

> 每条原则用环世界的概念重新解释。不需要计算机学位——你玩过环世界就足够了。

---

## 目录
- [SOLID 五原则](#solid-五原则)
- [高内聚低耦合](#高内聚低耦合)
- [DRY — 别重复自己](#dry--别重复自己)
- [KISS — 保持简单](#kiss--保持简单)
- [YAGNI — 别提前做](#yagni--别提前做)
- [组合优于继承](#组合优于继承)
- [关注点分离](#关注点分离)
- [最少知识原则](#最少知识原则)
- [防御式编程](#防御式编程)
- [快速失败](#快速失败)
- [增量更新](#增量更新)
- [技术债务](#技术债务)
- [何时反思架构](#何时反思架构)
- [重构纪律](#重构纪律)

---

## SOLID 五原则

### S — 单一职责：一个工作台只做一种东西

屠宰台屠宰，烹饪台做饭，研究台研究。你不会把三种功能塞进一个工作台——你的类也一样。

```csharp
// ❌ 一个类干了所有事（"上帝类"）
public class ModCore
{
    public void DrawUI() { ... }        // UI
    public void SendNetwork() { ... }   // 网络
    public void SaveData() { ... }      // 存储
    public void ProcessTrade() { ... }  // 交易逻辑
    public void PlaySound() { ... }     // 音效
}
// 3000 行后：没人知道这玩意到底在干什么

// ✅ 每个类只负责一件事
public class TradeWindow { ... }        // 只负责交易 UI
public class NetworkSender { ... }      // 只负责发消息
public class SaveManager { ... }        // 只负责读写存档
public class TradeProcessor { ... }     // 只负责交易逻辑
public class SoundManager { ... }       // 只负责音效
```

**判断法**：能用一句话说清楚这个类是干什么的吗？"这个类负责交易窗口的 UI 渲染"——单一职责。"这个类负责交易、音效、保存和网络通信"——该拆了。

---

### O — 开闭原则：想给突击步枪加配件，不用拆开枪本身

**对扩展开放，对修改关闭**。翻译：想要新功能时，写新代码，不改旧代码。

```csharp
// ❌ 每加一种新武器就得改核心方法
public float CalculateDamage(Weapon w)
{
    if (w.type == "rifle") return 15 * w.quality;
    if (w.type == "pistol") return 10 * w.quality;
    if (w.type == "shotgun") return 20 * w.quality;
    // 要加新武器？来改这里吧——迟早改出事
}

// ✅ 让每种武器自己算伤害
public interface IWeapon
{
    float GetDamage();
}
public class Rifle : IWeapon
{
    public float GetDamage() => 15 * QualityMultiplier;
}
public class Shotgun : IWeapon
{
    public float GetDamage() => 20 * QualityMultiplier;
}
// 新武器 = 新建一个类 = 不动已有代码
```

**判断标准**：新增一个业务能力，需要改动多少已有代码行？理想答案是 0。

---

### L — 里氏替换：所有肉都能喂动物

子类必须能无缝替换父类。简单肉、奢侈肉、人肉——只要是肉，喂动物的行为就应该一致。

```csharp
// ❌ 子类抛出父类没有的异常——调用的代码完全没预料到
public class Meat { public virtual int Nutrition => 5; }

public class HumanMeat : Meat
{
    public override int Nutrition =>
        throw new Exception("不能吃人肉！");  // 炸了！
}
// 任何代码拿到一个 Meat 并访问 Nutrition 都可能崩溃

// ✅ 子类行为不违背父类契约
public class HumanMeat : Meat
{
    public override int Nutrition => 5;  // 还是肉，照常返回营养值
    public bool IsHumanMeat => true;     // 额外信息放新属性
}
```

**判断法**：如果你的子类让调用方必须 `if (x is SomeSubclass)` 才能正常工作，就违反了 L。

---

### I — 接口隔离：小人不需要科研技能就不会去研究台

别强迫别人依赖他用不上的东西。接口要小而精。

```csharp
// ❌ 一个巨大的接口——实现者被迫处理不相关的东西
public interface IColonistAbilities
{
    void Cook();
    void Research();
    void Fight();
    void Hunt();
    void Mine();
    void Heal();
    void Build();
} // 一个只会做饭的小人被迫实现 Fight/Mine/Heal ——荒谬

// ✅ 拆成小接口——按需组合
public interface ICook { void Cook(); }
public interface IFight { void Fight(); }
public interface IHeal { void Heal(); }

public class ChefColonist : ICook { ... }                // 厨师只实现做饭
public class CombatMedic : IFight, IHeal { ... }         // 战地医生实现战斗+治疗
```

**判断法**：你的接口里有至少一个方法，某个实现类对它只抛 `NotImplementedException`？该拆了。

---

### D — 依赖倒置：依赖"武器"这个概念，不依赖"泵动霰弹枪"

依赖抽象（接口），不依赖具体（某个特定武器）。这样一来，你换武器不需要重写所有相关代码。

```csharp
// ❌ 猎人直接依赖特定武器
public class Hunter
{
    private PumpShotgun _gun = new PumpShotgun();  // 焊死了
    public void Hunt(Animal target)
    {
        _gun.Fire(target);
    }
}
// 想让猎人用步枪？改 Hunter 类的代码。想让他用弓？再改。每次都改。

// ✅ 依赖"武器"接口——什么武器都能用
public class Hunter
{
    private IWeapon _weapon;
    public Hunter(IWeapon weapon) { _weapon = weapon; }  // 传入什么用什么
    public void Hunt(Animal target) { _weapon.Attack(target); }
}

// 使用
var hunter = new Hunter(new PumpShotgun());  // 今天用霰弹枪
var hunter = new Hunter(new Bow());          // 明天用弓——Hunter 类一行没改
```

**这就是为什么 Mod 兼容性的核心是"依赖接口不依赖实现"**。别人 Mod 的具体类可能不存在——但你依赖的接口你可以自己定义。

---

## 高内聚低耦合

**类比**：好 Mod 像瑞士军刀——每个工具独立但配合默契。

- **高内聚**：相关的东西放一起。交易的所有逻辑在交易模块里，聊天在聊天模块里。一个类不应该"一半做 UI 一半连数据库"。
- **低耦合**：模块之间通过明确的小接口通信，而不是互相翻对方的内部字段。

```csharp
// ❌ 高耦合——模块之间互相翻对方的内部
public class TradeModule
{
    public List<TradeItem> Items;  // 公开字段，谁都能改
}
public class UIModule
{
    void Draw()
    {
        // UI 直接翻交易模块的内部列表
        foreach (var item in tradeModule.Items)  // 耦合！
        {
            tradeModule.Items.Remove(item);  // 更糟——直接修改别人的数据！
        }
    }
}

// ✅ 低耦合——通过方法调用通信
public class TradeModule
{
    private List<TradeItem> _items;
    public IReadOnlyList<TradeItem> GetItems() => _items.AsReadOnly();
    public bool TryRemoveItem(string id, out TradeItem removed) { ... }
}
public class UIModule
{
    void Draw()
    {
        foreach (var item in tradeModule.GetItems())  // 只读访问
        {
            if (Button("删除"))
                tradeModule.TryRemoveItem(item.Id, out _);  // 通过正规渠道
        }
    }
}
```

**经验法则**：要改一个模块的功能，你需要同时改几个其他模块？改动越少越好。一个模块内部重构，其他模块完全不知道——这是低耦合。

---

## DRY — 别重复自己

**类比**：别在三个不同的地方写同样的营养膏配方。以后改配方时，三个地方都要改——而且你肯定会漏一个。

```csharp
// ❌ 同样的逻辑出现在三个地方
public class CookMealJob { float NutriPerIngredient = 0.5f; ... }
public class MealUI { float displayNutri = ingredientCount * 0.5f; ... }
public class TradeHelper { if (thing is Meal) return ingr * 0.5f; ... }
// 0.5f 散落在各处——要改成 0.6f？准备 grep + 逐个确认吧

// ✅ 单一来源
public static class NutritionConsts
{
    public const float NutritionPerIngredient = 0.5f;
}
// 所有地方引用 NutritionConsts.NutritionPerIngredient
// 改一次，处处生效
```

**但不绝对**：两个碰巧长得很像但逻辑不同的代码**不应该**强行合并。DRY 消除的是"同一逻辑的多份拷贝"，不是"两段看起来像的代码"。

---

## KISS — 保持简单

**类比**：一个 3x3 的小房间够用，就别造 15x15 的豪华大厅——还附带没必要的复杂温控系统。

```csharp
// ❌ 过度设计——需求只是"存个开关状态"
public interface ISettingStore { T Get<T>(string key); void Set<T>(string key, T val); }
public abstract class SettingStoreBase : ISettingStore { ... }
public class JsonSettingStore : SettingStoreBase { ... }
public class XmlSettingStore : SettingStoreBase { ... }
public class SettingStoreFactory { ... }
// 需求只是存几个 bool 和 string ——一个 Dictionary 足够了

// ✅ 最简单的可行方案
private Dictionary<string, object> _settings = new();
public T Get<T>(string key, T fallback = default) =>
    _settings.TryGetValue(key, out var v) ? (T)v : fallback;
public void Set<T>(string key, T val) => _settings[key] = val;
```

**判断法**：你的代码里的接口/抽象类/工厂类的数量，是不是超过了实际业务类的数量？如果是，你在为还没出现的需求写代码。

---

## YAGNI — 别提前做

**类比**：游戏初期别造 8 个空房间等着招人——先解决温饱。

```csharp
// ❌ "以后可能会需要"——但两年了从来没用过
public enum MessageType
{
    Chat, Trade, System, Mail, Auction, Quest,
    Event, Notification, Error, Warning, Debug,
    VoiceCall, VideoCall, ScreenShare  // 环世界 —— 视频通话？
}

// ✅ 只做当前需要的东西。需要时再加。
public enum MessageType
{
    Chat,    // 现在需要
    Trade,   // 现在需要
    System,  // 现在需要
    // 缺什么再加什么——加一个枚举值 20 秒，不会破坏任何东西
}
```

**YAGNI 不等于不做扩展点**。留个接口 (interface) 是好的——这是开闭原则。但不要提前实现那个接口的 5 种变体——没有人用。

---

## 组合优于继承

**类比**：给小人加能力用"装备"（组合），而不是搞十层继承 `Colonist → Worker → Cook → GourmetChef → ...`

```csharp
// ❌ 继承链爆炸——每种组合都要新建一个类
public class Colonist { }
public class WorkingColonist : Colonist { }
public class CookingColonist : WorkingColonist { }
public class FightingColonist : WorkingColonist { }
// 那"会做饭又会打仗的小人"呢？CookingFightingColonist？
// N 种能力 = 2^N 个类——组合爆炸

// ✅ 组合——能力作为组件附加
public class Colonist
{
    private List<IAbility> _abilities = new();
    public void AddAbility(IAbility a) => _abilities.Add(a);
    public T GetAbility<T>() where T : IAbility =>
        _abilities.OfType<T>().FirstOrDefault();
}
public interface IAbility { }
public class CookingAbility : IAbility { ... }
public class FightingAbility : IAbility { ... }

// 做个小人：会做饭 + 会打仗
var colonist = new Colonist();
colonist.AddAbility(new CookingAbility());
colonist.AddAbility(new FightingAbility());
// 任意组合，不需要新建类
```

**判断法**：你的类名里如果有 "And"（如 `CookingAndFightingColonist`），说明你在用继承处理组合——该换了。

---

## 关注点分离

**类比**：烹饪和战斗不在一个工作台上。你的 UI 代码和网络代码也不该混一起。

```csharp
// ❌ UI、网络、逻辑全在同一段代码里
public void DrawTradeWindow(Rect rect)
{
    // UI 渲染
    Widgets.Label(rect, "物品列表");
    // 网络请求（在 Draw 里！每帧都发！）
    var packet = new TradeRequestPacket();
    NetworkManager.Send(packet);
    // 业务逻辑
    foreach (var item in items) { CalculatePrice(item); }
    // 存档操作
    SaveManager.Save("latest_trade", tradeData);
}

// ✅ 各层分离
public class TradeUI { public void Draw(Rect r) { ... } }       // 只画 UI
public class TradeService { public void RequestTrade(...) { ... } } // 业务逻辑
public class NetworkService { public void Send(Packet p) { ... } }  // 网络通信
public class SaveService { public void Save(string k, object d) { ... } } // 持久化
```

---

## 最少知识原则（Law of Demeter）

**类比**：别翻别人的口袋。你只跟你的直接朋友说话，不需要知道朋友的朋友口袋里有什么。

```csharp
// ❌ 链式调用——穿透多层对象
var weaponOwnerName = pawn.equipment.primary.WeaponDef.label;
// pawn → equipment → primary → WeaponDef → label
// 任何一环是 null 就崩溃。任何一环的内部结构变了，这行就炸。

// ✅ 只跟直接朋友说话
var weaponOwnerName = pawn.GetEquippedWeaponName();
// pawn 内部怎么实现的不关我事，我只要名字
```

**好处**：pawn 的内部结构可以随意重构——只要 `GetEquippedWeaponName()` 还在，所有调用方不受任何影响。

---

## 防御式编程

**类比**：永远假设输入可能不合法——哪怕你觉得"这种情况不可能发生"。

```csharp
// ❌ "肯定不会是 null" ——下一秒就炸
public void EquipWeapon(Pawn pawn, Thing weapon)
{
    pawn.equipment.AddEquipment(weapon);  // weapon 可能是 null！
}

// ✅ 先检查，给出明确错误信息
public void EquipWeapon(Pawn pawn, Thing weapon)
{
    if (pawn == null) throw new ArgumentNullException(nameof(pawn));
    if (weapon == null) throw new ArgumentNullException(nameof(weapon));
    if (weapon.def == null)
    {
        Log.Warning($"武器 {weapon.Label} 的 def 为 null，跳过装备");
        return;
    }
    pawn.equipment.AddEquipment(weapon);
}
```

**核心观点**：早检查、早报错、给出明确信息。比"不知道在哪、不知道为什么崩溃"好一万倍。

---

## 快速失败

**类比**：在游戏加载时就报告 Mod 冲突——别让玩家玩了 5 小时才发现存档坏了。

```csharp
// ❌ 悄悄吞掉配置错误，用默认值继续——玩家不知道出了问题
public void ApplySettings(XmlNode config)
{
    int maxItems = 100;  // 默认
    try { maxItems = int.Parse(config["maxItems"].InnerText); }
    catch { /* 用了默认值，玩家完全不知情 */ }
    StartMod(maxItems);
}

// ✅ 配置不对立刻报告，附带修复建议
public void ApplySettings(XmlNode config)
{
    var node = config["maxItems"];
    if (node == null)
    {
        Log.Error("缺少 maxItems 设置！在 Mod 设置中添加 maxItems 项。使用默认值 100 继续。");
        maxItems = 100;
        return;
    }
    if (!int.TryParse(node.InnerText, out var maxItems) || maxItems <= 0)
    {
        Log.Error($"maxItems 值无效: '{node.InnerText}'。应为正整数。使用默认值 100 继续。");
        maxItems = 100;
        return;
    }
    Log.Message($"maxItems 设置为 {maxItems}");
    StartMod(maxItems);
}
```

---

## 增量更新

**类比**：一次只扩建一间房，保证小人不会睡在废墟里。不要一次性推平整个基地重来。

```
❌ 瀑布式：设计 3 个月 → 写代码 3 个月 → 测试 1 个月 → 发布。中途需求变了直接白做。

✅ 增量式：
  第 1 周：聊天窗口能显示文字（可运行）
  第 2 周：加上发送功能（可运行）
  第 3 周：加上表情（可运行）
  每周结束都是能跑的状态——随时可以停下来，随时可以发布
```

```csharp
// ❌ "一次全做完"——所有功能同时开发，3 个月后第一次编译，全是错
public class MegaMod
{
    // TODO: 实现聊天、交易、邮箱、拍卖行、排行榜...
    // 300 个文件同时改动
}

// ✅ 增量交付——每步都能验证
// Step 1: 聊天显示（能跑，可以测，可以发 alpha）
// Step 2: 聊天发送（能跑，可以测，可以发 beta）
// Step 3: 通知系统（能跑，可以测）
// 每一步都是稳定的——在稳定的基础上做下一步
```

**核心规则**：每步做完的状态都可以发布。永远不回退超过一步。

---

## 技术债务

**类比**：房间里的污渍——今天不擦，明天不擦，下个月整个地板变色，要铲掉重铺。

| 技术债务的形式 | 环世界类比 | 代价 |
|---------------|-----------|------|
| 没写注释的复杂逻辑 | 没标记的捕食动物——以后踩到就咬你 | 没人敢改 |
| 复制粘贴的代码 | 三个炉灶各贴一份食谱——改配方跑断腿 | 改一处漏三处 |
| `// TODO: fix later` | 屋顶漏了只放个桶——下次暴雨淹全家 | 越积越多 |
| 绕过的硬编码 | 用木头临时撑墙——哪天木头烂了墙塌 | 脆弱的依赖 |
| 没测试的核心逻辑 | 没测试防御塔就直接突袭——全灭了才知道不行 | 生产环境爆炸 |

```csharp
// ❌ 借债——"临时方案"成了永久方案
// TODO: 正式发布前换成配置表
string serverUrl = "http://192.168.1.5:8080";  // 硬编码！两年后还在这。

// ✅ 每次迭代还一点债
// 第 1 周先写成可读的
string serverUrl = GetServerUrl(); // 至少是个方法了

// 第 2 周加配置
string GetServerUrl() => LoadedModManager.GetMod<MyMod>().GetSettings<MySettings>().ServerUrl;

// 第 3 周加验证 + 回退
string GetServerUrl()
{
    var url = settings.ServerUrl;
    if (string.IsNullOrEmpty(url)) { Log.Warning("ServerUrl 未配置，使用默认"); return "localhost"; }
    return url;
}
```

**原则**：借债没问题——你知道自己在借。但借了要记下来，定个时间还。永远不还的"临时方案"就是项目腐烂的开始。

---

## 何时反思架构

当你发现以下信号 **3 个以上同时出现**，停下来。不要继续写代码。先思考结构：

| 信号 | 含义 |
|------|------|
| 修一个 Bug 引出新 Bug | 模块耦合太紧，改一处影响多处 |
| 加个小功能要改 5+ 个文件 | 没有挂载点/接口抽象不足 |
| 一个类超过 300 行 | 职责太多，拆 |
| "这段代码是干嘛的" ——你自己写的，三个月后 | 命名不清晰 / 逻辑太绕 |
| "重构来不及了，先加个 if" | 技术债务在复利增长 |
| 两个"独立"功能代码互相引用 | 实际上不独立——要么拆开，要么正式合并 |
| 想写测试但发现写不了——依赖太多 | 耦合太高，先解耦 |
| 改了 A 模块，B 模块的测试失败 | A 和 B 有隐藏的耦合 |

**此时应该做的事**：
1. 画一张图：哪些模块存在？之间怎么连接的？（纸和笔就行）
2. 找出"连接最密"的那个点——那是你的重构目标
3. 设计一个接口把连接最密的部分断开
4. 小步重构——每步保持可编译可运行

---

## 重构纪律

**类比**：给基地改布局——不是拆光重建，是一间一间搬，每搬一间确认小人还有地方睡觉。

```
重构三步：
1. 保证现有行为（有测试最好，没测试至少手动验证过）
2. 做最小改动
3. 验证所有东西还正常工作

❌ "顺便重构一下" + "顺便加个新功能" = 不知道是重构导致的 Bug 还是新功能导致的 Bug
✅ 重构一趟，只做重构，不改行为。新功能新开一趟。
```

```csharp
// ❌ 重构混搭新功能
public void ProcessMessage(Packet pkt)  // 原来叫 HandlePacket
{
    // 顺便把参数名改了，顺便把错误处理逻辑改了，顺便加了邮件功能
    if (pkt.Type == "mail") { ... }  // 新功能混在重构里
}

// ✅ 第一步：纯重构（改名、拆方法，行为完全不变）
public void ProcessMessage(Packet packet)  // 只改参数名
{
    HandlePacketInternal(packet);  // 只拆方法，逻辑一模一样
}
// 验证行为不变。commit。

// ✅ 第二步：加新功能
public void HandlePacketInternal(Packet packet)
{
    // 现在加邮件功能
    if (packet.Type == "mail") HandleMail(packet);
}
// 如果炸了——一定是新功能的问题，不可能是重构的问题
```

---

## 总结：什么时候用什么原则

| 场景 | 优先使用的原则 |
|------|-------------|
| 开始新功能 | KISS → 做最简单版本 → 增量交付 |
| 代码里有 3 处重复 | DRY → 提取单一来源 |
| 新增功能要改旧代码 | 开闭原则 → 设计接口/挂载点 |
| 类太长了看不懂 | 单一职责 → 拆 |
| 两个模块互相缠绕 | 高内聚低耦合 → 设计清晰边界 |
| 连着修了好几个 Bug | 停下来反思架构 → 可能是设计问题不是代码问题 |
| 代码能跑但感觉不对 | 看看是否违反 SOLID 中的某一条 |
| "以后可能需要" | YAGNI → 现在不做 |
| 有人要调你的 API | 接口隔离 → 小接口 |
| Mod 要支持第三方扩展 | 依赖倒置 → 依赖接口不依赖实现 |
| 临时方案凑合上线 | 记一笔技术债务 → 定还债时间 |
| 想改进代码结构 | 重构纪律 → 小步改，每步验证，不和加功能混一起 |
