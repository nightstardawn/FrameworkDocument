# 数据管理与存档系统使用指南

## 概述

本系统由两部分组成：

- **DataCenter**（数据中心）— 管理所有运行时数据模块，提供统一访问入口、新游戏重置、存读档协调
- **SaveManager**（存档管理器）— 负责存档文件的读写，支持多槽位和 PlayerPrefs 后备

---

## 一、快速开始

### 1. 定义数据模块

```csharp
using Dawn.Framework;
using Newtonsoft.Json;

public class PlayerData : BaseData, ISaveable
{
    public int SaveVersion => 1;

    public string Name { get; set; }
    public int Level { get; set; }
    public long Gold { get; set; }

    public override void OnInit()
    {
        Name = "旅行者";
        Level = 1;
        Gold = 0;
    }

    public override void OnClear() => OnInit();

    public string OnCollectSave() => JsonConvert.SerializeObject(this);

    public void OnLoadSave(string json)
    {
        var loaded = JsonConvert.DeserializeObject<PlayerData>(json);
        Name = loaded.Name;
        Level = loaded.Level;
        Gold = loaded.Gold;
    }
}
```

### 2. 访问数据

```csharp
var player = DataCenter.Instance.Get<PlayerData>();
player.Gold += 100;
```

### 3. 存档 / 读档

```csharp
DataCenter.Instance.Save();   // 存到默认槽位 0
DataCenter.Instance.Load();   // 从默认槽位 0 读取
```

### 4. 新游戏

```csharp
DataCenter.Instance.NewGame(); // 清空所有数据，后续 Get<T>() 返回默认值
```

---

## 二、BaseData — 数据模块基类

所有运行时数据类继承 `BaseData`：

```csharp
public abstract class BaseData
{
    public virtual void OnInit() { }     // 首次创建时设置默认值
    public virtual void OnClear() { }    // 新游戏/读档前清空
    public virtual void OnDispose() { }  // 退出游戏时释放资源
}
```

### 生命周期

| 回调 | 触发时机 | 典型用途 |
|------|----------|----------|
| `OnInit()` | `Get<T>()` 首次访问时 | 设置默认值 |
| `OnClear()` | `NewGame()` / `LoadFromSave()` | 重置为初始状态 |
| `OnDispose()` | `Dispose()` 退出游戏时 | 释放非托管资源 |

---

## 三、ISaveable — 可存档接口

**不是所有 BaseData 都需要存档。** 只有实现 `ISaveable` 的模块才参与存读档流程。

```csharp
public interface ISaveable
{
    int SaveVersion { get; }        // 版本号，结构变化时递增
    string OnCollectSave();         // 收集数据 → JSON 字符串
    void OnLoadSave(string json);   // 从 JSON 字符串还原
}
```

**适合实现 ISaveable 的：** 玩家数据、背包、任务进度、技能等需要持久化的数据

**不需要 ISaveable 的：** 临时战斗数据、使用 PlayerPrefs 独立存储的系统设置

---

## 四、DataCenter API 一览

| 方法 | 说明 |
|------|------|
| `Get<T>()` | 获取数据模块（懒加载，首次访问自动创建并 OnInit） |
| `NewGame()` | 新游戏：清空所有模块，后续 Get 返回默认值 |
| `Save(int slotId = 0)` | 保存到指定槽位 |
| `Load(int slotId = 0)` | 从指定槽位读档 |
| `Delete(int slotId)` | 删除指定槽位的存档 |
| `HasSave(int slotId = 0)` | 检查存档是否存在 |
| `GetSaveInfo(int slotId = 0)` | 获取存档元信息 |
| `GetAllSaveInfos()` | 获取所有存档信息列表 |
| `Dispose()` | 退出游戏：OnClear + OnDispose 所有模块 |

---

## 五、SaveManager（内部工具，业务层通常不直接使用）

SaveManager 负责底层文件读写、PlayerPrefs 后备、多槽位管理。业务层通过 DataCenter 的统一入口间接调用。

---

## 六、槽位系统

### 单存档模式（默认）

