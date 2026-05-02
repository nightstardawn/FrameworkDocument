# UI 框架使用指南

## 快速开始

### 1. 初始化框架（这部分框架入口会在游戏开始阶段初始化好，业务逻辑上不需要处理）

在场景启动脚本中调用一次：

```csharp
// 使用 YooAsset 加载 UI Prefab（推荐）
ResLoaderManager.Instance.Init(new YooAssetProvider("DefaultPackage"));
UIManager.Instance.Init(new ResLoaderUIWindowLoader());

// 或使用 Resources 加载（无需额外配置）
UIManager.Instance.Init();
```

框架会自动创建 UIRoot Canvas、所有层级节点、全屏遮罩和 EventSystem。

### 2. 创建一个窗口

**三步走：做 Prefab → 生成/写脚本   → 打开窗口。**

```csharp
public class ShopWindow : UIWindow
{
    [SerializeField] private Button _btnClose;

    protected override void InitConfig()
    {
        Config.PrefabPath = "Shop/ShopWindow";    // 路径格式：文件夹/文件名
        Config.Layer      = UILayer.Normal;
        Config.Style      = WindowStyle.FixSize;
        Config.AnimType   = WindowAnimType.Scale;
        Config.MaskStyle  = MaskStyle.Black;
    }

    protected override void OnInitWindow(IUIWindowParam param)
    {
        _btnClose.onClick.AddListener(() => UIManager.Instance.HideWindow<ShopWindow>());
    }

    protected override void OnShow(IUIWindowParam param)
    {
        // 刷新 UI
    }
}
```

Prefab 命名为 `Shop_ShopWindow.prefab`，放在 YooAsset Collector 收集范围内的任意目录。

打开：

```csharp
UIManager.Instance.ShowWindow<ShopWindow>();
```

关闭：

```csharp
UIManager.Instance.HideWindow<ShopWindow>();
```

---

## 窗口配置（InitConfig）

在 `InitConfig()` 中设置以下字段：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| PrefabPath | string | null | **必填**，路径格式如 `"Shop/ShopWindow"`，内部自动转为资源地址 |
| Layer | int | UILayer.Normal | 窗口所在层级 |
| Style | WindowStyle | FixSize | 布局风格 |
| AnimType | WindowAnimType | None | 进出场动画 |
| MaskStyle | MaskStyle | None | 背后遮罩类型 |
| MaskAlpha | float | 0.6f | 遮罩透明度（0=全透明, 1=全黑） |
| ClickMaskToClose | bool | true | 点击遮罩是否关闭窗口 |
| AnimDuration | float | 0.25f | 动画时长（秒） |
| OpenSound | string | null | 打开音效 ID（可为空） |

### 布局风格（WindowStyle）

| 值 | 效果 |
|----|------|
| FullScreen | 全屏拉伸撑满父节点 |
| FixSize | 固定尺寸居中，保持 Prefab 中设计的宽高 |
| FullScreenPopup | 同 FullScreen 拉伸，逻辑上视为弹窗 |
| Custom | 不修改 anchor，完全由 Prefab 决定 |

### 动画类型（WindowAnimType）

所有动画遵循**从哪来回哪去**原则：

| 值 | 进场 | 退场 |
|----|------|------|
| None | 立即显示 | 立即隐藏 |
| Fade | 透明度 0→1 | 透明度 1→0 |
| Scale | 缩放 0→1（带回弹） | 缩放 1→0 |
| SlideUp | 从下方滑入 | 回到下方 |
| SlideDown | 从上方滑入 | 回到上方 |
| SlideLeft | 从左侧滑入 | 回到左侧 |
| SlideRight | 从右侧滑入 | 回到右侧 |

### 遮罩类型（MaskStyle）

| 值 | 效果 |
|----|------|
| None | 无遮罩，下层可点击 |
| Black | 黑色半透明遮罩 |
| Transparent | 透明但拦截点击（防穿透） |

设置 `ClickMaskToClose = false` 可实现模态弹窗效果（必须点关闭按钮才能关）。

---

## 窗口层级（UILayer）

从下到上渲染顺序：

