# 05 – Win32 API 参考手册

> 📁 目标读者：需要理解或修改 Tai 系统底层交互逻辑的开发者
> 🎯 目标：说清楚 Tai 用了哪些 Win32 API、为什么用、在哪里用、有哪些坑
> 📄 本文档遵循 [PlcDataStudio 文档规范]组织
> 关联文档：数据链路见 [02_DATA_FLOW.md](02_DATA_FLOW.md)，UI 控件见 [04_UI_CONTROLS_GUIDE.md](04_UI_CONTROLS_GUIDE.md)

---

## 1. 为什么 Tai 需要 Win32 API

Tai 的核心能力是「统计用户正在使用哪个应用」，而 WPF/.NET Framework 托管 API 无法完成以下事情：

| 需求 | 托管 API 的局限 | Tai 的 Win32 方案 |
|------|----------------|------------------|
| 感知**前台窗口切换** | .NET 无全局焦点事件 | `SetWinEventHook` 订阅系统事件 |
| 由窗口句柄反查**进程路径** | `Process` 类无法从 hwnd 反查 | `GetWindowThreadProcessId` + `OpenProcess` + `QueryFullProcessImageName` |
| 识别 **UWP 应用**真实进程 | UWP 宿主进程是 `ApplicationFrameHost` | `EnumChildWindows` 找子窗口真实 PID |
| 全局**键盘/鼠标监听**（睡眠检测、快捷键、弹层关闭） | WPF 只能收到本窗口输入 | `SetWindowsHookEx` 低级钩子 |
| 判断系统**是否在播放声音** | .NET 无音频会话 API | COM 接口 `IAudioMeterInformation` |
| 鼠标**穿透窗口** | WPF `IsHitTestVisible` 仍会遮挡 | `SetWindowLong` + `WS_EX_TRANSPARENT` |

**所有 P/Invoke 声明集中在 4 个文件：**

