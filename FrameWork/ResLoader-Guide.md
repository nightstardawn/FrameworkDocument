# ResLoader 资源加载模块使用指南

## 架构总览

```
业务层 (UI / Audio / Battle / 各模块)
    │
ResLoader (每个子系统独立实例，资源隔离，可指定默认 Provider)
    │
ResLoaderManager (全局单例，对象池，定时清理，管理多个 Provider)
    │
IAssetProvider (底层抽象)
    ├── YooAssetProvider (YooAsset 2.3.x)
    └── ResourcesAssetProvider (Resources.LoadAsync)
```

支持 **双 Provider 共存**，通过三级优先级选择使用哪个 Provider：

1. **单次调用指定** — 每个 Load 方法可传入 `providerType` 参数
2. **子加载器默认** — `GetResLoader()` 可指定该加载器的默认 Provider
3. **全局默认** — `ResLoaderManager.Init()` 设定全局默认 Provider

不传 `providerType` 参数时自动使用更高层级的默认值，**完全向后兼容**。

## 快速开始

### 获取加载器

```csharp
using Dawn.Framework;

// 每个子系统获取独立的 ResLoader
var battleLoader = ResLoaderManager.Instance.GetResLoader("Battle");
var audioLoader  = ResLoaderManager.Instance.GetResLoader("Audio");

// 可自定义资源存活宽限时间（默认5秒）
var uiLoader = ResLoaderManager.Instance.GetResLoader("UI", liveTime: 30f);

// 可指定该加载器的默认 Provider（不传则使用全局默认）
var configLoader = ResLoaderManager.Instance.GetResLoader("Config", 30f,
    defaultProvider: E_ASSET_PROVIDER.Resources);
```

### 加载资源

```csharp
// 加载 Prefab（lifeRef 传 this，资源生命周期绑定当前对象）
loader.LoadPrefabAsync(this, "TestSphere", (asset, _) =>
{
    var go = Instantiate(asset as GameObject);
});

// 带自定义数据透传
loader.LoadPrefabAsync(this, "Enemy", (asset, userData) =>
{
    var config = userData as EnemyConfig;
    var go = Instantiate(asset as GameObject);
}, myConfig);

// 显式指定使用 Resources 加载
loader.LoadPrefabAsync(this, "Config/DefaultConfig", (asset, _) =>
{
    // ...
}, providerType: E_ASSET_PROVIDER.Resources);

// 显式指定使用 YooAsset 加载
loader.LoadAudioClipAsync(this, "Audio/BGM_Main", (asset, _) =>
{
    // ...
}, providerType: E_ASSET_PROVIDER.YooAsset);
```

### 释放资源

```csharp
// 释放单个资源
loader.Release("TestSphere");

// 释放此加载器持有的所有资源
loader.ReleaseAll();
```

### 场景切换与内存清理

```csharp
// 切换场景时深度清理（强制清理所有可回收资源）
ResLoaderManager.Instance.OnChangeScene(() =>
{
    // 清理完成回调
});

// 强制内存清理（触发 Resources.UnloadUnusedAssets + GC）
ResLoaderManager.Instance.ForceClearMemory();
```

## 类型化加载方法

| 方法 | 资源类型 |
|------|---------|
| `LoadPrefabAsync` | GameObject |
| `LoadTextureAsync` | Texture2D |
| `LoadSpriteAsync` | Sprite |
| `LoadAudioClipAsync` | AudioClip |
| `LoadMaterialAsync` | Material |
| `LoadScriptableObjectAsync` | ScriptableObject |
| `LoadAsync<T>` | 泛型（任意 UnityEngine.Object） |

所有方法签名一致：

```csharp
void LoadXxxAsync(object lifeRef, string path, Action<UnityEngine.Object, object> onLoaded, object userData = null, E_ASSET_PROVIDER? providerType = null)
```

`providerType` 为可选参数，不传时按三级优先级自动选择 Provider。

## 业务层使用示例