不传 slotId，所有 API 默认使用槽位 0：

```csharp
DataCenter.Instance.Save();
DataCenter.Instance.Load();
```

### 多槽位模式

```csharp
DataCenter.Instance.Save(1);
DataCenter.Instance.Save(2);

// 展示存档列表
var allSaves = DataCenter.Instance.GetAllSaveInfos();
foreach (var info in allSaves)
    Debug.Log($"槽位 {info.SlotId}: {info.SaveTime}");

// 读取指定槽位
DataCenter.Instance.Load(1);
```

---

## 七、数据流向

```
新游戏:
  DataCenter.Instance.NewGame()
    → 清空所有模块
    → 后续 Get<T>() 自动 OnInit() 创建默认数据

存档:
  DataCenter.Instance.Save(slotId)
    → SaveManager 调用 DataCenter.CollectSaveData(slotId)
      → 遍历所有 ISaveable 模块，各自 OnCollectSave()
    → 组装 GameSaveData → 序列化 → 写入文件

读档:
  DataCenter.Instance.Load(slotId)
    → SaveManager 读取文件 → 反序列化 GameSaveData
    → DataCenter.LoadFromSave(saveData)
      → 清空所有模块
      → 按存档逐模块创建 → OnInit() → OnLoadSave(json)

退出:
  DataCenter.Instance.Dispose()
    → 所有模块 OnClear() + OnDispose()
```

---

## 八、存档文件格式

路径：`Application.persistentDataPath/save_slot_{slotId}.json`

```json
{
  "meta": {
    "slotId": 0,
    "saveTime": "2024-12-01T14:30:45+08:00",
    "timestampUtc": 1733034645,
    "frameworkVersion": "1.0.0"
  },
  "modules": {
    "PlayerData": {
      "typeName": "MyGame.PlayerData",
      "saveVersion": 1,
      "data": "{\"Name\":\"Hero\",\"Level\":25,\"Gold\":99999}"
    },
    "BagData": {
      "typeName": "MyGame.BagData",
      "saveVersion": 1,
      "data": "{\"capacity\":50,\"items\":[...]}"
    }
  }
}
```

每个模块各自序列化为独立 JSON 字符串，互不影响。

---

## 九、版本迁移

### 版本号机制

1. 保存时：`ISaveable.SaveVersion` 写入 `modules[name].saveVersion`
2. 加载时：比较文件中的版本号和代码中的版本号
3. 版本较旧时：输出日志提示，仍正常调用 `OnLoadSave`

### 业务层迁移示例

```csharp
public void OnLoadSave(string json)
{
    var loaded = JsonConvert.DeserializeObject<PlayerData>(json);
    Name = loaded.Name;
    Level = loaded.Level;
    Gold = loaded.Gold;

    // 新增字段自动为默认值，Newtonsoft.Json 处理
    // 如需特殊迁移逻辑，可在此检查并处理
}
```

---

## 十、存储后端

### JSON 文件存储（主）

- 路径：`Application.persistentDataPath`
- 文件名：`save_slot_{slotId}.json`
- 编码：UTF-8

### PlayerPrefs 后备

- 文件 I/O 失败时自动回退
- Key 格式：`Dawn_Save_Slot_{slotId}`
- 限制：部分平台有 1MB 大小限制

---

## 十一、独立测试

1. 创建空 GameObject，挂载 `SaveSystemTest` 脚本
2. Play 运行，检查 Console 输出 7 个测试阶段的结果
3. 检查 `Application.persistentDataPath` 下 JSON 文件

---

## 十二、注意事项

- BaseData 子类必须有无参公开构造函数
- 建议使用 `[JsonProperty]` 标记属性确保序列化名称稳定
- 不要在 BaseData 中存储 UnityEngine.Object 引用（Sprite、Texture 等）
- DataCenter 自动发现所有非抽象 BaseData 子类（反射，仅启动时一次）
- `NewGame()` 后必须重新通过 `Get<T>()` 获取引用，旧引用已失效
- 事件通知由业务层 Manager 负责，DataCenter/BaseData 本身不触发事件
