# AudioManager 使用指南

## 概述

`AudioManager` 是 Dawn.Framework 提供的全局音频管理器（MonoSingleton），统一管理游戏中的 **BGM**（背景音乐）、**SFX**（音效）和 **Ambient**（环境音）三类音频。

核心特性：
- BGM 支持淡入淡出、CrossFade 切换、暂停/恢复
- BGM 叠加层系统，可在主 BGM 基础上叠加额外音轨
- SFX 并发控制，同一音效限制最大同时播放数
- Ambient 环境音循环播放，支持多路叠加
- 分组音量控制（Master / BGM / SFX / Ambient），支持静音
- 音量设置自动持久化到 PlayerPrefs

## 前置条件

`AudioManager` 依赖 `ResLoaderManager` 进行音频资源加载。使用前请确保：

```csharp
// 初始化资源加载器（二选一）
ResLoaderManager.Instance.Init(new ResourcesAssetProvider());
// 或
ResLoaderManager.Instance.Init(new YooAssetProvider(packageName));
```

`AudioManager` 首次访问 `Instance` 时会自动初始化，无需手动创建。

## 快速开始

### 播放 BGM

```csharp
// 播放背景音乐（循环，0.5秒淡入）
AudioManager.Instance.PlayBGM("Audio/BGM_Main");

// 自定义参数
AudioManager.Instance.PlayBGM("Audio/BGM_Main", loop: true, fadeIn: 1f);
```

> 如果传入的路径与当前正在播放的 BGM 相同，调用会被忽略。

### 切换 BGM（CrossFade）

```csharp
// 当前 BGM 淡出 0.5秒，新 BGM 淡入 0.5秒（并行进行）
AudioManager.Instance.SwitchBGM("Audio/BGM_Battle");

// 自定义淡出和淡入时长
AudioManager.Instance.SwitchBGM("Audio/BGM_Battle", fadeOut: 1f, fadeIn: 1f);
```

### 停止 BGM

```csharp
// 淡出后停止
AudioManager.Instance.StopBGM();

// 自定义淡出时长
AudioManager.Instance.StopBGM(fadeOut: 1f);
```

### 暂停 / 恢复 BGM

```csharp
// 暂停（同时暂停所有 BGM 叠加层）
AudioManager.Instance.PauseBGM();

// 恢复
AudioManager.Instance.ResumeBGM();
```

## BGM 叠加层

叠加层允许在主 BGM 之上添加额外音轨（如战斗时叠加鼓点、进入特殊区域叠加旋律等）。

```csharp
// 添加叠加层，返回层 ID
int layerId = AudioManager.Instance.AddBGMLayer("Audio/BGM_Layer_Drums", loop: true, fadeIn: 0.3f);

// 移除指定叠加层
AudioManager.Instance.RemoveBGMLayer(layerId, fadeOut: 0.3f);

// 移除所有叠加层
AudioManager.Instance.RemoveAllBGMLayers(fadeOut: 0.3f);
```

叠加层与主 BGM 共享音量组（`E_AudioGroup.BGM`），受 PauseBGM/ResumeBGM 统一控制。

## 播放音效 SFX

```csharp
// 播放一次性音效（默认最大并发 3）
AudioManager.Instance.PlaySFX("Audio/SFX_Click");

// 自定义最大并发数
AudioManager.Instance.PlaySFX("Audio/SFX_Explosion", maxConcurrent: 5);
```

### 并发控制说明

`maxConcurrent` 限制同一路径音效的最大同时播放数：
- 当同一音效正在播放的数量 < maxConcurrent 时，正常播放新实例
- 当达到上限时，复用最早播放的 AudioSource（重头开始播放）
- 播放完毕后 AudioSource 自动回收到空闲池

这可以有效避免"机关枪效果"（大量相同音效同时播放导致音量叠加爆炸）。

## 环境音 Ambient

环境音用于持续循环播放的背景音（如森林鸟鸣、河流水声等），支持多路同时播放。