```csharp
public class BattleManager : MonoBehaviour
{
    private ResLoader _loader;

    private void Awake()
    {
        _loader = ResLoaderManager.Instance.GetResLoader("Battle");
    }

    public void SpawnEnemy(string prefabName)
    {
        _loader.LoadPrefabAsync(this, prefabName, (asset, _) =>
        {
            if (asset == null) return;
            Instantiate(asset as GameObject);
        });
    }

    private void OnDestroy()
    {
        _loader.ReleaseAll();
    }
}
```

## 与 UI 框架集成

通过 `ResLoaderUIWindowLoader` 桥接到 `UIManager`，初始化后正常使用 UIManager 即可：

```csharp
UIManager.Instance.ShowWindow<LobbyWindow>();
UIManager.Instance.HideWindow<LobbyWindow>();
```

### PrefabPath 路径转换

所有资源路径统一使用 `"文件夹/文件名"` 格式，`YooAssetProvider` 内部自动将 `/` 转为 `_` 作为 YooAsset 地址：

```csharp
protected override void InitConfig()
{
    // 代码中写路径格式（直觉清晰）
    Config.PrefabPath = "Lobby/LobbyWindow";
    // YooAssetProvider 内部自动转为 YooAsset 地址："Lobby_LobbyWindow"
    // 对应 Prefab 文件名：Lobby_LobbyWindow.prefab
}
```

| 代码中传入路径 | YooAsset 实际地址 | Prefab 文件名 |
|---------------|-----------------|--------------|
| `"Shop/ShopWindow"` | `Shop_ShopWindow` | `Shop_ShopWindow.prefab` |
| `"Battle/HUD"` | `Battle_HUD` | `Battle_HUD.prefab` |
| `"Effect/Explosion"` | `Effect_Explosion` | `Effect_Explosion.prefab` |

路径转换统一在 `YooAssetProvider` 中完成，所有使用 ResLoader 的模块（UIManager、GameObjectPoolManager 等）均无需单独处理。

`ResLoaderUIWindowLoader` 内部持有名为 `"UIWindow"` 的 ResLoader（liveTime=30s），自动管理窗口 Prefab 的加载与释放。

## 核心机制

### WeakReference 生命周期追踪

`lifeRef` 参数是资源的生命周期锚点。传入的对象被 `WeakReference` 持有：

```csharp
// 绑定到当前 MonoBehaviour，对象销毁后资源自动标记为可回收
loader.LoadPrefabAsync(this, "Effect", (asset, _) => { ... });

// 绑定到临时对象
var tempObj = new GameObject("TempOwner");
loader.LoadPrefabAsync(tempObj, "Effect", (asset, _) => { ... });
Destroy(tempObj, 5f);  // 5秒后销毁，资源随后自动清理
```

当所有 `lifeRef` 失效（被销毁/GC）后，资源进入延迟卸载流程。

**Unity Object 特殊处理**：`Destroy()` 后的 Unity 对象在 C# 层仍有引用但 `== null`，框架对此做了专门判断。

### 延迟卸载

资源不会在引用失效后立即卸载，而是经过一段宽限时间（`liveTime`），防止频繁加载/卸载抖动：

1. 首次检测到引用为空 → 记录死亡时间
2. 后续检测 → 判断 `当前时间 - 死亡时间 >= liveTime`
3. 超过宽限期 → 执行真正卸载

`liveTime` 在 `GetResLoader` 时设置（默认 5 秒）。

### 帧预算清理

`ResLoaderManager` 通过 `MonoMgr` 注册 Update 监听，每 **5 秒** 执行一次 `LoopCheck`：

- 单次最多卸载 **20** 个资源，避免卡顿
- 自动清理空的 ResLoader 实例
- 所有清理对用户透明，无需手动触发

### 多订阅者回调队列

同一资源被多处同时请求时，只触发一次实际加载，完成后批量通知：

```csharp
// 两个系统同时请求同一资源，只加载一次
loaderA.LoadPrefabAsync(objA, "SharedPrefab", onLoadedA);
loaderB.LoadPrefabAsync(objB, "SharedPrefab", onLoadedB);
// 加载完成后 onLoadedA 和 onLoadedB 都会被调用
```

### AssetItem 对象池

