# 性能、UI 与线程

## 测量正确的预算

优化前先分类路径：

- 每个 GUI 事件/帧；
- 每 Tick 或周期 Tick；
- 每 Pawn、Projectile、Building 或 Map；
- 仅加载阶段；
- 少见玩家命令；
- 后台 I/O 或计算。

分配只是成本之一，还包括 CPU、Draw Call、材质创建、寻路、Def 查询、日志、反射和集合抖动。优化测得的热点，在其他位置保持清晰。

## IMGUI

`DoWindowContents`、Gizmo、Overlay 和 Inspector 可能高频执行，并在多个 EventType 下重复调用。热 UI 路径中：

- 输入变化时预计算过滤/排序数据；
- 缓存昂贵文本测量、Regex、Texture 和派生模型，并明确失效条件；
- 对大型列表虚拟化或裁剪；
- 避免重复 Def 扫描和材质创建；
- 保持事件处理确定性，并恢复 GUI 全局状态。

`new` 不一定产生堆分配，值类型和编译器行为会影响结果。LINQ 也不是禁用项；只有分析或检查表明其在热路径造成明显成本时才移除。

## Tick 工作

延迟语义允许时使用 `TickRare`、`TickLong`、错峰、脏队列或事件驱动更新。不要让每个实体执行全地图或全 Pawn 扫描。Def 数据库就绪后缓存 Def 查询，不要过早初始化。

## 线程

除非项目证明安全，否则假设 Verse 和 Unity 对象只能在主线程访问。后台任务只能处理不可变快照和纯数据，不得接触 Map、Pawn、DefDatabase、Texture、GUI、线程安全未知的日志设施或 Scribe 状态。

通过项目已验证的 Dispatcher/生命周期 Hook 返回结果，应用前重新验证引用的游戏状态，并支持游戏卸载或 Mod 关闭时取消。`LongEventHandler.ExecuteWhenFinished` 属于长事件生命周期；未验证上下文前不要把它当通用运行时 Dispatcher。

Harmony Patch 运行在调用者线程上；被 Patch 的方法不会因此自动获得主线程安全性。

## 网络或外部进程

定义超时、取消、重试/退避、最大队列、消息大小、认证边界和故障隔离。不得在游戏线程等待远程 I/O。重复失败时限制日志和用户通知。

## 性能证据

报告场景、地图/Pawn/Projectile 数量、Mod 列表、持续时间、Profiler、基线、结果和方差。游戏外微基准不能证明游戏内改善。

