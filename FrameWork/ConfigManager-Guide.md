# 配置表管理系统使用指南

## 概述

配置表系统负责加载游戏策划数据（数值表、道具表、关卡表等），使用 JSON 格式存储，通过 YooAsset 体系加载，支持热更新。

系统包含两部分：
- **ConfigManager** — 运行时配置加载与查询
- **Excel 转 JSON 工具** — 编辑器内 Excel → JSON + C# 数据类一键转换

---

## 一、添加新配置表的流程

### 1. 编写 Excel 文件

在 `Assets/GameData/Excel/` 的**子文件夹**中创建 `.xlsx` 文件。

由于 YooAsset 使用「父文件夹名 + 文件名」拼接作为寻址 key，JSON 文件必须放在子文件夹中以区分不同配置表。转换工具会自动镜像 Excel 的目录结构到 JSON 和 C# 输出路径。

```
Assets/GameData/Excel/
├── Item/
│   └── ItemConfig.xlsx
├── Skill/
│   └── SkillLevelConfig.xlsx
└── Monster/
    └── MonsterConfig.xlsx
```

### 2. Excel 表头格式（5行）

| 行号 | 内容 | 说明 |
|------|------|------|
| 第1行 | 字段描述 | 中文说明，程序不读取 |
| 第2行 | 字段名 | C# 属性名（PascalCase） |
| 第3行 | 数据类型 | 见下方类型列表 |
| 第4行 | 键标记 | `key` / `key1` / `key2` / 空 |
| 第5行起 | 数据 | 实际配置数据 |

### 3. 运行转换工具

菜单 `Tools/配置表工具/Excel 转 JSON`，点击"转换全部"。

工具会递归扫描 Excel 文件夹中所有子文件夹的 `.xlsx` 文件，并镜像目录结构输出：
- `Assets/GameData/Json/Item/ItemConfig.json`
- `Assets/ScriptGenerate/Game/Data/Item/ItemConfig.cs`

### 4. YooAsset Collector 设置

在 YooAsset 的 AssetBundle Collector 中添加：
- 收集路径：`Assets/GameData/Json`
- 寻址规则：选择包含父文件夹名的寻址模式
- 启用递归收集子文件夹

YooAsset 寻址映射规则：ConfigPath `"Item/ItemConfig"` → YooAsset 地址 `"Item_ItemConfig"`（`/` 替换为 `_`）。

### 5. 在代码中使用

```csharp
// 加载
ConfigManager.Instance.LoadConfigAsync<Config.ItemConfig>();

// 查询
var item = ConfigManager.Instance.GetConfig<Config.ItemConfig>(1001);
```

---

## 二、支持的数据类型

### 基础类型

| 类型名 | C# 类型 | Excel 填写示例 |
|--------|---------|----------------|
| `int` | int | `100` |
| `long` | long | `9999999999` |
| `float` | float | `3.14` |
| `bool` | bool | `0` 或 `1` |
| `string` | string | `你好` |

### 数组类型（元素用 `|` 分隔）

| 类型名 | C# 类型 | Excel 填写示例 |
|--------|---------|----------------|
| `array_int` | int[] | `1|2|3` |
| `array_long` | long[] | `100|200|300` |
| `array_float` | float[] | `0.1|0.2|0.3` |
| `array_string` | string[] | `苹果|香蕉|橘子` |

### 向量类型（分量用 `;` 分隔）

| 类型名 | C# 类型 | Excel 填写示例 |
|--------|---------|----------------|
| `vector2_int` | Vector2Int | `2;3` |
| `vector3_int` | Vector3Int | `2;3;4` |
| `vector2_float` | Vector2 | `0.5;1.0` |
| `vector3_float` | Vector3 | `0.5;1.0;2.0` |

### 向量数组类型（向量间用 `|`，分量用 `;`）

| 类型名 | C# 类型 | Excel 填写示例 |
|--------|---------|----------------|
| `vector2_array_int` | Vector2Int[] | `1;2|3;4|5;6` |
| `vector3_array_int` | Vector3Int[] | `1;2;3|4;5;6` |
| `vector2_array_float` | Vector2[] | `0.1;0.2|0.3;0.4` |
| `vector3_array_float` | Vector3[] | `0.1;0.2;0.3|0.4;0.5;0.6` |

---

## 三、键标记规则

### 单键表

第4行中只有一列标记 `key`，该列为主键（int 类型）。

```
| 道具ID | 名称 | 价格 |
| Id     | Name | Price|
| int    | string | int |
| key    |        |     |
| 1001   | 铁剑  | 100  |
```

### 双键表

第4行中两列分别标记 `key1` 和 `key2`，组合为复合主键。

```
| 技能ID  | 等级  | 伤害值 |
| SkillId | Level | Damage |
| int     | int   | int    |
| key1    | key2  |        |
| 101     | 1     | 50     |
| 101     | 2     | 80     |
```

---

## 四、ConfigManager API

### 加载

```csharp
// 加载单张表
ConfigManager.Instance.LoadConfigAsync<Config.ItemConfig>((success) => { });

// 加载全部已注册表（适合游戏启动时）
ConfigManager.Instance.LoadAllAsync(
    (current, total) => { /* 进度回调 */ },
    () => { /* 全部完成 */ }
);
```

### 查询（单键表）

```csharp
// 按 Id 获取
var item = ConfigManager.Instance.GetConfig<Config.ItemConfig>(1001);

// 安全获取
if (ConfigManager.Instance.TryGetConfig<Config.ItemConfig>(1001, out var item))
{ }

// 获取全部
var allItems = ConfigManager.Instance.GetAllConfigs<Config.ItemConfig>();
```