| 常量 | 值 | 用途 |
|------|----|------|
| BackGroup | 0 | 背景层（背景图、场景 UI） |
| Main | 10 | 主界面层 |
| Normal | 20 | 普通弹窗层（大部分弹窗在这里） |
| TopResource | 30 | 顶部资源浮层（金币、体力显示） |
| LoadingMask | 50 | 加载遮罩（网络请求时） |
| PopLayer | 60 | 非窗口弹出（飘字、飞道具特效） |
| GuideForbid | 70 | 引导屏蔽层 |
| Loading | 80 | 全屏 Loading |
| Guide | 90 | 新手引导层 |
| Network | 100 | 网络提示层（断网弹窗） |
| TopMost | 110 | 最顶层（强制更新弹窗） |

---

## 窗口参数传递

需要传参时，创建一个实现 `IUIWindowParam` 的类：

```csharp
public class ShopWindowParam : IUIWindowParam
{
    public int TabIndex;
    public string ItemId;
}
```

传参打开：

```csharp
UIManager.Instance.ShowWindow<ShopWindow>(new ShopWindowParam
{
    TabIndex = 2,
    ItemId = "sword_01"
});
```

窗口内接收：

```csharp
protected override void BeforeShow(IUIWindowParam param)
{
    if (param is ShopWindowParam p)
    {
        SwitchTab(p.TabIndex);
        HighlightItem(p.ItemId);
    }
}
```

不需要传参时直接调用 `ShowWindow<T>()` 即可，param 为 null。

---

## 窗口生命周期

```
[首次创建]
  InitConfig()              ← 填写配置（必须实现）
  AutoBindComponents()      ← 自动绑定控件引用（由生成工具 override）
  OnInitWindow(param)       ← 绑定按钮、获取组件引用（只调用一次）

[每次显示]
  BeforeShow(param)         ← 接收参数，刷新数据
  OnShow(param)             ← 窗口已可见，注册事件监听

[重新获得焦点]
  OnFocus(param)            ← 上层窗口关闭后回到栈顶

[每次隐藏]
  AfterHide(param)          ← 移除事件监听、停止计时器

[销毁]
  OnDispose(param)          ← 释放资源（仅主动销毁时调用）
```

### 关键区别：关闭 vs 销毁

- **HideWindow**：`SetActive(false)` + 缓存实例，下次打开直接复用，不触发 `OnDispose`
- **DestroyWindow**：销毁 GameObject + 清除缓存，触发 `OnDispose`

因此：

- `OnInitWindow` 中绑定按钮事件**不需要**在关闭时移除，实例不销毁，下次复用
- 需要在每次隐藏时清理的逻辑（如事件监听、定时器），放在 `AfterHide` 中
- 需要在每次显示时刷新的逻辑，放在 `BeforeShow` 或 `OnShow` 中

### 各生命周期的推荐用法

| 方法 | 适合做什么 | 不适合做什么 |
|------|-----------|-------------|
| InitConfig | 填写配置 | 任何逻辑代码 |
| OnInitWindow | 绑按钮、GetComponent | 刷新数据（只调用一次） |
| BeforeShow | 接收参数、准备数据 | 操作 UI 显示（窗口尚未可见） |
| OnShow | 刷新 UI、注册事件 | — |
| OnFocus | 刷新可能变化的数据 | 重新绑定按钮 |
| AfterHide | 注销事件、停定时器 | 释放资源（窗口还在缓存中） |
| OnDispose | 释放资源 | 通常不需要写 |

---

## 常用 API

### 打开 / 关闭

```csharp
// 打开（无参数）
UIManager.Instance.ShowWindow<ShopWindow>();

// 打开（带参数）
UIManager.Instance.ShowWindow<ShopWindow>(new ShopWindowParam { TabIndex = 1 });

// 打开（带回调，在显示前设置数据）
UIManager.Instance.ShowWindow<ShopWindow>(window =>
{
    window.SetData(someData);
});

// 关闭
UIManager.Instance.HideWindow<ShopWindow>();
```

### 查询