`AssetItem`（资源跟踪项）由 `ResLoaderManager` 统一通过 Queue 对象池管理，避免频繁 new/GC。

## 初始化配置

### ResLoaderManager 初始化

先通过 `RegisterProvider` 注册需要的 Provider，再调用 `Init()` 完成初始化：

```csharp
// 推荐：注册两个 Provider，开发者按需选择
ResLoaderManager.Instance.RegisterProvider(E_ASSET_PROVIDER.Resources, new ResourcesAssetProvider());
ResLoaderManager.Instance.RegisterProvider(E_ASSET_PROVIDER.YooAsset, new YooAssetProvider("DefaultPackage"));
ResLoaderManager.Instance.Init(E_ASSET_PROVIDER.YooAsset);

// 只注册 YooAsset
ResLoaderManager.Instance.RegisterProvider(E_ASSET_PROVIDER.YooAsset, new YooAssetProvider("DefaultPackage"));
ResLoaderManager.Instance.Init(E_ASSET_PROVIDER.YooAsset);

// 只注册 Resources
ResLoaderManager.Instance.RegisterProvider(E_ASSET_PROVIDER.Resources, new ResourcesAssetProvider());
ResLoaderManager.Instance.Init(E_ASSET_PROVIDER.Resources);

// 不注册任何 Provider，默认使用 ResourcesAssetProvider
ResLoaderManager.Instance.Init();
```

> **注意**：`RegisterProvider()` 必须在 `Init()` 之前调用，Init 后注册会报错。

### E_ASSET_PROVIDER 枚举

```csharp
public enum E_ASSET_PROVIDER
{
    Resources,  // Unity Resources 系统
    YooAsset    // YooAsset 资源包系统
}
```

### IAssetProvider 底层抽象

```csharp
public interface IAssetProvider
{
    void LoadAssetAsync(string path, Type type, Action<UnityEngine.Object> onLoaded);
    void UnloadAsset(string path);
    void UnloadAllAssets();
}
```

**ResourcesAssetProvider**

- 使用 `Resources.LoadAsync` 加载
- 路径格式：Resources 相对路径，不含扩展名（如 `"Test/TestCube"`）
- 内置缓存字典，避免重复加载

**YooAssetProvider**

- 使用 YooAsset 2.3.x `ResourcePackage.LoadAssetAsync` 加载
- 内部自动将路径中的 `/` 转为 `_`（如 `"Effect/Explosion"` → `"Effect_Explosion"`）
- 支持多包：优先从主包加载，未找到则查找其他已初始化的包
- 编辑器下使用 `EditorSimulateMode`，无需打 AB 包

### 生产环境入口（GameStart）

```csharp
public class GameStart : MonoBehaviour
{
    private IEnumerator Start()
    {
        DontDestroyOnLoad(gameObject);

        // 1. YooAsset 初始化（含三步激活 manifest）
        yield return InitYooAsset();

        // 2. 资源管理器（注册 Provider 后初始化）
        ResLoaderManager.Instance.RegisterProvider(E_ASSET_PROVIDER.Resources, new ResourcesAssetProvider());
        ResLoaderManager.Instance.RegisterProvider(E_ASSET_PROVIDER.YooAsset, new YooAssetProvider("DefaultPackage"));
        ResLoaderManager.Instance.Init(E_ASSET_PROVIDER.YooAsset);

        // 3. UI 框架
        UIManager.Instance.Init(new ResLoaderUIWindowLoader());
    }
}
```

## YooAsset 配置指南

### AssetBundle Collector 设置

1. Unity 菜单 → `YooAsset` → `AssetBundle Collector`
2. Package 名设为 `DefaultPackage`
3. 添加 Group 和 Collector：
   - **Collect Path**：资源目录（如 `Assets/Art`）
   - **Collector Type**：`MainAssetCollector`
   - **Address Rule**：`AddressByFileName`（地址为文件名，不含扩展名）
4. 点击右上角 **Save** 保存配置

### YooAsset 初始化三步（缺一不可）