```csharp
// 播放环境音（循环，1秒淡入），返回环境音 ID
int ambientId = AudioManager.Instance.PlayAmbient("Audio/Ambient_Forest", fadeIn: 1f);

// 停止指定环境音
AudioManager.Instance.StopAmbient(ambientId, fadeOut: 1f);

// 停止所有环境音
AudioManager.Instance.StopAllAmbient(fadeOut: 1f);
```

## 音量控制

### 设置和获取音量

```csharp
// 设置音量（0~1）
AudioManager.Instance.SetVolume(E_AudioGroup.Master, 0.8f);
AudioManager.Instance.SetVolume(E_AudioGroup.BGM, 0.6f);
AudioManager.Instance.SetVolume(E_AudioGroup.SFX, 1f);
AudioManager.Instance.SetVolume(E_AudioGroup.Ambient, 0.5f);

// 获取音量
float bgmVol = AudioManager.Instance.GetVolume(E_AudioGroup.BGM);
```

### 静音控制

```csharp
// 设置静音
AudioManager.Instance.SetMute(E_AudioGroup.BGM, true);

// 查询静音状态
bool isMuted = AudioManager.Instance.IsMuted(E_AudioGroup.BGM);
```

### 音量计算规则

实际音量 = Master音量 x 分组音量 x 静音系数

具体公式（参见 `AudioVolumeStore.GetEffectiveVolume`）：

```
effectiveVolume = IsMuted(Master) || IsMuted(group) ? 0 : Master * GroupVolume
```

- 当 Master 或对应分组任一被静音时，实际音量为 0
- 修改 Master 音量会立即影响所有分组
- 修改某分组音量只影响该分组

## 全局控制

```csharp
// 暂停所有音频（BGM + BGM叠加层 + SFX + Ambient）
AudioManager.Instance.PauseAll();

// 恢复所有音频
AudioManager.Instance.ResumeAll();

// 停止所有音频并清理所有状态
AudioManager.Instance.StopAll();
```

> `StopAll()` 会彻底清理所有播放状态（包括 BGM 叠加层、SFX 空闲池等），适用于场景切换时的完全重置。

## 音量分组表格

| 分组 | 枚举值 | 影响范围 | 说明 |
|------|--------|----------|------|
| Master | `E_AudioGroup.Master` | 全局 | 总音量，影响所有子分组 |
| BGM | `E_AudioGroup.BGM` | 主BGM + BGM叠加层 | 背景音乐音量 |
| SFX | `E_AudioGroup.SFX` | 所有音效 | 音效音量 |
| Ambient | `E_AudioGroup.Ambient` | 所有环境音 | 环境音音量 |

## 持久化说明

音量设置通过 `AudioVolumeStore` 自动保存到 **PlayerPrefs**，与游戏存档（DataCenter/SaveManager）独立。

- 保存时机：每次调用 `SetVolume()` 或 `SetMute()` 时立即写入 PlayerPrefs
- 加载时机：`AudioManager` 初始化时自动调用 `AudioVolumeStore.LoadAll()`
- 存储 Key：`AudioVolume_{分组名}` / `AudioMute_{分组名}`
- 默认值：音量 1.0，静音 false

> 因为是 PlayerPrefs 存储，音量偏好属于"玩家全局设置"，不随存档槽位变化。

## 资源路径说明

音频资源路径格式与 ResLoader 一致：

- **Resources 模式**：相对于 `Resources/` 文件夹的路径，不含扩展名
  - 例：`Resources/Audio/BGM_Main.mp3` → 路径为 `"Audio/BGM_Main"`
- **YooAsset 模式**：配置在 AssetBundle Collector 中的资源路径
  - YooAssetProvider 内部会将 `/` 转换为 `_` 作为资源定位地址

AudioManager 内部使用独立的 ResLoader 实例（名称 "Audio"，延迟释放 30 秒），无需外部管理资源生命周期。