```csharp
// 是否正在显示
bool visible = UIManager.Instance.IsWindowVisible<ShopWindow>();

// 获取窗口实例（可能为 null）
var window = UIManager.Instance.GetWindow<ShopWindow>();
```

### 销毁

```csharp
// 销毁单个窗口（释放内存）
UIManager.Instance.DestroyWindow<ShopWindow>();

// 关闭 Normal 层所有弹窗并销毁（返回主界面时用）
UIManager.Instance.BackToMainMenu();

// 关闭并销毁所有窗口（切大场景时用）
UIManager.Instance.CloseAllWindows();
```

### Loading 和等待遮罩

```csharp
// 全屏 Loading
UIManager.Instance.StartLoading();
UIManager.Instance.StopLoading();

// 等待遮罩（网络请求时阻止用户操作，支持引用计数）
UIManager.Instance.ShowWaitMask();   // 计数 +1
UIManager.Instance.HideWaitMask();   // 计数 -1，归零时真正隐藏
```

---

## 典型场景示例

### 全屏主界面

```csharp
public class MainWindow : UIWindow
{
    [SerializeField] private Button _btnShop;

    protected override void InitConfig()
    {
        Config.PrefabPath = "Main/MainWindow";
        Config.Layer      = UILayer.Main;
        Config.Style      = WindowStyle.FullScreen;
        Config.AnimType   = WindowAnimType.Fade;
        Config.MaskStyle  = MaskStyle.None;
    }

    protected override void OnInitWindow(IUIWindowParam param)
    {
        _btnShop.onClick.AddListener(() =>
            UIManager.Instance.ShowWindow<ShopWindow>());
    }
}
```

### 居中弹窗（带参数 + 遮罩）

```csharp
public class ItemDetailParam : IUIWindowParam
{
    public int ItemId;
}

public class ItemDetailWindow : UIWindow
{
    [SerializeField] private Button _btnClose;
    [SerializeField] private TMP_Text _txtName;

    protected override void InitConfig()
    {
        Config.PrefabPath      = "ItemDetail/ItemDetailWindow";
        Config.Layer            = UILayer.Normal;
        Config.Style            = WindowStyle.FixSize;
        Config.AnimType         = WindowAnimType.Scale;
        Config.MaskStyle        = MaskStyle.Black;
        Config.ClickMaskToClose = true;
    }

    protected override void OnInitWindow(IUIWindowParam param)
    {
        _btnClose.onClick.AddListener(() =>
            UIManager.Instance.HideWindow<ItemDetailWindow>());
    }

    protected override void BeforeShow(IUIWindowParam param)
    {
        if (param is ItemDetailParam p)
            _txtName.text = GetItemName(p.ItemId);
    }
}
```

### 侧边栏滑入面板

```csharp
public class SettingsWindow : UIWindow
{
    protected override void InitConfig()
    {
        Config.PrefabPath = "Settings/SettingsWindow";
        Config.Layer      = UILayer.Normal;
        Config.Style      = WindowStyle.Custom;
        Config.AnimType   = WindowAnimType.SlideRight;
        Config.MaskStyle  = MaskStyle.Transparent;
    }
}
```

### 模态弹窗（必须点按钮关闭）

```csharp
protected override void InitConfig()
{
    Config.MaskStyle        = MaskStyle.Black;
    Config.ClickMaskToClose = false;  // 点击遮罩不关闭
}
```

---

## 资源加载器（IUIWindowLoader）

### PrefabPath 与资源地址的关系

代码中 `Config.PrefabPath` 使用路径格式书写（`"文件夹/文件名"`），`YooAssetProvider` 内部自动将 `/` 转换为 `_` 后作为 YooAsset 资源地址：

| Config.PrefabPath | 转换后资源地址 | 对应 Prefab 文件名 |
|-------------------|--------------|-------------------|
| `"Shop/ShopWindow"` | `Shop_ShopWindow` | `Shop_ShopWindow.prefab` |
| `"Battle/HUD"` | `Battle_HUD` | `Battle_HUD.prefab` |
| `"Test/TestPanel"` | `Test_TestPanel` | `Test_TestPanel.prefab` |

