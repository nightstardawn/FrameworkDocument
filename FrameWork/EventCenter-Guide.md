# 事件中心使用指南

## 快速开始

### 1. 定义事件类型

在 `EventType` 中声明事件，每个事件自动分配唯一 int ID：

```csharp
public static class EventType
{
    private static int _counter = 0;
    public static int GetIndex() => ++_counter;

    // === 玩家相关 ===
    public static readonly int PlayerDead       = GetIndex(); // 1
    public static readonly int PlayerLevelUp    = GetIndex(); // 2
    public static readonly int PlayerHpChanged  = GetIndex(); // 3

    // === UI 相关 ===
    public static readonly int ScoreChanged     = GetIndex(); // 4
    public static readonly int CoinChanged      = GetIndex(); // 5

    // === 系统相关 ===
    public static readonly int GamePause        = GetIndex(); // 6
    public static readonly int GameResume       = GetIndex(); // 7
}
```

### 2. 监听事件

```csharp
// 无参事件
EventCenter.Instance.AddEvent(EventType.PlayerDead, OnPlayerDead);

// 带参数事件
EventCenter.Instance.AddEvent<int>(EventType.ScoreChanged, OnScoreChanged);
```

### 3. 触发事件

```csharp
// 无参
EventCenter.Instance.TriggerEvent(EventType.PlayerDead);

// 带参数
EventCenter.Instance.TriggerEvent<int>(EventType.ScoreChanged, 100);
```

### 4. 移除监听

```csharp
EventCenter.Instance.RemoveEvent(EventType.PlayerDead, OnPlayerDead);
EventCenter.Instance.RemoveEvent<int>(EventType.ScoreChanged, OnScoreChanged);
```

---

## API 一览

### 添加监听（AddEvent）

| 方法签名 | 说明 |
|---------|------|
| `AddEvent(int eventId, Action callback)` | 无参 |
| `AddEvent<T>(int eventId, Action<T> callback)` | 1 个参数 |
| `AddEvent<T1, T2>(int eventId, Action<T1, T2> callback)` | 2 个参数 |
| `AddEvent<T1, T2, T3>(int eventId, Action<T1, T2, T3> callback)` | 3 个参数 |
| `AddEvent<T1, T2, T3, T4>(int eventId, Action<T1, T2, T3, T4> callback)` | 4 个参数 |

### 移除监听（RemoveEvent）

签名与 AddEvent 一一对应，参数相同。

### 触发事件（Trigger）

| 方法签名 | 说明 |
|---------|------|
| `Trigger(int eventId)` | 无参触发 |
| `Trigger<T>(int eventId, T arg)` | 1 个参数 |
| `Trigger<T1, T2>(int eventId, T1 arg1, T2 arg2)` | 2 个参数 |
| `Trigger<T1, T2, T3>(int eventId, T1 arg1, T2 arg2, T3 arg3)` | 3 个参数 |
| `Trigger<T1, T2, T3, T4>(int eventId, T1 arg1, T2 arg2, T3 arg3, T4 arg4)` | 4 个参数 |

### 工具方法

| 方法 | 说明 |
|------|------|
| `RemoveAllEvents(int eventId)` | 移除某个事件的全部监听 |
| `Clear()` | 清空所有事件的所有监听 |

---

## 典型场景示例

### 场景一：玩家死亡通知多个系统

```csharp
// ---- EventType 定义 ----
public static readonly int PlayerDead = GetIndex();

// ---- UIManager（监听方） ----
void OnEnable()
{
    EventCenter.Instance.AddEvent(EventType.PlayerDead, ShowGameOverUI);
}

void OnDisable()
{
    EventCenter.Instance.RemoveEvent(EventType.PlayerDead, ShowGameOverUI);
}

void ShowGameOverUI()
{
    UIManager.Instance.ShowWindow<GameOverWindow>();
}

// ---- AudioManager（监听方） ----
void OnEnable()
{
    EventCenter.Instance.AddEvent(EventType.PlayerDead, PlayDeathSound);
}

void OnDisable()
{
    EventCenter.Instance.RemoveEvent(EventType.PlayerDead, PlayDeathSound);
}

// ---- Player（触发方） ----
void Die()
{
    EventCenter.Instance.TriggerEvent(EventType.PlayerDead);
}
```

