# Tai 项目完整数据链路分析

> 分析时间：2026-10-10
> 版本：基于当前 master 分支最新代码
> 关联文档：项目概览见 [`01_PROJECT_OVERVIEW.md`](./01_PROJECT_OVERVIEW.md)

---

## 零、应用启动完整链路

> 追踪从 `App` 构造函数到首屏渲染的完整过程，涵盖 DI 容器初始化、服务启动、主窗口显示。

### 阶段一：DI 容器初始化（App 构造函数）

```
App() [UI/App.xaml.cs]
  │
  ├── DispatcherUnhandledException += App_DispatcherUnhandledException  全局异常兜底
  │
  └── ConfigureServices(IServiceCollection)
      │
      ├── 数据库
      │   └── IDatabase → Database (单例，SQLite 读写分离连接池)
      │
      ├── 窗口监控
      │   ├── IAppManager → AppManager (单例)
      │   ├── IWindowManager → WindowManager (单例)
      │   ├── IAppTimerServicer → AppTimerServicer (单例)
      │   └── IAppObserver → AppObserver (单例)
      │
      ├── Web 服务
      │   ├── IWebServer → WebServer (单例，WebSocket 8908端口)
      │   ├── IWebData → WebData (单例)
      │   └── IWebFilter → WebFilter (单例)
      │
      ├── 数据处理
      │   ├── IData → Data (单例)
      │   ├── IAppData → AppData (单例)
      │   └── ICategorys → Categorys (单例)
      │
      ├── 核心编排
      │   └── IMain → Main (单例，注入上述所有服务)
      │
      ├── 辅助服务
      │   ├── IAppConfig → AppConfig (单例)
      │   ├── IDateTimeObserver → DateTimeObserver (单例)
      │   └── ISleepdiscover → Sleepdiscover (单例)
      │
      └── UI 层服务
          ├── IUIServicer → UIServicer / IThemeServicer → ThemeServicer
          ├── IStatusBarIconServicer → StatusBarIconServicer
          ├── IAppContextMenuServicer → AppContextMenuServicer
          ├── IWebSiteContextMenuServicer → WebSiteContextMenuServicer
          ├── IMainServicer → MainServicer / IInputServicer → InputServicer
          └── MainViewModel (单例)
```

**关键点**：此阶段仅注册类型映射（`services.AddSingleton<I, T>()`），服务实例尚未创建。实际实例化在 `serviceProvider.GetService()` 时触发（Lazy）。

### 阶段二：启动与首屏渲染

```
App.OnStartup(StartupEventArgs e)
  │
  ├── ▶ 1. IsRuned() 检查
  │   └── Mutex 互斥体防止多开实例
  │
  ├── ▶ 2. serviceProvider.GetService<IMainServicer>().Start(isSelfStart)
  │   └── MainServicer.Start()
  │       ├── _statusBarIconServicer.Init()         系统托盘图标初始化
  │       ├── main.OnStarted += Main_OnStarted      注册启动完成回调
  │       └── main.Run()                            启动核心服务 (见下方)
  │
  ├── ▶ 3. serviceProvider.GetService<MainWindow>()
  │   └── DataContext = 单例 MainViewModel
  │
  └── ▶ 4. mainWindow.Show()
      └── 首屏渲染 (根据配置 StartPage 导航到对应页面)
```

### imain.Run() 执行流程

```
Main.Run() [Core/Servicers/Instances/Main.cs]
  │
  ├── Task.Run 后台初始化：
  │   ├── CreateDirectory()                         创建 Data/ 目录
  │   ├── new TaiDbContext().SelfCheck()            数据库自检（SQLite 表存在性）
  │   ├── appData.Load()                            加载所有 AppModel 到内存缓存
  │   ├── categories.Load()                         加载所有 CategoryModel
  │   └── AppState.IsLoading = false                标记加载完成
  │
  ├── appConfig.Load()                             读取/创建 AppConfig.json
  │   ├── File.Exists? → JsonConvert.DeserializeObject<ConfigModel>
  │   └── No File → CreateDefaultConfig() → Save()
  │
  ├── config = appConfig.GetConfig()               获取配置快照
  ├── UpdateConfigIgnoreProcess()                  构建忽略进程列表
  ├── UpdateConfigProcessWhiteList()               构建白名单列表
  ├── _webFilter.Init()                            初始化 URL 过滤器
  │
  ├── Start()                                      启动监控服务
  │   ├── _appTimer.Start()                       先启动计时器（必须比Observer先启动）
  │   ├── appObserver.Start()                     启动前台窗口监听
  │   │   └── SetWinEventHook(0x0003, ...)        Win32 全局钩子
  │   ├── if IsWebEnabled: _webServer.Start()     按需启动 WebSocket 服务
  │   └── if IsSleepWatch: sleepdiscover.Start()  按需启动睡眠监测
  │
  └── OnStarted?.Invoke(this, EventArgs.Empty)    通知 UI 层初始化完成
      └── MainServicer.Main_OnStarted()
          ├── themeServicer.Init()                 加载主题
          ├── inputServicer.Start()                键盘监听（F5刷新）
          ├── appContextMenuServicer.Init()        应用右键菜单
          └── _webSiteContext.Init()              网站右键菜单
```

---

## 一、整体架构分层