路径转换统一在 `YooAssetProvider` 中完成，所有使用 ResLoader 的模块（UIManager、GameObjectPoolManager 等）均无需单独处理。

### 默认加载器

不传参调用 `UIManager.Instance.Init()` 时使用 `ResourcesUIWindowLoader`，从 `Resources/` 目录加载。

### ResLoaderUIWindowLoader（推荐）

通过 ResLoader 资源管理系统加载 UI Prefab，支持 YooAsset：

```csharp
// 初始化
ResLoaderManager.Instance.Init(new YooAssetProvider("DefaultPackage"));
UIManager.Instance.Init(new ResLoaderUIWindowLoader());
```

内部持有名为 `"UIWindow"` 的 ResLoader（liveTime=30s），自动管理 Prefab 缓存与释放。

### 自定义加载器

实现 `IUIWindowLoader` 接口即可替换为任意加载方案：

```csharp
public interface IUIWindowLoader
{
    void LoadWindow(string prefabPath, Action<GameObject> onLoaded);
    void UnloadWindow(string prefabPath);
}
```

---

## UIModule（窗口子模块）

当窗口逻辑变复杂时，可以用 UIModule 把一组控件封装为独立模块，由 Window 主动调用其方法来刷新 UI。

UIModule 是一个**控件容器**，负责：
1. 通过 `AutoBindComponents()` 自动绑定子控件引用
2. 对外暴露 public 方法供宿主 Window 调用（刷新 UI、业务逻辑等）

### 创建 Module

继承 `UIModule`，挂在窗口 Prefab 的子节点上：

```csharp
public partial class ChatModule : UIModule
{
    public void RefreshMessages(List<string> messages)
    {
        // 使用自动绑定的控件刷新 UI
    }

    private void OnClick_BtnSend()
    {
        // 发送消息逻辑
    }
}
```

### 挂载到窗口

Module 通过脚本自动生成工具自动注册（见下方「一键生成脚本」章节），也可以手动在 Window 的 `RegisterModules()` 中注册。

```
MyWindow (UIWindow 子类)
  ├── M_ChatModule       ← 自动识别为 Module，生成字段并注册
  ├── M_InventoryModule  ← 同上
  └── ...
```

### 访问宿主窗口

Module 通过 `Owner` 属性访问宿主 UIWindow：

```csharp
public void DoSomething()
{
    Debug.Log($"我的宿主是: {Owner.GetType().Name}");
}
```

---

## 一键生成脚本

框架提供 Editor 工具，可通过 Hierarchy 右键菜单一键生成 UIWindow / UIModule 的 partial class 脚本。

### 基本用法

1. 在 Hierarchy 中选中一个 UI 对象（如 `LobbyWindow`）
2. 右键 → **GameObject > UI Framework > 生成(刷新) UIWindow 脚本**
3. 在弹出的路径选择窗口中选择脚本存放路径（如 `UI/Lobby`）
4. 点击「确认生成」

工具会生成两个文件：

| 文件 | 路径 | 说明 |
|------|------|------|
| 主脚本 | `Assets/Script/UI/Lobby/LobbyWindow.cs` | 手动编写业务逻辑，**不会被覆盖** |
| 副脚本 | `Assets/ScriptGenerate/UI/Lobby/LobbyWindowGenerate.cs` | 自动生成，每次刷新时**重新覆盖** |

生成 UIModule 脚本同理：右键 → **生成(刷新) UIModule 脚本**。

### 对象命名规则

工具通过子对象的**名称前缀**来决定如何处理：

| 前缀 | 含义 | 行为 |
|------|------|------|
| `Y_` | 控件 | 按优先级检测组件，生成对应类型的字段。不继续遍历子对象 |
| `M_` | Module | 生成 Module 类型字段并自动注册到 `RegisterModules()`。不继续遍历子对象 |
| 无前缀 | 普通容器 | 不生成字段，继续向下遍历子对象 |

**示例层级：**