### 场景二：分数变化（带参数）

```csharp
// ---- EventType ----
public static readonly int ScoreChanged = GetIndex();

// ---- ScoreUI（监听方） ----
void OnEnable()
{
    EventCenter.Instance.AddEvent<int>(EventType.ScoreChanged, OnScoreChanged);
}

void OnDisable()
{
    EventCenter.Instance.RemoveEvent<int>(EventType.ScoreChanged, OnScoreChanged);
}

void OnScoreChanged(int newScore)
{
    _txtScore.text = newScore.ToString();
}

// ---- GameLogic（触发方） ----
void AddScore(int amount)
{
    _totalScore += amount;
    EventCenter.Instance.TriggerEvent<int>(EventType.ScoreChanged, _totalScore);
}
```

### 场景三：多参数事件（伤害信息）

```csharp
// ---- EventType ----
public static readonly int DamageDealt = GetIndex();

// ---- DamagePopup（监听方） ----
void OnEnable()
{
    EventCenter.Instance.AddEvent<Vector3, int, bool>(
        EventType.DamageDealt, ShowDamageNumber);
}

void OnDisable()
{
    EventCenter.Instance.RemoveEvent<Vector3, int, bool>(
        EventType.DamageDealt, ShowDamageNumber);
}

void ShowDamageNumber(Vector3 position, int damage, bool isCritical)
{
    // 在指定位置弹出伤害数字
}

// ---- Combat（触发方） ----
void DealDamage(Vector3 hitPos, int dmg, bool crit)
{
    EventCenter.Instance.TriggerEvent<Vector3, int, bool>(
        EventType.DamageDealt, hitPos, dmg, crit);
}
```

### 场景四：切场景时清理事件

```csharp
void OnSceneUnloaded(Scene scene)
{
    EventCenter.Instance.Clear();
}
```

---

## 与 UI 框架配合使用

推荐在 UIWindow 的生命周期中注册和注销事件：

```csharp
public class BattleHUDWindow : UIWindow
{
    protected override void OnShow(IUIWindowParam param)
    {
        EventCenter.Instance.AddEvent<int>(EventType.PlayerHpChanged, OnHpChanged);
        EventCenter.Instance.AddEvent<int>(EventType.ScoreChanged, OnScoreChanged);
    }

    protected override void AfterHide(IUIWindowParam param)
    {
        EventCenter.Instance.RemoveEvent<int>(EventType.PlayerHpChanged, OnHpChanged);
        EventCenter.Instance.RemoveEvent<int>(EventType.ScoreChanged, OnScoreChanged);
    }

    void OnHpChanged(int hp) { /* 刷新血条 */ }
    void OnScoreChanged(int score) { /* 刷新分数 */ }
}
```

---

## 注意事项

1. **泛型类型必须匹配**：`AddEvent<int>` 注册的事件，必须用 `Trigger<int>` 触发，类型不一致会静默失败
2. **必须成对注册/注销**：在 `OnEnable`/`OnShow` 中注册，在 `OnDisable`/`AfterHide` 中注销，避免对象销毁后仍被回调
3. **使用方法引用而非 Lambda**：Lambda 每次创建新委托实例，无法通过 `RemoveEvent` 移除
4. **避免在回调中触发同一事件**：可能导致无限递归
5. **切场景时考虑清理**：如果事件不需要跨场景保留，在场景卸载时调用 `Clear()`

### 正确 vs 错误用法

```csharp
// 正确：使用方法引用，可以移除
EventCenter.Instance.AddEvent(EventType.GamePause, OnPause);
EventCenter.Instance.RemoveEvent(EventType.GamePause, OnPause);

// 错误：Lambda 无法移除
EventCenter.Instance.AddEvent(EventType.GamePause, () => { Time.timeScale = 0; });
// 下面这行无法移除上面的 Lambda，因为是不同的委托实例
EventCenter.Instance.RemoveEvent(EventType.GamePause, () => { Time.timeScale = 0; });
```

---

## 文件结构

```
Assets/Script/frame/Event/
├── EventCenter.cs      # 事件中心单例（继承 BaseManager<EventCenter>）
└── EventType.cs        # 事件 ID 定义（静态类 + GetIndex 自增）
```
