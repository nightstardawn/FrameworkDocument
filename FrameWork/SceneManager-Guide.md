# SceneManager 场景管理模块

## 概述

SceneManager 提供协程驱动的异步场景切换，支持：
- 主场景切换（Single 模式）：带过渡动画、进度回调、事件通知、资源清理
- 叠加场景加载/卸载（Additive 模式）：轻量级，无过渡和清理

## 快速开始

### 1. 初始化

在 GameStart 中初始化 SceneManager，注入默认过渡实现：

```csharp
// YourLoadingWindow 是你创建的加载界面，需继承 UIWindow 并实现 ILoadingWindow
SceneManager.Instance.Init(new DefaultSceneTransition<YourLoadingWindow>());
```

### 2. 创建加载窗口

创建一个 UIWindow 子类，实现 ILoadingWindow 接口：

```csharp
public class YourLoadingWindow : UIWindow, ILoadingWindow
{
    // UI 组件引用（由代码生成工具绑定或手动获取）
    private Slider _progressBar;
    private TMP_Text _progressText;

    protected override void InitConfig()
    {
        Config.PrefabPath = "UI/Loading/YourLoadingWindow";
        Config.Layer = UILayer.Loading;        // sortingOrder=80，覆盖所有游戏内容
        Config.AnimType = WindowAnimType.None; // 加载界面建议无动画，立即显示
    }

    public void SetProgress(float progress)
    {
        if (_progressBar != null) _progressBar.value = progress;
        if (_progressText != null) _progressText.text = $"{(int)(progress * 100)}%";
    }
}
```

### 3. 切换场景

```csharp
// 基本用法
SceneManager.Instance.ChangeScene("BattleScene");

// 带完成回调
SceneManager.Instance.ChangeScene("LobbyScene", onComplete: () =>
{
    Debug.Log("场景加载完成");
});

// 使用自定义过渡（临时替换默认过渡）
SceneManager.Instance.ChangeScene("BattleScene", new MyCustomTransition());
```

### 4. Additive 加载

```csharp
// 叠加加载
SceneManager.Instance.LoadSceneAdditive("EnvironmentScene", () =>
{
    Debug.Log("环境场景叠加完成");
});

// 卸载叠加场景
SceneManager.Instance.UnloadSceneAdditive("EnvironmentScene");
```

## 切换流程

主场景切换（ChangeScene）的完整流程：

1. **防重入检查** — 如果正在切换中，忽略本次请求
2. **transition.OnTransitionIn()** — 显示过渡画面（如加载界面淡入）
3. **EventCenter 广播 BeforeSceneUnload** — 各系统收到通知，自行清理
4. **LoadSceneAsync (Single)** — 异步加载新场景（allowSceneActivation=false）
5. **transition.OnProgress()** — 持续更新加载进度 [0, 1]
6. **ResLoaderManager.OnChangeScene()** — 资源深度清理（同步）
7. **激活新场景** — allowSceneActivation = true
8. **EventCenter 广播 AfterSceneLoaded** — 各系统收到通知，执行初始化
9. **transition.OnTransitionOut()** — 隐藏过渡画面

## 事件

在 EventType 中定义，通过 EventCenter 监听：

| 事件 | 触发时机 | 参数类型 | 参数说明 |
|------|---------|---------|---------|
| `EventType.BeforeSceneUnload` | 旧场景卸载前 | `string` | 旧场景名 |
| `EventType.AfterSceneLoaded` | 新场景加载完成后 | `string` | 新场景名 |

监听示例：

```csharp
EventCenter.Instance.AddEvent<string>(EventType.BeforeSceneUnload, sceneName =>
{
    // 旧场景即将卸载，执行清理
});

EventCenter.Instance.AddEvent<string>(EventType.AfterSceneLoaded, sceneName =>
{
    // 新场景已就绪，执行初始化
});
```

## 自定义过渡

实现 ISceneTransition 接口即可：

```csharp
public class FadeTransition : ISceneTransition
{
    public void OnTransitionIn(Action onComplete)
    {
        // 淡入黑屏，完成后调用 onComplete
    }

    public void OnProgress(float progress)
    {
        // 可选：更新进度显示
    }

    public void OnTransitionOut(Action onComplete)
    {
        // 淡出黑屏，完成后调用 onComplete
    }
}
```

## API 参考

### SceneManager（MonoSingleton 单例）

| 方法 | 说明 |
|------|------|
| `Init(ISceneTransition)` | 初始化，注入默认过渡实现 |
| `ChangeScene(sceneName, transition?, onComplete?)` | 主场景切换 |
| `LoadSceneAdditive(sceneName, onComplete?)` | Additive 加载 |
| `UnloadSceneAdditive(sceneName, onComplete?)` | Additive 卸载 |
| `IsLoading` | 是否正在切换中 |
| `CurrentSceneName` | 当前主场景名 |

### ISceneTransition

| 方法 | 说明 |
|------|------|
| `OnTransitionIn(onComplete)` | 显示过渡画面，完成后回调 |
| `OnProgress(progress)` | 更新进度 [0, 1] |
| `OnTransitionOut(onComplete)` | 隐藏过渡画面，完成后回调 |

### ILoadingWindow

| 方法 | 说明 |
|------|------|
| `SetProgress(progress)` | 更新加载进度 [0, 1] |
