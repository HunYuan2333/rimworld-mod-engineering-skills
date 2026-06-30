# 设计模式（环世界 Mod 版）

> 设计模式是"常见问题的标准解法"。你不必知道所有模式，但下面这 10 个在 Mod 开发中经常出现。每个模式配一个环世界类比和 Mod 场景。

---

## 目录
- [观察者模式](#观察者模式-observer) — "有消息通知我"
- [策略模式](#策略模式-strategy) — "不同情况不同做法"
- [工厂模式](#工厂模式-factory) — "给我造一个东西"
- [单例模式](#单例模式-singleton) — "全局只有一个 + 陷阱"
- [装饰器模式](#装饰器模式-decorator) — "不改变本体，只加一层皮"
- [命令模式](#命令模式-command) — "操作排队"
- [适配器模式](#适配器模式-adapter) — "让旧的能插新的插座"
- [外观模式](#外观模式-facade) — "简化复杂系统"
- [状态模式](#状态模式-state) — "不同心情不同行为"
- [模板方法](#模板方法-template-method) — "流程框架写好，细节你来填"

---

## 观察者模式 (Observer)

**环世界类比**：广播站。"有袭击！"——所有听到广播的小人各自反应。广播站不关心谁在听、怎么反应。

**Mod 场景**：Tick 事件、存档事件、网络消息到达——都是观察者模式。

```csharp
// ✅ RimWorld 中每天都会用到的模式
public class MyMod
{
    public void OnGameStart()
    {
        // 订阅：告诉广播站"有 Tick 时通知我"
        Find.TickManager.Tick += OnTick;
        // 订阅存档事件
        LongEventHandler.ExecuteWhenFinished(OnGameLoaded);
    }

    public void OnGameEnd()
    {
        // 取消订阅：告诉广播站"别再通知我了"
        Find.TickManager.Tick -= OnTick;  // 别忘！
    }

    private void OnTick()
    {
        // 收到通知，做自己的事
        if (Find.TickManager.TicksGame % 60 == 0)
            DoSomethingEverySecond();
    }
}
```

**自定义观察者**（让你的 Mod 能广播事件给其他 Mod）：

```csharp
public class MyEventBus
{
    // 定义事件
    public event Action<TradeCompletedEventArgs> TradeCompleted;

    // 触发通知
    public void CompleteTrade(TradeData data)
    {
        TradeCompleted?.Invoke(new TradeCompletedEventArgs(data));
    }
}

// 其他 Mod 订阅
myEventBus.TradeCompleted += (args) =>
{
    Log.Message($"交易完成: {args.Data.TotalValue} 银");
};
```

**要点**：+= 必须配 -=。用 lambda 订阅会导致无法取消——缓存为命名委托。

---

## 策略模式 (Strategy)

**环世界类比**：小人的"应对威胁"行为——看到敌人时，有的逃跑，有的进攻，有的装作没看见。同一种触发，不同策略。

**Mod 场景**：多种计算方式、多种排序规则、多种 AI 行为。

```csharp
// ✅ 不同武器不同伤害计算策略
public interface IDamageStrategy
{
    float CalculateDamage(float baseDamage, Pawn target);
}

public class ArmorPiercing : IDamageStrategy
{
    public float CalculateDamage(float dmg, Pawn target)
    {
        // 穿甲：忽略 50% 护甲
        return dmg * 0.5f + dmg * 0.5f * (1 - target.GetArmor() * 0.5f);
    }
}

public class AreaDamage : IDamageStrategy
{
    public float CalculateDamage(float dmg, Pawn target)
    {
        // 范围伤害：周围每个敌人 +10%
        var nearby = target.Map.GetPawnsAround(target.Position, 3);
        return dmg * (1 + nearby.Count * 0.1f);
    }
}

// 使用
public class Weapon
{
    private IDamageStrategy _strategy;
    public Weapon(IDamageStrategy strategy) { _strategy = strategy; }
    public float Hit(Pawn target) => _strategy.CalculateDamage(BaseDamage, target);
}

var rifle = new Weapon(new ArmorPiercing());
var grenade = new Weapon(new AreaDamage());
```

---

## 工厂模式 (Factory)

**环世界类比**：ThingDef 是环世界的工厂——你不直接 new 物品，你告诉 Def 系统"给我一个这东西"，它给你造好。

**Mod 场景**：创建复杂对象、读取配置生成实例。

```csharp
// ✅ 简单工厂——根据配置创建不同的东西
public static class WeaponFactory
{
    public static IWeapon Create(WeaponDef def)
    {
        return def.weaponClass switch
        {
            "Rifle" => new Rifle(def.baseDamage, def.range),
            "Shotgun" => new Shotgun(def.baseDamage, def.pelletCount),
            "Bow" => new Bow(def.baseDamage, def.drawTime),
            _ => throw new ArgumentException($"未知武器类型: {def.weaponClass}")
        };
    }
}

// 使用
var weapon = WeaponFactory.Create(ThingDef.Named("Gun_AssaultRifle"));
```

**要点**：工厂让调用方不需要知道"这个东西怎么构造"——只告诉工厂"我要什么"，工厂负责拼装。

---

## 单例模式 (Singleton)

**环世界类比**：游戏里只有一个 `Find.World`。单例 = 全局唯一。

**⚠️ 警告**：单例是 Mod 冲突的常见来源。除非确实全局唯一，否则用依赖注入。

```csharp
// ✅ 适用场景：Mod 的设置管理器——确实只需要一个
public class MyModSettings
{
    private static MyModSettings _instance;
    public static MyModSettings Instance =>
        _instance ??= LoadedModManager.GetMod<MyMod>().GetSettings<MyModSettings>();

    private MyModSettings() { }  // 私有构造函数——外部不能 new
}

// ✅ 更好的方式（避免静态状态）：通过依赖注入传递
public class MyWindow
{
    private readonly MyModSettings _settings;
    public MyWindow(MyModSettings settings)  // 传进来
    {
        _settings = settings;
    }
}
```

**单例常见问题**：其他 Mod 的状态残留、测试困难、隐式耦合。**当你觉得"这个东西应该是单例"时，先问自己"能不能传进去而不是全局拿？"**

---

## 装饰器模式 (Decorator)

**环世界类比**：给武器加配件——枪本身没变，但威力/射程/looks 全变了。一层层叠加，不改枪本身。

**Mod 场景**：Harmony 补丁本质就是装饰器——不改原方法代码，在它前后加逻辑。

```csharp
// ✅ 数据层面的装饰器——给消息加时间戳，不改消息本身
public interface IMessage
{
    string Content { get; }
}

public class ChatMessage : IMessage
{
    public string Content { get; }
    public ChatMessage(string content) { Content = content; }
}

// 装饰器：加时间戳
public class TimestampedMessage : IMessage
{
    private readonly IMessage _inner;
    public string Content => $"[{DateTime.Now:T}] {_inner.Content}";
    public TimestampedMessage(IMessage inner) { _inner = inner; }
}

// 装饰器：加发送者名字颜色
public class ColoredMessage : IMessage
{
    private readonly IMessage _inner;
    private readonly string _color;
    public string Content => $"<color={_color}>{_inner.Content}</color>";
    public ColoredMessage(IMessage inner, string color) { _inner = inner; _color = color; }
}

// 组合装饰器——像叠配件
var msg = new ChatMessage("你好");
var stamped = new TimestampedMessage(msg);
var colored = new ColoredMessage(stamped, "green");
// 输出: <color=green>[14:30:05] 你好</color>
```

---

## 命令模式 (Command)

**环世界类比**：小人的工作队列——"先去砍树，然后搬铁，最后做饭"。每个操作是一个命令对象，可以被排队、取消、重做。

**Mod 场景**：操作队列、Undo/Redo、网络指令队列。

```csharp
// ✅ 把操作封装成对象——可以排队、撤销
public interface ICommand
{
    void Execute();
    void Undo();
    string Description { get; }
}

public class PlaceBlueprintCommand : ICommand
{
    private readonly IntVec3 _pos;
    private readonly ThingDef _def;
    private Thing _built;

    public string Description => $"建造 {_def.label}";

    public void Execute()
    {
        _built = GenSpawn.Spawn(_def, _pos, Find.CurrentMap);
    }

    public void Undo()
    {
        if (_built != null) _built.Destroy();
    }
}

// 命令队列管理器
public class CommandManager
{
    private Stack<ICommand> _undoStack = new();
    private Queue<ICommand> _pendingCommands = new();

    public void Enqueue(ICommand cmd)
    {
        _pendingCommands.Enqueue(cmd);
    }

    public void ProcessQueue()
    {
        while (_pendingCommands.Count > 0)
        {
            var cmd = _pendingCommands.Dequeue();
            cmd.Execute();
            _undoStack.Push(cmd);
        }
    }

    public void Undo()
    {
        if (_undoStack.TryPop(out var cmd))
            cmd.Undo();
    }
}
```

---

## 适配器模式 (Adapter)

**环世界类比**：转换插头——你在国外用国内电器，需要一个转换头。电器本身不变，插座本身不变，转换头在中间翻译。

**Mod 场景**：新版 RimWorld 改了类名/方法名——写个适配器翻译，不改已有的 5000 行业务代码。

```csharp
// ✅ 旧版 Mod 的存储接口——新版本 RimWorld 改了存储方式
// 不改业务代码，加一个适配器

// 旧接口（你的 Mod 里一直用的）
public interface IModSaveSystem
{
    void SaveData(string key, byte[] data);
    byte[] LoadData(string key);
}

// 新版 RimWorld 的存储类（改名了，参数格式也变了）
// 叫 DataStore 而不是原来的 SaveManager
public class RimWorldV16DataStore
{
    public void Write(string category, string id, string json) { ... }
    public string Read(string category, string id) { ... }
}

// 适配器：翻译新 API 到旧接口
public class SaveSystemAdapter : IModSaveSystem
{
    private readonly RimWorldV16DataStore _store;

    public SaveSystemAdapter(RimWorldV16DataStore store) { _store = store; }

    public void SaveData(string key, byte[] data)
    {
        var json = Convert.ToBase64String(data);       // 格式转换
        _store.Write("MyMod", key, json);               // 调用新 API
    }

    public byte[] LoadData(string key)
    {
        var json = _store.Read("MyMod", key);
        return Convert.FromBase64String(json);          // 转换回来
    }
}

// 所有旧代码不用改——它们还是调用 IModSaveSystem
```

---

## 外观模式 (Facade)

**环世界类比**：工作台的操作按钮——你点一下"烹饪简单食物"，不需要知道里面涉及了多少个步骤（选食材→检查库存→计算营养值→生成物品→扣除材料...）。外观就是一个简化的门面。

**Mod 场景**：封装 RimWorld 的复杂 API，让你的代码只需要调用一两个方法。

```csharp
// ❌ 每次发起交易都要写 10 行
var packet = new TradePacket();
packet.senderId = Find.CurrentMap.PlayerColony.playerId;
packet.items = tradeWindow.SelectedItems.Select(i => i.ToProto()).ToList();
packet.timestamp = DateTimeOffset.UtcNow.ToUnixTimeSeconds();
var data = ProtoBuf.Serialize(packet);
networkManager.Send("TradeChannel", data);
Log.Message($"发送交易: {packet.items.Count} 件物品");

// ✅ 外观模式——一行调用
public class TradeFacade
{
    private NetworkManager _network;
    public TradeFacade(NetworkManager network) { _network = network; }

    public void SendTrade(IEnumerable<TradeItem> items)
    {
        var packet = BuildPacket(items);
        var data = ProtoBuf.Serialize(packet);
        _network.Send("TradeChannel", data);
        Log.Message($"发送交易: {packet.items.Count} 件物品");
    }

    private TradePacket BuildPacket(IEnumerable<TradeItem> items) { ... }
}

// 调用方只需要
tradeFacade.SendTrade(tradeWindow.SelectedItems);
```

**外观 vs 适配器**：适配器是让两个已有接口协作（翻译）。外观是给复杂系统提供一个简化的入口（隐藏复杂度）。

---

## 状态模式 (State)

**环世界类比**：小人的心理状态——"开心""压力大""崩溃边缘""狂暴"。同一个小人，不同状态下行为完全不同。

**Mod 场景**：连接状态（连接中/已连接/重连中/断开）、游戏阶段、Mod 工作模式。

```csharp
// ✅ 连接状态机——每种状态自己的行为
public interface IConnectionState
{
    void OnEnter(MyNetworkMod mod);
    void OnUpdate(MyNetworkMod mod);
    void OnExit(MyNetworkMod mod);
    string StatusText { get; }
}

public class ConnectingState : IConnectionState
{
    public string StatusText => "连接中...";
    public void OnEnter(MyNetworkMod mod) { mod.ShowSpinner(true); }
    public void OnUpdate(MyNetworkMod mod)
    {
        if (mod.TryConnect()) mod.SetState(new ConnectedState());
        else if (mod.RetryCount > 5) mod.SetState(new DisconnectedState("重试超限"));
    }
    public void OnExit(MyNetworkMod mod) { mod.ShowSpinner(false); }
}

public class ConnectedState : IConnectionState
{
    public string StatusText => "已连接";
    public void OnEnter(MyNetworkMod mod) { mod.LoadServerData(); }
    public void OnUpdate(MyNetworkMod mod)
    {
        if (!mod.IsConnected) mod.SetState(new ReconnectingState());
    }
    public void OnExit(MyNetworkMod mod) { mod.SaveSessionData(); }
}

// 切换状态
public class MyNetworkMod
{
    private IConnectionState _state;
    public void SetState(IConnectionState newState)
    {
        _state?.OnExit(this);
        _state = newState;
        _state.OnEnter(this);
    }
}
```

**好处**：每个状态是独立的类——改一个不影响其他。新增一种状态？新建一个类，不改已有逻辑。

---

## 模板方法 (Template Method)

**环世界类比**：烹饪流程是模板——"准备食材 → 加工 → 装盘"。具体怎么做每道菜不同（炒/煮/烤），但大流程都一样。

**Mod 场景**：处理不同类型消息/数据的基础流程相同，但具体步骤不同。

```csharp
// ✅ 基础流程在父类，具体步骤在子类
public abstract class RecipeProcessor
{
    // 模板方法——定义骨架
    public void Process()
    {
        ValidateIngredients();
        Prepare();
        Cook();
        Plate();
        CleanUp();
    }

    protected abstract void Prepare();   // 每种菜不同
    protected abstract void Cook();      // 每种菜不同

    private void ValidateIngredients() { ... }  // 所有菜一样
    private void Plate() { ... }                // 所有菜一样
    private void CleanUp() { ... }              // 所有菜一样
}

public class FineMeal : RecipeProcessor
{
    protected override void Prepare() { /* 切蔬菜 */ }
    protected override void Cook() { /* 中火煎 */ }
}

public class LavishMeal : RecipeProcessor
{
    protected override void Prepare() { /* 切肉+蔬菜+调味 */ }
    protected override void Cook() { /* 慢火炖 */ }
}
```

**要点**：模板方法的特点是——把不变的部分锁在父类里，只留必须变化的部分给子类。减少了"子类忘记调用 base 方法"这类错误。

---

## 快速选择指南

| 你遇到了 | 考虑用 |
|----------|--------|
| 多个地方需要知道同一件事发生了 | 观察者 |
| 同一种行为有不同实现方式 | 策略 |
| 创建对象的逻辑很复杂/多变 | 工厂 |
| 全局只能有一个实例 | 单例（但先想能不能不用） |
| 想不改原代码就加功能 | 装饰器 |
| 操作需要排队/撤销/重做 | 命令 |
| 新旧 API 不兼容，想隔离变化 | 适配器 |
| API 太复杂，想提供简单入口 | 外观 |
| 行为随内部状态变化 | 状态 |
| 流程骨架相同，步骤不同 | 模板方法 |

---

## 不要过度使用

一个刚入门的 Mod 不需要工厂+策略+装饰器全部上阵。你需要的是：

1. 先让代码能跑（KISS）
2. 发现重复/混乱/改不动（识别问题）
3. 再引入合适的模式（对症下药）

**设计模式是药——有症状才用，没症状别先吃。**
