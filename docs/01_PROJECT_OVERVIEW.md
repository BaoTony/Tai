# Tai 项目概览与架构标准

> 编写时间：2026-10-10
> 版本：基于当前 master 分支最新代码
> 目标读者：新加入的开发维护人员
> 关联文档：数据链路分析见 [`02_DATA_FLOW.md`](./02_DATA_FLOW.md)

---

## 一、项目简介

**Tai** 是一款 Windows 平台的时间统计工具，用于追踪电脑**软件使用时长**和**网站浏览时长**。项目采用 WPF (.NET Framework 4.8) + Entity Framework 6 + SQLite 架构，通过 Win32 API 监控前台窗口变化，并通过 WebSocket 接收浏览器插件上报的网页浏览数据。

### 核心功能

- 自动统计桌面应用前台使用时长（精确到秒）
- 浏览器网站浏览时长统计（需安装 Chrome/Edge 插件）
- 多维度数据展示：概览（今日/本周）、统计（按天/周/月/年）、详细、汇总图表
- 应用/网站分类管理与自动分类（目录匹配）
- 关联进程同步统计
- 睡眠/离开检测，自动暂停统计
- 数据导出（Excel / CSV）
- 应用/网站白名单、忽略列表（支持正则）
- 开机自启、深色/浅色主题

---

## 二、技术栈

| 类别 | 技术 | 版本 |
|------|------|------|
| 运行时 | .NET Framework | 4.8 |
| UI | WPF | - |
| DI | Microsoft.Extensions.DependencyInjection | 6.0.0 |
| ORM | Entity Framework 6 (Code First) | 6.3.0 |
| 数据库 | SQLite (System.Data.SQLite) | - |
| JSON | Newtonsoft.Json | - |
| WebSocket | WebSocketSharp | - |
| Excel 导出 | NPOI + CsvHelper | 2.5.6 / 29.0.0 |

---

## 三、解决方案结构

```
Tai/
├── Tai.sln                      # 解决方案
├── Core/                        # 核心层（服务接口、服务实现、数据模型、工具库）
│   ├── Enums/                   # 枚举：AppType, BrowserType, SleepStatus, WebPageInfoType
│   ├── Event/                   # 事件参数与委托定义
│   ├── Librarys/                # 工具库：SQLite, Browser(UrlHelper, Favicon), File/Process/Icon/Win32
│   ├── Models/                  # 数据模型：App/Db/Config/WebPage/Data/AppObserver
│   └── Servicers/
│       ├── Interfaces/          # 服务接口（IAppObserver, IData, IWebData, IMain...）
│       └── Instances/           # 服务实现
├── UI/                          # WPF 主程序
│   ├── Controls/                # 自定义控件（Charts/DatePickerBar/Navigation/Select/Toggle...）
│   ├── ViewModels/              # 页面 ViewModel
│   ├── Views/                   # 页面 XAML + code-behind
│   ├── Servicers/               # UI 层服务（MainServicer/Theme/StatusBarIcon/ContextMenu...）
│   ├── Models/                  # UI 层页面模型
│   └── Themes/                  # XAML 主题 (Dark/Light)
├── TaiBug/                      # 崩溃弹窗独立程序
├── Updater/                     # 自动更新独立程序
└── docs/                        # 项目文档
```

### 依赖方向（强制单向）

```
UI → Core
TaiBug → (独立，无依赖)
Updater → (独立，无依赖)
Core：不引用 UI，纯业务逻辑与数据访问
```

- **Core** 包含所有服务接口与实现、数据模型、工具库、Win32 API 封装。
- **UI** 引用 Core，通过 `Microsoft.Extensions.DependencyInjection` 在 `App.xaml.cs` 中完成 DI 注册。
- **TaiBug** / **Updater** 是独立 EXE，通过进程启动调用。

---

## 四、核心服务与 DI 注册

