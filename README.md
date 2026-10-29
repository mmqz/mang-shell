# mang-shell

> 标书工作台桌面壳（tao + wry + WebView2），VSCode 风格。
>
> **本仓库是为了求助一个问题：自绘标题栏的拖拽和双击最大化失效。**

---

## 问题

```
✅ 窗口能开，UI 正常
✅ 窗口按钮（最小化/最大化/关闭）能点
✅ 窗口边缘能拖动调整大小
❌ 按住标题栏空白处拖 -> 窗口不动
❌ 双击标题栏 -> 不最大化
```

### 已经确认的约束

1. **WebView2 铺满整个客户区**（包括 32px 标题带），所以鼠标点在标题带上时，事件落在 WebView 子窗口
2. **父窗口的 WM_NCHITTEST 拿不到这次点击**——系统先问 WebView，再问父链
3. 已尝试：JS mousedown → IPC → Rust `ReleaseCapture()` + `SendMessage(WM_NCLBUTTONDOWN, HTCAPTION)` ——**仍不生效**
4. 已尝试：父窗口 SetWindowSubclass 拦 WM_NCHITTEST 返回 HTCAPTION ——**不生效**（因为事件根本不到父窗口）

---

## 代码位置

| 文件 | 作用 |
|---|---|
| `crates/mang-app/src/chrome.rs` | **自绘标题栏核心**：SetWindowSubclass、WM_NCHITTEST、WM_NCCALCSIZE、window_action |
| `crates/mang-app/src/main.rs` | 窗口创建、WebView 创建、IPC 分发 |
| `web-ui/mang/app.js` | UI 侧：标题栏 mousedown/dblclick 绑定 |
| `web-ui/mang/index.html` | 布局骨架（菜单栏 = 标题栏） |
| `web-ui/mang/styles.css` | 样式 |

---

## 关键实现（当前版本）

### Rust 侧：窗口去边框

```rust
WindowBuilder::new()
    .with_decorations(false)          // 去系统标题栏
    .with_undecorated_shadow(false)   // tao 保留 rgrc，整窗 = 客户区
```

### Rust 侧：子类拦截（chrome.rs）

```rust
SetWindowSubclass(hwnd, Some(proc_), SUBCLASS_ID, 0);

// proc_ 里：
match msg {
    WM_NCCALCSIZE => 客户区 = 整窗,
    WM_NCHITTEST => hit_test(...),   // 标题带返回 HTCAPTION
    WM_LBUTTONDOWN | ... => try_dispatch(...),
}
```

### Rust 侧：拖动动作

```rust
"drag" => {
    ReleaseCapture();
    SendMessageW(hwnd, WM_NCLBUTTONDOWN,
        Some(WPARAM(HTCAPTION as usize)), Some(LPARAM(0)));
}
```

### JS 侧：发起拖动

```javascript
menubarEl.addEventListener("mousedown", (e) => {
  if (e.button !== 0) return;
  if (!isTitlebarBlank(e.target)) return;
  if (e.detail > 1) return;
  e.preventDefault();
  ipc("window.action", { action: "drag" });
});
```

---

## 怀疑的方向（欢迎指点）

| # | 怀疑 | 理由 |
|---|---|---|
| 1 | `ReleaseCapture()` 时机不对 | WebView 可能还没释放鼠标捕获 |
| 2 | `SendMessageW` 被跨线程阻塞 | IPC handler 跑在 WebView2 消息循环线程，不是 UI 线程 |
| 3 | WebView2 自己处理了 `WM_NCHITTEST` 并返回 HTCLIENT | 需要给 WebView 的 HWND 挂子类 |
| 4 | `WM_NCLBUTTONDOWN` 需要真实的鼠标坐标 | 传了 `LPARAM(0)`，可能被判为无效位置 |
| 5 | 应该在 `WM_NCHITTEST` 返回 `HTCAPTION` 而不是手动 SendMessage | 但事件不到父窗口 |

---

## 环境

```
Windows 11 / tao 0.37 / wry 0.57 / WebView2 156.x
```

---

## 构建

```bash
# Rust 壳
cd crates/mang-app
cargo build --release

# UI
cd web-ui
npm i
node mang-build.mjs

# 运行
../target/release/mang-app.exe --workspace <某个目录>
```

---

## License

MIT