```csharp
// 1. 创建包并初始化
var package = YooAssets.CreatePackage("DefaultPackage");
var initOp = package.InitializeAsync(params);
yield return initOp;

// 2. 请求版本号（激活 manifest 必需）
var versionOp = package.RequestPackageVersionAsync();
yield return versionOp;

// 3. 更新 manifest（激活 manifest 必需）
var manifestOp = package.UpdatePackageManifestAsync(versionOp.PackageVersion);
yield return manifestOp;
```

> **重要**：即使在 EditorSimulateMode 下，第 2、3 步也是必需的。
> 缺少这两步会导致 `Can not found active package manifest!` 异常。
> 参考：[YooAsset FAQ](https://www.yooasset.com/docs/FAQ)

### 地址格式对照

| Address Rule | 代码中传的路径 | 示例 |
|-------------|--------------|------|
| `AddressByFileName` | 文件名（不含扩展名） | `"TestSphere"` |
| `AddressByFolderAndFileName` | 目录/文件名 | `"FrameWorkTest/TestSphere"` |
| `AddressDisable` | 完整资源路径 | `"Assets/Art/FrameWorkTest/TestSphere.prefab"` |

> **UI Prefab 命名约定**：UI 窗口 Prefab 使用 `文件夹_文件名` 命名（如 `Shop_ShopWindow.prefab`），
> 配合 `AddressByFileName`，YooAsset 地址为 `Shop_ShopWindow`。
> 代码中 `Config.PrefabPath = "Shop/ShopWindow"`，`ResLoaderUIWindowLoader` 自动将 `/` 转为 `_`。

## API 一览

### ResLoaderManager（全局单例）

| 方法 | 说明 |
|------|------|
| `RegisterProvider(type, provider)` | 注册 Provider（必须在 Init 之前调用） |
| `Init(defaultProvider)` | 初始化，设定默认 Provider 类型 |
| `GetResLoader(name, liveTime, defaultProvider)` | 获取/创建子系统加载器（可指定默认 Provider） |
| `OnChangeScene(callback)` | 场景切换深度清理（遍历所有已注册 Provider） |
| `ForceClearMemory()` | 强制 GC + UnloadUnusedAssets |

| 属性 | 类型 | 说明 |
|------|------|------|
| `IsInitialized` | bool | 是否已初始化 |
| `DefaultProviderType` | E_ASSET_PROVIDER | 全局默认 Provider 类型 |

### ResLoader（子系统加载器）

| 方法 | 说明 |
|------|------|
| `LoadAsync<T>(lifeRef, path, onLoaded, userData, providerType)` | 泛型异步加载 |
| `LoadPrefabAsync(...)` | 加载 Prefab |
| `LoadTextureAsync(...)` | 加载 Texture2D |
| `LoadSpriteAsync(...)` | 加载 Sprite |
| `LoadAudioClipAsync(...)` | 加载 AudioClip |
| `LoadMaterialAsync(...)` | 加载 Material |
| `LoadScriptableObjectAsync(...)` | 加载 ScriptableObject |
| `Release(path)` | 释放单个资源 |
| `ReleaseAll()` | 释放所有资源 |

| 属性 | 类型 | 说明 |
|------|------|------|
| `LoaderName` | string | 加载器名称 |
| `LiveTime` | float | 资源存活宽限时间（秒） |
| `IsEmpty` | bool | 是否无任何资源在使用 |

## 注意事项

1. **必须先 Init**：在调用 `GetResLoader` 前必须先调用 `ResLoaderManager.Instance.Init()`。
2. **lifeRef 不要传 null**：否则资源无法被自动追踪回收。
3. **回调中检查 null**：加载失败时 asset 为 null，始终做判空处理。
4. **YooAsset 三步初始化**：`InitializeAsync` → `RequestPackageVersionAsync` → `UpdatePackageManifestAsync`，EditorSimulateMode 下也不能省略。
5. **Address Rule 与路径一致**：Collector 中的 Address Rule 决定了代码中传的路径格式。
6. **场景切换时清理**：在切场景时调用 `OnChangeScene()` 防止资源泄漏。
7. **OnDestroy 中释放**：业务系统销毁时应调用 `ReleaseAll()` 主动释放持有的资源。
