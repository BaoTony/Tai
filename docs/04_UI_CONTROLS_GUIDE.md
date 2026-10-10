# 04 – UI 自定义控件与主题开发指南

> 📁 目标读者：需要修改 UI 界面或新增控件的开发者  
> 🎯 目标：理解 Tai 的自定义控件体系，能够独立开发新控件或修改主题

---

## 1. 概述

Tai 没有使用任何第三方 UI 组件库（如 MahApps、MaterialDesign），而是**完全自研了一套 WPF 自定义控件库**，位于 [UI/Controls/](file:///d:/dev/store/Tai/UI/Controls/)。这套体系涵盖了：

- **20+ 个自定义控件**，分布在 12 个子目录中
- **6 种图表类型**（List / Card / Month / Column / Radar / Pie）
- **深色/浅色双主题**，通过资源字典动态切换
- **基于依赖属性的 MVVM 友好设计**

学习本指南前，建议先阅读 [docs/02_DATA_FLOW.md](02_DATA_FLOW.md) 了解数据如何流到 UI 层。

---

## 2. 控件架构总览

### 2.1 目录结构

```
UI/Controls/
├── Base/                    # 🔧 基元控件 — 项目中最小粒度的 UI 单元
│   ├── Icon.cs(.xaml)       #   图标控件（支持多种内置图标类型）
│   ├── Text.cs(.xaml)       #   文本控件（统一字体、排版）
│   ├── Img.cs(.xaml)        #   图片控件
│   ├── CheckBox.cs(.xaml)   #   复选框
│   ├── View.cs(.xaml)       #   条件渲染容器
│   ├── EmptyData.cs(.xaml)  #   空数据占位
│   ├── Placeholder.cs(.xaml)#   加载占位
│   ├── IconSelect.cs(.xaml) #   图标选择器
│   ├── ColorSelect.cs(.xaml)#   颜色选择器
│   ├── IconConverter.cs     #   图标转换工具类
│   ├── IconTypes.cs         #   图标类型枚举
│   └── DiffValue.cs         #   差异值显示
│
├── Button/                  # 🔘 按钮
│   ├── Button.cs(.xaml)     #   标准按钮
│   └── IconButton.cs(.xaml) #   图标按钮
│
├── Charts/                  # 📊 图表系统（核心控件族）
│   ├── Charts.cs(.xaml)     #   图表容器（入口控件）
│   ├── ChartsType.cs        #   图表类型枚举
│   ├── ChartDataValueType.cs#   图表数据值类型
│   ├── ChartsItemTypeList.cs(.xaml)   # 列表图
│   ├── ChartsItemTypeCard.cs(.xaml)   # 卡片图
│   ├── ChartsItemTypeMonth.cs(.xaml)  # 月份热力图
│   ├── ChartItemTypeColumn.cs(.xaml)  # 柱形图
│   ├── ChartsItemTypeRadar.cs(.xaml)  # 雷达图
│   ├── ChartsItemTypePie.cs           # 饼图（Canvas 绘制）
│   └── Model/
│       ├── ChartsDataModel.cs    # 图表数据模型
│       ├── ChartBadgeModel.cs    # 徽标模型
│       ├── ChartBadgeType.cs     # 徽标类型
│       └── ChartColumnInfoModel.cs# 柱图列宽/位置信息
│
├── Navigation/              # 🧭 侧边导航栏
│   ├── Navigation.cs(.xaml) #   导航容器
│   ├── NavigationItem.cs(.xaml)# 导航项
│   └── Models/
│       └── NavigationItemModel.cs
│
├── Select/                  # 📋 选择器
│   ├── Select.cs(.xaml)     #   普通下拉选择
│   ├── Option.cs(.xaml)     #   选择项
│   ├── ImageSelect.cs(.xaml)#   图片选择
│   ├── DateSelect.cs(.xaml) #   日期选择
│   ├── DateSelectType.cs    #   日期选择类型
│   └── SelectItemModel.cs   #   选择项模型
│
├── DatePickerBar/           # 📅 日期选择栏
│   ├── DatePickerBar.cs(.xaml)
│   ├── DatePickerBarItem.cs(.xaml)
│   └── DatePickerShowType.cs
│
├── SettingPanel/            # ⚙ 设置面板
│   ├── SettingPanel.cs(.xaml)
│   ├── SettingPanelItem.cs(.xaml)
│   └── SettingPanelMultiItem.cs(.xaml)
│
├── List/                    # 📃 列表
│   ├── BaseList.cs(.xaml)
│   └── BaseListItem.cs(.xaml)
│
├── Window/                  # 🪟 窗口
│   ├── DefaultWindow.cs(.xaml)# 标准窗口（含标题栏）
│   ├── DefaultWindowCommands.cs
│   └── HideWindow.cs(.xaml) #   隐藏窗口（系统托盘模式）
│
├── Tabbar/                  # 📑 标签栏
│   └── Tabbar.cs(.xaml)
│
├── Toggle/                  # 🔄 开关
│   ├── Toggle.cs(.xaml)
│   └── ToggleTextPosition.cs
│
├── Input/                   # ⌨ 输入框
│   └── InputBox.cs(.xaml)
│
├── Expander/                # 📂 折叠面板
│   └── Expander.cs(.xaml)
│
├── Converters/              # 🔄 值转换器（全局共享）
│   ├── DateTimeConverter.cs
│   ├── TimeConverter.cs
│   ├── HextoColorConverter.cs
│   ├── ToStringConverter.cs
│   └── GetCountConverter.cs
│
├── Models/
│   └── PageModel.cs         # 页面缓存模型
│
├── Command.cs               # 命令封装（跨控件共享）
├── PageContainer.cs(.xaml)  # 页面容器（路由 + 缓存）
└── TPage.cs(.xaml)          # 页面基类
```

### 2.2 继承体系

```
System.Windows.Controls.Control          ← WPF 原生控件基类
├── Charts                               ← 图表容器
├── ChartsItemTypeList / Card / Month
│   / ChartItemTypeColumn / Radar        ← 各图表子控件
├── Navigation                           ← 导航栏
├── NavigationItem                       ← 导航项
├── Select / ImageSelect / DateSelect    ← 选择器
├── DatePickerBar / DatePickerBarItem    ← 日期栏
├── SettingPanel / ...Item / ...MultiItem ← 设置面板
├── BaseList / BaseListItem              ← 列表
├── DefaultWindow / HideWindow           ← 窗口
├── Tabbar                               ← 标签栏
├── Toggle                               ← 开关
├── InputBox                             ← 输入框
├── Expander                             ← 折叠面板
├── Button / IconButton                  ← 按钮
├── Icon / Text / Img / CheckBox         ← 基元控件
├── EmptyData / Placeholder              ← 状态占位
├── View                                 ← 条件渲染容器
├── PageContainer                        ← 页面容器
└── TPage                                ← 页面基类

System.Windows.Controls.ContentControl   ← WPF 内容控件基类
└── View                                 ← 条件渲染（同时继承 Control）

System.Windows.Controls.Canvas           ← WPF 画布基类
└── ChartsItemTypePie                    ← 饼图（完全自绘制）
```

> 📌 **关键设计**：所有自定义控件继承 `Control`，通过 `DefaultStyleKey` 绑定到 `Themes/` 下的 XAML 主题文件。不依赖 `UserControl`——这是 WPF 自定义控件库的标准做法，保证模板可完全替换。

---

## 3. 主题系统

### 3.1 双层主题架构

Tai 的主题系统由**两层**资源字典构成：

```
第一层：控件默认样式             第二层：颜色/字体资源
─────────────────────────     ───────────────────────
UI/Themes/Generic.xaml   ←→   UI/Resources/Themes/Light.xaml
  ├── Charts/Charts.xaml           └── 引用了 Common.xaml（公共画笔/样式）
  ├── Navigation/Navigation.xaml   UI/Resources/Themes/Dark.xaml
  ├── Button/Button.xaml           └── 引用了 Common.xaml（公共画笔/样式）
  ├── ... (所有控件主题)
  └── PageContainer.xaml
```

- **第一层** (`Themes/`)：每个自定义控件的默认样式（ControlTemplate），声明控件视觉树结构。引用第二层的资源键获取颜色。
- **第二层** (`Resources/Themes/`)：`Light.xaml` 和 `Dark.xaml`，定义颜色画笔、字体等视觉资源。切换主题时替换这一层即可。

### 3.2 主题切换流程

```
用户点击设置 > 主题切换
    │
    ▼
SettingPageViewModel → appConfig.SetConfig()
    │
    ▼
AppConfig.ConfigChanged 事件广播
    │
    ▼
ThemeServicer.AppConfig_ConfigChanged()           [UI/Servicers/ThemeServicer.cs]
    │
    ├─ 1. 从 MergedDictionaries 中移除旧主题资源字典
    ├─ 2. 加载新主题资源字典：
    │       Light → Resources/Themes/Light.xaml
    │       Dark  → Resources/Themes/Dark.xaml
    │
    ├─ 3. 触发 OnThemeChanged 事件
    │       ├── MainWindow 响应：更新窗口标题栏颜色（Win32 API）
    │       └── ChartsItemTypePie 响应：重新绘制饼图颜色
    │
    └─ 4. 保存主题设置到 Tai.config
```

**`ThemeServicer`**（[UI/Servicers/ThemeServicer.cs](file:///d:/dev/store/Tai/UI/Servicers/ThemeServicer.cs)）是主题管理的核心类：
- 构造函数中注册了 `IAppConfig.ConfigChanged` 事件监听
- `LoadTheme(themeName)` 负责加载/替换资源字典
- `isRefresh` 参数用于同一主题下的颜色微调刷新

### 3.3 如何修改主题颜色

主题资源定义在：
- [UI/Resources/Themes/Light.xaml](file:///d:/dev/store/Tai/UI/Resources/Themes/Light.xaml) — 浅色主题
- [UI/Resources/Themes/Dark.xaml](file:///d:/dev/store/Tai/UI/Resources/Themes/Dark.xaml) — 深色主题
- [UI/Resources/Themes/Common.xaml](file:///d:/dev/store/Tai/UI/Resources/Themes/Common.xaml) — 公共样式（两主题共享）

**修改颜色的步骤**：
1. 找到目标控件的 XAML 主题文件（`Themes/` 下对应目录）
2. 确认它使用的资源键名（如 `{DynamicResource PrimaryBrush}`）
3. 到 `Dark.xaml` / `Light.xaml` 中找到该资源键，修改其颜色值

> ⚠️ 注意：主题资源使用 `DynamicResource` 标记，确保切换主题时自动刷新；不可改为 `StaticResource`。

---

## 4. 图表系统详解

### 4.1 架构设计

`Charts` 是 Tai 最复杂的控件体系，采用**容器 + 策略**模式：

```
Charts（容器控件）
  │
  ├─ 依赖属性 ChartsType：决定使用哪种子控件渲染
  │     ├── List   → ChartsItemTypeList   列表视图
  │     ├── Card   → ChartsItemTypeCard   卡片视图
  │     ├── Month  → ChartsItemTypeMonth  月历热力图
  │     ├── Column → ChartItemTypeColumn  柱形图
  │     ├── Radar  → ChartsItemTypeRadar  雷达图
  │     └── Pie    → ChartsItemTypePie    饼图
  │
  ├─ 依赖属性 Data：IEnumerable<ChartsDataModel>
  ├─ 依赖属性 IsLoading：加载状态
  ├─ 依赖属性 ClickCommand：点击回调
  └─ 依赖属性 ShowLimit / MaxValueLimit：显示控制
```

### 4.2 ChartsDataModel — 统一数据模型

[UI/Controls/Charts/Model/ChartsDataModel.cs](file:///d:/dev/store/Tai/UI/Controls/Charts/Model/ChartsDataModel.cs) 是所有图表共用的数据契约：

| 属性 | 类型 | 用途 |
|------|------|------|
| `Name` | `string` | 数据项名称（显示在图表上） |
| `Value` | `double` | 单一数值（List/Card/Pie 使用） |
| `Values` | `double[]` | 多数值（Radar/Column 使用） |
| `ColumnNames` | `string[]` | 列名（绑定到 Values 各项的标签） |
| `Tag` | `string` | 附加标签 |
| `Icon` | `string` | 图标路径/标识 |
| `Color` | `string` | 自定义颜色 |
| `DateTime` | `DateTime` | 时间戳（Month 类型专用） |
| `MaxValue` | `double` | 最大值（进度条/比例计算参考） |
| `BadgeList` | `List<ChartBadgeModel>` | 徽标列表（显示附加信息） |
| `Data` | `object` | 原始数据引用（点击回传） |
| `IsSelected` | `bool` | 选中状态 |
| `PopupText` | `string` | 悬浮提示文本 |

### 4.3 各图表类型特点

| 类型 | 枚举值 | 类名 | 基类 | 特点 |
|------|--------|------|------|------|
| 列表 | `List` | `ChartsItemTypeList` | `Control` | 最常用，显示应用/网站排名列表 |
| 卡片 | `Card` | `ChartsItemTypeCard` | `Control` | 卡片式布局，带进度条和图标 |
| 月历 | `Month` | `ChartsItemTypeMonth` | `Control` | 类似 GitHub 贡献热力图，按日着色 |
| 柱形 | `Column` | `ChartItemTypeColumn` | `Control` | 时间段分组柱状图，支持多列对比 |
| 雷达 | `Radar` | `ChartsItemTypeRadar` | `Control` | 多边形雷达图，适合多维度对比 |
| 饼图 | `Pie` | `ChartsItemTypePie` | `Canvas` | **直接在 Canvas 上用 Shape 绘制** |

> 📌 `Pie` 是唯一不继承 `Control` 而继承 `Canvas` 的图表——它完全通过代码动态创建扇形 `Path`，不依赖 XAML 模板。

### 4.4 Charts 使用示例

在页面 XAML 中使用 Charts 控件：

```xml
<!-- 引入命名空间 -->
<controls:Charts
    ChartsType="List"
    Data="{Binding AppDatas}"
    IsLoading="{Binding IsLoading}"
    ClickCommand="{x:Static model:MainCommand.ClickItemCommand}">
    <!-- 通过附加属性传递点击命令 -->
</controls:Charts>
```

在 ViewModel 中构建数据：

```csharp
var chartData = new List<ChartsDataModel>
{
    new ChartsDataModel
    {
        Name = "Visual Studio",
        Value = 3600,  // 秒
        Icon = "VisualStudio",
        Data = rawAppModel  // 原始数据，点击回传
    }
};
```

---

## 5. 页面导航系统

### 5.1 PageContainer — 页面路由与缓存

[UI/Controls/PageContainer.cs](file:///d:/dev/store/Tai/UI/Controls/PageContainer.cs) 是页面系统的核心，实现了**基于 URI 的页面路由**和**页面缓存**：

```
PageContainer 核心依赖属性：
├── Uri              → 当前页面路径（如 "IndexPage"），变更即触发页面切换
├── IndexUriList     → 首页路径列表（用于"回首页"判定）
├── Title            → 页面标题（显示在导航栏上方）
├── Content          → 当前页面实例
└── ServiceProvider  → DI 容器引用
```

**页面切换流程**：

```
Uri 属性变更
    │
    ├─ 1. 检查 PageCache（Dictionary<string, PageModel>）是否已有缓存
    │      ├── 有缓存：恢复页面 + 还原滚动位置
    │      └── 无缓存：通过 ServiceProvider 创建新页面实例
    │
    ├─ 2. Navigation.SelectedItem 同步更新（侧边导航高亮）
    │
    └─ 3. Content 属性更新 → 触发 RenderTransform 淡入动画
```

**页面缓存机制**：首次访问的页面通过 DI 容器创建 ViewModel 和 View，缓存在 `PageCache` 中。离开页面时只保存滚动位置，不销毁实例。这意味着切换到新页面时页面不会被 GC，切换回来也不需要重新加载数据。

### 5.2 TPage — 页面基类

所有页面（`IndexPage`、`DetailPage` 等）都继承自 `TPage`（[UI/Controls/TPage.cs](file:///d:/dev/store/Tai/UI/Controls/TPage.cs)），TPage 继承标准的 WPF `Page`，统一了页面的生命周期管理和样式应用。

### 5.3 Navigation — 侧边导航栏

[UI/Controls/Navigation/Navigation.cs](file:///d:/dev/store/Tai/UI/Controls/Navigation/Navigation.cs) 管理左侧导航菜单：

- 数据源：`ObservableCollection<NavigationItemModel>`（由 MainWindow 的 ViewModel 填充）
- 选中项变化 → 更新 `PageContainer.Uri` → 触发页面切换
- 响应式宽度：窗口宽度 < 700px 时自动折叠导航栏
- 支持右键菜单（`ItemContextMenu`）

---

## 6. 基元控件（Base/）

基元控件是项目中使用频率最高的 UI 组件，其他控件（如 Charts、Navigation）内部都大量使用它们。

### 6.1 Icon — 图标控件

[UI/Controls/Base/Icon.cs](file:///d:/dev/store/Tai/UI/Controls/Base/Icon.cs) 支持多种内置图标类型（`IconTypes` 枚举），通过 `IconConverter` 将图标类型转换为对应的几何图形。不依赖外部图标字体库。

### 6.2 View — 条件渲染容器

[UI/Controls/Base/View.cs](file:///d:/dev/store/Tai/UI/Controls/Base/View.cs) 是一个有趣的轻量控件：

- `Condition` 属性：支持 `=value` 格式的条件表达式
- `Value` 属性：当 `Condition` 为空时，直接按 bool 值决定是否显示内容
- 内部通过设置 `Visibility` 实现条件渲染，避免在 XAML 中写大量 `Visibility` 绑定转换器

```xml
<!-- 仅当 IsAdmin 为 true 时显示 -->
<base:View Value="{Binding IsAdmin}">
    <Button Content="管理员操作" />
</base:View>
```

---

## 7. 如何新增一个控件

### 7.1 标准流程

以新增一个「标签（Tag）」控件为例：

```
步骤 1：创建 C# 类
  → UI/Controls/Base/Tag.cs
    继承 Control，定义依赖属性（Text、Color、Closable 等）
    设置 DefaultStyleKey = typeof(Tag)

步骤 2：创建主题 XAML
  → UI/Themes/Base/Tag.xaml
    定义 ControlTemplate，使用 {DynamicResource} 引用主题色

步骤 3：在 Generic.xaml 中注册
  → UI/Themes/Generic.xaml
    添加 <ResourceDictionary Source=".../Themes/Base/Tag.xaml"/>

步骤 4：在页面中使用
  → 引入命名空间 xmlns:base="clr-namespace:UI.Controls.Base"
    <base:Tag Text="示例" Color="Blue" />
```

### 7.2 新增图表类型

新增图表类型的流程比普通控件更复杂：

1. 在 `ChartsType` 枚举中添加新值
2. 创建 `ChartsItemTypeXxx` 类，继承 `Control`（或 `Canvas`）
3. 在 `Charts.cs` 的 `OnPropertyChanged` 回调中添加新类型的分支逻辑
4. 创建对应的 `Themes/Charts/ChartsItemTypeXxx.xaml` 主题
5. 在 `Generic.xaml` 中注册

---

## 8. 页面内协作组件

View 层有一些「协作组件」不在 Controls 目录中，但配合页面使用：

| 文件 | 位置 | 用途 |
|------|------|------|
| `DetailPage.xaml` 内部控件 | 页面内联定义 | 详情页的时间段编辑弹窗、日志列表等 |
| `ChartPage.xaml` | `DatePickerBar` 嵌入 | 图表页的时间范围选择 |
| `IndexPage.xaml` | `Charts + DatePickerBar` | 首页仪表盘组合 |

这些页面的 XAML 本身包含了大量内联定义的交互逻辑，修改时要在对应的 `.xaml` 文件中操作。

---

## 9. 建议学习路线

| 天数 | 学习内容 | 目标文件 |
|------|----------|----------|
| **第 1 天** | 主题系统 | [UI/Servicers/ThemeServicer.cs](file:///d:/dev/store/Tai/UI/Servicers/ThemeServicer.cs)、[UI/Resources/Themes/Dark.xaml](file:///d:/dev/store/Tai/UI/Resources/Themes/Dark.xaml)、[UI/Resources/Themes/Light.xaml](file:///d:/dev/store/Tai/UI/Resources/Themes/Light.xaml)、[UI/Themes/Generic.xaml](file:///d:/dev/store/Tai/UI/Themes/Generic.xaml) |
| **第 2 天** | 图表系统 — 数据模型与容器 | [UI/Controls/Charts/Charts.cs](file:///d:/dev/store/Tai/UI/Controls/Charts/Charts.cs)、[UI/Controls/Charts/Model/ChartsDataModel.cs](file:///d:/dev/store/Tai/UI/Controls/Charts/Model/ChartsDataModel.cs)、[UI/Controls/Charts/ChartsType.cs](file:///d:/dev/store/Tai/UI/Controls/Charts/ChartsType.cs) |
| **第 3 天** | 图表系统 — 各类型实现 | [UI/Controls/Charts/ChartsItemTypeList.cs](file:///d:/dev/store/Tai/UI/Controls/Charts/ChartsItemTypeList.cs)、[UI/Controls/Charts/ChartsItemTypeCard.cs](file:///d:/dev/store/Tai/UI/Controls/Charts/ChartsItemTypeCard.cs)、[UI/Controls/Charts/ChartsItemTypePie.cs](file:///d:/dev/store/Tai/UI/Controls/Charts/ChartsItemTypePie.cs) (+ 对应 XAML) |
| **第 4 天** | 导航与页面系统 | [UI/Controls/PageContainer.cs](file:///d:/dev/store/Tai/UI/Controls/PageContainer.cs)、[UI/Controls/Navigation/Navigation.cs](file:///d:/dev/store/Tai/UI/Controls/Navigation/Navigation.cs)、[UI/Controls/TPage.cs](file:///d:/dev/store/Tai/UI/Controls/TPage.cs) |
| **第 5 天** | 基元控件与其他 | [UI/Controls/Base/](file:///d:/dev/store/Tai/UI/Controls/Base/) 下全部文件、[UI/Controls/Select/](file:///d:/dev/store/Tai/UI/Controls/Select/)、[UI/Controls/SettingPanel/](file:///d:/dev/store/Tai/UI/Controls/SettingPanel/) |