```
LobbyWindow
  ├── Panel                    ← 无前缀，继续遍历
  │   ├── Y_TxtTitle           ← 生成 TMP_Text 字段
  │   ├── Y_BtnStart           ← 生成 Button 字段 + onClick 绑定
  │   ├── Y_SldVolume          ← 生成 Slider 字段 + onValueChanged 绑定
  │   └── Y_ImgAvatar          ← 生成 RawImage 字段（无默认事件）
  ├── M_ChatModule             ← 生成 ChatModule 字段 + 自动注册
  └── Footer                   ← 无前缀，继续遍历
      └── Y_BtnSetting         ← 生成 Button 字段 + onClick 绑定
```

### 组件检测优先级

当一个 Y_ 对象上有多个 UI 组件时，按以下优先级取**第一个**匹配的：

| 优先级 | 组件 | 字段类型 |
|--------|------|----------|
| 1 | TMP_InputField | TMP_InputField |
| 2 | TMP_Dropdown | TMP_Dropdown |
| 3 | Button | Button |
| 4 | ScrollRect | ScrollRect |
| 5 | Slider | Slider |
| 6 | Toggle | Toggle |
| 7 | RawImage | RawImage |
| 8 | TMP_Text | TMP_Text |
| 9 | Image | Image |

### 字段命名

字段名与对象名**完全一致**（包含前缀），均为 `private`：

```csharp
private TMP_Text Y_TxtTitle;
private Button Y_BtnStart;
private ChatModule M_ChatModule;
```

### 默认事件绑定

交互类控件会自动生成事件绑定。副脚本生成 `BindEvents()` 方法，主脚本生成对应的空回调方法。

| 组件 | 事件 | 回调前缀 | 参数 |
|------|------|----------|------|
| Button | onClick | OnClick | 无 |
| Slider | onValueChanged | OnValueChanged | float value |
| Toggle | onValueChanged | OnValueChanged | bool isOn |
| TMP_Dropdown | onValueChanged | OnValueChanged | int index |
| TMP_InputField | onValueChanged | OnValueChanged | string text |
| TMP_InputField | onEndEdit | OnEndEdit | string text |
| ScrollRect | onValueChanged | OnValueChanged | Vector2 value |

回调方法名规则：`{回调前缀}_` + 去掉 `Y_` 前缀后的对象名。

**示例：**

- `Y_BtnStart`（Button）→ `OnClick_BtnStart()`
- `Y_SldVolume`（Slider）→ `OnValueChanged_SldVolume(float value)`
- `Y_TglMute`（Toggle）→ `OnValueChanged_TglMute(bool isOn)`
- `Y_InputName`（TMP_InputField）→ `OnValueChanged_InputName(string text)` + `OnEndEdit_InputName(string text)`

Image、RawImage、TMP_Text 等非交互组件不会生成事件绑定。

### 自定义事件（UICustomEvent）

默认事件对同类型组件统一绑定。如需对**特定对象**添加额外事件（如某个按钮的长按），使用自定义事件系统。

#### 使用方式

1. 在 Inspector 中为目标 Y_ 对象添加对应的自定义事件组件（如 `UILongPressEvent`）
2. 生成/刷新脚本 → 工具自动检测并生成额外的字段和事件绑定
3. 在主脚本中实现回调方法

#### 内置自定义事件

框架在 `Assets/Script/frame/UI/Core/UIEvent/` 中提供以下事件组件：

| 组件 | 事件 | 回调前缀 | 参数 | 说明 |
|------|------|----------|------|------|
| UILongPressEvent | onLongPress | OnLongPress | 无 | 长按（Inspector 可配置 holdDuration） |
| UIDoubleClickEvent | onDoubleClick | OnDoubleClick | 无 | 双击（Inspector 可配置 clickInterval） |
| UIBeginDragEvent | onBeginDrag | OnBeginDrag | PointerEventData | 开始拖拽 |
| UIDragEvent | onDrag | OnDrag | PointerEventData | 拖拽中 |
| UIEndDragEvent | onEndDrag | OnEndDrag | PointerEventData | 拖拽结束 |
| UIPointerEnterEvent | onPointerEnter | OnPointerEnter | PointerEventData | 指针进入 |
| UIPointerExitEvent | onPointerExit | OnPointerExit | PointerEventData | 指针离开 |
| UIPointerDownEvent | onPointerDown | OnPointerDown | PointerEventData | 按下 |
| UIPointerUpEvent | onPointerUp | OnPointerUp | PointerEventData | 抬起 |
| UISelectEvent | onSelect | OnSelect | BaseEventData | 获得焦点 |
| UIDeselectEvent | onDeselect | OnDeselect | BaseEventData | 失去焦点 |

