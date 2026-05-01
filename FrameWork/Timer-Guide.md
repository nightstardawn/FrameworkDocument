# Timer 计时器模块使用指南

## 快速开始

### 创建并启动计时器

```csharp
using Dawn.Framework;

// 每1秒执行一次（无限循环）
TimerTask timer = Timer.CreateAndStartTimer(1f, () =>
{
    Debug.Log("每秒触发一次");
});

// 每0.5秒执行一次，共执行10次
TimerTask timer = Timer.CreateAndStartTimer(0.5f, 10, () =>
{
    Debug.Log("触发了");
});
```

### 延迟执行

```csharp
// 2秒后执行一次回调
Timer.Delay(2f, () =>
{
    Debug.Log("延迟完成");
});

// 不受 TimeScale 影响的延迟（暂停菜单中也会触发）
Timer.Delay(3f, () => Debug.Log("触发"), useUnscaledTime: true);

// 返回句柄，可提前取消
TimerTask task = Timer.Delay(5f, OnComplete);
task.Remove(); // 取消延迟
```

### 带初始延迟的计时器

```csharp
// 延迟3秒后开始，每1秒执行一次，共5次
TimerTask timer = Timer.CreateAndStartTimer(1f, 5, OnTick, delay: 3f);

// 延迟2秒后开始，每0.5秒无限循环
TimerTask timer = Timer.CreateAndStartTimer(0.5f, 0, OnTick, delay: 2f);

// 仅创建（稍后手动启动），启动后先等待1秒再开始计时
TimerTask timer = Timer.CreateTimer(1f, 10, OnTick, delay: 1f);
timer.Start();

// 查询延迟状态
Debug.Log(timer.IsInDelay);  // 是否仍在延迟等待阶段
Debug.Log(timer.Delay);      // 初始延迟时间
```

### 不受 TimeScale 影响

```csharp
// 最后一个参数传 true，即使 Time.timeScale = 0 也会正常计时
TimerTask timer = Timer.CreateAndStartTimer(1f, OnTick, true);
TimerTask timer = Timer.CreateAndStartTimer(1f, 5, OnTick, true);
```

### 仅创建（稍后手动启动）

```csharp
TimerTask timer = Timer.CreateTimer(2f, OnTick);
// ... 某个时机再启动
timer.Start();
```

## 生命周期控制

```csharp
TimerTask timer = Timer.CreateAndStartTimer(1f, OnTick);

timer.Pause();    // 暂停
timer.Resume();   // 恢复
timer.Reset();    // 重置（清零已过时间和执行次数，保持当前运行/暂停状态）
timer.Remove();   // 移除（不可恢复）
```

## 属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `Delay` | float | 初始延迟时间 |
| `IsInDelay` | bool | 是否仍在延迟等待阶段 |
| `IsRunning` | bool | 是否正在运行 |
| `IsPaused` | bool | 是否暂停中 |
| `IsRemoved` | bool | 是否已被移除 |
| `ElapsedTime` | float | 当前周期内已过时间 |
| `RemainingTime` | float | 当前周期内剩余时间 |
| `ExecutedCount` | int | 已执行次数 |
| `TotalCount` | int | 总执行次数（0=无限） |
| `Interval` | float | 间隔时间 |
| `UseUnscaledTime` | bool | 是否不受TimeScale影响 |

## 全局管理

```csharp
// 移除所有计时器
Timer.RemoveAllTimers();
```

## API 一览

### Timer（静态类）

| 方法 | 说明 |
|------|------|
| `Delay(float delay, Action callback, bool useUnscaledTime = false)` | 延迟指定时间后执行一次回调 |
| `CreateAndStartTimer(float interval, Action callback, bool useUnscaledTime = false)` | 创建并启动无限循环计时器 |
| `CreateAndStartTimer(float interval, int count, Action callback, bool useUnscaledTime = false)` | 创建并启动有限次数计时器 |
| `CreateAndStartTimer(float interval, int count, Action callback, float delay, bool useUnscaledTime = false)` | 创建并启动带初始延迟的计时器 |
| `CreateTimer(float interval, Action callback, bool useUnscaledTime = false)` | 仅创建无限循环计时器 |
| `CreateTimer(float interval, int count, Action callback, bool useUnscaledTime = false)` | 仅创建有限次数计时器 |
| `CreateTimer(float interval, int count, Action callback, float delay, bool useUnscaledTime = false)` | 仅创建带初始延迟的计时器 |
| `RemoveAllTimers()` | 移除所有计时器 |

### TimerTask（计时器句柄）

| 方法 | 说明 |
|------|------|
| `Start()` | 启动计时器 |
| `Pause()` | 暂停计时器 |
| `Resume()` | 恢复计时器 |
| `Reset()` | 重置（清零时间、次数和延迟） |
| `Remove()` | 移除计时器 |

## 注意事项

1. **Timer 依赖 MonoMgr**：TimerManager 通过 `MonoMgr.Instance.AddUpdateListener()` 驱动每帧更新，确保 MonoMgr 可用。
2. **Remove 不可恢复**：调用 `Remove()` 后计时器将被标记为已移除，无法再 Start/Resume。
3. **有限次数计时器自动移除**：达到执行次数上限后自动调用 Remove。
4. **回调中可安全操作**：可以在回调中 Pause/Remove 当前计时器或创建新计时器。
5. **外部管理生命周期**：持有 TimerTask 引用，在适当时机调用 Remove 避免泄漏。
