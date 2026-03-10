---
title: Unity UIToolkit数据驱动UI实践
published: 2026-03-11
updated: 2026-03-11
description: 'Start to Learn about Unity UIToolkit features'
image: ''
tags: [Demo, Unity, UIToolkit]
category: 'Unity'
draft: false 
---

# Unity UIToolkit数据驱动UI实践

> 基于 `ui test` 项目的源码分析

---

## 项目概述

这是一个 Unity UIToolkit (UITK) 的演示项目，用于展示角色数据面板的 UI 系统。项目包含以下核心功能：
- 角色数据展示（头像、名称、等级、属性）
- 队伍数据管理
- 运行时 UI 切换（空格键显示/隐藏）

---

## 核心架构

### 文件结构

```
Assets/
├── Scripts/
│   ├── Data/
│   │   ├── CharacterData.cs        # 角色数据 (ScriptableObject)
│   │   ├── CharacterStatsData.cs   # 角色属性数据 (ScriptableObject)
│   │   ├── CharaterStats.cs        # 属性数据结构
│   │   └── PartyData.cs            # 队伍数据 (ScriptableObject)
│   └── UI/
│       ├── CharacterDataPanel.cs   # 角色面板组件
│       └── PartyDataScreen.cs      # 队伍数据屏幕 (MonoBehaviour)
└── Data/
    ├── UI Documents/
    │   ├── PartyDataScreen.uxml    # 主界面布局
    │   └── CharacterDataPanel.uxml # 角色面板布局
    └── Style Sheets/
        └── PartyDataScreen.uss     # 样式定义
```

---

## UIToolkit 核心概念

### 1. VisualElement (视觉元素)

所有 UI 元素的基类，类似于 HTML 的 DOM 元素。

```csharp
// 继承 VisualElement 创建自定义组件
public partial class CharacterDataPanel : VisualElement
```

### 2. UXML (UI Markup Language)

类似 HTML 的 UI 声明式标记语言，用于定义 UI 结构。

```xml
<engine:VisualElement name="CharacterDataPanel">
    <engine:VisualElement name="AvatarContainer">
        <engine:VisualElement name="Avatar" />
    </engine:VisualElement>
    <engine:Label name="NameLabel" />
    <engine:VisualElement name="StatsContainer">
        <!-- 多个属性容器 -->
    </engine:VisualElement>
</engine:VisualElement>
```

### 3. USS (Unity Style Sheets)

类似 CSS 的样式表语言，用于定义 UI 外观。

```css
Label {
    font-size: 25px;
    color: rgb(255, 255, 255);
    -unity-text-align: middle-center;
}

#StatsContainer Label {
    background-color: rgba(0, 0, 0, 0);
    color: rgb(0, 0, 0);
}
```

### 4. UIDocument

用于运行时 UI 的核心组件，挂在 GameObject 上提供 UI 文档的根节点。

```csharp
rootVisualElement = GetComponent<UIDocument>().rootVisualElement;
```

---

## 数据驱动 UI 模式

### ScriptableObject 数据层

项目使用 ScriptableObject 作为数据存储，实现数据与 UI 分离。

#### CharacterData.cs
```csharp
[CreateAssetMenu(menuName = ("CharacterData"), fileName = ("CharacterData_"))]
public class CharacterData : ScriptableObject
{
    [SerializeField] Texture2D characterAvatarImage;
    [SerializeField] string characterName;
    [SerializeField, Range(1, CharacterMaxLevel)] int characterStartLevel = 1;
    [SerializeField] CharacterStatsData characterStatsData;
    
    // 属性封装
    public Texture2D CharacterAvatarImage => characterAvatarImage;
    public string CharacterName => characterName;
    public int CharacterLevel { get; set; }
    public CharacterStats CharacterStats => characterStatsData.GetCurrentCharacterLevel(characterLevel);
}
```

#### CharacterStatsData.cs
```csharp
[CreateAssetMenu(menuName = ("CharacterStatsData"), fileName = ("CharacterStatsData_"))]
public class CharacterStatsData : ScriptableObject
{
    [SerializeField] TextAsset characterStatsCSV;
    [SerializeField] List<CharacterStats> characterStatsList = new List<CharacterStats>();
    
    // 从 CSV 加载数据
    public void LoadStats()
    {
        string[] lines = characterStatsCSV.text.Split('\n');
        for (int i = 1; i < lines.Length; i++)
        {
            string[] values = line.Split(',');
            CharacterStats stats = new CharacterStats
            {
                initiative = int.Parse(values[0]),
                maxHp = int.Parse(values[1]),
                // ...
            };
            characterStatsList.Add(stats);
        }
    }
}
```

---

## UI 组件实现

### CharacterDataPanel (自定义 VisualElement)

