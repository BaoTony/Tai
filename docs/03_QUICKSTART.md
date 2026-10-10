# 03 – 开发者快速入门

> 📁 目标读者：首次接触 Tai 代码的开发者  
> 🎯 目标：30 分钟内完成环境搭建并运行项目  
> 📄 本文档遵循 [PlcDataStudio 文档规范]组织

---

## 1. 环境要求

| 依赖 | 版本要求 | 说明 |
|------|----------|------|
| 操作系统 | Windows 10+ | Tai 依赖 Win32 API，不支持其他平台 |
| .NET Framework | **4.8** | 开发目标框架（不是 .NET Core/.NET 5+） |
| Visual Studio | 2022 / 2019 | 需安装 `.NET 桌面开发` 工作负载 |
| Git | 任意版本 | 用于克隆仓库和管理分支 |

### 1.1 .NET Framework 4.8 安装验证

```powershell
# 查看已安装的 .NET Framework 版本
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP\v4\Full" | Select-Object Release, Version
```

- Release >= `528040` → 4.8 已安装
- 未安装则在 Tai 启动时会无响应，去 [Microsoft 官网](https://dotnet.microsoft.com/en-us/download/dotnet-framework/net48) 下载 Runtime

---

## 2. 获取代码与编译

### 2.1 目录结构速查

```
Tai/
├── Tai.sln                    # 解决方案，一般双击此文件打开
├── Core/                      # 核心业务层
│   ├── Librarys/              #   工具库（Win32 P/Invoke、配置读写、SQLite上下文等）
│   ├── Servicers/
│   │   ├── Interfaces/        #   服务接口（IAppObserver、IData、IWebData…）
│   │   └── Instances/         #   服务实现（AppObserver、Data、WebServer…）
│   └── Models/                #   数据模型与 DTO
├── UI/                        # WPF 主程序
│   ├── MainWindow.xaml(.cs)   #   主窗口入口
│   ├── App.xaml(.cs)          #   应用启动 + DI 注册
│   ├── Views/                 #   XAML 页面视图
│   │   ├── IndexPage.xaml     #     首页/仪表盘
│   │   ├── DetailPage.xaml    #     应用详情页
│   │   ├── ChartPage.xaml     #     图表统计页
│   │   ├── CategoryPage.xaml  #     分类管理页
│   │   ├── DataPage.xaml      #     数据导出/清理页
│   │   ├── SettingPage.xaml   #     设置页
│   │   └── WebSiteDetailPage.xaml # 网站详情页
│   ├── ViewModels/            #   MVVM ViewModel
│   ├── Controls/              #   自定义 UI 控件库（导航、图表、选择器等）
│   ├── Themes/                #   控件默认主题 XAML
│   └── Resources/Themes/     #   深色/浅色主题资源字典
├── TaiBug/                    # 崩溃弹窗独立程序
└── Updater/                   # 自动更新独立程序
```

### 2.2 编译运行

```powershell
# 方法一：Visual Studio
#   双击 Tai/Tai.sln → 右键 Tai 项目设为启动项目 → F5

# 方法二：命令行 MSBuild（VS 开发者命令提示符中执行）
msbuild /t:Build /p:Configuration=Debug Tai\Tai.sln

# 运行产物
.\UI\bin\Debug\Tai.exe
```

### 2.3 ⚠️ 必须以管理员权限运行

Tai 的核心功能依赖 `SetWinEventHook`（前景窗口监控 Win32 API），**必须以管理员权限运行**。Visual Studio 调试时同理：

> VS → 右键 「Tai」项目 → 属性 → 安全性 → 勾选「启用 ClickOnce 安全性设置」  
> 或者在 `app.manifest` 中配置 `<requestedExecutionLevel level="requireAdministrator" />`

---

## 3. 运行调试关键位置

### 3.1 程序入口

| 文件 | 关键方法 | 作用 |
|------|----------|------|
| [UI/App.xaml.cs](file:///d:/dev/store/Tai/UI/App.xaml.cs) | `OnStartup()` | DI 容器构建 + 所有服务注册 |
| [Core/Servicers/Instances/Main.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/Main.cs) | `Run()` | 启动所有核心服务（AppObserver、WebServer、Sleepdiscover...） |
| [UI/MainWindow.xaml.cs](file:///d:/dev/store/Tai/UI/MainWindow.xaml.cs) | `MainWindow_Loaded()` | 主窗口加载 → 导航到首页 |

### 3.2 数据库位置

```
运行目录\Data\data.db       # SQLite 数据库文件（EF6 Code First 自动创建）
运行目录\Data\Tai.config     # 用户配置文件（JSON）
```

- 数据库由 EF6 的 `CreateDatabaseIfNotExists` 策略在首次访问时自动创建
- 开发调试时如需重置数据：直接删除 `Data` 目录，重启程序即可重新生成
- 查看数据库内容：[DB Browser for SQLite](https://sqlitebrowser.org/) 或 VS Code 插件

### 3.3 WebSocket 调试

Tai 在 `ws://localhost:8908/TaiWebSentry` 上启动 WebSocket 服务，用于接收浏览器插件推送的网页浏览记录。

**调试技巧**（无需实际安装浏览器插件）：

```powershell
# 使用 websocat 或 wscat 发送测试 JSON：
# 消息格式见 Core/Models/WebPage/NotifyWeb.cs
```

也可以直接用浏览器 Console 连接测试：

```javascript
const ws = new WebSocket('ws://localhost:8908/TaiWebSentry');
ws.onopen = () => ws.send(JSON.stringify({
    Url: 'https://example.com/test-page',
    Title: '测试页面'
}));
```

---

## 4. 核心依赖 NuGet 包

| 包名 | 用途 | 关键版本 |
|------|------|----------|
| `EntityFramework` | ORM，Code First 操作 SQLite | 6.4.4 |
| `System.Data.SQLite` | SQLite ADO.NET Provider | — |
| `Microsoft.Extensions.DependencyInjection` | DI 容器 | 6.0 |
| `WebSocketSharp` | WebSocket 服务端（浏览器插件通信） | — |
| `Newtonsoft.Json` | JSON 序列化（配置文件读写） | — |
| `NPOI` | Excel 导出（.xlsx） | 2.5.6 |
| `CsvHelper` | CSV 导出 | 29.0 |

---

## 5. 从页面出发的代码导航

如果你只想改某个页面，直接定位到对应文件：

| 页面 | View | ViewModel | 依赖的核心服务 |
|------|------|-----------|----------------|
| 🏠 首页仪表盘 | [UI/Views/IndexPage.xaml](file:///d:/dev/store/Tai/UI/Views/IndexPage.xaml) | [UI/ViewModels/IndexPageViewModel.cs](file:///d:/dev/store/Tai/UI/ViewModels/IndexPageViewModel.cs) | `IData`, `ICategorys` |
| 📊 图表统计 | [UI/Views/ChartPage.xaml](file:///d:/dev/store/Tai/UI/Views/ChartPage.xaml) | [UI/ViewModels/ChartPageViewModel.cs](file:///d:/dev/store/Tai/UI/ViewModels/ChartPageViewModel.cs) | `IData`, `IWebData` |
| 📱 应用详情 | [UI/Views/DetailPage.xaml](file:///d:/dev/store/Tai/UI/Views/DetailPage.xaml) | [UI/ViewModels/DetailPageViewModel.cs](file:///d:/dev/store/Tai/UI/ViewModels/DetailPageViewModel.cs) | `IAppData`, `IData` |
| 🌐 网站详情 | [UI/Views/WebSiteDetailPage.xaml](file:///d:/dev/store/Tai/UI/Views/WebSiteDetailPage.xaml) | [UI/ViewModels/WebSiteDetailPageViewModel.cs](file:///d:/dev/store/Tai/UI/ViewModels/WebSiteDetailPageViewModel.cs) | `IWebData` |
| 📂 分类管理 | [UI/Views/CategoryPage.xaml](file:///d:/dev/store/Tai/UI/Views/CategoryPage.xaml) | [UI/ViewModels/CategoryPageViewModel.cs](file:///d:/dev/store/Tai/UI/ViewModels/CategoryPageViewModel.cs) | `ICategorys` |
| ⚙ 设置 | [UI/Views/SettingPage.xaml](file:///d:/dev/store/Tai/UI/Views/SettingPage.xaml) | [UI/ViewModels/SettingPageViewModel.cs](file:///d:/dev/store/Tai/UI/ViewModels/SettingPageViewModel.cs) | `IAppConfig` |
| 📤 数据管理 | [UI/Views/DataPage.xaml](file:///d:/dev/store/Tai/UI/Views/DataPage.xaml) | [UI/ViewModels/DataPageViewModel.cs](file:///d:/dev/store/Tai/UI/ViewModels/DataPageViewModel.cs) | `IData`, `IWebData` |

---

## 6. 常见调试问题

### Q1：启动后状态栏没有图标？
- 检查 `Data/Tai.config` 是否被错误修改
- 查看 `%APPDATA%` 或运行目录下是否有 `error.log` 文件

### Q2：无法统计部分软件？
- **必须以管理员权限运行**，否则 `SetWinEventHook` 无法捕获高权限进程
- 检查该软件是否在过滤列表中（`设置 → 行为 → 过滤应用`）

### Q3：网站浏览统计不工作？
- 确认已在 `设置 → 常规 → 功能` 中启用网站统计
- 安装浏览器扩展并确认已连接（扩展图标显示连接状态）
- 手动检查 `ws://localhost:8908/TaiWebSentry` 是否可访问

### Q4：启动后直接闪退？
- 确认 .NET Framework 4.8 已安装（见 1.1 节）
- 以管理员身份运行命令提示符，从命令行启动查看错误日志：
  ```powershell
  .\Tai.exe
  ```

### Q5：EF6 数据库迁移报错？
- 删除 `Data/data.db`，让 EF6 重新建库
- Tai 使用的是 `CreateDatabaseIfNotExists` 策略，不涉及显式 Migration

---

## 7. 编译产物结构

```
bin/Debug/ 或 bin/Release/
├── Tai.exe                  # 主程序
├── Tai.exe.config           # .NET Framework 配置（绑定重定向等）
├── TaiBug.exe               # 崩溃弹窗程序
├── Updater.exe              # 自动更新程序
├── EntityFramework.dll      # EF6 运行时
├── System.Data.SQLite.dll   # SQLite 运行时
│     └── x86/
│     └── x64/               # 分平台原生二进制（SQLite.Interop.dll）
├── *.dll                    # 其他依赖（NPOI、WebSocketSharp 等）
└── Data/                    # 运行时数据（首次运行后生成）
    ├── data.db              #   SQLite 数据库
    └── Tai.config           #   用户配置
```

---

## 8. 下一步

完成入门后，建议按以下顺序深入学习：

| 顺序 | 文档 | 内容 |
|------|------|------|
| 1 | [docs/02_DATA_FLOW.md](02_DATA_FLOW.md) | 10 条完整数据链路详解 + 分步学习路线 |
| 2 | [docs/04_UI_CONTROLS_GUIDE.md](04_UI_CONTROLS_GUIDE.md) | 自定义 UI 控件体系与主题开发 |
| 3 | [docs/01_PROJECT_OVERVIEW.md](01_PROJECT_OVERVIEW.md) | 项目架构全貌与设计决策 |