#### 自定义事件生成示例

层级：
```
LobbyWindow
  └── Y_BtnConfirm          ← Button + UILongPressEvent
```

生成的**副脚本**：
```csharp
private Button Y_BtnConfirm;
private UILongPressEvent Y_BtnConfirm_UILongPressEvent;

protected override void AutoBindComponents()
{
    var cache = new Dictionary<string, Transform>();
    UIBindHelper.CollectChildren(transform, cache);
    Y_BtnConfirm = cache["Y_BtnConfirm"].GetComponent<Button>();
    Y_BtnConfirm_UILongPressEvent = cache["Y_BtnConfirm"].GetComponent<UILongPressEvent>();
    BindEvents();
}

private void BindEvents()
{
    Y_BtnConfirm.onClick.AddListener(OnClick_BtnConfirm);
    Y_BtnConfirm_UILongPressEvent.onLongPress.AddListener(OnLongPress_BtnConfirm);
}
```

生成的**主脚本**回调桩：
```csharp
private void OnClick_BtnConfirm() { }
private void OnLongPress_BtnConfirm() { }
```

自定义事件字段命名规则：`{控件字段名}_{事件组件类名}`。

#### 创建自己的自定义事件

继承 `UICustomEvent` 基类，声明 `UnityEvent` 字段并实现 `GetEventBindings()`：

```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;
using UnityEngine.EventSystems;

namespace Dawn.Framework
{
    public class UISwipeEvent : UICustomEvent, IDragHandler, IEndDragHandler
    {
        public float swipeThreshold = 50f;
        public UnityEvent onSwipe = new UnityEvent();

        private Vector2 _startPos;

        public void OnDrag(PointerEventData eventData) { _startPos = eventData.pressPosition; }
        public void OnEndDrag(PointerEventData eventData)
        {
            if (Vector2.Distance(_startPos, eventData.position) > swipeThreshold)
                onSwipe?.Invoke();
        }

        public override List<EventBindingInfo> GetEventBindings() => new List<EventBindingInfo>
        {
            new EventBindingInfo { EventMember = "onSwipe", CallbackPrefix = "OnSwipe" }
        };
    }
}
```

挂载到 Y_ 对象后，生成工具会自动识别并生成绑定代码。

### 生成的代码示例

**主脚本**（手动编辑，首次生成后不再覆盖）：

```csharp
using UnityEngine;
using UnityEngine.UI;
using UnityEngine.EventSystems;
using TMPro;
using Dawn.Framework;

namespace UI.Lobby
{
    public partial class LobbyWindow : UIWindow
    {
        protected override void InitConfig()
        {
            Config.PrefabPath = "";   // 路径格式："文件夹/窗口名"
            Config.Layer = UILayer.Normal;
            Config.Style = WindowStyle.FixSize;
            Config.AnimType = WindowAnimType.None;
            Config.MaskStyle = MaskStyle.None;
            // ... 其他配置项使用默认值
        }

        protected override void OnInitWindow(IUIWindowParam param) { }
        protected override void BeforeShow(IUIWindowParam param) { }
        protected override void OnShow(IUIWindowParam param) { }
        protected override void OnFocus(IUIWindowParam param) { }
        protected override void AfterHide(IUIWindowParam param) { }
        protected override void OnDispose(IUIWindowParam param) { }

        private void OnClick_BtnStart() { }
        private void OnLongPress_BtnStart() { }
        private void OnValueChanged_SldVolume(float value) { }
        private void OnClick_BtnSetting() { }
    }
}
```

**副脚本**（自动生成，刷新时覆盖）：