```
┌───────────────────────────────────────────────────────────────────────┐
│  UI 层（Views / ViewModels / Controls / Themes）                       │
│  IndexPage · DataPage · DetailPage · ChartPage · CategoryPage         │
│  SettingPage · WebSiteDetailPage · CategoryAppListPage …              │
│  MainWindow（DefaultWindow, 托盘图标）                                  │
│  页面导航：MainViewModel.Uri 属性驱动 PageContainer 动态切换            │
└───────────────────────────────┬───────────────────────────────────────┘
                                │ 绑定 / 命令 / ServiceProvider.GetService
┌───────────────────────────────▼───────────────────────────────────────┐
│  UI Servicers 层                                                       │
│  MainServicer（启动编排）· ThemeServicer（主题）· InputServicer（键盘） │
│  StatusBarIconServicer（托盘）· AppContextMenuServicer（菜单）          │
│  WebSiteContextMenuServicer（网站菜单）· UIServicer（对话框/Toast）      │
└───────────────────────────────┬───────────────────────────────────────┘
                                │ DI 注入（构造函数依赖 Core 接口）
┌───────────────────────────────▼───────────────────────────────────────┐
│  Core Servicers 层（业务逻辑核心）                                      │
│                                                                         │
│  Main（编排中心）                                                       │
│  ├── AppObserver（前台窗口监控）  ──→  AppManager + WindowManager       │
│  ├── AppTimerServicer（1秒计时）  ──→  订阅 AppObserver 事件            │
│  ├── WebServer（WebSocket 8908）  ──→  接收浏览器插件数据               │
│  ├── Sleepdiscover（睡眠检测）    ──→  鼠标+键盘+电源钩子               │
│  ├── DateTimeObserver（跨小时）   ──→  59分59秒触发                     │
│  ├── Data（应用数据读写）         ──→  DailyLog + HoursLog             │
│  ├── WebData（网站数据读写）      ──→  WebSite + WebUrl + WebBrowseLog  │
│  ├── AppData（应用缓存）          ──→  内存 List<AppModel>              │
│  ├── Categorys（分类管理）        ──→  CategoryModel CRUD               │
│  ├── WebFilter（URL 过滤）        ──→  域名/正则匹配                    │
│  └── AppConfig（配置管理）        ──→  AppConfig.json 读写              │
└───────────────────────────────┬───────────────────────────────────────┘
                                │ EF6 DbContext / Win32 API
┌───────────────────────────────▼───────────────────────────────────────┐
│  基础设施层                                                             │
│  TaiDbContext（EF6 Code First）  │  Win32API（user32/kernel32/psapi）  │
│  SQLite 数据库文件 (Data/data.db) │  窗口句柄/进程信息/图标提取         │
└───────────────────────────────────────────────────────────────────────┘
```

**依赖方向（强制，单向）**：`UI → Core → 基础设施`。Core 不引用 UI；UI 仅通过接口消费 Core 服务。

---

## 二、链路一：应用使用时间追踪（核心链路）

> 这是项目最重要的数据链路，从 Windows 窗口焦点变化到数据库持久化、再到 UI 展示的完整流程。

### 2.1 数据流总览

```
Win32 前台窗口变化
  │
  ▼
[AppObserver]  SetWinEventHook(EVENT_SYSTEM_FOREGROUND)
  │  ForegroundEventCallback(hwnd)
  ▼
[AppManager]   GetWindowThreadProcessId → OpenProcess → GetModuleBaseName
  │  返回 AppInfo(PID, ProcessName, Description, ExecutablePath, Type)
  ▼
[WindowManager] GetWindowText / GetWindowRect / GetWindowClassName
  │  返回 WindowInfo(ClassName, Title, Handle, Width, Height, X, Y)
  ▼
[AppObserver]  触发 OnAppActiveChanged(AppActiveChangedEventArgs)
  │
  ▼
[AppTimerServicer] 订阅 OnAppActiveChanged
  │  ├── 进程切换? → StopTimer() → InvokeEvent() → StartTimer()
  │  ├── IsStatistical() 判断：排除系统组件、空进程
  │  └── Timer(1s) 累加 _appDuration
  ▼
触发 OnAppDurationUpdated(AppDurationUpdatedEventArgs)
  │  { Duration, App, Window, ActiveTime, EndTime }
  ▼
[Main._appTimer_OnAppDurationUpdated]
  │  ├── IsCheckApp(): 默认忽略检查 → 配置忽略检查 → 白名单检查
  │  ├── 新应用：Iconer.ExtractFromFile → AppData.AddApp()
  │  ├── 已有应用：更新图标/描述 → AppData.UpdateApp()
  │  ├── data.UpdateAppDuration(process, duration, startTime)
  │  ├── HandleLinks(): 关联进程同步
  │  └── DispatchCategory(): 自动分类匹配
  ▼
[Data.UpdateAppDuration]
  │  ├── 时段拆分：duration → Dictionary<DateTime, int> (按小时)
  │  ├── 日期拆分：duration → Dictionary<DateTime, int> (按天)
  │  └── 写入 SQLite：
  │       ├── AppModel.TotalTime += duration
  │       ├── DailyLogModel (按天聚合，上限86400秒)
  │       └── HoursLogModel (按小时聚合，上限3600秒)
  ▼
UI 查询展示
  │  ├── IndexPageVM: GetTodaylogList() / GetThisWeeklogList()
  │  ├── DataPageVM: GetDateRangelogList() / GetCategoryRangeData()
  │  ├── DetailPageVM: GetProcessMonthLogList() / GetAppDayData()
  │  └── ChartPageVM: GetCategoryYearData() / GetAppYearData()
```

### 2.2 分步详解

#### 步骤1：Windows 前台窗口监听 — AppObserver

**文件**：[AppObserver.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/AppObserver.cs)

```
工作流程：
1. Start() 调用 SetWinEventHook(0x0003, 0x0003, ...)
   - 0x0003 = EVENT_SYSTEM_FOREGROUND（前台窗口变化）
2. Win32 回调 ForegroundEventCallback(hwnd, ...)
   - 先调用 HandleForegroundWindow() 获取当前窗口
3. 调用 GetAppInfoEventArgs(hwnd, activeTime):
   - _appManager.GetAppInfo(hwnd) → AppInfo
   - _windowManager.GetWindowInfo(hwnd) → WindowInfo
4. 触发 OnAppActiveChanged(AppActiveChangedEventArgs)

特殊处理：
- 系统组件（SystemComponent）：启动 1秒延迟检测 Timer
- 延迟检测后若非系统组件则再次触发事件
```

#### 步骤2：进程信息获取 — AppManager

**文件**：[AppManager.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/AppManager.cs)

