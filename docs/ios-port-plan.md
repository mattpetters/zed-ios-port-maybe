# Zed for iOS — Implementation Plan

**Status:** Planning · **Last updated:** 2026-05-25 · **Branch:** `ios` (forked off `zed-industries/zed@main`)

This document is the implementation plan for running **real Zed** on iOS (iPhone/iPad).
It is the detailed companion to the milestone summary in the project [README](../README.md).

---

## TL;DR strategy

1. **Base off clean upstream Zed**, not the Android port. We *learn from* the Android
   port (`Dylanmurzello/zed-android-port`) as a reference for how a mobile platform wires
   into gpui — we do not inherit its Linux-userland/Termux scaffolding, which is a dead end
   on iOS.
2. **Add a new `crates/gpui_ios` crate**, modeled byte-for-byte on the structure of the
   existing `crates/gpui_macos`. iOS Metal ≈ macOS Metal, so the renderer is *reused*; only
   the windowing/input/lifecycle layer is rewritten AppKit → UIKit.
3. **Leverage `itsbalamurali/gpui-mobile`** as the concrete reference for the iOS-specific
   pieces (UIKit view + `CADisplayLink` + `UITextInput` + touch), since it already
   implements `gpui::Platform` for iOS against *stock* gpui.
4. **Stub everything the iOS kernel forbids** — language servers, the extension host,
   terminal PTYs, the Node runtime, task spawning, the auto-updater. These are
   kernel-enforced restrictions, not App Store policy, so they apply even when sideloaded.
5. **Route git through libgit2 in-process.** Desktop Zed shells out to a bundled `git`
   binary for network ops; iOS cannot spawn it, so remote git must go through libgit2 with
   `https`+`ssh` transport and Keychain-backed credentials.

Target for now: **dev/sideload + simulator** (Xcode signing). App Store distribution is a
later concern; the kernel constraints above already bound the design.

---

## Why fork upstream Zed (and what we keep from the Android port)

The current fork (`mattpetters/zed-ios-port-maybe`) descends from an Android port that boots
Zed as a Linux process inside a bundled Linux userland (Vulkan via the Adreno driver,
Termux bootstrap, Storage Access Framework glue). **None of that translates to iOS:**

| iOS reality | Consequence |
| --- | --- |
| No Linux userland | The Android port's entire boot strategy is unusable |
| No JIT / no `fork`+`exec` of arbitrary binaries (kernel-enforced) | No language servers, no WASM extension host, no terminal, no bundled `git` binary |
| Hard app sandbox | File access via document picker + security-scoped bookmarks only |
| Metal, not Vulkan | But gpui already has a Metal renderer in `gpui_macos` |

What the Android port (`crates/gpui_android`) is genuinely **useful for** is its *module
shape* — it proves out exactly which surfaces a mobile gpui platform must implement:
`platform · window · display · dispatcher · events · keyboard · ime · touch · clipboard ·
cursor · frame_timing`. We use that as a checklist, plus its IME/touch-handling approach,
and discard the rest.

---

## Architecture

### The integration seam

Upstream Zed already uses a **split-crate platform layout**
(`gpui_macos`, `gpui_linux`, `gpui_windows`, `gpui_platform`, `gpui_wgpu`, `gpui_web`).
Platform selection happens in exactly one place:

`crates/gpui_platform/src/gpui_platform.rs`:

```rust
pub fn current_platform(headless: bool) -> Rc<dyn Platform> {
    #[cfg(target_os = "macos")]
    { Rc::new(gpui_macos::MacPlatform::new(headless)) }
    #[cfg(target_os = "windows")]
    { /* ... */ }
    #[cfg(any(target_os = "linux", target_os = "freebsd"))]
    { gpui_linux::current_platform(headless) }
    // ADD:
    // #[cfg(target_os = "ios")]
    // { Rc::new(gpui_ios::IosPlatform::new(headless)) }
}
```

**iOS integration is one new `#[cfg(target_os = "ios")]` arm here**, plus the new crate it
points at. The rest of gpui and the entire editor stack are platform-agnostic.