| 文件 | 职责 |
|------|------|
| [Core/Librarys/Win32API.cs](file:///d:/dev/store/Tai/Core/Librarys/Win32API.cs) | 集中封装的公共 API（主要入口） |
| [Core/Servicers/Instances/AppObserver.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/AppObserver.cs) | 前台窗口切换事件钩子 |
| [Core/Servicers/Instances/AppManager.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/AppManager.cs) | 进程信息查询、UWP 识别 |
| [UI/Controls/Window/DefaultWindow.cs](file:///d:/dev/store/Tai/UI/Controls/Window/DefaultWindow.cs) | GDI 对象释放、窗口穿透 |

---

## 2. 前台窗口监控（核心链路）

这是 Tai 统计功能的起点，见 [02_DATA_FLOW.md](02_DATA_FLOW.md) 的「窗口监控链路」。

### 2.1 涉及 API

| API | DLL | 作用 | 声明位置 |
|-----|-----|------|---------|
| `SetWinEventHook` | user32 | 订阅系统级窗口事件 | [Win32API.cs#L20](file:///d:/dev/store/Tai/Core/Librarys/Win32API.cs#L20)、[AppObserver.cs#L25](file:///d:/dev/store/Tai/Core/Servicers/Instances/AppObserver.cs#L25) |
| `UnhookWinEvent` | user32 | 取消订阅 | 同上附近 |
| `GetForegroundWindow` | user32 | 获取当前前台窗口句柄 | [Win32API.cs#L14](file:///d:/dev/store/Tai/Core/Librarys/Win32API.cs#L14) |
| `SetForegroundWindow` | user32 | 置前台（当前代码未调用，仅声明） | [Win32API.cs#L16](file:///d:/dev/store/Tai/Core/Librarys/Win32API.cs#L16) |

### 2.2 工作机制

```
AppObserver.Start()
  │
  └── SetWinEventHook(0x0003, 0x0003, ...)   // 0x0003 = EVENT_SYSTEM_FOREGROUND
        │
        ▼ （用户切换窗口时，系统回调）
  ForegroundEventCallback(hwnd)
        │
        ├── AppManager.GetAppInfo(hwnd)    // hwnd → 进程信息（见 §3）
        ├── WindowManager.GetWindowInfo(hwnd)  // hwnd → 标题/尺寸/类名（见 §6）
        └── 触发 OnAppActiveChanged 事件 → 进入统计链路
```

### 2.3 注意点

- **委托必须存为字段**（[AppObserver.cs#L34](file:///d:/dev/store/Tai/Core/Servicers/Instances/AppObserver.cs#L34) 的 `_foregroundEventDelegate`），否则 GC 回收委托后系统回调会崩溃。这是 P/Invoke 回调的经典坑。
- **延迟复检机制**：当切到系统组件（如开始菜单）时，启动 1 秒定时器 `DelayDetect`，因为系统组件弹出后焦点可能很快还给真实应用，避免记录一次无意义的「系统组件使用」。
- `SetWinEventHook` 的 `eventMin = eventMax = 0x0003` 表示只订阅前台切换这一种事件，减少回调开销。
- 重复声明：`SetWinEventHook` 在 `Win32API.cs` 和 `AppObserver.cs` 中各声明了一份（签名相同），属于历史遗留，新代码应复用 `Win32API` 中的声明。

---

## 3. 进程信息获取（hwnd → 进程路径）

### 3.1 涉及 API

| API | DLL | 作用 | 声明位置 |
|-----|-----|------|---------|
| `GetWindowThreadProcessId` | user32 | 由窗口句柄取 PID | [Win32API.cs#L28](file:///d:/dev/store/Tai/Core/Librarys/Win32API.cs#L28)、[AppManager.cs#L22](file:///d:/dev/store/Tai/Core/Servicers/Instances/AppManager.cs#L22) |
| `OpenProcess` | kernel32 | 打开进程句柄 | [AppManager.cs#L24](file:///d:/dev/store/Tai/Core/Servicers/Instances/AppManager.cs#L24) |
| `QueryFullProcessImageName` | kernel32 | 取进程 exe 完整路径 | [AppManager.cs#L26](file:///d:/dev/store/Tai/Core/Servicers/Instances/AppManager.cs#L26) |
| `GetModuleBaseName` / `GetModuleFileNameExA` | psapi | 取模块名（备选方案） | [AppManager.cs#L31](file:///d:/dev/store/Tai/Core/Servicers/Instances/AppManager.cs#L31)、[Win32API.cs#L31](file:///d:/dev/store/Tai/Core/Librarys/Win32API.cs#L31) |
| `CloseHandle` | kernel32 | 关闭句柄 | [AppManager.cs#L37](file:///d:/dev/store/Tai/Core/Servicers/Instances/AppManager.cs#L37) |
| `GetGUIThreadInfo` | user32 | 取线程焦点窗口（用于全屏 UWP） | [AppManager.cs#L64](file:///d:/dev/store/Tai/Core/Servicers/Instances/AppManager.cs#L64) |

### 3.2 标准调用链

```csharp
// 1. hwnd → PID
GetWindowThreadProcessId(hwnd, out int processId);

// 2. PID → 进程句柄（0x001F0FFF = PROCESS_ALL_ACCESS，权限拉满）
IntPtr processHandle = OpenProcess(0x001F0FFF, false, processId);

// 3. 进程句柄 → exe 路径
var buffer = new StringBuilder(4096);   // 缓冲区给足，长路径不截断
QueryFullProcessImageName(processHandle, 0, buffer, ref capacity);
```

完整实现见 [AppManager.GetAppExecutablePath](file:///d:/dev/store/Tai/Core/Servicers/Instances/AppManager.cs#L179)。

### 3.3 权限常量

| 常量 | 值 | 用途 |
|------|-----|------|
| `PROCESS_QUERY_INFORMATION` | `0x0400` | 查询进程信息（最小权限） |
| `PROCESS_VM_READ` | `0x0010` | 读进程内存（UWP 识别时用） |
| `0x001F0FFF` | `PROCESS_ALL_ACCESS` | Tai 实际用的（取路径场景） |

> ⚠️ **坑**：对受保护进程（如以管理员身份运行而 Tai 不是）`OpenProcess` 会返回 `IntPtr.Zero`，此时 `executablePath` 为空字符串，`GetAppInfo` 仍能返回进程名但无路径和描述。这是预期降级，不是 bug。

---

## 4. UWP 应用识别（最绕的一段逻辑）

### 4.1 问题

UWP 应用的前台窗口宿主进程是 `ApplicationFrameHost.exe`，直接按 §3 查到的是宿主而不是真实应用（如「设置」「计算器」）。

### 4.2 解法：枚举子窗口找真实 PID

核心思路：UWP 窗口的子窗口属于**另一个进程**（真实应用进程），枚举子窗口并比较 PID 即可。

```csharp
// WINDOWINFO 携带 ownerpid（宿主 PID），遍历子窗口时回填 childpid
EnumChildWindows(hWnd, EnumChildWindowsCallback, pWindowinfo);
// 回调里：若子窗口 PID != ownerpid → 找到真实进程
```

涉及 API：`EnumChildWindows`（[Win32API.cs#L54](file:///d:/dev/store/Tai/Core/Librarys/Win32API.cs#L54)）、`GetWindowThreadProcessId`、`OpenProcess`、`QueryFullProcessImageName`。

### 4.3 三级降级策略

[AppManager.GetAppInfo](file:///d:/dev/store/Tai/Core/Servicers/Instances/AppManager.cs#L78) 中对 `ApplicationFrameHost` 的处理：

1. **直接取**：`GetUWPAPPExecutablePath` 枚举子窗口拿真实路径
2. **轮询等待**：刚启动的 UWP 子窗口尚未就绪，每 100ms 重试，最长 5 秒（`_outTime`）
3. **全屏兜底**：仍失败（可能是全屏 UWP）→ `GetGUIThreadInfo(0)` 取当前 GUI 线程焦点窗口的 PID（`GetFullScreenUWPPID`，[AppManager.cs#L267](file:///d:/dev/store/Tai/Core/Servicers/Instances/AppManager.cs#L267)）

> ⚠️ **坑**：`Marshal.AllocHGlobal` 分配的内存在所有 `return null` 分支前必须 `FreeHGlobal`，当前实现有泄漏隐患（子进程未找到的分支未释放）。修改这段代码时留意。

### 4.4 系统组件判定

`IsSystemComponent`（[AppManager.cs#L277](file:///d:/dev/store/Tai/Core/Servicers/Instances/AppManager.cs#L277)）用两份名单排除系统组件：

- **窗口类名**：`Progman`（桌面）、`WorkerW`、`Shell_TrayWnd`（任务栏）等
- **进程名**：`ShellExperienceHost`、`StartMenuExperienceHost`、`SearchHost`、`LockApp`

特殊处理：`explorer` 进程既可能是文件管理器也可能是桌面/任务栏，需要先用 `GetClassName` 判定类名，类名为空时退化为「标题长度 ≤ 0 即系统组件」。

---

## 5. 全局输入钩子（键盘 + 鼠标）

### 5.1 涉及 API

| API | 作用 |
|-----|------|
| `SetWindowsHookEx` | 安装低级钩子（`WH_KEYBOARD_LL = 13` / `WH_MOUSE_LL = 14`） |
| `UnhookWindowsHookEx` | 卸载钩子 |
| `CallNextHookEx` | 把事件传递给下一个钩子（**必须调用**，否则可能阻塞系统输入链） |
| `GetModuleHandle` | 取当前模块句柄（钩子安装需要） |

封装方法：`Win32API.SetKeyboardHook(proc)` / `Win32API.SetMouseHook(proc)`（[Win32API.cs#L247](file:///d:/dev/store/Tai/Core/Librarys/Win32API.cs#L247)）。

### 5.2 三个使用场景

| 调用方 | 钩子类型 | 用途 |
|--------|---------|------|
| [Sleepdiscover.cs](file:///d:/dev/store/Tai/Core/Servicers/Instances/Sleepdiscover.cs) | 键盘 + 鼠标 | 睡眠/离开检测：有输入则判定用户活跃 |
| [InputServicer.cs](file:///d:/dev/store/Tai/UI/Servicers/InputServicer.cs) | 键盘 | 全局按键事件转发（`OnKeyDownInput` / `OnKeyUpInput`） |
| [ColorSelect.cs](file:///d:/dev/store/Tai/UI/Controls/Base/ColorSelect.cs)、[IconSelect.cs](file:///d:/dev/store/Tai/UI/Controls/Base/IconSelect.cs)、[DateSelect.cs](file:///d:/dev/store/Tai/UI/Controls/Select/DateSelect.cs) | 鼠标 | 弹层打开时监听全局点击，点在控件外则关闭弹层 |

### 5.3 弹层关闭的典型模式

```csharp
// 弹层打开时挂钩子，关闭时卸载
if (IsOpen) hook = Win32API.SetMouseHook(mouseProc);
else        Win32API.UnhookWindowsHookEx(hook);

// 回调里：双击/滚轮时检查鼠标是否在控件内，不在就关闭
if (wParam == (IntPtr)Win32API.WM_LBUTTONDBLCLK || wParam == (IntPtr)Win32API.WM_WHEEL)
{
    if (!IsInControl()) IsOpen = false;
}
return Win32API.CallNextHookEx(hookMouseID, nCode, wParam, lParam);  // 必须传递
```

### 5.4 关键常量（[Win32API.cs#L234](file:///d:/dev/store/Tai/Core/Librarys/Win32API.cs#L234)）

| 常量 | 值 | 含义 |
|------|-----|------|
| `WH_KEYBOARD_LL` | 13 | 低级键盘钩子 |
| `WH_MOUSE_LL` | 14 | 低级鼠标钩子 |
| `WM_KEYDOWN` / `WM_KEYUP` | 0x0100 / 0x0101 | 普通键按下/抬起 |
| `WM_SYSKEYDOWN` / `WM_SYSKEYUP` | 0x0104 / 0x0105 | 系统键（Alt 组合） |
| `WM_LBUTTONDBLCLK` | 0x0202 | 左键双击 |
| `WM_WHEEL` | 0x020A | 滚轮 |

> ⚠️ **坑**：
> 1. 低级钩子回调在**系统级超时**（默认 300ms，注册表 `LowLevelHooksTimeout`）内必须返回，耗时操作要异步化，否则系统会静默摘除钩子。
> 2. 回调委托同样要防止 GC 回收（各调用方都存为实例字段）。
> 3. `Sleepdiscover.Start()` 中 `SetKeyboardHook` 的返回值未保存，意味着键盘钩子永远不会被卸载——进程生命周期内可接受，但不要模仿到短生命周期场景。

---

## 6. 窗口信息获取

[WindowManager](file:///d:/dev/store/Tai/Core/Servicers/Instances/WindowManager.cs) 把以下 API 组合成 `WindowInfo` 模型（标题、类名、尺寸、位置）：

| API | 作用 | 封装 |
|-----|------|------|
| `GetWindowRect` | 窗口矩形（含 `RECT` 结构体，带 Width/Height 计算属性） | [Win32API.cs#L294](file:///d:/dev/store/Tai/Core/Librarys/Win32API.cs#L294)，安全包装在 `#L322` |
| `GetClassName` | 窗口类名 | 安全包装 `GetWindowClassName`，[Win32API.cs#L347](file:///d:/dev/store/Tai/Core/Librarys/Win32API.cs#L347) |
| `GetWindowTextLength` + `GetWindowText` | 窗口标题 | [WindowManager.cs#L41](file:///d:/dev/store/Tai/Core/Servicers/Instances/WindowManager.cs#L41) |

> ⚠️ **坑**：`GetWindowTitle` 中缓冲区用了 `长度 × 2`，因为 `GetWindowTextLength` 返回的**不含**终止符且对 Unicode 标题可能低估，乘 2 是经验性防御写法。

另外 `GetCursorPos`（封装为 `GetCursorPosition`，[Win32API.cs#L137](file:///d:/dev/store/Tai/Core/Librarys/Win32API.cs#L137)）被 `Sleepdiscover` 用于比对鼠标位置是否移动——静止不动且无输入即判定离开。

---

## 7. 系统声音检测（COM 互操作）

`IsWindowsPlayingSound`（[Win32API.cs#L149](file:///d:/dev/store/Tai/Core/Librarys/Win32API.cs#L149)）不用 P/Invoke，而是**手工声明 COM 接口**访问 Windows Core Audio API：

```
MMDeviceEnumerator (ComImport, coclass)
  → IMMDeviceEnumerator.GetDefaultAudioEndpoint(eRender, eMultimedia)   // 默认扬声器
    → IMMDevice.Activate(IAudioMeterInformation.GUID)
      → IAudioMeterInformation.GetPeakValue()   // 峰值电平 > 1E-08 即在放声音
```

用途：`Sleepdiscover` 判定睡眠时，播放声音期间不计为离开（用户在看电影/听歌）。

> ⚠️ **坑**：
> - 接口用 `InterfaceIsIUnknown` + 精确 GUID 声明，方法表只定义了用到的成员（其余以注释「the rest is not defined/needed」占位）——**顺序和签名必须与 vtable 一致**，新增成员时不能插错位置。
> - 峰值电平在暂停播放后不是精确 0 而是约 1E-09 的微小值，所以阈值取 `1E-08` 而不是 `> 0`（代码注释里有说明）。

---

## 8. UI 层辅助 API

位于 [DefaultWindow.cs](file:///d:/dev/store/Tai/UI/Controls/Window/DefaultWindow.cs)：

### 8.1 `DeleteObject`（gdi32）— 图标转换防泄漏

`ToImageSource`（[#L288](file:///d:/dev/store/Tai/UI/Controls/Window/DefaultWindow.cs#L288)）把 `System.Drawing.Icon` 转成 WPF `ImageSource`：

```csharp
IntPtr hBitmap = bitmap.GetHbitmap();   // 申请 GDI 句柄
// ... CreateBitmapSourceFromHBitmap ...
DeleteObject(hBitmap);                  // 必须释放，否则 GDI 句柄泄漏
```

> ⚠️ `GetHbitmap` 拿到的句柄 .NET 不会自动释放，GDI 句柄上限 10000/进程，泄漏会导致 UI 花屏或崩溃。

### 8.2 `SetWindowLong` / `GetWindowLong` — 鼠标穿透窗口

```csharp
// OnSourceInitialized 中，IsThruWindow = true 时：
uint extendedStyle = GetWindowLong(hwnd, GWL_EXSTYLE);        // GWL_EXSTYLE = -20
SetWindowLong(hwnd, GWL_EXSTYLE, extendedStyle | WS_EX_TRANSPARENT);  // 0x20
```

`WS_EX_TRANSPARENT` 让窗口对鼠标事件「透明」——点击穿透到下层窗口，用于覆盖层类窗口（见 [04_UI_CONTROLS_GUIDE.md](04_UI_CONTROLS_GUIDE.md) 的 DefaultWindow 章节）。

---

## 9. 调用关系速查矩阵

| API | AppObserver | AppManager | WindowManager | Sleepdiscover | InputServicer | 弹层控件 | DefaultWindow |
|-----|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| SetWinEventHook | ✅ | | | | | | |
| GetForegroundWindow | ✅ | | | | | | |
| GetWindowThreadProcessId | | ✅ | | | | | |
| OpenProcess / QueryFullProcessImageName | | ✅ | | | | | |
| EnumChildWindows | | ✅ (UWP) | | | | | |
| GetGUIThreadInfo | | ✅ (全屏UWP) | | | | | |
| GetClassName | | ✅ (系统组件判定) | ✅ | | | | |
| GetWindowRect / GetWindowText | | | ✅ | | | | |
| GetCursorPos | | | | ✅ | | | |
| 键盘钩子 (SetWindowsHookEx) | | | | ✅ | ✅ | | |
| 鼠标钩子 (SetWindowsHookEx) | | | | ✅ | | ✅ | |
| COM 音频检测 | | | | ✅ | | | |
| DeleteObject | | | | | | | ✅ |
| SetWindowLong (穿透) | | | | | | | ✅ |

---

## 10. 通用注意事项（改这段代码前必读）

1. **委托防 GC**：所有传给 Win32 的回调委托必须存为字段（`WinEventDelegate`、`LowLevelKeyboardProc`），局部变量会被 GC 回收导致 `ExecutionEngineException`。
2. **句柄配对**：`OpenProcess` → `CloseHandle`；`SetWindowsHookEx` → `UnhookWindowsHookEx`；`SetWinEventHook` → `UnhookWinEvent`；`AllocHGlobal` → `FreeHGlobal`。写新代码时检查所有 early-return 分支。
3. **64 位注意**：`SetWindowLong`/`GetWindowLong` 在 64 位下严格说应该用 `SetWindowLongPtr`，Tai 目标平台为 x64 时当前写法对 `GWL_EXSTYLE`（32 位样式值）仍可用，但新增指针类索引（如 `GWLP_WNDPROC`）必须换 `*Ptr` 版本。
4. **权限降级**：对高权限进程的操作会静默失败（返回 `IntPtr.Zero` / 空串），调用方都有判空降级，新增代码也要保持这个习惯，不要抛异常打断统计链路。
5. **新增 P/Invoke 的归属**：通用 API 加到 [Win32API.cs](file:///d:/dev/store/Tai/Core/Librarys/Win32API.cs)；仅单个服务使用的就地声明在该服务内并放 `#region win32api` 中（项目现有惯例）。