```csharp
[UxmlElement]
public partial class CharacterDataPanel : VisualElement
{
    readonly TemplateContainer templateContainer;
    readonly List<VisualElement> statsContainers;
    
    // 方式1: 通过代码动态创建
    public CharacterDataPanel()
    {
        templateContainer = Resources.Load<VisualTreeAsset>("CharacterDataPanel").Instantiate();
        templateContainer.style.flexGrow = 1.0f;
        hierarchy.Add(templateContainer);
    }
    
    // 方式2: 通过 UI Builder 创建时传入数据
    public CharacterDataPanel(CharacterData characterData) : this()
    {
        userData = characterData;
        templateContainer.Q("Avatar").style.backgroundImage = characterData.CharacterAvatarImage;
        templateContainer.Q<Label>("NameLabel").text = characterData.CharacterName;
        
        statsContainers = templateContainer.Query("StatContainer").ToList();
        UpdateCharacterStats();
    }
}
```

**关键点**:
- 使用 `Resources.Load<VisualTreeAsset>()` 加载 UXML
- 使用 `.Instantiate()` 创建 VisualElement 实例
- 使用 `Q<T>()` 查询子元素
- 使用 `userData` 存储自定义数据

### PartyDataScreen (MonoBehaviour 控制器)

```csharp
public class PartyDataScreen : MonoBehaviour
{
    [SerializeField] PartyData partyData;
    private VisualElement rootVisualElement;
    
    void Awake()
    {
        rootVisualElement = GetComponent<UIDocument>().rootVisualElement;
        var bodyContainer = rootVisualElement.Q("BodyContainer");
        
        foreach (CharacterData characterData in partyData.CharacterDataList)
        {
            var characterDataPanel = new CharacterDataPanel(characterData);
            characterDataPanel.style.flexBasis = Length.Percent(25.0f);
            bodyContainer.Add(characterDataPanel);
        }
    }
    
    void Update()
    {
        if (Input.GetKeyDown(KeyCode.Space))
        {
            rootVisualElement.style.display = 
                rootVisualElement.style.display == DisplayStyle.Flex ? 
                DisplayStyle.None : DisplayStyle.Flex;
        }
    }
}
```

---

## 元素查询与操作

### 查询元素

```csharp
// 按名称查询
var element = root.Q("ElementName");

// 按类型查询
var labels = root.Query<Label>();

// 带条件的查询
var containers = root.Query<VisualElement>("StatContainer");
```

### 元素层级操作

```csharp
// 添加子元素
parent.Add(child);

// 插入子元素
parent.Insert(index, child);

// 移除子元素
parent.Remove(child);

// 清空所有子元素
parent.Clear();
```

### 样式操作

```csharp
// 设置样式
element.style.flexGrow = 1.0f;
element.style.backgroundImage = texture;

// 读取样式
var display = element.style.display;
```

---

## 布局系统

### Flexbox 布局

UIToolkit 使用类似 CSS Flexbox 的布局系统。

```xml
<engine:VisualElement name="BodyContainer" style="flex-grow: 1; flex-direction: row;" />
```

常用属性：
- `flex-direction`: `row` | `column`
- `flex-grow`: 扩展比例
- `flex-basis`: 基础尺寸
- `justify-content`: 主轴对齐
- `align-items`: 交叉轴对齐

---

## 事件系统

### 注册事件

```csharp
// 鼠标点击事件
button.RegisterCallback<ClickEvent>(evt => { /* 处理点击 */ });

// 拖拽事件
element.RegisterCallback<DragExitedEvent>(evt => { /* 处理拖拽 */ });
```

---

## 资源加载

### 运行时加载

```csharp
// 加载 VisualTreeAsset (UXML)
var vta = Resources.Load<VisualTreeAsset>("CharacterDataPanel");
var element = vta.Instantiate();

// 加载样式表 (USS)
var uss = Resources.Load<StyleSheet>("MyStyle");
root.styleSheets.Add(uss);
```

---

## 最佳实践

1. **数据与 UI 分离**: 使用 ScriptableObject 存储数据，VisualElement 仅负责展示
2. **组件化**: 将可复用的 UI 部分封装为自定义 VisualElement
3. **样式集中管理**: 使用 USS 统一管理样式，支持主题切换
4. **资源预加载**: 对于大量 UI，使用 Resources.Load 配合对象池

---

## 扩展阅读

- [Unity UIToolkit 官方文档](https://docs.unity3d.com/Manual/UIElements.html)
- [UXML 参考](https://docs.unity3d.com/2021.2/Documentation/Manual/UIE-UXML.html)
- [USS 参考](https://docs.unity3d.com/2021.2/Documentation/Manual/UIE-Stylesheet.html)

---

## 项目源码路径

- 项目位置: `E:\Unity\Projects\ui test`
- 脚本: `Assets\Scripts\`
- UI 资源: `Assets\Data\UI Documents\`
- 样式: `Assets\Data\Style Sheets\`
