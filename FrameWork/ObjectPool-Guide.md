# 对象池模块使用指南

## 概述

对象池通过复用已创建的对象避免频繁 `Instantiate`/`Destroy` 带来的 GC 压力和实例化开销。

本模块提供三种池：

| 类型 | 用途 | 适用对象 |
|------|------|----------|
| `ObjectPool<T>` | 无约束基础池 | 任意有无参构造函数的类型 |
| `PoolObject<T>` | 带自动重置的池 | 实现 `IPoolObject` 的业务数据类 |
| `GameObjectPoolManager` | GameObject 池管理器 | 预制体实例（子弹、特效、敌人等） |

---

## 一、GameObject 对象池

### 快速开始

```csharp
using Dawn.Framework;

// 从池中取出对象（池空时自动异步加载）
GameObjectPoolManager.Instance.Pop("Effect/Explosion", (go) =>
{
    go.transform.position = hitPos;
});

// 归还对象到池（自动识别路径，无需手动传入）
GameObjectPoolManager.Instance.Push(go);

// 也可以手动指定路径归还
GameObjectPoolManager.Instance.Push("Effect/Explosion", go);
```

### 预加载

```csharp
// 提前加载到池中，避免首次使用时的加载延迟
GameObjectPoolManager.Instance.Preload("Bullet/Arrow");

// 预加载完成后的回调
GameObjectPoolManager.Instance.Preload("Bullet/Arrow", (go) =>
{
    Debug.Log("预加载完成");
});
```

### 数量上限与强制复用

```csharp
// maxNum = 30：最多同时存在30个此类对象
// 超出时强制复用最早取出的对象，并通过 onForceRecycle 通知旧持有者
GameObjectPoolManager.Instance.Pop("Bullet/Arrow", (bullet) =>
{
    bullet.GetComponent<Bullet>().Fire(direction);
},
maxNum: 30,
onForceRecycle: (oldBullet) =>
{
    // 旧子弹被抢回前的清理工作
    oldBullet.GetComponent<Bullet>().StopAllCoroutines();
});

// maxNum = -1（默认）：无上限，按需无限创建
GameObjectPoolManager.Instance.Pop("NPC/Villager", (npc) =>
{
    npc.transform.position = spawnPos;
});
```

**maxNum 设计原则：**

| maxNum 值 | 行为 | 适用场景 |
|-----------|------|----------|
| -1 | 无上限，池空就创建新对象 | 敌人、NPC 等数量由设计控制的对象 |
| > 0 | 超出上限时强制复用最早的对象 | 子弹、特效、飘字等短生命周期对象 |

**onForceRecycle 回调使用场景：**

| 场景 | 回调中做什么 |
|------|-------------|
| 子弹正在飞行 | 停止飞行协程、清空目标引用 |
| 特效绑定了跟随 | 解除跟随绑定 |
| 音效正在播放 | `AudioSource.Stop()` |
| 对象注册了事件 | `EventCenter.RemoveListener` |

如果对象是无状态的（纯视觉装饰物），`onForceRecycle` 可以不传。

### 销毁池

```csharp
// 销毁指定路径的池
GameObjectPoolManager.Instance.DestroyPool("Bullet/Arrow");

// 销毁所有池（切换场景时调用）
GameObjectPoolManager.Instance.DestroyAll();
```

### 自动回收机制

池中所有对象归还后，若 **30秒内无任何 Pop/Push 操作**，池将自动销毁，无需手动管理。

---

## 二、泛型对象池

### ObjectPool\<T\> — 无约束基础池

```csharp
using Dawn.Framework;

// 创建池
var listPool = new ObjectPool<List<int>>();

// 取出
List<int> list = listPool.Pop();
list.Add(1);
list.Add(2);

// 归还（调用方负责重置状态）
list.Clear();
listPool.Push(list);
```

### PoolObject\<T\> — 带自动重置的池

#### 步骤1：定义可池化类，实现 IPoolObject

```csharp
using Dawn.Framework;

public class DamageInfo : IPoolObject
{
    public int damage;
    public GameObject attacker;
    public GameObject target;

    public void ResetInfo()
    {
        damage = 0;
        attacker = null;
        target = null;
    }
}
```

#### 步骤2：使用 PoolObject\<T\>

```csharp
private PoolObject<DamageInfo> _damagePool = new PoolObject<DamageInfo>();

// 取出
DamageInfo info = _damagePool.Pop();
info.damage = 50;
info.attacker = player;
info.target = enemy;

// 传递使用...
EventCenter.Instance.Trigger(EventType.OnDamage, info);

// 归还（自动调用 ResetInfo，无需手动清理）
_damagePool.Push(info);
```

---

## 三、API 一览

### GameObjectPoolManager（单例）

| 方法 | 说明 |
|------|------|
| `Pop(string path, Action<GameObject> callback, bool autoActive = true, int maxNum = -1, Action<GameObject> onForceRecycle = null)` | 从池中取出对象 |
| `Push(GameObject go)` | 归还对象（自动识别路径） |
| `Push(string path, GameObject go)` | 归还对象（手动指定路径） |
| `Preload(string path, Action<GameObject> callback = null)` | 预加载到池中 |
| `DestroyPool(string path)` | 销毁指定池 |
| `DestroyAll()` | 销毁所有池 |

### ObjectPool\<T\>（无约束）

| 方法 | 说明 |
|------|------|
| `Pop()` | 取出对象（栈空时 new） |
| `Push(T item)` | 归还对象（调用方负责重置） |
| `Clear()` | 清空池 |
| `Count` | 池中空闲对象数量 |

### PoolObject\<T\>（带自动重置）

| 方法 | 说明 |
|------|------|
| `Pop()` | 取出对象（栈空时 new） |
| `Push(T item)` | 归还对象（自动调用 ResetInfo） |
| `Clear()` | 清空池 |
| `Count` | 池中空闲对象数量 |

### IPoolObject 接口

```csharp
public interface IPoolObject
{
    void ResetInfo();  // 归还前的状态清理
}
```

---

## 四、注意事项

1. **归还后不要继续使用对象** — 已归还的对象可能被其他地方取出复用。
2. **资源路径格式** — 使用 `"父文件夹/文件名"` 格式（如 `"Effect/Explosion"`），内部自动转换为 YooAsset 地址。
3. **maxNum 仅首次创建池时生效** — 同一路径第二次 Pop 时传入的 maxNum 不会覆盖已创建的池。
4. **依赖 ResLoaderManager** — 使用 GameObject 池前确保 `ResLoaderManager.Instance.Init()` 已调用。
5. **IPoolObject.ResetInfo() 务必清理委托/事件引用** — 防止内存泄漏。
6. **场景切换时调用 DestroyAll()** — 避免引用已销毁场景中的对象。

---

## 五、编辑器调试

运行时在 Hierarchy 面板中观察 `[GameObjectPoolManager]` 节点：

```
[GameObjectPoolManager]
├── [Pool] Bullet/Arrow [Pool:5 Use:3]      ← 池中5个空闲，3个在外使用
├── [Pool] Effect/Explosion [Pool:0 Use:10]  ← 10个全在使用中
└── [Pool] Sound/HitSE [Pool:2 Use:0]       ← 2个空闲，30秒后自动销毁
```

在 DebugLogSystem 日志模块配置中启用 `Pool` 模块可查看详细日志。
