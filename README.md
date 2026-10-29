# mang-shell

> 标书工作台桌面壳（tao + wry + WebView2），VSCode 风格。
>
> **本仓库是为了求助一个问题：自绘标题栏的拖拽和双击最大化失效。**

---

## 问题

```
✅ 窗口能开，UI 正常
✅ 窗口按钮（最小化/最大化/关闭）能点  ← 说明 IPC 通道是通的
✅ 窗口边缘能拖动调整大小              ← 说明 WM_NCHITTEST 部分生效
❌ 按住标题栏空白处拖 -> 窗口不动
❌ 双击标题栏 -> 不最大化
```

---

## 已确认的事实

### 1. WebView2 铺满整个客户区

```rust
WebViewBuilder::new()
    // ... 没有 with_bounds，默认填满
    .build(window)?;   // window 是 tao 的 &Window
```

包括 32px 标题带。所以鼠标点在标题带上时，**事件落在 WebView 子窗口而不是父窗口**。

### 2. 父窗口的 WM_NCHITTEST 拿不到标题带的点击

系统先问 WebView，再问父链。WebView 返回 HTCLIENT 就结束了。

### 3. 已尝试的方案（都不生效）

| # | 方案 | 结果 |
|---|---|---|
| 1 | 父窗口 `SetWindowSubclass` 拦 `WM_NCHITTEST` 返回 `HTCAPTION` | ❌ 事件不到父窗口 |
| 2 | JS `mousedown` → IPC → Rust `ReleaseCapture()` + `SendMessage(WM_NCLBUTTONDOWN, HTCAPTION)` | ❌ 不生效 |
| 3 | 子窗口遍历 `ChildWindowFromPointEx` 判「是否要点」 | ❌ 判定总为 true（命中 WebView） |

---

## 代码位置

| 文件 | 作用 | 行数 |
|---|---|---|
| `crates/mang-app/src/chrome.rs` | **标题栏核心**：SetWindowSubclass / WM_NCHITTEST / WM_NCCALCSIZE / window_action | ~650 |
| `crates/mang-app/src/shell.rs` | 窗口创建（tao WindowBuilder 配置） | 75 |
| `crates/mang-app/src/main.rs` | WebView 创建 + IPC 分发 | ~600 |
| `web-ui/mang/app.js` | UI 侧事件绑定 | ~2500 |
| `web-ui/mang/index.html` | 布局骨架（菜单栏 = 标题栏） | 381 |

---

## 关键代码

### 窗口创建（去边框）

```rust
// shell.rs
WindowBuilder::new()
    .with_decorations(false)          // 去系统标题栏
    .with_undecorated_shadow(false)   // 关键：让 tao 原样保留 rgrc，整窗 = 客户区
    .with_inner_size(LogicalSize::new(1440.0, 900.0))
```

**为什么 `with_undecorated_shadow(false)`**：tao 的 `WM_NCCALCSIZE` 在「无装饰 + 有阴影」时会把客户区按阴影内缩，导致 WebView 顶在窗口第 1 像素，把整条标题带盖住。

### 子类拦截（chrome.rs）

```rust
SetWindowSubclass(hwnd, Some(proc_), SUBCLASS_ID, 0);

unsafe extern "system" fn proc_(hwnd, msg, wparam, lparam, _, _) -> LRESULT {
    match msg {
        WM_NCCALCSIZE => { /* 客户区 = 整窗 */ }
        WM_NCHITTEST => hit_test_passthrough(hwnd, lparam),
        WM_LBUTTONDOWN | WM_LBUTTONDBLCLK | WM_NCLBUTTONDBLCLK
        | WM_RBUTTONDOWN | WM_MBUTTONDOWN => {
            if try_dispatch(hwnd, msg, wparam, lparam) { return LRESULT(0); }
            DefSubclassProc(hwnd, msg, wparam, lparam)
        }
        _ => DefSubclassProc(hwnd, msg, wparam, lparam),
    }
}
```

### 命中测试（返回 HTCAPTION）

```rust
fn hit_test(hwnd: HWND, lparam: LPARAM) -> LRESULT {
    let sx = (lparam.0 & 0xFFFF) as u16 as i16 as i32;      // 屏幕坐标
    let sy = ((lparam.0 >> 16) & 0xFFFF) as u16 as i16 as i32;
    let wr = hit_rect(hwnd);   // GetWindowRect（与系统派发 WM_NCHITTEST 同坐标系）

    // 左右下三条边 -> 缩放手柄
    // ...

    // 标题带 -> HTCAPTION
    if in_caption_band(hwnd, sx, sy) {
        return LRESULT(HTCAPTION as isize);
    }
    LRESULT(HTCLIENT as isize)
}
```

### 拖动动作（IPC 触发，当前不生效）

```rust
"drag" => {
    let _ = ReleaseCapture();
    let _ = SendMessageW(
        hwnd,
        WM_NCLBUTTONDOWN,
        Some(WPARAM(HTCAPTION as usize)),
        Some(LPARAM(0)),          // ⚠️ 坐标传 0，可能是问题
    );
    true
}
```

### JS 侧

```javascript
const isTitlebarBlank = (target) => {
    if (target.closest("button, a, input, select, textarea, [role=menuitem]")) return false;
    const bar = target.closest("#menubar, #titlebar");
    if (!bar) return false;
    if (target.closest(".menu-pop, .dropdown, [data-popover]")) return false;
    return true;
};

const onTitlebarMouseDown = (e) => {
    if (e.button !== 0) return;
    if (!isTitlebarBlank(e.target)) return;
    if (e.detail > 1) return;
    e.preventDefault();
    ipc("window.action", { action: "drag" });
};

menubarEl.addEventListener("mousedown", onTitlebarMouseDown);
menubarEl.addEventListener("dblclick", onTitlebarDblclick);
```

---

## 怀疑的方向（欢迎指点）

| # | 怀疑 | 理由 | 怎么验证 |
|---|---|---|---|
| **1** | **`SendMessageW` 被跨线程调用，在错误的线程进拖动循环** | IPC handler 跑在 WebView2 的消息泵线程，不是 tao 事件循环线程 | 把动作投到 EventLoopProxy，在 UI 线程执行 |
| **2** | **WebView2 自己处理 WM_NCHITTEST 返回 HTCLIENT** | 需要给 WebView 的 HWND 也挂子类返回 HTTRANSPARENT | 给 WebView HWND `SetWindowSubclass` |
| 3 | `LPARAM(0)` 坐标无效 | `WM_NCLBUTTONDOWN` 可能要看坐标 | 传真实鼠标屏幕坐标 |
| 4 | `ReleaseCapture()` 时机 | WebView 可能还持有捕获 | 在 `WM_MOUSEMOVE` 里再 ReleaseCapture |
| 5 | 应该让父窗口的 `WM_NCHITTEST` 生效，而不是手动 SendMessage | 但事件不到父窗口 | 需要方案 2 配合 |

---

## 我最想知道的

**WebView2 子窗口是否能通过 `SetWindowSubclass` 拦截 `WM_NCHITTEST`，并对标题带区域返回 `HTTRANSPARENT`？**

如果能，父窗口的 `hit_test` 就会收到这条消息并返回 `HTCAPTION`，拖动自然生效。

---

## 环境

```
Windows 11 (10.0.26100)
tao 0.37.2 / wry 0.57.1 / WebView2 Runtime 156.0.4314.50
Rust 1.99
```

---

## 构建

```bash
cd crates/mang-app
cargo build --release

cd ../../web-ui
npm i
node mang-build.mjs

../crates/mang-app/target/release/mang-app.exe --workspace <任意目录>
```

---

## License

MIT