### The new `crates/gpui_ios` crate

Modeled on `crates/gpui_macos/src/`. Disposition of each module:

| `gpui_macos` module | iOS disposition |
| --- | --- |
| `metal_renderer.rs` | **Reuse** — Metal API is identical on iOS |
| `metal_atlas.rs` | **Reuse** |
| `shaders.metal` | **Reuse** |
| `text_system.rs` | **Reuse** — CoreText is cross-Apple |
| `open_type.rs` | **Reuse** |
| `keyboard.rs` | **Mostly reuse** — adapt key mapping for soft keyboard |
| `dispatcher.rs` | **Reuse** — GCD/`dispatch2` works on iOS |
| `display.rs` / `display_link.rs` | **Rewrite** — `UIScreen` + `CADisplayLink` (not `CVDisplayLink`) |
| `window.rs` | **Rewrite** — `UIWindow`/`UIView` + `CAMetalLayer` (not `NSWindow`/`NSView`) |
| `events.rs` | **Rewrite** — `UITouch`/`UIPress` (not `NSEvent`) |
| `platform.rs` | **Rewrite** — `UIApplication` lifecycle; no app menu |
| `pasteboard.rs` | **Rewrite** — `UIPasteboard` (not `NSPasteboard`) |
| `window_appearance.rs` | **Rewrite** — `UITraitCollection` dark mode |
| `screen_capture.rs` | **Drop** |
| IME (new) | **New** — `UITextInput`/`UIKeyInput` conformance bridged to gpui input |

Dependency swap in `Cargo.toml`: `objc2-app-kit` → `objc2-ui-kit`; keep `metal`,
`core-text`, `core-graphics`, `core-foundation`, `objc`/`objc2`, `raw-window-handle`,
`dispatch2`. `gpui-mobile`'s iOS code is the reference implementation for the rewritten
modules.

### What gets stubbed (feature-gated off on iOS)

These compile to no-ops or return "unsupported" on `target_os = "ios"`:

- **`lsp` / language servers** — no subprocess. (Later: in-process WASM analyzers, out of scope.)
- **`extension_host`** — WASM/process extensions disabled.
- **`terminal` / `terminal_view`** — no PTY.
- **`node_runtime`** — no Node binary.
- **`task` spawning**, **`remote`/SSH dev server**, **auto-updater**.
- The **bundled `git` binary** path (see Git section).

The bulk of "make full Zed boot" is this stubbing work, not the editor itself — the
`editor`/`text`/`rope`/`language` crates run unchanged once gpui renders.

---

## Git strategy

`crates/git` uses `git2` (libgit2), but the workspace pins it
`default-features = false, features = ["vendored-libgit2"]` — i.e. **no network transport
compiled in.** Desktop Zed performs clone/fetch/push by shelling out to a bundled `git`
binary (`GitBinary` in `crates/git/src/repository.rs`, `path_for_auxiliary_executable("git")`).

iOS cannot spawn that binary, so:

1. **Enable libgit2 transport for the iOS target** — build `git2` with `https` (Secure
   Transport on Apple, or vendored OpenSSL) and `ssh` (libssh2) features under
   `cfg(target_os = "ios")`.
2. **Route every network op through libgit2** (`Remote::fetch`/`push`, `Repository::clone`)
   instead of `GitBinary`. Audit each `GitBinary::new(...)` call site in `repository.rs`
   (and `commit.rs`, `blame.rs`) and provide a libgit2 path for the iOS build.
3. **Credentials via libgit2's credential callback ↔ iOS Keychain.** HTTPS tokens and SSH
   keys stored in Keychain; no `GIT_ASKPASS` subprocess.

Local ops (status / stage / commit / diff) already go through git2 in-process and port with
little change — they are the M5 milestone; remote is M6.

---

## Filesystem & app entry

- **Files:** Zed's `RealFs` works within the app sandbox. Opening user folders outside the
  sandbox uses `UIDocumentPickerViewController` + **security-scoped bookmarks** persisted so
  a project reopens with access. (Dev/sideload target — we don't yet harden for App Store
  document-scope rules.)