```
GetAppInfo(IntPtr hwnd):
1. GetWindowThreadProcessId(hwnd, out pid)      获取进程ID
2. GetAppProcessName(pid):
   - OpenProcess(QUERY_INFORMATION | VM_READ)   打开进程
   - GetModuleBaseName()                         获取进程名
3. 特殊处理：
   - ApplicationFrameHost → 找到实际UWP子窗口 → 递归获取
   - SystemSettings → AppType.SystemComponent
4. 缓存 Dictionary<string, AppInfo> _apps       避免重复查询
5. 返回 AppInfo(PID, Process, Description, ExecutablePath, Type)
```

**返回数据模型**：[AppInfo.cs](file:///d:/dev/store/Tai/Core/Models/AppObserver/AppInfo.cs)

```
public class AppInfo {
    int PID;                    // 进程ID
    AppType Type;              // Win32 / UWP / SystemComponent
    string Process;             // 进程名称 (如 "chrome")
    string Description;         // 文件描述 (如 "Google Chrome")
    string ExecutablePath;      // 完整路径 (如 "C:\...\chrome.exe")
    IntPtr WindowHandle;        // 窗口句柄
}
```

#### 步骤3：窗口信息获取 — WindowManager

**文件**：[WindowManager.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/WindowManager.cs)

```
GetWindowInfo(IntPtr handle):
1. GetWindowText()             窗口标题
2. GetWindowRect()             窗口位置/大小
3. GetWindowClassName()        窗口类名
4. 返回 WindowInfo(ClassName, Title, Handle, Width, Height, X, Y)
```

#### 步骤4：计时服务 — AppTimerServicer

**文件**：[AppTimerServicer.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/AppTimerServicer.cs)

```
核心逻辑：
1. 订阅 _appObserver.OnAppActiveChanged
2. 进程切换时：
   - StopTimer()：_timer.Stop(), 记录 _endTime = DateTime.Now
   - InvokeEvent()：调用 GetAppDuration() 组装事件参数并触发
   - StartTimer()：_timer.Start(), 记录 _startTime = DateTime.Now, _appDuration = 0
3. Timer(1s) 每 1000ms：_appDuration++
4. IsStatistical() 判断：
   - 排除 SystemComponent
   - 排除空进程名
   - 排除空 ExecutablePath
5. 防重复：_lastInvokeEventArgs 比较防止重复触发
```

#### 步骤5：核心编排 — Main

**文件**：[Main.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/Main.cs)

```
_appTimer_OnAppDurationUpdated(e):
  └── UpdateAppDuration(e)
      ├── IsCheckApp(app.Process, app.Description, app.ExecutablePath):
      │   ├── 默认忽略: "Tai", "Taskmgr", "SearchHost", "dwm", "LockApp"...
      │   ├── 缓存忽略: IgnoreProcessCacheList
      │   ├── 配置忽略: ConfigIgnoreProcessList (精确名称)
      │   ├── 正则忽略: ConfigIgnoreProcessRegxList (匹配进程名或路径)
      │   └── 白名单检查: IsWhiteList → ProcessWhiteList 匹配
      │
      ├── data.UpdateAppDuration(app.Process, duration, startTime)
      ├── HandleLinks(app.Process, duration, startTime)  关联进程同步
      └── DispatchCategory(app.Process, app.ExecutablePath)  自动分类
```

**过滤规则判断逻辑**（`IsCheckApp`）：
```
输入: (processName, description, file)
  ├── file为空 或 processName为空 → false
  ├── DefaultIgnoreProcess 包含 → false
  ├── IgnoreProcessCacheList 包含 → false
  ├── ConfigIgnoreProcessList 包含 → false
  ├── ConfigIgnoreProcessRegxList 匹配 → 加入缓存, false
  ├── IsWhiteList == true:
  │   ├── _configProcessNameWhiteList 包含 → true
  │   ├── _configProcessRegexWhiteList 匹配 → true
  │   └── 都不匹配 → false
  └── 通过 → true
```

#### 步骤6：数据持久化 — Data

**文件**：[Data.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/Data.cs)

```
UpdateAppDuration(string process_, int duration_, DateTime startTime_):
1. 过滤无效值: string.IsNullOrEmpty 或 duration <= 0 → return

2. 时段拆分（按小时）:
   例: duration=5000秒, startTime=10:30:00
   ├── 10:00～11:00: max=1800秒(30分钟) → 实际1800秒
   ├── 11:00～12:00: 3200秒 > 3600 → 3600秒
   └── 结果: {10:00: 1800, 11:00: 3600}

3. 日期拆分（按天）:
   例: duration=100000秒 (跨天场景)
   ├── 当天剩余秒数计算
   └── 跨天部分按每天86400秒拆分

4. 写入数据库 (TaiDbContext):
   ├── App.TotalTime += duration_                 累计总时长
   ├── DailyLogModel.Time 更新/新增                每日 (上限86400)
   └── HoursLogModel.Time 更新/新增                每小时 (上限3600)
```

#### 步骤7：自动分类 — Main.DispatchCategory

**文件**：[Main.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/Main.cs)（第507行起）

```
DispatchCategory(string processName_, string executablePath_):
1. 获取 AppModel
2. 获取所有启用目录匹配的分类 (IsDirectoryMath == true)
3. 遍历分类的 DirectoryList (JSON List<string>)
4. 用 Regex.IsMatch(executablePath_, pattern) 匹配
5. 匹配成功 → app.CategoryID = category.ID → appData.UpdateApp()
```

#### 步骤8：关联进程同步 — Main.HandleLinks

**文件**：[Main.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/Main.cs)（第373行起）

```
HandleLinks(string processName, int seconds, DateTime time):
1. 读取 config.Links (List<LinkModel>)
2. 查找包含 processName 的关联组
3. 遍历关联组中其他进程
4. 如果其他进程正在运行 (IsProcessRuning):
   → data.UpdateAppDuration(linkProcess, seconds, time) 同步更新
```

### 2.3 学习步骤

| 步骤 | 任务 | 目标文件 | 预期耗时 |
|------|------|----------|----------|
| 1 | 理解 Win32 窗口监控机制 | [AppObserver.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/AppObserver.cs) | 30min |
| 2 | 理解进程信息获取流程 | [AppManager.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/AppManager.cs) | 30min |
| 3 | 理解计时与事件触发逻辑 | [AppTimerServicer.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/AppTimerServicer.cs) | 30min |
| 4 | 理解核心编排与过滤规则 | [Main.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/Main.cs) (全文) | 60min |
| 5 | 理解时段/日期拆分算法 | [Data.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/Data.cs) UpdateAppDuration方法 | 45min |
| 6 | 理解数据模型 | [AppModel.cs](file:///d:/dev/store/Tai/Core/Models/AppModel.cs), [DailyLogModel.cs](file:///d:/dev/store/Tai/Core/Models/DailyLogModel.cs), [HoursLogModel.cs](file:///d:/dev/store/Tai/Core/Models/HoursLogModel.cs) | 20min |
| 7 | 理解UI层数据查询展示 | [IndexPageVM.cs](file:///d:/dev/store/Tai/UI/ViewModels/IndexPageVM.cs), [DataPageVM.cs](file:///d:/dev/store/Tai/UI/ViewModels/DataPageVM.cs) | 45min |

---

## 三、链路二：网页浏览追踪

> 通过浏览器插件 + WebSocket 接收浏览数据，过滤后写入数据库。

### 3.1 数据流总览

```
浏览器插件 (Chrome/Edge Extension)
  │ 监控标签页切换和停留时间
  │ WebSocket → ws://localhost:8908/TaiWebSentry
  ▼
[WebServer] OnMessage(MessageEventArgs e)
  │ JsonConvert.DeserializeObject<NotifyWeb>(e.Data)
  │  { Url, Title, Icon(favicon URL), ActiveTime(unix), Duration }
  ▼
[WebSocketEvent.Invoke(log)]
  │ 静态事件：触发 OnWebLog(NotifyWeb)
  ▼
[Main.WebSocketEvent_OnWebLog]
  │
  ├── WebFilter.IsIgnore(args.Url)?
  │   ├── 域名精确匹配 (Behavior.IgnoreURLList)
  │   ├── 正则匹配 (支持 *, $, !, (, ), {, [, ^, |)
  │   ├── 默认过滤: ^chrome://, ^edge://, ^view-source:
  │   └── 命中 → return (丢弃)
  │
  ├── _webData.AddUrlBrowseTime(site, duration, activeDateTime)
  │   └── 见下方 WebData 链路
  │
  └── Task.Run: 异步下载 Favicon
      ├── FaviconDownloader.DownloadAsync(iconUrl, saveName)
      └── _webData.UpdateUrlFavicon(site, localPath)
  ▼
[WebData.AddUrlBrowseTime]
  │
  ├── UrlHelper.GetDomain(url)                       提取域名
  ├── 查找/创建 WebSiteModel (按 Domain)             站点记录
  │   └── 新站点: Title=UrlHelper.GetName(url), Domain
  ├── GetCreateUrl(site)                             查找/创建 WebUrlModel
  │   └── Title, Url, IconFile
  ├── 查找/创建 WebBrowseLogModel (按 LogTime + UrlId)
  │   └── Duration 累加/新建 (上限3600秒)
  ├── site.Duration += nowTimeDuration               站点累计时长
  └── 递归: nextTimeDuration > 0 → AddUrlBrowseTime(next)
  ▼
UI 查询展示 (IndexPageVM / DataPageVM / WebSiteDetailPageVM)
```

### 3.2 关键组件详解

#### WebServer — WebSocket 服务器

**文件**：[WebServer.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/WebServer.cs)

```
WebSocketSharp.Server 框架：
- 端口: 8908
- 路径: /TaiWebSentry
- Start(): new WebSocketServer(8908) → AddWebSocketService → Start()
- Stop(): _webSocket.Stop()
- SendMsg("sleep" / "wake"): Broadcast() 通知浏览器插件
- OnMessage(): 反序列化 NotifyWeb JSON → WebSocketEvent.Invoke()
```

#### WebFilter — URL 过滤

**文件**：[WebFilter.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/WebFilter.cs)

```
IsIgnore(string url):
1. string.IsNullOrEmpty(url) → true
2. _ignoreURLCache.Contains(url) → true (缓存命中)
3. IsDomainIgnore(url):
   - UrlHelper.GetDomain(url) → 提取域名
   - ignoreURLList.Contains(domain) → true
4. IsRegexIgnore(url):
   - 正则字符检测: [\*|\$|\!|\(|\)|\{|\|\[|\^|\|]
   - 默认规则: ^chrome://(.*), ^edge://(.*), ^view-source:(.*)
   - 匹配任一规则 → 加入缓存, true
5. 通过所有检查 → false
```

#### WebData — 网站数据持久化

**文件**：[WebData.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/WebData.cs)

```
AddUrlBrowseTime(Site site_, int duration_, DateTime? dateTime_):
1. 时段拆分逻辑：
   - 当前时刻最大时长: (60 - dateTime.Minute) * 60 秒
   - 超过最大时长 → 递归调用 (带下一小时时间戳)
2. 数据库操作 (TaiDbContext):
   ├── WebSites: 按 Domain 查找/创建
   │     Title = UrlHelper.GetName(url)
   ├── WebUrls: GetCreateUrl() 查找/创建
   │     Title, Url, IconFile
   └── WebBrowserLogs: 按 LogTime+UrlId 查找/更新/创建
         Duration 累加 (上限3600)
3. site.Duration 同步更新
4. 递归处理超出的 nextTimeDuration
```

### 3.3 数据模型关系

```
NotifyWeb (浏览器上报JSON)
  ├── Url                  → WebUrlModel.Url
  ├── Title                → WebUrlModel.Title / WebSiteModel.Title
  ├── Icon                 → 下载到本地 → WebUrlModel.IconFile / WebSiteModel.IconFile
  ├── ActiveTime (unix)    → ActiveDateTime (本地时间)
  └── Duration             → WebBrowseLogModel.Duration

WebSiteModel (站点聚合)     WebUrlModel (具体URL)
  ├── Domain (域名)          ├── Url (完整链接)
  ├── Duration (累计时长)    ├── Title
  └── CategoryID             └── IconFile

WebBrowseLogModel (浏览记录)
  ├── LogTime (YYYY-MM-DD HH:00:00)
  ├── Duration (秒)
  ├── UrlId → WebUrlModel
  └── SiteId → WebSiteModel
```

### 3.4 学习步骤

| 步骤 | 任务 | 目标文件 | 预期耗时 |
|------|------|----------|----------|
| 1 | 理解 WebSocket 通信机制 | [WebServer.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/WebServer.cs) | 20min |
| 2 | 理解浏览器上报数据格式 | [NotifyWeb.cs](file:///d:/dev/store/Tai/Core/Models/WebPage/NotifyWeb.cs) | 10min |
| 3 | 理解 WebFilter 过滤逻辑 | [WebFilter.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/WebFilter.cs) | 20min |
| 4 | 理解网站数据写入流程 | [WebData.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/WebData.cs) AddUrlBrowseTime方法 | 45min |
| 5 | 理解 URL 工具方法 | [UrlHelper.cs](file:///d:/dev/store/Tai/Core/Librarys/Browser/UrlHelper.cs) | 20min |
| 6 | 理解 网站数据模型 | [WebSiteModel.cs](file:///d:/dev/store/Tai/Core/Models/Db/WebSiteModel.cs), [WebUrlModel.cs](file:///d:/dev/store/Tai/Core/Models/Db/WebUrlModel.cs), [WebBrowseLogModel.cs](file:///d:/dev/store/Tai/Core/Models/Db/WebBrowseLogModel.cs) | 15min |
| 7 | 理解 Favicon 下载 | [FaviconDownloader.cs](file:///d:/dev/store/Tai/Core/Librarys/Browser/Favicon/FaviconDownloader.cs) | 15min |
| 8 | 理解 UI 层网站数据展示 | [IndexPageVM.cs](file:///d:/dev/store/Tai/UI/ViewModels/IndexPageVM.cs) Web 相关部分, [WebSiteDetailPageVM.cs](file:///d:/dev/store/Tai/UI/ViewModels/WebSiteDetailPageVM.cs) | 30min |

---

## 四、链路三：配置管理

> 配置文件读取、修改、保存、事件通知的完整链路。

### 4.1 数据流

```
AppConfig.Load() [Core/Servicers/Instances/AppConfig.cs]
  │
  ├── 读取 Data/AppConfig.json
  ├── JsonConvert.DeserializeObject<ConfigModel>
  │   └── ConfigModel
  │       ├── Links: List<LinkModel>         关联进程配置
  │       ├── General: GeneralModel          常规设置
  │       └── Behavior: BehaviorModel        行为设置
  │
  ├── 文件不存在 → CreateDefaultConfig() → Save()
  ├── CheckOption(config)                    递归检查所有属性非null
  └── CopyToOldConfig()                      备份旧配置 (用于变更对比)
  │
  ▼
UI 修改配置 (SettingPageVM)
  │ 修改 config 对象属性
  ▼
AppConfig.Save()
  │ File.WriteAllText(fileName, JsonConvert.SerializeObject(config))
  │ ConfigChanged?.Invoke(oldConfig, config)        触发变更事件
  └── CopyToOldConfig()                             更新旧配置副本
  │
  ▼
订阅者响应 ConfigChanged 事件：
  │
  ├── Main.AppConfig_ConfigChanged:
  │   ├── SystemCommon.SetStartup(IsStartatboot)    开机自启
  │   ├── UpdateConfigIgnoreProcess()               更新忽略进程
  │   ├── UpdateConfigProcessWhiteList()            更新白名单
  │   └── HandleWebServiceConfig()                  Web服务启停
  │
  └── WebFilter._appConfig_ConfigChanged:
      └── _ignoreURLCache.Clear()                   清空过滤缓存
```

### 4.2 配置模型结构

```
ConfigModel
├── Links: List<LinkModel>
│     ├── Name: string                    关联名称
│     └── ProcessList: List<string>       关联进程列表
├── General: GeneralModel
│     ├── IsStartatboot: bool             开机自启
│     ├── Theme: int (0=浅色, 1=深色)     主题模式
│     ├── ThemeColor: string              主题颜色 (#2b20d9)
│     ├── StartPage: int                  启动页
│     ├── IsWebEnabled: bool              网站浏览统计
│     ├── IndexPageFrequentUseNum: int    最为频繁显示条数
│     └── IndexPageMoreNum: int           更多显示条数
└── Behavior: BehaviorModel
      ├── IsSleepWatch: bool              睡眠监测
      ├── IgnoreProcessList: List<string> 忽略进程
      ├── IgnoreURLList: List<string>     忽略URL
      ├── IsWhiteList: bool               应用白名单
      └── ProcessWhiteList: List<string>  白名单列表
```

### 4.3 学习步骤

| 步骤 | 任务 | 目标文件 | 预期耗时 |
|------|------|----------|----------|
| 1 | 理解配置模型结构 | [ConfigModel.cs](file:///d:/dev/store/Tai/Core/Models/Config/ConfigModel.cs), [GeneralModel.cs](file:///d:/dev/store/Tai/Core/Models/Config/GeneralModel.cs), [BehaviorModel.cs](file:///d:/dev/store/Tai/Core/Models/Config/BehaviorModel.cs) | 15min |
| 2 | 理解配置读写逻辑 | [AppConfig.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/AppConfig.cs) | 30min |
| 3 | 理解配置变更的响应链 | [Main.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/Main.cs) AppConfig_ConfigChanged 方法 | 20min |
| 4 | 理解 SettingPage UI 配置编辑 | [SettingPageVM.cs](file:///d:/dev/store/Tai/UI/ViewModels/SettingPageVM.cs) | 30min |

---

## 五、链路四：睡眠监测

> 检测用户是否离开电脑，自动暂停/恢复统计。

### 5.1 数据流

```
[Sleepdiscover] [Core/Servicers/Instances/Sleepdiscover.cs]
  │
  ├── DispatcherTimer (5分钟间隔)
  │   └── 检测鼠标位置是否有变化 → 无变化则 Sleep
  │
  ├── Win32 键盘钩子 (LowLevelKeyboardProc)
  │   └── 检测最后按键时间 → 超时则 Sleep
  │
  ├── Win32 鼠标钩子 (LowLevelMouseProc)
  │   └── 检测鼠标活动 → 重置休眠倒计时
  │
  ├── SystemEvents.PowerModeChanged
  │   └── Suspend/Resume 事件 → Sleep/Wake
  │
  └── SystemEvents.SessionSwitch
      └── 远程桌面断开/连接 → Sleep/Wake
  │
  ▼
SleepStatusChanged(SleepStatus status)
  │
  ▼
Main.Sleepdiscover_SleepStatusChanged
  │
  ├── SleepStatus.Sleep (进入睡眠):
  │   ├── _webServer.SendMsg("sleep")        通知浏览器插件暂停
  │   ├── Stop()                              停止所有监控服务
  │   └── UpdateAppDuration()                 保存当前应用时长
  │
  └── SleepStatus.Wake (唤醒):
      ├── _webServer.SendMsg("wake")          通知浏览器插件恢复
      └── Start()                             恢复所有监控服务
```

### 5.2 学习步骤

| 步骤 | 任务 | 目标文件 | 预期耗时 |
|------|------|----------|----------|
| 1 | 理解睡眠检测机制 | [Sleepdiscover.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/Sleepdiscover.cs) | 30min |
| 2 | 理解睡眠状态响应的编排 | [Main.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/Main.cs) Sleepdiscover_SleepStatusChanged 方法 | 15min |

---

## 六、链路五：跨日/跨小时触发 (DateTimeObserver)

> 确保在整点前后正确触发数据切换。

### 6.1 数据流

```
[DateTimeObserver] [Core/Servicers/Instances/DateTimeObserver.cs]
  │
  ├── Start():
  │   └── 计算到下一个 HH:59:59 的毫秒差
  │   └── Timer.Interval = diffMilliseconds
  │   └── Timer.Start()
  │
  ├── Timer_Tick (到达59:59):
  │   ├── timer.Stop()
  │   ├── OnDateTimeChanging?.Invoke(this, nowTime)
  │   ├── Thread.Sleep(2000)                   等待2秒确保进入下一小时
  │   └── SetTimer()                           重新计算下一个整点时间差
  │
  └── Stop():
      └── timer.Stop() + timer.Dispose()
```

**设计意图**：`AppTimerServicer` 的 `Data.UpdateAppDuration` 内部按小时拆分数据，`DateTimeObserver` 确保在每小时59分59秒触发事件，提醒系统即将跨小时，保证时长正确分配到正确的时段。

### 6.2 学习步骤

| 步骤 | 任务 | 目标文件 | 预期耗时 |
|------|------|----------|----------|
| 1 | 理解时间观察者机制 | [DateTimeObserver.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/DateTimeObserver.cs) | 15min |

---

## 七、链路六：软件更新

> 独立 Updater 进程，通过 GitHub API 检查新版本。

### 7.1 数据流

```
SettingPageVM.OnCheckUpdate()
  │
  ├── 复制 Updater.exe + Newtonsoft.Json.dll 到 Update/ 目录
  ├── ProcessHelper.Run(updaterCacheExePath, ...)
  │
  ▼
[Updater.exe] 独立进程
  ├── GithubRelease.cs → GitHub API 获取最新 Release 信息
  ├── 版本比较：当前版本 vs 线上版本
  ├── 有新版本：
  │   ├── 下载 *.zip
  │   ├── Unzip.cs → 解压覆盖当前目录
  │   └── 提示用户重启
  └── 无新版本：
      └── 提示已是最新
```

### 7.2 学习步骤

| 步骤 | 任务 | 目标文件 | 预期耗时 |
|------|------|----------|----------|
| 1 | 理解 Updater 整体流程 | [Updater/MainWindow.xaml.cs](file:///d:/dev/store/Tai/Updater/MainWindow.xaml.cs) | 15min |
| 2 | 理解 GitHub Release 获取 | [Updater/GithubRelease.cs](file:///d:/dev/store/Tai/Updater/GithubRelease.cs) | 15min |
| 3 | 理解解压覆盖逻辑 | [Updater/Unzip.cs](file:///d:/dev/store/Tai/Updater/Unzip.cs) | 10min |
| 4 | 理解 UI 层升级入口 | [SettingPageVM.cs](file:///d:/dev/store/Tai/UI/ViewModels/SettingPageVM.cs) OnCheckUpdate 方法 | 10min |

---

## 八、链路七：数据导出

> 将统计数据导出为 Excel / CSV 文件。

### 8.1 数据流

```
SettingPageVM.OnExportData()
  │ 弹出日期范围选择
  ▼
IData.ExportToExcel(string dir, DateTime start, DateTime end)
  │ 查询时间范围内所有 DailyLog 数据
  │ 按 AppModel 聚合
  │
  ├── Npoi.Mapper → 生成 .xlsx
  └── CsvHelper → 生成 .csv
  │
  ▼
文件保存到用户指定目录
```

### 8.2 学习步骤

| 步骤 | 任务 | 目标文件 | 预期耗时 |
|------|------|----------|----------|
| 1 | 理解数据导出逻辑 | [Data.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/Data.cs) ExportToExcel 方法 | 20min |

---

## 九、链路八：数据清空

> 清空指定应用或时间范围的数据。

### 9.1 数据流

```
DetailPageVM / SettingPageVM
  │ 用户点击"清空"按钮
  ▼
IData.Clear(int appID, DateTime month)
  │ 删除指定 app 在某个月的所有 DailyLog + HoursLog
  │
IData.ClearRange(DateTime start, DateTime end)
  │ 删除整个时间范围的所有数据
```

### 9.2 学习步骤

| 步骤 | 任务 | 目标文件 | 预期耗时 |
|------|------|----------|----------|
| 1 | 理解数据清空逻辑 | [Data.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/Data.cs) Clear/ClearRange 方法 | 10min |

---

## 十、链路九：应用/网站分类管理

> 分类的增删改查 + 自动分类 + 分类展示。

### 10.1 数据流

```
应用分类 (Categorys):
  ├── Load(): 从数据库加载所有 CategoryModel
  ├── Create(CategoryModel): 新建分类
  ├── Update(CategoryModel): 更新分类
  ├── Delete(CategoryModel): 删除分类
  └── GetCategories(): 获取所有分类

网站分类 (WebData):
  ├── GetWebSiteCategories(): 获取所有网站分类
  ├── CreateWebSiteCategory(WebSiteCategoryModel)
  ├── UpdateWebSiteCategory(WebSiteCategoryModel)
  ├── DeleteWebSiteCategory(WebSiteCategoryModel)
  ├── GetWebSites(int categoryId): 按分类获取网站
  └── UpdateWebSitesCategory(int[] siteIds, int categoryId): 批量设置

自动分类 (Main.DispatchCategory):
  └── CategoryModel.DirectoryList 正则匹配 app 文件路径
      → 匹配成功自动设置 App.CategoryID
```

### 10.2 学习步骤

| 步骤 | 任务 | 目标文件 | 预期耗时 |
|------|------|----------|----------|
| 1 | 理解分类模型 | [CategoryModel.cs](file:///d:/dev/store/Tai/Core/Models/CategoryModel.cs), [WebSiteCategoryModel.cs](file:///d:/dev/store/Tai/Core/Models/Db/WebSiteCategoryModel.cs) | 10min |
| 2 | 理解应用分类管理 | [Categorys.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/Categorys.cs) | 20min |
| 3 | 理解网站分类管理 | [WebData.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/WebData.cs) Category 相关方法 | 15min |
| 4 | 理解分类 UI 管理页面 | [CategoryPageVM.cs](file:///d:/dev/store/Tai/UI/ViewModels/CategoryPageVM.cs) | 30min |

---

## 十一、链表十：崩溃处理 (TaiBug)

> 独立进程，捕获主程序崩溃并显示友好错误提示。

### 11.1 数据流

```
App.App_DispatcherUnhandledException
  │
  ├── Logger.Error("[程序崩溃异常] " + e.Exception)
  ├── Logger.Save(true)                             强制保存日志
  ├── ProcessHelper.Run("TaiBug.exe", ...)           启动崩溃弹窗
  └── Shutdown()                                    退出主程序
  │
  ▼
[TaiBug.exe] 独立 WPF 窗口
  │ 显示崩溃提示信息
  │ 提供日志查看入口
```

### 11.2 学习步骤

| 步骤 | 任务 | 目标文件 | 预期耗时 |
|------|------|----------|----------|
| 1 | 理解全局异常捕获 | [App.xaml.cs](file:///d:/dev/store/Tai/UI/App.xaml.cs) App_DispatcherUnhandledException | 10min |
| 2 | 理解 TaiBug 崩溃弹窗 | [TaiBug/App.xaml.cs](file:///d:/dev/store/Tai/TaiBug/App.xaml.cs), [TaiBug/MainWindow.xaml.cs](file:///d:/dev/store/Tai/TaiBug/MainWindow.xaml.cs) | 15min |

---

## 十二、数据库访问层

> SQLite + EF6 Code First 的读写分离连接池。

### 12.1 架构

```
IDatabase 接口
  ├── GetReaderContext(): TaiDbContext    获取只读连接
  ├── GetWriterContext(): TaiDbContext    获取写入连接
  └── CloseWriter(): void                 释放写入连接
  │
Database 实现 [Core/Servicers/Instances/Database.cs]
  ├── 读写分离：Reader 和 Writer 使用不同连接
  ├── Writer 串行化：同一时间只有一个 Writer
  └── SQLite BusyTimeout = 60秒

TaiDbContext [Core/Librarys/SQLite/TaiDbContext.cs]
  ├── DailyLog: DbSet<DailyLogModel>
  ├── HoursLog: DbSet<HoursLogModel>
  ├── App: DbSet<AppModel>
  ├── Categorys: DbSet<CategoryModel>
  ├── WebSites: DbSet<WebSiteModel>
  ├── WebSiteCategories: DbSet<WebSiteCategoryModel>
  ├── WebBrowserLogs: DbSet<WebBrowseLogModel>
  └── WebUrls: DbSet<WebUrlModel>
```

### 12.2 学习步骤

| 步骤 | 任务 | 目标文件 | 预期耗时 |
|------|------|----------|----------|
| 1 | 理解 DbContext 结构 | [TaiDbContext.cs](file:///d:/dev/store/Tai/Core/Librarys/SQLite/TaiDbContext.cs) | 15min |
| 2 | 理解读写分离连接池 | [Database.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/Database.cs) | 20min |
| 3 | 理解 SQLite 配置 | [SQLiteConfiguration.cs](file:///d:/dev/store/Tai/Core/Librarys/SQLite/SQLiteConfiguration.cs), [SQLiteBuilder.cs](file:///d:/dev/store/Tai/Core/Librarys/SQLite/SQLiteBuilder.cs) | 15min |

---

## 十三、UI 层数据展示链路

> 各页面的数据查询与展示流程。

### 13.1 概览页 (IndexPage)

```
IndexPageVM.LoadData()
  │
  ├── TabbarSelectedIndex == 0 (今日):
  │   ├── IData.GetTodaylogList()
  │   │   └── db.DailyLog.Where(m => m.Date == today)
  │   ├── IWebData.GetDateRangeWebSiteList(today, today)
  │   └── 转换为 ChartsDataModel → Charts 控件渲染
  │
  └── TabbarSelectedIndex == 1 (本周):
      ├── IData.GetThisWeeklogList()
      └── IWebData.GetDateRangeWebSiteList(weekStart, weekEnd)
```

### 13.2 统计页 (DataPage)

```
DataPageVM.LoadData(date, dataType)
  │
  ├── 按天 (TabbarSelectedIndex == 0):
  │   └── IData.GetDateRangelogList(date, date.AddDays(1))
  │       或 IData.GetCategoryHoursData(date)
  │
  ├── 按月 (TabbarSelectedIndex == 1):
  │   └── IData.GetCategoryRangeData(startOfMonth, endOfMonth)
  │
  └── 按年 (TabbarSelectedIndex == 2):
      └── IData.GetCategoryYearData(date)
```

### 13.3 应用详情页 (DetailPage)

```
DetailPageVM.LoadData()
  │
  ├── 按天: IData.GetAppDayData(appID, date)
  ├── 按周:
  │   ├── 本周: IData.GetAppRangeData(appID, weekStart, weekEnd)
  │   └── 上周: IData.GetAppRangeData(appID, lastWeekStart, lastWeekEnd)
  ├── 按月: IData.GetProcessMonthLogList(appID, month)
  └── 按年: IData.GetAppYearData(appID, date)
```

### 13.4 汇总图表页 (ChartPage)

```
ChartPageVM.LoadData()
  │
  ├── ChartDataMode.Id == 1 (默认视图):
  │   └── 复合图表：柱状图 + 饼图 + 列表
  ├── ChartDataMode.Id == 2 (汇总视图):
  │   └── 所有应用/网站按分类聚合
  └── ChartDataMode.Id == 3 (分类视图):
      └── 按分类展示雷达图/饼图
```

### 13.5 学习步骤

| 步骤 | 任务 | 目标文件 | 预期耗时 |
|------|------|----------|----------|
| 1 | 通读所有 ViewModel | `UI/ViewModels/*.cs` (8个文件) | 60min |
| 2 | 理解 Charts 控件体系 | `UI/Controls/Charts/` 下所有文件 | 45min |
| 3 | 理解页面导航机制 | [PageContainer.cs](file:///d:/dev/store/Tai/UI/Controls/PageContainer.cs), [MainViewModel](file:///d:/dev/store/Tai/UI/ViewModels/MainViewModel.cs) | 20min |

---

## 十四、完整学习路线总结

```
第1天：项目概览 + 启动链路
  ├── README.md                                 了解项目定位
  ├── docs/01_PROJECT_OVERVIEW.md               阅读架构概述
  └── docs/02_DATA_FLOW.md §零 (启动链路)       理解程序启动流程

第2天：核心链路（上）— 应用追踪
  ├── docs/02_DATA_FLOW.md §二 (链路一)         通读应用追踪链路
  ├── AppObserver.cs → AppManager.cs            阅读窗口监控代码
  ├── AppTimerServicer.cs                       阅读计时逻辑
  └── Main.cs (IsCheckApp + UpdateAppDuration) 阅读核心编排

第3天：核心链路（下）— 数据持久化 + 网站追踪
  ├── Data.cs (UpdateAppDuration)              阅读时段/日期拆分算法
  ├── docs/02_DATA_FLOW.md §三 (链路二)         通读网站追踪链路
  ├── WebServer.cs → WebFilter.cs              阅读 Web 接收端
  └── WebData.cs (AddUrlBrowseTime)            阅读网站数据写入

第4天：辅助链路 + 数据库
  ├── docs/02_DATA_FLOW.md §四～§九             通读剩余数据链路
  ├── AppConfig.cs                              理解配置管理
  ├── Sleepdiscover.cs                          理解睡眠检测
  ├── TaiDbContext.cs                           理解数据库模型
  └── Database.cs                               理解读写分离

第5天：UI 层 + 实践
  ├── App.xaml.cs (ConfigureServices)           理解 DI 注册
  ├── IndexPageVM.cs → DataPageVM.cs            阅读主要 ViewModel
  ├── UI/Controls/Charts/                       理解图表控件体系
  └── 实践：尝试添加一个新的过滤规则
            或修改统计页面展示逻辑
```

---

## 十五、关键工具类速查

| 工具类 | 文件 | 用途 |
|--------|------|------|
| `Win32API` | [Win32API.cs](file:///d:/dev/store/Tai/Core/Librarys/Win32API.cs) | Win32 API P/Invoke 封装（窗口/进程/图标） |
| `ProcessHelper` | [ProcessHelper.cs](file:///d:/dev/store/Tai/Core/Librarys/ProcessHelper.cs) | 进程启动与管理 |
| `Iconer` | [Iconer.cs](file:///d:/dev/store/Tai/Core/Librarys/Iconer.cs) | 从 EXE 文件提取图标 |
| `FileHelper` | [FileHelper.cs](file:///d:/dev/store/Tai/Core/Librarys/FileHelper.cs) | 文件路径/目录工具 |
| `Logger` | [Logger.cs](file:///d:/dev/store/Tai/Core/Librarys/Logger.cs) | 日志记录（文件 + Debug） |
| `RegexHelper` | [RegexHelper.cs](file:///d:/dev/store/Tai/Core/Librarys/RegexHelper.cs) | 正则匹配工具 |
| `Time` | [Time.cs](file:///d:/dev/store/Tai/Core/Librarys/Time.cs) | 时间格式化/计算工具 |
| `UrlHelper` | [UrlHelper.cs](file:///d:/dev/store/Tai/Core/Librarys/Browser/UrlHelper.cs) | URL 解析（域名提取/名称生成） |
| `SystemCommon` | [SystemCommon.cs](file:///d:/dev/store/Tai/Core/Librarys/SystemCommon.cs) | 开机自启/系统信息 |
| `FaviconDownloader` | [FaviconDownloader.cs](file:///d:/dev/store/Tai/Core/Librarys/Browser/Favicon/FaviconDownloader.cs) | 网站 Favicon 下载 |