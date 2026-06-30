# Harmony 补丁最佳实践

> Harmony 是 RimWorld Mod 的核心工具——让你在不动原版代码的前提下修改游戏行为。但它也是 Mod 冲突的第一来源。用好 = 你的 Mod 和所有人共存。用错 = 你的 Mod 和别人互炸。

---

## 补丁类型：什么时候用哪种

| 补丁类型 | 用途 | 关键语义 |
|----------|------|----------|
| **Prefix** | 在原方法执行**前**插入 | 返回 `false` = **跳过原方法**，你的代码全权负责 |
| **Postfix** | 在原方法执行**后**插入 | 原方法已执行完毕，你可以修改 `__result` |
| **Transpiler** | 修改方法的 **IL 代码** | 最强大也最容易出错——谨慎使用 |
| **Finalizer** | 类似 finally 块 | 原方法异常时也执行，用于清理 |

---

## Prefix 核心规则

```csharp
// ✅ 标准 Prefix 模板
[HarmonyPatch(typeof(TargetClass), "TargetMethod")]
public static class TargetMethod_Patch
{
    // Prefix 返回 bool：false = 跳过原方法
    public static bool Prefix(TargetClass __instance, ref bool __result, ...)
    {
        // 如果需要跳过原方法
        if (ShouldSkip(__instance))
        {
            __result = false;  // 设置原方法的返回值
            return false;      // 告诉 Harmony：别执行原方法了
        }

        // 正常执行原方法
        return true;
    }
}
```

**关键陷阱**：

```csharp
// ❌ Prefix 返回 false 但没有设置 __result
public static bool Prefix(ref int __result)
{
    // 原方法签名是 int DoWork()——调用方期待一个 int
    return false;  // 跳过了原方法但 __result 未赋值！
    // → 调用方得到一个未初始化的 int → 随机值 → 神秘 Bug
}

// ✅ 跳过原方法时必须设置 __result
public static bool Prefix(ref int __result)
{
    __result = 42;  // 先设好返回值
    return false;   // 再跳过
}
```

---

## Postfix 核心规则

```csharp
// ✅ Postfix 可以读取和修改原方法的返回值
[HarmonyPatch(typeof(TargetClass), "TargetMethod")]
public static class TargetMethod_Patch
{
    public static void Postfix(TargetClass __instance, ref int __result)
    {
        // __instance：调用该方法的对象（静态方法则没有）
        // __result：原方法的返回值（ref 可修改）
        if (__result < 0)
            __result = 0;  // 修正返回值
    }
}
```

**重要**：Postfix 在**原方法执行后**运行。原方法的所有副作用（状态变更、事件触发）已经发生。Postfix 只能"追加行为"或"修正结果"。

---

## 特殊参数命名约定

Harmony 通过参数**名称**识别特殊注入。这些是区分大小写的：

| 参数名 | 用途 | 适用补丁类型 |
|--------|------|-------------|
| `__instance` | 引用调用该方法的对象 | Prefix, Postfix |
| `__result` | 原方法的返回值（`ref` 可修改） | Prefix, Postfix |
| `__state` | 从 Prefix 传数据到 Postfix | Prefix → Postfix |
| `___fields` | 用 Traverse 的私有字段访问（三个下划线） | Prefix, Postfix |

```csharp
// ✅ __state 传递数据：Prefix → Postfix
public static void Prefix(object __instance, out bool __state)
{
    __state = ((SomeClass)__instance).SomeField != null;
}

public static void Postfix(object __instance, bool __state)
{
    if (__state)
    {
        // Prefix 告诉我们的信息
    }
}
```

> **注意**：`___fields` 是三个下划线（`___`），用于注入 Traverse（私有字段访问器）。这是实验性特性——不要在生产代码中依赖它；用反射替代。

---

## Priority：控制多个补丁的执行顺序

```csharp
// 多个 Mod 都 patch 了同一个方法时，Priority 决定顺序
[HarmonyPriority(100)]  // 数字越小越先执行
public static bool Prefix()
{
    // 我的补丁优先于别人
}
```

**惯例**：
- `Priority = 0`（默认）：正常补丁
- `Priority = 100`：需要较早执行的补丁
- `Priority = -100`：需要较晚执行的补丁
- `Priority = int.MaxValue`：最后一个 Prefix（通常用于"兜底"逻辑）

**原则**：如果你不知道应该设多少，就不设。只有在遇到明确的执行顺序问题时才调整 Priority。

---

## 补丁的热路径性能

补丁方法本身也在热路径上——如果 `Tick()` 方法每秒跑 60 次，你的 Prefix 也是每秒 60 次。

```csharp
// ❌ 在 Tick 的 Patch 里做复杂操作
[HarmonyPatch(typeof(TickManager), "DoSingleTick")]
public static bool Prefix()
{
    // 每帧都反射！极慢！
    var field = typeof(Game).GetField("someField", BindingFlags.NonPublic);
    var value = field.GetValue(Current.Game);
    // ...
}

// ✅ 反射结果缓存为 static
[HarmonyPatch(typeof(TickManager), "DoSingleTick")]
public static class DoSingleTick_Patch
{
    private static readonly FieldInfo _someField =
        typeof(Game).GetField("someField", BindingFlags.NonPublic);  // 只反射一次

    public static void Prefix()
    {
        var value = _someField.GetValue(Current.Game);  // 只有 GetValue 的开销
    }
}
```

**规则**：补丁方法遵守所有 IMGUI 性能规则（见 [imgui-performance.md](imgui-performance.md)）。

---

## 最小化补丁范围

