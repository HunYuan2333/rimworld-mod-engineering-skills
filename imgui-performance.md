# IMGUI 性能：Draw 路径零分配指南

> RimWorld 使用 Unity 的 IMGUI（立即模式 GUI）。你的 Draw 方法**每帧执行一次**。一帧给 UI 的时间可能不到 10ms。每帧分配对象 → GC 在关键时刻回收 → 间歇性卡顿。

---

## 核心原则

**Draw 路径上的内存分配必须为零。** 这不是"尽量少分配"——是零分配。

Draw 路径包括：`DoWindowContents()`、`Draw()`、`DoButton()`、`CalcHeight()`、`CalcWidth()`——以及它们调用的所有方法。

---

## 什么会分配内存（你未必知道）

### 1. `new` 任何引用类型

```csharp
// ❌ 每帧 new——最常见也最严重
var content = new GUIContent("hello");        // 每帧
var rect = new Rect(0, 0, 100, 100);          // Rect 是 struct——没问题
var list = new List<Message>();                // 每帧！
var builder = new StringBuilder();             // 每帧！
```

**修复**：移到字段，复用。

```csharp
private GUIContent _cachedContent = new GUIContent();  // 只分配一次
private List<Message> _tempList = new List<Message>();  // 复用
private StringBuilder _builder = new StringBuilder();   // 复用

public void Draw(Rect inRect)
{
    _cachedContent.text = "hello";  // 只改值，不 new
    _builder.Clear();               // 清空复用，不抛掉重建
    // ...
}
```

### 2. LINQ 在 Draw 路径上

```csharp
// ❌ 每个 LINQ 操作都分配枚举器
var count = messages.Where(m => m.IsNew).Count();       // Where 分配
var sorted = messages.OrderBy(m => m.Time).ToList();    // OrderBy + ToList 分配
var names = messages.Select(m => m.Sender).ToList();    // Select + ToList 分配
```

**修复**：在数据变更时计算，存字段。

```csharp
private int _newMessageCount;  // 数据变更时更新
private List<Message> _sortedMessages;  // 数据变更时排序

public void OnDataChanged()
{
    _newMessageCount = 0;
    _sortedMessages = new List<Message>();
    foreach (var m in messages)  // foreach 在外面——不是每帧
    {
        if (m.IsNew) _newMessageCount++;
        _sortedMessages.Add(m);
    }
    _sortedMessages.Sort((a, b) => a.Time.CompareTo(b.Time));
}

public void Draw(Rect inRect)
{
    // Draw 里直接用缓存的值——零分配
    Widgets.Label(rect, $"新消息: {_newMessageCount}");
}
```

### 3. 属性 getter 里做计算

```csharp
// ❌ 属性 getter 每帧被 UI 调用——每次访问都分配
public string BadgeText
{
    get
    {
        return messages.Where(m => m.IsUnread).Count().ToString();
        // Where 枚举器 + Count 枚举器 + ToString 分配字符串 + 每帧！
    }
}
```

**修复**：缓存 + 推送更新。

```csharp
private string _badgeText = "";
public string BadgeText => _badgeText;  // 直接返回缓存

public void OnMessagesChanged()
{
    int unread = 0;
    foreach (var m in messages) if (m.IsUnread) unread++;
    _badgeText = unread > 0 ? unread.ToString() : "";
}
```

### 4. foreach 在热路径上（老 Mono）

```csharp
// ❌ Unity 的老 Mono 运行时——foreach 在 List 上分配枚举器
public void Draw(Rect inRect)
{
    foreach (var msg in messages)  // 分配枚举器！
    {
        Widgets.Label(rect, msg.Text);
    }
}
```

**修复**：用 `for`。

```csharp
// ✅ for 循环——零分配
public void Draw(Rect inRect)
{
    for (int i = 0; i < messages.Count; i++)
    {
        Widgets.Label(rect, messages[i].Text);
    }
}
```

> **注**：新版 .NET 运行时已修复此问题，但 RimWorld 的 Unity 2019 用的是老 Mono。不要冒险——热路径上默认用 `for`。

### 5. 字符串拼接

```csharp
// ❌ 字符串在 C# 中不可变——每次 + 都是 new 一个新字符串
var text = "玩家: " + playerName + " | 状态: " + status + " | 物品: " + itemCount;
// 至少分配了 4 个临时字符串
```

**修复**：用 `StringBuilder` 或缓存。

```csharp
// ✅ StringBuilder 复用
private StringBuilder _sb = new StringBuilder();
private string _cachedStatusText;

void UpdateStatusCache()
{
    _sb.Clear();
    _sb.Append("玩家: ").Append(playerName)
       .Append(" | 状态: ").Append(status)
       .Append(" | 物品: ").Append(itemCount);
    _cachedStatusText = _sb.ToString();
}
```

### 6. 委托 / Lambda 分配

```csharp
// ❌ lambda 表达式 = 分配委托对象
GUIUtility.GetRect(100, 20, () => "myWidget");  // 每帧！
Widgets.DrawIf(() => showSection, DrawSection);   // 每帧！
```

**修复**：缓存委托。

