# 06 – 浏览器扩展协议

> 📁 目标读者：需要修改浏览器扩展、对接 WebSocket 服务或新增浏览器支持的开发者
> 🎯 目标：完整描述 Tai 客户端与浏览器扩展之间的通信协议、数据格式与交互时序
> 关联文档：服务端数据入库链路见 [02_DATA_FLOW.md](02_DATA_FLOW.md)，URL 过滤配置见 [01_PROJECT_OVERVIEW.md](01_PROJECT_OVERVIEW.md)

---

## 1. 架构总览

Tai 通过「**本地 WebSocket 服务 + 浏览器扩展**」统计网页浏览时长，与桌面应用统计（Win32 窗口监控，见 [05_WIN32_API_REFERENCE.md](05_WIN32_API_REFERENCE.md)）是两条独立链路。

```
┌─────────────────────────┐          ws://127.0.0.1:8908/TaiWebSentry         ┌──────────────────────────┐
│  Chrome 扩展 (MV3)       │  ◄────────────────────────────────────────────►  │  Tai 客户端               │
│  WebExtensions/Chrome/   │                                                 │  WebServer (WebSocketSharp)│
│                          │   C→S: "ping" / {Url,Title,Icon,Duration,...}   │                            │
│  service-worker.js       │   S→C: "sleep" / "wake"                         │  → WebFilter → WebData     │
└─────────────────────────┘                                                 └──────────────────────────┘
```