**原则**：能 patch 具体的方法就不 patch 通用的。能 patch 深层方法就不 patch 表面的。

```csharp
// ❌ Patch Update()——每帧触发，影响范围巨大
[HarmonyPatch(typeof(Pawn), "Tick")]  // 每个 Pawn 每帧都跑
// 你的补丁代码会被每个小人每帧执行一次——几百个小人 = 灾难

// ✅ Patch 具体的事件处理方法
[HarmonyPatch(typeof(JobDriver_Cook), "FinishAction")]
// 只有在烹饪完成时触发——频率低，影响小
```

**判断法**：你的补丁一分钟触发多少次？如果大于 3600（每秒 60 帧 × 60 秒），必须严审性能。

---

## 与其他 Mod 共存的技巧

```csharp
// ❌ 硬检查：你的 Mod 特定处理排斥所有人
public static bool Prefix(Pawn __instance)
{
    if (__instance.def.defName == "MyCustomPawn")  // 只认自己
    {
        // 你的逻辑
        return false;  // 跳过——其他 Mod 的补丁也没机会跑了！
    }
    return true;
}

// ✅ 让行为可组合——你不关心的 case 放行给别人
public static bool Prefix(Pawn __instance)
{
    if (__instance.def.defName != "MyCustomPawn")
        return true;  // 不是我关心的——放行，让其他 Mod 的补丁也能跑

    // 你的逻辑
    return false;
}
```

**原则**：前缀补丁里，**尽早放行**。你不处理的 case 立即 `return true`，不要阻碍其他 Mod。

---

## 常见反模式

### 1. 补丁一切，而不是只补需要的地方

```csharp
// ❌ Patch 了所有 Thing 的 SpawnSetup——几十个子类，不确定影响范围
[HarmonyPatch(typeof(Thing), "SpawnSetup")]

// ✅ Patch 特定子类
[HarmonyPatch(typeof(Building_Bed), "SpawnSetup")]
```

### 2. Prefix 当 Postfix 用

```csharp
// ❌ 在 Prefix 里做了原方法之后才该做的事情
public static bool Prefix(...)
{
    DoStuffBefore();        // 正确：Prefix 的职责
    DoStuffAfterMethod();   // 错误：原方法还没执行！状态还不存在！
    return true;
}

// ✅ 分到 Prefix 和 Postfix
public static void Prefix() { DoStuffBefore(); }
public static void Postfix() { DoStuffAfterMethod(); }
```

### 3. 在补丁里分配对象

```csharp
// ❌ 补丁在热路径上——跟 Draw 一样遵守零分配规则
public static void Prefix()
{
    var list = new List<Pawn>();  // 分配！
    var regex = new Regex("...");  // 分配！
}
```

### 4. 补丁方法抛异常

```csharp
// ❌ 补丁方法里的未捕获异常会导致原方法静默失败或崩溃
public static void Prefix()
{
    SomeMethodThatMightThrow();  // 危险！外面没有 try-catch
}

// ✅ 加保护
public static void Prefix()
{
    try
    {
        SomeMethodThatMightThrow();
    }
    catch (Exception ex)
    {
        Log.Error($"[MyMod] Prefix error: {ex}");
        // 不要重新抛出——让游戏继续运行
    }
}
```

### 5. 依赖补丁执行顺序但不设 Priority

```csharp
// ❌ 你的 Postfix 需要别人的 Prefix 先执行——但没声明 Priority
// 别人的补丁 Priority 改了 → 你的就炸了

// ✅ 明确声明依赖
[HarmonyPriority(200)]  // 晚于大多数 Prefix，早于大多数 Postfix
public static void Postfix() { ... }
```

---

## Transpiler 警告

Transpiler 操作 IL 指令——这是 Harmony 最强大的功能，也是最容易产生神秘 Bug 的。

**仅在以下场景使用 Transpiler**：
- 需要在方法**中间**插入/删除逻辑（Prefix 和 Postfix 只能加在头和尾）
- 需要修改循环内的行为
- 需要替换方法调用（如把 `Debug.Log` 替换成你的日志）

**使用 Transpiler 时**：
- 写清楚注释，解释每步 IL 操作的意图
- 用 `CodeMatcher` 和 `CodeInstruction` 辅助类（Harmony 提供）
- 测试多种情况下原方法的响应
- **不要用 Transpiler 做 Prefix/Postfix 能做的事**

---

## 调试技巧

```csharp
// 临时加——验证你的补丁是否被调用了
[HarmonyDebug]  // 输出详细的补丁调用信息到日志
[HarmonyPatch(...)]
public static void Prefix()
{
    // 这会产生很多日志——只在调试时用
}
```

```csharp
// 打印被补丁方法的调用栈
Log.Warning($"Called from: {Environment.StackTrace}");
```

---

## 快速清单

写 Harmony 补丁时对照：

- [ ] 用具体类型 patch 而不是 `typeof(Thing)`/`typeof(Pawn)` 等基类？
- [ ] Prefix 返回 false 时设置了 `__result`？
- [ ] 热路径上的补丁有没有分配对象？
- [ ] 反射调用是否已缓存为 `static readonly`？
- [ ] 有没有在补丁方法里 `try-catch` 保护？
- [ ] 不需要处理的情况是否尽早 `return true`（放行给其他 Mod）？
- [ ] Priority 是否只在有明确顺序需求时才设？
- [ ] 是否能用 Prefix/Postfix 替代 Transpiler？
- [ ] `__instance` 参数名是否正确（两个下划线）？
- [ ] 补丁类的访问修饰符是 `public static`？