### 查询（双键表）

```csharp
// 按 Key1 + Key2 获取
var skill = ConfigManager.Instance.GetConfig<Config.SkillLevelConfig>(101, 2);

// 获取某个 Key1 下的所有数据
var allLevels = ConfigManager.Instance.GetConfigsByKey1<Config.SkillLevelConfig>(101);
```

### 状态管理

```csharp
// 是否已加载
bool loaded = ConfigManager.Instance.IsLoaded<Config.ItemConfig>();

// 卸载
ConfigManager.Instance.UnloadConfig<Config.ItemConfig>();
```

---

## 五、特殊规则

- 字段名以 `#` 开头的列会被跳过（策划备注列）
- 空单元格：数值=0，字符串=""，数组=空数组
- Excel 临时文件（`~$` 开头）自动跳过
- 配置类通过 `[ConfigPath]` Attribute 自动注册，无需手动代码注册

---

## 六、目录结构与 YooAsset 寻址

### 文件组织

```
Assets/GameData/Excel/           ← Excel 源文件（不打包）
├── Item/
│   └── ItemConfig.xlsx
├── Skill/
│   └── SkillLevelConfig.xlsx
└── Monster/
    └── MonsterConfig.xlsx

Assets/GameData/Json/            ← 生成的 JSON（YooAsset 收集）
├── Item/
│   └── ItemConfig.json
├── Skill/
│   └── SkillLevelConfig.json
└── Monster/
    └── MonsterConfig.json

Assets/ScriptGenerate/Game/Data/ ← 生成的 C# 数据类
├── Item/
│   └── ItemConfig.cs
├── Skill/
│   └── SkillLevelConfig.cs
└── Monster/
    └── MonsterConfig.cs
```

### 寻址映射

转换工具自动为 C# 数据类生成 `[ConfigPath("子文件夹/文件名")]`。运行时 ConfigManager 通过 ResLoader 加载时，YooAssetProvider 会将路径中的 `/` 替换为 `_` 作为 YooAsset 的寻址地址：

| ConfigPath | YooAsset 地址 | 对应文件 |
|------------|---------------|----------|
| `Item/ItemConfig` | `Item_ItemConfig` | `Json/Item/ItemConfig.json` |
| `Skill/SkillLevelConfig` | `Skill_SkillLevelConfig` | `Json/Skill/SkillLevelConfig.json` |

### 多级子文件夹

支持多级嵌套，例如 `Excel/Battle/Skill/SkillConfig.xlsx` 会生成：
- JSON: `Json/Battle/Skill/SkillConfig.json`
- C# 类: `Data/Battle/Skill/SkillConfig.cs`
- ConfigPath: `"Battle/Skill/SkillConfig"`
- YooAsset 地址: `"Battle_Skill_SkillConfig"`

---

## 七、工具配置

菜单 `Tools/配置表工具/Excel 转 JSON` 打开工具面板，可配置：
- Excel 源文件夹路径（默认 `Assets/GameData/Excel`）
- JSON 输出路径（默认 `Assets/GameData/Json`）
- C# 数据类输出路径（默认 `Assets/ScriptGenerate/Game/Data`）
- 生成的数据类命名空间（默认 `Config`）

转换时工具会递归扫描 Excel 文件夹下所有子文件夹中的 `.xlsx` 文件，并将目录结构同步镜像到 JSON 和 C# 输出目录。

点击"重置路径"可将所有路径恢复为默认值。

---

## 八、独立测试

框架提供了独立测试脚本 `Assets/External/GameFrame/frame/Config/Example/ConfigSystemTest.cs`。

使用方式：
1. 在场景中创建空 GameObject，挂载 `ConfigSystemTest` 脚本
2. 确保已通过 Excel 工具生成 JSON 和 C# 数据类
3. 确保 YooAsset Collector 已配置 JSON 目录
4. 运行场景

测试脚本会自动完成 YooAsset 和 ResLoaderManager 初始化（无需依赖 GameStart），然后依次测试：
- 自动发现配置表数量
- 加载全部配置表（含进度输出）
- 单键表查询
- 双键表查询

所有日志输出使用 `DebugLogSystem.Log(E_LOG_MODULE.Config, ...)`，可通过日志模块配置面板控制是否显示。

---

## 九、生成的 C# 类示例

### 单键表

```csharp
// Auto-Generated. Do not edit.
using Newtonsoft.Json;
using Dawn.Framework;

namespace Config
{
    [ConfigPath("Item/ItemConfig")]
    public class ItemConfig : IConfigData
    {
        [JsonProperty("Id")]
        public int Id { get; set; }

        [JsonProperty("Name")]
        public string Name { get; set; }

        [JsonProperty("Price")]
        public int Price { get; set; }

        [JsonProperty("Tags")]
        public int[] Tags { get; set; }
    }
}
```

### 双键表

```csharp
// Auto-Generated. Do not edit.
using Newtonsoft.Json;
using Dawn.Framework;

namespace Config
{
    [ConfigPath("Skill/SkillLevelConfig")]
    public class SkillLevelConfig : IDualKeyConfigData
    {
        [JsonProperty("SkillId")]
        public int SkillId { get; set; }

        [JsonProperty("Level")]
        public int Level { get; set; }

        [JsonProperty("Damage")]
        public int Damage { get; set; }

        public int Key1 => SkillId;
        public int Key2 => Level;
    }
}
```
