# C# 陷阱 + 老 Mono 特有问题

> 如果你是因为 RimWorld Mod 才开始学 C#——这份清单是为你写的。每条陷阱包含：通俗解释、为什么坑、怎么绕。

---

## 值类型 vs 引用类型

**通俗解释**：值类型像"写在纸上直接给你的数字"。引用类型像"写了地址的纸条——你得按地址去找到真东西"。

```csharp
// 值类型：struct（int, float, bool, Vector3, Color, Rect...）
int a = 5;
int b = a;   // b 得到 a 的**副本**
b = 10;      // a 还是 5 ——各管各的

// 引用类型：class（string, List<T>, 你定义的大部分 class）
List<int> list1 = new List<int> { 1, 2, 3 };
List<int> list2 = list1;   // list2 和 list1 指向**同一个列表**
list2.Add(4);              // list1 里也多了 4 ！
```

**坑 #1 — struct 陷阱**：

```csharp
// ❌ struct 是值类型，修改副本不影响原值
public struct PawnData { public int Health; }

PawnData data = GetPawnData();
data.Health = 50;  // 改了副本！原值没变！

// ✅ 正确做法：重新赋值
PawnData data = GetPawnData();
data.Health = 50;
SetPawnData(data);  // 把修改后的副本写回去
```

**规则**：`struct` 变量被传递时总是复制。如果你改了 struct 的字段但没有"放回去"，修改就丢了。

**坑 #2 — class 的意外共享**：

```csharp
// ❌ 两个对象共享了同一个列表引用
public class Colonist
{
    public List<Skill> Skills { get; set; }
}

var colonist1 = new Colonist { Skills = new List<Skill>() };
var colonist2 = new Colonist { Skills = colonist1.Skills };  // 共享同一个列表！
colonist2.Skills.Add(new Skill("射击"));   // colonist1 也多了射击技能！
```

---

## 装箱拆箱

**通俗解释**：值类型放进引用类型的盒子（装箱），再从盒子里拿出来（拆箱）。这在老 Mono 上既慢又分配内存。

```csharp
// ❌ 装箱——每行都分配内存
int count = 42;
object box = count;                    // 装箱：值类型塞进 object
string text = $"数量: {count}";        // 字符串插值也可能触发装箱
var method = typeof(MyClass).GetMethod("DoStuff");
method.Invoke(instance, new object[] { count });  // 参数装箱！

// ✅ 避免
string text = $"数量: {count.ToString()}";  // 显式 ToString 避免一些装箱
// 反射调用尽量缓存 MethodInfo，避免每次 Invoke
```

**Mod 中的雷区**：`Dictionary<string, object>`——每个值类型放进去都装箱。考虑用泛型 `Dictionary<string, T>`。

---

## foreach vs for（老 Mono 特别重要）

**通俗解释**：老 Mono 上 `foreach` 在遍历 `List<T>` 时会在堆上分配一个迭代器对象。`for` 不会。

```csharp
// ❌ 老 Mono 上 foreach List<T>——分配迭代器
foreach (var msg in messageList)  // 分配了！
{
    ProcessMessage(msg);
}

// ✅ for——零分配
for (int i = 0; i < messageList.Count; i++)
{
    ProcessMessage(messageList[i]);
}
```

> **新版**.NET Core 已修复此问题，foreach 在 List<T> 和数组上不再分配。但 RimWorld 的 Unity 2019 用的老 Mono **仍然分配**。

**规则**：热路径（Draw/Update/回调）上用 `for`。冷路径上用 `foreach` 没问题。

**数组上的 foreach 在 Mono 上也不分配**——所以 `foreach (var c in someCharArray)` 是安全的。

---

## 委托分配

**通俗解释**：lambda `() => ...` 和匿名方法 `delegate { ... }` 会分配一个委托对象。

```csharp
// ❌ 每帧分配委托
public void Draw(Rect inRect)
{
    if (Widgets.ButtonText(rect, "确定", () => OnConfirm()))  // 每帧 new 委托！
    { }
    // 传 lambda 给任何方法——分配
}

// ✅ 缓存委托
private static readonly Action _onConfirm = () => OnConfirm();  // 全局一次

public void Draw(Rect inRect)
{
    if (Widgets.ButtonText(rect, "确定", _onConfirm))  // 复用，不分配
    { }
}
```

**需要特别注意的**：事件订阅如果用 lambda，**无法取消订阅**。

```csharp
// ❌ 致命：lambda 导致无法取消订阅
someEvent += (s, e) => DoStuff();  // 这个委托永远没法 -=
someEvent -= (s, e) => DoStuff();  // 这是另一个对象！取消不了！

// ✅ 缓存为字段
private EventHandler _handler;
public void Init() { _handler = (s, e) => DoStuff(); someEvent += _handler; }
public void Cleanup() { someEvent -= _handler; }  // 引用相同，能取消
```

---

## 字符串拼接

C# 的 `string` 是**不可变的**——每次修改都创建一个新字符串。

```csharp
// ❌ 循环里 + 字符串——每次都 new
string result = "";
for (int i = 0; i < 1000; i++)
{
    result += $"行 {i}\n";  // 每次循环创建一个新字符串！
}
// 1000 次循环 = 1000 次分配，总 GC 压力巨大

// ✅ 用 StringBuilder
private StringBuilder _sb = new StringBuilder();  // 复用
_sb.Clear();
for (int i = 0; i < 1000; i++)
{
    _sb.Append("行 ").Append(i).Append('\n');
}
string result = _sb.ToString();  // 只分配一次最终结果
```

