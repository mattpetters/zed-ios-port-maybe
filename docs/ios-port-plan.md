# Zed for iOS — Implementation Plan

**Status:** Planning · **Last updated:** 2026-05-26 · **Branch:** `ios` (forked off `zed-industries/zed@main`)

This document is the implementation plan for running **real Zed** on iOS (iPhone/iPad) as a
clean, fast, **stable** editor — primarily for editing markdown and code over your iOS Files
providers. It is the detailed companion to the milestone summary in the project [README](../README.md).

---

## Why this project (the actual motivation)

Obsidian mobile's plugin instability makes editing a vault on iOS miserable. The goal is Zed's
editing quality on iOS: open a folder from the **Files** app (iCloud / Google Drive / local),
browse it, search it, and edit fast and stably — with markdown preview for vault use.

**Git is out of scope.** It's handled by GitSync / Working Copy on the side. This is a
deliberate scope cut: it drops the single biggest technical risk in earlier drafts of this plan
(libgit2 network transport, which iOS can't do the way desktop Zed does).

---

## TL;DR strategy

1. **Base off clean upstream Zed**, not the Android port. We *learn from* the Android
   port (`Dylanmurzello/zed-android-port`) as a reference for how a mobile platform wires
   into gpui — we do not inherit its Linux-userland/Termux scaffolding, a dead end on iOS.
2. **Add a new `crates/gpui_ios` crate**, modeled on the structure of the existing
   `crates/gpui_macos`. iOS Metal ≈ macOS Metal, so the renderer is *reused*; only the
   windowing/input/lifecycle layer is rewritten AppKit → UIKit.
3. **Leverage `itsbalamurali/gpui-mobile`** as the concrete reference for the iOS-specific
   pieces (UIKit view + `CADisplayLink` + `UITextInput` + touch).
4. **Open folders through the iOS Files browser** — `UIDocumentPicker` + security-scoped
   bookmarks, so any provider (iCloud, Google Drive, On My iPhone) works and reopens with access.
5. **Stub everything the iOS kernel forbids** — language server binaries, the extension host,
   terminal PTYs, the Node runtime, task spawning. Kernel-enforced, so they apply even sideloaded.
6. **Markdown is the primary workload** — reuse `crates/markdown_preview` for the preview pane
   and tree-sitter for highlighting/outline. That *is* the true-to-desktop markdown experience.

Target for now: **dev/sideload + simulator** (Xcode signing). App Store distribution is a
later concern; the kernel constraints above already bound the design.

### Non-goals
- **Git** — GitSync / Working Copy. No libgit2 network transport, no on-device git client.
- **Terminal**, **extension host**, **task spawning**, **Node runtime** — kernel-forbidden.
- **On-device LSP server binaries** — see the Language Features section; the stretch path is
  BYO-host remote LSP, explicitly *not* a hosted SaaS.

---

## Why fork upstream Zed (and what we keep from the Android port)

The fork (`mattpetters/zed-ios-port-maybe`) originally descended from an Android port that boots
Zed as a Linux process inside a bundled Linux userland (Vulkan via the Adreno driver, Termux
bootstrap, Storage Access Framework glue). **None of that translates to iOS:**

| iOS reality | Consequence |
| --- | --- |
| No Linux userland | The Android port's entire boot strategy is unusable |
| No JIT / no `fork`+`exec` of arbitrary binaries (kernel-enforced) | No language servers, no WASM extension host, no terminal |
| Hard app sandbox | File access via document picker + security-scoped bookmarks only |
| Metal, not Vulkan | But gpui already has a Metal renderer in `gpui_macos` |

What the Android port (`crates/gpui_android`) is genuinely **useful for** is its *module shape* —
it proves out which surfaces a mobile gpui platform must implement: `platform · window ·
display · dispatcher · events · keyboard · ime · touch · clipboard · cursor · frame_timing`.
We use that as a checklist, plus its IME/touch approach, and discard the rest.

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
`dispatch2`. `gpui-mobile`'s iOS code is the reference for the rewritten modules.

### What gets stubbed (feature-gated off on iOS)

These compile to no-ops or return "unsupported" on `target_os = "ios"`:

- **`lsp` / language servers** — no subprocess (see Language Features below).
- **`extension_host`** — WASM/process extensions disabled.
- **`terminal` / `terminal_view`** — no PTY.
- **`node_runtime`** — no Node binary.
- **`task` spawning**, **`remote`/SSH dev server** (until repurposed for BYO-host LSP),
  **auto-updater**.
- **`git`** — out of scope entirely; the git panel / blame / commit UI is gated off.

The bulk of "make full Zed boot" is this stubbing work, not the editor itself — the
`editor`/`text`/`rope`/`language`/`markdown_preview` crates run unchanged once gpui renders.

---

## Files & project access

- **Open a folder via the iOS Files browser.** `UIDocumentPickerViewController` returns a
  security-scoped URL for any provider (iCloud Drive, Google Drive, On My iPhone, etc.).
  Persist a **security-scoped bookmark** so the project reopens with access after a restart.
  Wrap access in `startAccessingSecurityScopedResource` / `stopAccessing...`.
- **Provider quirks:** cloud-provider files may download lazily or go stale; handle
  not-yet-materialized files and coordinate reads via `NSFileCoordinator`.
- Zed's `RealFs` works within the granted scope. The project panel / file tree / multi-buffer
  tabs come from upstream once the fs layer is wired to the bookmarked root.

---

## Search

Reuse Zed's project search:
- **Fuzzy file finder** (open-by-name) and **full-text project-wide search** across the opened
  folder, both from upstream `search` / `project` crates.
- No subprocess `rg` on iOS — Zed's search runs in-process, so this ports without a binary.

---

## Markdown & preview

The vault use-case makes markdown first-class:
- **Highlighting / outline / folding** via tree-sitter markdown (in-process, free).
- **Preview render** by reusing `crates/markdown_preview` (verify it renders without
  desktop-only affordances).
- **`[[wikilink]]` navigation** for Obsidian vaults is *not* native Zed — it's custom work.
  Open question: full Obsidian-isms (wikilinks, backlinks, embeds, tags) vs. plain markdown only.

---

## Language features

iOS cannot spawn LSP server binaries (kernel-enforced, even sideloaded), so desktop-style LSP
isn't available on-device. Tiers:

1. **On-device, default — tree-sitter.** Symbol outline, code folding, structural selection,
   bracket matching, indentation. For markdown (the workload), this plus preview *is* the
   true-to-desktop experience.
2. **Stretch — BYO-host remote LSP.** Connect to a language server running on *your own*
   Mac/server over the network, adapting Zed's existing `remote`/SSH dev-server plumbing.
   Privacy-friendly, no infra to run, no uploading your source anywhere.

> **Not a hosted SaaS.** A hosted remote-LSP service is explicitly not the play: it's a thin
> slice of what Codespaces/Gitpod/Coder already sell as full environments, the unit economics
> are ugly (LSPs sit multi-GB warm per project), it requires ingesting the user's source, and
> for markdown the value is ~nil. If there's money in this, it's selling the app — not an LSP
> backend.

3. **Not pursued — in-process WASM LSP.** wasmtime without JIT is interpreted (slow), few real
   LSPs ship WASM builds. High effort, poor payoff on mobile.

---

## App entry

An Xcode iOS app target (Swift/`UIApplicationDelegate` + `UIScene`) that links the Rust core
built as a `staticlib`/`cdylib` and calls the gpui application entry — analogous to how the
Android port uses `android_main`. Build via `cargo` for `aarch64-apple-ios` (device) and
`aarch64-apple-ios-sim` / `x86_64-apple-ios` (simulator), wired into Xcode (XcodeGen optional,
mirroring gpui-mobile's setup).

---

## Milestones

M0–M1 are prerequisite platform plumbing; M2 onward are the product milestones you feel.
Each is independently demoable with a concrete acceptance check.

### M0 — Build skeleton
- Repo content = upstream Zed (`ios` branch off `upstream/main`). ✅ done as part of this plan.
- Add Xcode app target + Rust `staticlib`; cargo cross-compile profile for iOS targets.
- **Accept:** `cargo build -p gpui --target aarch64-apple-ios-sim` (and device) succeeds; an
  empty Swift host app launches on the simulator.

### M1 — Pixels on screen
- New `crates/gpui_ios` crate; `IosPlatform` + `IosWindow` (UIKit view + `CAMetalLayer`),
  `CADisplayLink` frame loop, reused Metal renderer, `UITextInput` IME; new `current_platform` arm.
- **Accept:** a trivial gpui view (colored quad / text) renders and animates on the simulator,
  with keyboard input echoing through `UITextInput`.

### M2 — Open a folder (first product milestone)
- `UIDocumentPicker` + security-scoped bookmarks; wire the bookmarked root into `RealFs`;
  project panel / file tree.
- **Accept:** pick a folder from iCloud / Google Drive / On My iPhone, browse it, and reopen the
  app with access still granted.

### M3 — Edit files
- Open / edit / save markdown + code: rope, multi-cursor, find/replace (inherited from `editor`).
- iOS soft keyboard + `UITextInput` IME + `UITouch` selection/scroll.
- **Accept:** edit and save a file on a physical iPhone with working selection, code input
  (autocorrect off), and syntax highlighting.

### M4 — Search the project
- Fuzzy file finder + full-text project-wide search across the opened folder.
- **Accept:** find a file by fuzzy name and grep a string across the vault from the device.

### M5 — Markdown + preview
- Tree-sitter markdown highlighting; reuse `markdown_preview` for a render pane; `[[wikilink]]`
  navigation.
- **Accept:** open a vault note, toggle a live preview, and follow a wikilink to another note.

### M6 — Language features
- On-device tree-sitter: symbol outline, code folding, structural selection.
- *(Stretch: BYO-host remote LSP per the Language Features section.)*
- **Accept:** outline + folding work for a code file on-device.

---

## Risks & open questions

- **Platform layer is the real cost** — nothing renders until `gpui_ios` exists. M0–M1 dominate.
- **Metal renderer assumptions** — `gpui_macos`'s renderer may bake in AppKit-isms
  (`CAMetalLayer` setup, drawable sizing, Retina scale via `NSScreen`). Expect a thin
  abstraction over the layer/scale source rather than a pure copy.
- **`objc2-app-kit` → `objc2-ui-kit`** surface differences in event/cursor/appearance code.
- **Text input fidelity** — `UITextInput` is a large protocol; matching gpui's input model
  (marked text, selection rects, autocorrect suppression for code) is non-trivial. This is
  where gpui-mobile's iOS IME code is most valuable.
- **Files-provider behavior** — lazy download, stale cache, file coordination for iCloud/Drive.
- **Markdown preview reuse** — `markdown_preview` may assume desktop affordances; verify on iOS.
- **Obsidian-isms** — decide wikilink/backlink/tag scope (M5) vs. plain markdown only.
- **App lifecycle** — backgrounding/foregrounding, memory pressure, scene reconnection.

---

## References

- Upstream Zed — https://github.com/zed-industries/zed
- gpui-mobile (iOS/Android `gpui::Platform` impl) — https://github.com/itsbalamurali/gpui-mobile
- zed-android-port (module-shape reference) — https://github.com/Dylanmurzello/zed-android-port
- Zed mobile tracking issues — zed-industries/zed#12039, #43206
- Key files: `crates/gpui_platform/src/gpui_platform.rs`, `crates/gpui_macos/src/*`,
  `crates/markdown_preview`, `crates/search`, `crates/project`
