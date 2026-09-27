# gpui-pre-macos (patched)

Zed's `gpui_macos` crate (gpui-pre 0.3.6 snapshot of zed@bcf6582), with a
patch for a macOS frame-source stall bug.

上游：<https://crates.io/crates/gpui-pre-macos>（zed-industries/zed 快照）

## 本仓库相对上游的改动

macOS 上窗口重绘的唯一驱动是 display link（vsync），而它的启动完全依赖
occlusion / key-status / screen 变化等事件驱动。一旦恢复事件丢失（或屏幕
重配置瞬态 `screen()` 为 nil 导致启动失败），帧源会永久停摆，窗口画面冻结
（典型表现：窗口失焦/遮挡后再回来，界面停更，但应用逻辑仍在运行）。

gpui 核心为此提供了兜底机制——`PlatformWindow::frame_waker()` /
`schedule_frame()`（见 gpui `InvalidationHandler::wake_platform` 的文档，
其用途正是"平台停止为空闲窗口请求帧时唤醒帧源"），但 mac 平台对二者
都是 trait 默认空实现。

本补丁：

- `src/display_link.rs`：`WindowFrameSource` 新增 `is_running()`
- `src/window.rs`：实现 `frame_waker()` 与 `schedule_frame()`——帧源停了，
  在有内容更新（invalidation / flush_effects）时自动重启。
  安全性由 `start_display_link` 内部的 occlusion 门控保证
  （读取 NSWindow 实时遮挡状态，不依赖可能丢失的事件）

另有若干 `eprintln!("[gpui-diag] ...")` 诊断日志（事件驱动，平时静默），
以及 `Cargo.toml` 增加 `[lints.rust] deprecated = "allow"`
（恢复 cargo 对 registry 依赖 cap-lints 的同等行为，
静默上游 pin 在已废弃 cocoa 0.26 生态产生的 1200+ 条 deprecation 噪音）。

## License

Apache-2.0（与上游一致，见 LICENSE-APACHE）
