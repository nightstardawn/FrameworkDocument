# DebugLogSystem 日志模块使用指南

## 快速开始

### 基本调用

```csharp
using Dawn.Framework;

DebugLogSystem.Log(E_LOG_MODULE._default_, "游戏初始化完成");
DebugLogSystem.LogWarning(E_LOG_MODULE.Network, "连接超时");
DebugLogSystem.LogError(E_LOG_MODULE.UI, "找不到预制体");
```

### 输出格式

```
[14:32:05][_default_] 游戏初始化完成
[14:32:06][Network] 连接超时
[14:32:07][UI] 找不到预制体
```

时间格式可在编辑器面板中切换为时间戳：

```
[1714380725][_default_] 游戏初始化完成
```

## 模块枚举

```csharp
[Flags]
public enum E_LOG_MODULE
{
    _default_ = 1 << 0,
    UI        = 1 << 1,
    Network   = 1 << 2,
    Audio     = 1 << 3,
    Timer     = 1 << 4,
    Event     = 1 << 5,
    Res       = 1 << 6,
    Pool      = 1 << 7,
}
```

扩展新模块时，在枚举中追加新值即可，面板会自动显示。

## 编辑器配置面板

菜单路径：`全局配置控制 → 日志模块配置`

面板功能：
- **时间显示格式**：下拉框选择 `Time`（HH:mm:ss）或 `Timestamp`（Unix 时间戳）
- **模块开关**：勾选框控制每个模块的日志是否输出到 Console
- **全选 / 全不选**：快速切换所有模块
- **保存**：点击后配置才会生效并持久化

配置存储在本地 EditorPrefs 中，不会进入版本控制，不影响团队其他成员。

## API 一览

### DebugLogSystem（静态类）

| 方法 | 说明 |
|------|------|
| `Log(E_LOG_MODULE module, string message)` | 输出普通日志 |
| `LogWarning(E_LOG_MODULE module, string message)` | 输出警告日志 |
| `LogError(E_LOG_MODULE module, string message)` | 输出错误日志 |

### E_LOG_TIME_FORMAT（时间格式枚举）

| 值 | 说明 |
|------|------|
| `Time` | HH:mm:ss 格式（如 14:32:05） |
| `Timestamp` | Unix 时间戳（如 1714380725） |

## 打包剥离

所有 `DebugLogSystem` 方法均标记 `[Conditional("UNITY_EDITOR")]`：

- **Editor 模式**：正常输出日志
- **打包后**：编译器自动剥离所有调用点，零运行时开销，无需手动处理

## 扩展新模块

1. 在 `E_LOG_MODULE` 枚举中追加新值：

```csharp
public enum E_LOG_MODULE
{
    // ... 已有模块
    MyModule = 1 << 8,  // 新增（下一个可用位）
}
```

2. 回到编辑器，打开 `全局配置控制 → 日志模块配置`，新模块会自动出现在列表中。

## 注意事项

1. **自动初始化**：`LogEditorConfig` 通过 `[InitializeOnLoadMethod]` 在编辑器加载时自动注册到 DebugLogSystem，无需手动调用 Init。
2. **配置不共享**：基于 EditorPrefs 存储，每位开发者独立配置，不进入 Git。
3. **Flags 枚举**：新模块值必须使用 `1 << N` 形式，不可使用递增整数。
4. **保存才生效**：面板中修改后必须点击"保存"按钮，否则关闭窗口后修改丢失。