```csharp
// ✅ 缓存委托
private static readonly Func<string> _myWidgetLabel = () => "myWidget";
private bool _showSection;
// Draw 里:
GUIUtility.GetRect(100, 20, _myWidgetLabel);
```

### 7. Regex

```csharp
// ❌ Draw 里 new Regex——每帧编译正则表达式！
var regex = new Regex(@"<[^>]+>");
var clean = regex.Replace(rawText, "");
```

**修复**：`static readonly` + `RegexOptions.Compiled`。

```csharp
// ✅ 预编译，全局复用
private static readonly Regex TagRegex = new Regex(
    @"<[^>]+>", RegexOptions.Compiled);

public void Draw(Rect inRect)
{
    var clean = TagRegex.Replace(rawText, "");
}
```

---

## 脏标记模式（最重要的优化模式）

**思路**：数据变化时设置一个 `bool _dirty = true`。Draw 里检查——脏了才重算，不脏直接用缓存。

```csharp
public class ChatWindow
{
    private List<ChatMessage> _allMessages = new();
    private List<ChatMessage> _visibleMessages = new();  // 缓存
    private float _cachedHeight;
    private bool _dirty = true;

    // 数据变更时
    public void AddMessage(ChatMessage msg)
    {
        _allMessages.Add(msg);
        _dirty = true;  // 标记需要重新计算
    }

    // 每次 Draw 可能跑很多次，但只在脏时才重算
    public float CalcHeight(Rect rect)
    {
        if (_dirty) RebuildCache();
        return _cachedHeight;
    }

    private void RebuildCache()
    {
        _visibleMessages.Clear();
        _cachedHeight = 0;
        for (int i = 0; i < _allMessages.Count; i++)
        {
            _visibleMessages.Add(_allMessages[i]);
            _cachedHeight += MeasureHeight(_allMessages[i]);
        }
        _dirty = false;
    }
}
```

**脏标记适用场景**：排序、过滤、高度/宽度计算——任何"数据不怎么变但每帧都要读"的计算。

---

## 大型列表的滚动优化

聊天消息列表、交易物品列表——可能有几百条。不要每帧遍历全部条目计算布局。

```csharp
// ❌ 每帧遍历 500 条消息计算可见的
public void Draw(Rect inRect)
{
    float y = 0;
    foreach (var msg in allMessages)  // 500 条全遍历
    {
        float h = CalcHeight(msg);    // 每帧 × 500 次
        if (y + h > scrollPos && y < scrollPos + viewHeight)
            DrawMessage(msg, y);      // 只有 20 条真正可见
        y += h;
    }
}

// ✅ 缓存每条消息的高度 + 二分查找可见范围
private float[] _cachedHeights;  // 数据变化时重建
private int _firstVisible;       // 缓存

void UpdateVisibility(Rect viewRect, float scrollPos)
{
    // 二分查找滚动位置——O(log N)，不是 O(N)
    _firstVisible = BinarySearchByHeight(scrollPos);
    // 只遍历可见的消息
}

public void Draw(Rect inRect)
{
    float y = _cachedYOffsets[_firstVisible];
    for (int i = _firstVisible; i < _visibleMessages.Count && y < inRect.yMax; i++)
    {
        DrawMessage(_visibleMessages[i], y);
        y += _cachedHeights[i];
    }
}
```

---

## 如何检测你的分配

### RimWorld 开发者模式 Profiler

1. 选项 → 启用开发者模式
2. 顶部出现 Dev 工具栏 → 点 Performance Profiler
3. 打开你的 Mod 窗口 → 操作几下 → 看 GC.Alloc 列
4. 你的 Draw 方法在 Profiler 中的分配应为 **0**

### 简易自查法

在方法上临时加：

```csharp
private long _lastAlloc;
public void Draw(Rect inRect)
{
    long alloc = GC.GetTotalMemory(false);
    if (_lastAlloc > 0 && alloc != _lastAlloc)
        Log.Warning($"Draw 分配了 {alloc - _lastAlloc} 字节！");
    _lastAlloc = alloc;
    // 你原来的 Draw 逻辑
}
```

> 调试完记得删掉——这段代码本身有开销。

---

## 清单

每次写新的 Window/Draw 方法时，对照检查：

- [ ] 有没有 `new GUIContent` / `new List` / `new Regex`？
- [ ] 有没有 LINQ（`.Where()` `.Select()` `.OrderBy()` `.ToList()` `.Count()`）？
- [ ] 属性 getter（如 `BadgeText`）是否在每帧做计算？
- [ ] 热路径上是否用了 `foreach` 而非 `for`？
- [ ] 有没有字符串 `+` 拼接在循环里？
- [ ] 有没有 lambda/匿名方法作为参数传入？
- [ ] 大列表是否每帧全部遍历？
- [ ] 布局计算（`CalcHeight`/`CalcWidth`）是否用了脏标记缓存？
- [ ] 所有 `Regex` 是否是 `static readonly` + `RegexOptions.Compiled`？

**Draw 路径零分配不是优化——是基础要求。** 10 个 Mod 每个在 Draw 里分配 1KB → 加起来 10KB/帧 → 每秒 600KB GC 压力 → 所有 Mod 一起卡。