- **App entry:** an Xcode iOS app target (Swift/`UIApplicationDelegate` + `UIScene`) that
  links the Rust core built as a `staticlib`/`cdylib` and calls the gpui application entry —
  analogous to how the Android port uses `android_main`. Build via `cargo` for
  `aarch64-apple-ios` (device) and `aarch64-apple-ios-sim` / `x86_64-apple-ios` (simulator),
  wired into Xcode (XcodeGen optional, mirroring gpui-mobile's setup).

---

## Milestones

Each milestone is independently demoable and has a concrete acceptance check.

### M0 — Rebase & build skeleton
- Repo content = upstream Zed (`ios` branch off `upstream/main`). ✅ done as part of this plan.
- Add Xcode app target + Rust `staticlib`; cargo cross-compile profile for iOS targets.
- **Accept:** `cargo build -p gpui --target aarch64-apple-ios-sim` (and device) succeeds; an
  empty Swift host app launches on the simulator.

### M1 — Pixels on screen
- New `crates/gpui_ios` crate; `IosPlatform` + `IosWindow` (UIKit view + `CAMetalLayer`),
  `CADisplayLink` frame loop, reused Metal renderer; new `current_platform` iOS arm.
- **Accept:** a trivial gpui view (colored quad / text) renders and animates on the simulator.

### M2 — Boot the Zed workspace
- Feature-gate off LSP, extension host, terminal, node, tasks, updater, remote on iOS.
- Bring up the Zed app shell with an empty workspace.
- **Accept:** Zed's workspace UI renders on device/simulator without crashing on a missing
  subsystem.

### M3 — Editor parity
- Open / edit / save a real file: rope, tree-sitter highlighting, multi-cursor, find/replace
  (inherited from the `editor` crate).
- iOS soft keyboard + `UITextInput` IME + `UITouch` selection/scroll.
- **Accept:** edit and save a source file on a physical iPhone with working selection,
  autocorrect-off code input, and syntax highlighting.

### M4 — Files & project
- `UIDocumentPicker` + security-scoped bookmarks; project/file panel; multi-buffer tabs.
- **Accept:** open a folder, browse it, edit multiple files across app restarts.

### M5 — Git (local)
- status / stage / commit / diff via libgit2 in-process; git panel + diff view.
- **Accept:** stage and commit a change to a local repo entirely on-device.

### M6 — Git (remote)
- libgit2 `https`+`ssh` transport; clone / fetch / pull / push; Keychain credentials.
- **Accept:** clone a GitHub repo over HTTPS with a token, commit, and push from the device.

---

## Risks & open questions

- **Metal renderer assumptions** — `gpui_macos`'s renderer may bake in AppKit-isms
  (`CAMetalLayer` setup, drawable sizing, Retina scale via `NSScreen`). Expect a thin
  abstraction over the layer/scale source rather than a pure copy.
- **`objc2-app-kit` → `objc2-ui-kit`** surface differences in event/cursor/appearance code.
- **libgit2 transport build** — Secure Transport vs vendored OpenSSL for `https`; libssh2
  cross-compile for iOS.
- **Text input fidelity** — `UITextInput` is a large protocol; matching gpui's input model
  (marked text, selection rects, autocorrect suppression for code) is non-trivial. This is
  where gpui-mobile's iOS IME code is most valuable.
- **App lifecycle** — backgrounding/foregrounding, memory pressure, scene reconnection.

---

## References

- Upstream Zed — https://github.com/zed-industries/zed
- gpui-mobile (iOS/Android `gpui::Platform` impl) — https://github.com/itsbalamurali/gpui-mobile
- zed-android-port (module-shape reference) — https://github.com/Dylanmurzello/zed-android-port
- Zed mobile tracking issues — zed-industries/zed#12039, #43206
- Key files: `crates/gpui_platform/src/gpui_platform.rs`, `crates/gpui_macos/src/*`,
  `crates/git/src/repository.rs`