**小规模拼接**（5 个以内）直接用 `+` 没问题。大规模/循环内必须用 `StringBuilder`。

---

## yield return 的隐藏分配

```csharp
// ❌ yield return 创建状态机——有分配开销
public IEnumerable<Pawn> GetColonists()
{
    foreach (var p in allPawns)
        if (p.IsColonist)
            yield return p;  // 编译器生成的状态机在堆上
}

// ✅ 热路径上构造普通 List 返回（清楚的一次性分配）
public List<Pawn> GetColonists()
{
    var result = new List<Pawn>();
    for (int i = 0; i < allPawns.Count; i++)
        if (allPawns[i].IsColonist)
            result.Add(allPawns[i]);
    return result;
}
```

**权衡**：`yield return` 的代码更简洁，适合冷路径（存档加载、配置解析）。热路径（Draw/Update）上避免。

---

## IDisposable 资源

持有这些资源的类**必须**实现 `IDisposable` 并在不使用的时候释放：

| 资源类型 | 示例 |
|----------|------|
| 定时器 | `System.Threading.Timer` |
| 文件 | `FileStream`、`StreamReader`、`StreamWriter` |
| 网络 | 网络连接、Socket |
| 线程 | `Thread` |
| 图形 | `Texture2D`（非 Unity 管理的） |

```csharp
// ✅ 标准模式
public class MyModComponent : IDisposable
{
    private Timer _timer;
    private FileStream _logFile;

    public void Start()
    {
        _timer = new Timer(OnTick, null, 0, 1000);
        _logFile = File.Open("log.txt", FileMode.Append);
    }

    public void Dispose()
    {
        _timer?.Dispose();
        _timer = null;
        _logFile?.Dispose();
        _logFile = null;
    }
    // 调用方必须确保 Dispose() 被调用
}
```

---

## Nullable<T> 的装箱

```csharp
// ❌ Nullable<int> 装箱——有值时会装箱成 int
int? maybeValue = 42;
object obj = maybeValue;  // 装箱！分配了！

// ✅ 避免
if (maybeValue.HasValue)
{
    int val = maybeValue.Value;  // 直接拿值类型，不装箱
}
```

---

## 反射的性能代价

反射（`GetType()`、`GetMethod()`、`GetField()`、`Invoke()`）比直接调用慢 100-1000 倍。

```csharp
// ❌ 每帧反射——灾难
public void Update()
{
    var method = typeof(SomeClass).GetMethod("DoWork");
    method.Invoke(instance, null);  // 每帧！极慢！
}

// ✅ 缓存反射结果
private static readonly MethodInfo _doWorkMethod =
    typeof(SomeClass).GetMethod("DoWork");  // 只反射一次

public void Update()
{
    _doWorkMethod.Invoke(instance, null);  // 还是慢（Invoke），但至少不重复反射了
}

// ✅ 更好的方式——用委托替代 Invoke
private static readonly Action _doWork =
    (Action)Delegate.CreateDelegate(typeof(Action), instance, _doWorkMethod);

public void Update()
{
    _doWork();  // 接近原生调用速度
}
```

**规则**：反射只做一次，结果缓存。热路径上不反射。

---

## ExposeData 存档注意事项

```csharp
// ❌ 往存档里塞太多东西
public override void ExposeData()
{
    Scribe_Collections.Look(ref _allMessages, "messages", LookMode.Deep);
    // 如果有 5000 条聊天记录，存档 20MB！
}

// ✅ 存档只存真正需要持久化的
public override void ExposeData()
{
    // 只存最近的 100 条
    var toSave = _allMessages.TakeLast(100).ToList();
    Scribe_Collections.Look(ref toSave, "messages", LookMode.Deep);

    // UI 状态、缓存、临时数据——不要存！
}
```

**规则**：`ExposeData` 里的数据越少越好。UI 状态、缓存、临时计算结果——这些不该进存档。

---

## 老 Mono（Unity 2019）特有坑

| 问题 | 影响 | 规避 |
|------|------|------|
| `List<T>.ForEach()` 分配委托 | 每帧分配 | 用 `for` |
| `foreach` 在 `List<T>` 上分配 | 每帧分配 | 用 `for` |
| `?.` 空传播在某些情况有 bug | 偶发崩溃 | 显式判空 |
| `nameof()` 在 attribute 里有时生成错误 | 编译期问题 | 直接写字符串 |
| 字符串插值在某些路径上有分配 | 轻微分配 | 热路径上用 `Append` |
| 没有 `Span<T>` / `ValueTuple` 等现代特性 | 无法使用高性能 API | 用数组和 struct |

---

## 快速诊断清单

遇到以下情况时检查对应项：

| 症状 | 检查 |
|------|------|
| 内存一直涨 | 事件未取消订阅 / IDisposable 未实现 / 静态集合无限增长 |
| 间歇性卡顿 | Draw 路径分配 / foreach 在热路径上 / 未缓存的 LINQ |
| 修改 struct 字段没效果 | 使用的是副本，没有写回 |
| 两个对象诡异联动 | 共享了引用类型的成员 |
| 反射调用极慢 | 未缓存 MethodInfo / 在热路径上 |
| 存档文件巨大 | ExposeData 存了太多临时数据 |
| 偶尔 NullReferenceException | 值类型 Nullable 装箱 / 多线程竞态 |