```csharp
// Auto-generated. Do not edit manually.
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;
using UnityEngine.EventSystems;
using TMPro;
using Dawn.Framework;

namespace UI.Lobby
{
    public partial class LobbyWindow
    {
        private TMP_Text Y_TxtTitle;
        private Button Y_BtnStart;
        private UILongPressEvent Y_BtnStart_UILongPressEvent;
        private Slider Y_SldVolume;
        private RawImage Y_ImgAvatar;
        private Button Y_BtnSetting;
        private ChatModule M_ChatModule;

        protected override void AutoBindComponents()
        {
            var cache = new Dictionary<string, Transform>();
            UIBindHelper.CollectChildren(transform, cache);

            Y_TxtTitle = cache["Y_TxtTitle"].GetComponent<TMP_Text>();
            Y_BtnStart = cache["Y_BtnStart"].GetComponent<Button>();
            Y_BtnStart_UILongPressEvent = cache["Y_BtnStart"].GetComponent<UILongPressEvent>();
            Y_SldVolume = cache["Y_SldVolume"].GetComponent<Slider>();
            Y_ImgAvatar = cache["Y_ImgAvatar"].GetComponent<RawImage>();
            Y_BtnSetting = cache["Y_BtnSetting"].GetComponent<Button>();

            M_ChatModule = cache["M_ChatModule"].GetComponent<ChatModule>();
            RegisterModules(M_ChatModule);

            BindEvents();
        }

        private void BindEvents()
        {
            Y_BtnStart.onClick.AddListener(OnClick_BtnStart);
            Y_BtnStart_UILongPressEvent.onLongPress.AddListener(OnLongPress_BtnStart);
            Y_SldVolume.onValueChanged.AddListener(OnValueChanged_SldVolume);
            Y_BtnSetting.onClick.AddListener(OnClick_BtnSetting);
        }
    }
}
```

### 刷新流程

当界面上添加了新控件后，再次右键 → 「生成(刷新)」：

- **副脚本**：重新生成，包含最新的控件字段和绑定
- **主脚本**：不修改。如果有新增的 Button 回调方法缺失，Console 会输出警告提示手动添加

### 命名空间

命名空间根据脚本路径自动生成：

| 脚本路径 | 命名空间 |
|----------|----------|
| `Assets/Script/UI/Lobby/` | `UI.Lobby` |
| `Assets/Script/UI/Chat/` | `UI.Chat` |
| `Assets/Script/Gameplay/HUD/` | `Gameplay.HUD` |

### 同名冲突

如果不同层级下有同名的 Y_ 或 M_ 对象，工具会弹出错误提示，要求重命名后重试。

### Window vs Module 生成差异

| | UIWindow | UIModule |
|--|----------|----------|
| 主脚本内容 | InitConfig + 6 个生命周期方法 + 事件回调 | 空类体 + 事件回调 |
| Canvas 校验 | 不在 Canvas 下时弹出警告 | 无限制 |
| RegisterModules | 副脚本中自动调用 | 不适用 |

### 自动绑定时机

`AutoBindComponents()` 在 UIWindow 的 `InternalInit()` 中被调用，时机位于 `ApplyStyle()` 之后、`OnInitWindow()` 之前：

```
InitConfig() → ApplyStyle() → AutoBindComponents() → OnInitWindow()
```

因此在 `OnInitWindow` 中可以直接使用所有自动绑定的控件字段。

---

## 注意事项

1. **Init 必须先调用**：使用任何 UI 功能前必须调用 `UIManager.Instance.Init()`
2. **Prefab 必须挂脚本**：窗口 Prefab 的根节点必须挂对应的 UIWindow 子类脚本
3. **按钮绑定在 OnInitWindow 中**：只会调用一次，窗口缓存复用时不会重复绑定
4. **关闭不等于销毁**：HideWindow 只是隐藏并缓存，不会触发 OnDispose
5. **参数在隐藏后清空**：`Param` 在 AfterHide 之后被置为 null
6. **动画不受 TimeScale 影响**：所有动画使用 `Time.unscaledDeltaTime`
7. **同一窗口不会重复打开**：已显示的窗口再次调用 ShowWindow 不会重复显示
8. **同一窗口不会重复加载**：正在加载中的窗口再次调用 ShowWindow 会被忽略