服务注册集中在 [App.xaml.cs](file:///d:/dev/store/Tai/UI/App.xaml.cs) 的 `ConfigureServices()` 方法中。

### 4.1 Core 层服务（单例）

| 接口 | 实现 | 职责 |
|------|------|------|
| `IMain` | `Main` | 核心编排：启动/停止所有监控服务、处理计时更新、自动分类、关联进程 |
| `IAppObserver` | `AppObserver` | 通过 Win32 SetWinEventHook 监听前台窗口切换 |
| `IAppManager` | `AppManager` | 根据窗口句柄获取进程信息（PID、进程名、路径） |
| `IWindowManager` | `WindowManager` | 获取窗口标题、类名、位置/大小 |
| `IAppTimerServicer` | `AppTimerServicer` | 1秒精度计时器，统计应用前台时长 |
| `IData` | `Data` | 应用使用数据的数据库读写（DailyLog + HoursLog） |
| `IWebData` | `WebData` | 网站浏览数据的数据库读写（WebSite + WebUrl + WebBrowseLog） |
| `IWebServer` | `WebServer` | WebSocket 服务器（端口 8908），接收浏览器插件数据 |
| `IWebFilter` | `WebFilter` | URL 过滤（域名/正则匹配） |
| `IAppConfig` | `AppConfig` | 配置管理：JSON 文件读写、变更通知 |
| `IAppData` | `AppData` | 应用信息缓存（内存列表 + 数据库同步） |
| `ICategorys` | `Categorys` | 应用分类管理（CRUD） |
| `IDatabase` | `Database` | 数据库连接池管理（读写分离） |
| `ISleepdiscover` | `Sleepdiscover` | 睡眠/离开检测（鼠标+键盘+电源事件） |
| `IDateTimeObserver` | `DateTimeObserver` | 跨小时触发（帮助时段数据切分） |

### 4.2 UI 层服务（单例）

| 接口 | 实现 | 职责 |
|------|------|------|
| `IMainServicer` | `MainServicer` | UI 启动入口，协调 Core 服务启动与 UI 初始化 |
| `IThemeServicer` | `ThemeServicer` | 深色/浅色主题切换 |
| `IStatusBarIconServicer` | `StatusBarIconServicer` | 系统托盘图标与右键菜单 |
| `IAppContextMenuServicer` | `AppContextMenuServicer` | 应用列表右键菜单构建 |
| `IWebSiteContextMenuServicer` | `WebSiteContextMenuServicer` | 网站列表右键菜单构建 |
| `IUIServicer` | `UIServicer` | 通用 UI 操作（对话框、Toast 通知等） |
| `IInputServicer` | `InputServicer` | 全局键盘输入监听（如 F5 刷新） |

---

## 五、MVVM 与页面导航

### 5.1 页面列表

| 页面 View | ViewModel | 用途 |
|-----------|-----------|------|
| `IndexPage` | `IndexPageVM` | 概览页：今日/本周使用最频繁应用 |
| `DataPage` | `DataPageVM` | 统计页：按天/月/年查看所有应用/网站 |
| `DetailPage` | `DetailPageVM` | 应用详情：查看单个应用的详细统计 |
| `WebSiteDetailPage` | `WebSiteDetailPageVM` | 网站详情：查看单个网站的详细统计 |
| `ChartPage` | `ChartPageVM` | 汇总图表：柱状图/饼图/雷达图等多种视图 |
| `CategoryPage` | `CategoryPageVM` | 分类管理：应用/网站分类的增删改查 |
| `CategoryAppListPage` | `CategoryAppListPageVM` | 分类下应用列表 |
| `CategoryWebSiteListPage` | `CategoryWebSiteListPageVM` | 分类下网站列表 |
| `SettingPage` | `SettingPageVM` | 设置页：常规/行为/关联进程/关于 |

### 5.2 导航方式

使用 `MainViewModel` 作为导航枢纽，通过 `Uri` 属性切换页面：

```csharp
// 跳转到应用详情页
main.Data = appModel;
main.Uri = nameof(DetailPage);

// 跳转到网站详情页
main.Data = webSiteModel;
main.Uri = nameof(WebSiteDetailPage);
```

页面容器 [PageContainer](file:///d:/dev/store/Tai/UI/Controls/PageContainer.cs) 根据 `Uri` 动态创建对应页面实例。

---

## 六、数据库模型 ER 关系

```
CategoryModel (应用分类)          WebSiteCategoryModel (网站分类)
  │ 1:N                              │ 1:N
  ▼                                  ▼
AppModel (应用)                   WebSiteModel (网站)
  │ 1:N                              │ 1:N
  ├── DailyLogModel (每日统计)       ├── WebBrowseLogModel (浏览记录)
  └── HoursLogModel (时段统计)       └── WebUrlModel (具体URL)
```

数据库文件位置：`运行目录/Data/data.db`，EF6 Code First 自动建表。

---

## 七、配置文件

| 文件 | 位置 | 格式 | 用途 |
|------|------|------|------|
| `AppConfig.json` | `Data/` | JSON | 应用设置（常规/行为/关联进程/忽略列表） |

配置模型：`ConfigModel` → `GeneralModel` + `BehaviorModel` + `Links(List<LinkModel>)`

配置变更通过 `AppConfig.ConfigChanged` 事件通知 `Main` 和 `WebFilter` 等订阅者。

---

## 八、学习路径建议

1. **第一天**：理解项目结构 → 阅读 [02_DATA_FLOW.md](./02_DATA_FLOW.md) 链路一和链路二
2. **第二天**：深入 Core 服务 → 逐文件阅读 `Main.cs`、`AppObserver.cs`、`AppTimerServicer.cs`、`Data.cs`
3. **第三天**：理解数据库模型 → 阅读 `TaiDbContext.cs` 和所有 `Models/Db/` 下的模型
4. **第四天**：理解 UI 层 → 阅读 `App.xaml.cs`（DI 注册）→ `MainWindow` → 各 ViewModel
5. **第五天**：实践任务 → 尝试添加一个新的过滤规则或修改统计逻辑