| 端 | 代码位置 | 技术 |
|----|---------|------|
| 服务端 | [Core/Servicers/Instances/WebServer.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/WebServer.cs) | C# + [WebSocketSharp](file:///d:/dev/store/Tai/packages/) 库 |
| 客户端 | [WebExtensions/Chrome/service-worker.js](file:///d:/dev/store/Tai/WebExtensions/Chrome/service-worker.js) | Chrome Extension Manifest V3 |

**关键参数：**

| 项 | 值 | 说明 |
|----|-----|------|
| 端口 | `8908` | 硬编码，两端不可配 |
| 路径 | `/TaiWebSentry` | 通过 `AddWebSocketService<WebServer>("/TaiWebSentry")` 注册 |
| 协议 | `ws://`（非加密） | `new WebSocketServer(8908, false)`，第二个参数 false = 不用 TLS |
| 序列化 | JSON（Newtonsoft.Json） | 仅统计数据消息 |
| 浏览器支持 | 仅 Chrome（MV3，>= 116） | [manifest.json](file:///d:/dev/store/Tai/WebExtensions/Chrome/manifest.json) |

---

## 2. 消息协议

协议共 4 种消息，**全部是纯文本帧**（WebSocket Text Frame），无二进制、无消息 ID、无应答机制。

### 2.1 扩展 → Tai

#### (1) 心跳 `ping`

```
ping
```

- 每 **10 秒**发送一次（`keepAlive()`，[service-worker.js#L105](file:///d:/dev/store/Tai/WebExtensions/Chrome/service-worker.js#L105)）
- 真实目的不是检测连接，而是**防止 MV3 Service Worker 30 秒无活动被 Chrome 休眠**（代码注释有说明）
- 服务端收到后尝试按 `NotifyWeb` 反序列化会失败，被静默 `catch` 吞掉（[WebServer.cs#L62](file:///d:/dev/store/Tai/Core/Servicers/Instances/WebServer.cs#L62)）——属于约定内的无害行为

#### (2) 浏览时长上报（核心消息）

```json
{
  "Url": "https://github.com/trending",
  "Title": "Trending repositories on GitHub today",
  "Icon": "https://github.githubassets.com/favicons/favicon.svg",
  "Duration": 125,
  "ActiveTime": 1760000000
}
```

| 字段 | 类型 | 单位 | 含义 |
|------|------|------|------|
| `Url` | string | — | 完整 URL（含协议和路径） |
| `Title` | string | — | 页面标题（`tab.title`） |
| `Icon` | string | — | favicon URL（`tab.favIconUrl`），可能为空 |
| `Duration` | int | **秒** | 本次停留时长 |
| `ActiveTime` | int | **Unix 秒时间戳** | 开始浏览的时刻（`new Date().getTime() / 1000` 取整） |

服务端字段映射：[NotifyWeb.cs](file:///d:/dev/store/Tai/Core/Models/WebPage/NotifyWeb.cs)，其中 `ActiveDateTime` 属性把 `ActiveTime` 按 **UTC 1970 纪元 + 秒数 → ToLocalTime()** 转成本地时间。

> ⚠️ **发送时机**（扩展端 `calDuration()`）：URL 切换、浏览器窗口失焦、收到 `sleep` 消息时。**不是实时流式上报**，而是离开页面时一次性结算。

### 2.2 Tai → 扩展

纯文本广播（`WebSocketServices.Broadcast`，发给所有已连接客户端）：

| 消息 | 触发时机 | 扩展端动作 |
|------|---------|-----------|
| `sleep` | Tai 判定用户离开/睡眠（Sleepdiscover，见 [05_WIN32_API_REFERENCE.md](05_WIN32_API_REFERENCE.md) §5） | 置 `isSleep = true`，立即 `calDuration()` 结算当前页面，此后忽略所有标签页事件 |
| `wake` | 用户恢复活跃 | 置 `isSleep = false`，恢复统计 |

发送位置：[Main.cs#L254](file:///d:/dev/store/Tai/Core/Servicers/Instances/Main.cs#L254)、[Main.cs#L266](file:///d:/dev/store/Tai/Core/Servicers/Instances/Main.cs#L266)。

---

## 3. 扩展端工作机制

### 3.1 状态机

```
        connect()                    onclose
   ┌──────────────┐  onopen   ┌──────────────┐
   │  未连接       │ ────────► │  已连接       │
   │ (inactive 图标)│ ◄──────── │ (active 图标) │
   └──────────────┘           └──────┬───────┘
        ▲                            │ 收到 sleep / 重连失败≥5次
        │                     ┌──────▼───────┐
        │                     │  睡眠中       │  忽略标签页事件
        │                     └──────┬───────┘
        │                            │ 收到 wake
        └────────────────────────────┘
```

### 3.2 核心定时器

| 定时器 | 间隔 | 作用 | 代码 |
|--------|------|------|------|
| `keepAlive` | 10s | 发 `ping`，保活 Service Worker | [L105](file:///d:/dev/store/Tai/WebExtensions/Chrome/service-worker.js#L105) |
| `startWatchFocus` | 1s | 轮询 `chrome.windows.getCurrent().focused`，检测浏览器是否在前台 | [L284](file:///d:/dev/store/Tai/WebExtensions/Chrome/service-worker.js#L284) |
| `startRenofity` | 10s | 重发失败队列中最早的一条 | [L314](file:///d:/dev/store/Tai/WebExtensions/Chrome/service-worker.js#L314) |
| 自动重连 | 10s | 断线后循环重连，失败 5 次进入睡眠态 | [L84](file:///d:/dev/store/Tai/WebExtensions/Chrome/service-worker.js#L84) |

### 3.3 统计时序（一次浏览的完整生命周期）

```
用户打开/切换标签页
  │  chrome.tabs.onActivated / onUpdated(complete)
  ▼
setActive(tab)  ──►  activePage = { url, title, icon, startTime: now }
  │
  │ （用户浏览中，每秒检查浏览器窗口焦点）
  │
  ▼ 触发结算（三选一）
  ① 切到另一个 URL        → onActivePage 发现 url 变了
  ② 浏览器窗口失焦         → startWatchFocus 检测到 focused = false
  ③ 收到 Tai 的 sleep 消息
  │
  ▼
calDuration()
  │  duration = (now - startTime) / 1000   // 秒
  │  发送 { Url, Title, Icon, Duration, ActiveTime }
  ▼
activePage = null  ──►  等待下一次 setActive
```

### 3.4 断线容错

发送失败（未连接）时数据进 `notifyFailList` 数组，每 10 秒由 `renotify()` 取出**最早一条**重发：

- 队列**只存在内存**，浏览器重启后丢失
- 重发的是**原始记录**（ActiveTime 仍是当时的时间戳），服务端按历史时间入库，不影响统计准确性
- 源码中有一段被注释掉的「同 URL 合并 Duration」逻辑（[L220-224](file:///d:/dev/store/Tai/WebExtensions/Chrome/service-worker.js#L220)），当前不做合并，同 URL 多条独立入库

---

## 4. 服务端处理链路

```
WebServer.OnMessage(e.Data)
  │  JsonConvert.DeserializeObject<NotifyWeb>   // 失败（如 "ping"）静默丢弃
  ▼
WebSocketEvent.Invoke(args)  ──静态事件──►  Main.WebSocketEvent_OnWebLog
  │                                             │
  │                                             ├─ ① WebFilter.IsIgnore(url) ─ 命中则丢弃
  │                                             ├─ ② WebData.AddUrlBrowseTime(site, duration, activeDateTime) ─ 入库
  │                                             └─ ③ Task.Run: FaviconDownloader.DownloadAsync(icon) ─ 下载图标
  │                                                  → WebData.UpdateUrlFavicon(site, path)
  ▼
（反序列化异常吞掉，不回复任何消息）
```

关键代码：[Main.cs#L485-516](file:///d:/dev/store/Tai/Core/Servicers/Instances/Main.cs#L485)。

### 4.1 URL 过滤规则（WebFilter）

[WebFilter.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/WebFilter.cs) 三级判定，命中即忽略：

| 层级 | 规则 | 来源 |
|------|------|------|
| 1 | 空 URL | — |
| 2 | 域名精确匹配（如 `bilibili.com`） | 用户配置 `Behavior.IgnoreURLList` |
| 3 | 正则匹配（含 `*$!(){}\[^|]` 即视为正则） | 用户配置 + **内置默认**：`^chrome://`、`^edge://`、`^view-source:` |

带缓存：命中过的 URL 进 `_ignoreURLCache`，配置变更时清空。

### 4.2 favicon 处理

[FaviconDownloader.cs](file:///d:/dev/store/Tai/Core/Librarys/Browser/Favicon/FaviconDownloader.cs)：

- 保存路径：`{程序根目录}/WebFavicons/{站点名}{yyyyMM}.ico`（站点名由 [UrlHelper.GetName](file:///d:/dev/store/Tai/Core/Librarys/Browser/UrlHelper.cs#L53) 生成，内置 google/github/bilibili 等别名表）
- **按月分文件**（`yyyyMM` 后缀），同站点同月只下载一次（`File.Exists` 直接返回）
- 不支持 SVG：把 `.svg` 替换成 `.png` 再下载
- 下载失败仅记日志，不影响时长统计入库

---

## 5. 安装与调试

### 5.1 安装扩展

1. 启动 Tai 客户端（确认 8908 端口监听：`netstat -ano | findstr 8908`）
2. Chrome 打开 `chrome://extensions`，开启「开发者模式」
3. 「加载已解压的扩展程序」→ 选择 [WebExtensions/Chrome/](file:///d:/dev/store/Tai/WebExtensions/Chrome/) 目录
4. 工具栏图标变为 **socket-active**（绿色）表示已连接；**socket-inactive**（灰色）表示未连接

### 5.2 调试

| 端 | 方法 |
|----|------|
| 扩展 | `chrome://extensions` → Tai Sentry → 「Service Worker」→ 看 console（`已连接Tai!` / `notify` / `ping` 日志） |
| 服务端 | Visual Studio 输出窗口，或日志文件（`Logger`） |
| 抓包 | WS 是明文本地流量，可用 Wireshark loopback 或 Chrome DevTools → Network → WS |

### 5.3 快速验证协议（无需扩展）

用任意 WebSocket 客户端连 `ws://127.0.0.1:8908/TaiWebSentry` 发送：

```json
{"Url":"https://www.baidu.com/s?wd=test","Title":"test_百度搜索","Icon":"","Duration":60,"ActiveTime":1760000000}
```

然后看 Tai 统计页是否出现 baidu.com 的记录。

---

## 6. 协议局限与扩展建议

| 局限 | 现状 | 改造建议 |
|------|------|---------|
| **无认证** | 任何本地进程都能连 8908 上报伪造数据 | 首次连接握手时下发 token，后续消息携带 |
| **无应答** | 扩展发完不知道服务端是否收到，只能靠失败队列盲重发 | 服务端回 `ack`，扩展收不到再入队 |
| **非加密** | 明文 ws://，本地环路风险低但可被抓包 | 内网场景可接受；如需加固换 wss:// |
| **仅 Chrome** | manifest V3 + `chrome.tabs` API | Edge 可直接加载同目录；Firefox 需处理 MV3 SW 兼容 |
| **端口硬编码** | 两端都写死 8908，冲突只能改代码重编 | 服务端写注册表/配置文件，扩展启动时读 native messaging 或固定约定 |
| **`ping` 被当垃圾消息吞掉** | 依赖反序列化失败 + 空 catch，语义不清晰 | 服务端显式判断 `e.Data == "ping"` 再跳过 |
| **失败队列在内存** | 浏览器崩溃/重启后未上报数据丢失 | 持久化到 `chrome.storage.local` |

---

## 7. 相关文件清单

| 文件 | 职责 |
|------|------|
| [WebExtensions/Chrome/manifest.json](file:///d:/dev/store/Tai/WebExtensions/Chrome/manifest.json) | 扩展清单（MV3，权限仅 `tabs`） |
| [WebExtensions/Chrome/service-worker.js](file:///d:/dev/store/Tai/WebExtensions/Chrome/service-worker.js) | 扩展全部逻辑（连接/统计/上报） |
| [Core/Servicers/Instances/WebServer.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/WebServer.cs) | WS 服务端，消息入口 |
| [Core/Librarys/Browser/WebSocketEvent.cs](file:///d:/dev/store/Tai/Core/Librarys/Browser/WebSocketEvent.cs) | 静态事件转发（解耦 WebServer 与 Main） |
| [Core/Models/WebPage/NotifyWeb.cs](file:///d:/dev/store/Tai/Core/Models/WebPage/NotifyWeb.cs) | 上报数据模型 |
| [Core/Models/WebPage/Site.cs](file:///d:/dev/store/Tai/Core/Models/WebPage/Site.cs) | 入库站点模型 |
| [Core/Servicers/Instances/WebFilter.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/WebFilter.cs) | URL 忽略过滤 |
| [Core/Librarys/Browser/Favicon/FaviconDownloader.cs](file:///d:/dev/store/Tai/Core/Librarys/Browser/Favicon/FaviconDownloader.cs) | favicon 下载 |
| [Core/Librarys/Browser/UrlHelper.cs](file:///d:/dev/store/Tai/Core/Librarys/Browser/UrlHelper.cs) | 域名提取、站点命名 |
| [Core/Servicers/Instances/Main.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/Main.cs) | 消息消费、sleep/wake 广播 |
