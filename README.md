# iZed? Zios? zedios?

**An experiment in running [Zed](https://github.com/zed-industries/zed) — the real editor, not a reskin — on iOS.**

This branch is a *clean fork of upstream `zed-industries/zed`*. We add an iOS platform layer
to gpui and stub out what the iOS kernel forbids, rather than booting Zed inside a Linux
userland (the approach the [Android port](https://github.com/Dylanmurzello/zed-android-port)
takes — which doesn't translate to iOS). We *learn* from that Android port and lift the
iOS-specific pieces from [`itsbalamurali/gpui-mobile`](https://github.com/itsbalamurali/gpui-mobile).

> Status: planning / early. See [`docs/ios-port-plan.md`](docs/ios-port-plan.md) for the full plan.

## The idea in one diagram

```
Zed editor stack (editor, text, rope, language, git)   <- runs unchanged
            |
          gpui  <- platform-agnostic UI framework
            |
  gpui_platform::current_platform()   <- one #[cfg] seam
            |
   +--------+---------+--------------+
 gpui_macos      gpui_linux       gpui_ios  <-- NEW (this project)
        |                            |
        +---- shares Metal renderer -+   (iOS Metal == macOS Metal API)
                                     |
                 UIKit view . CADisplayLink . UITextInput . UITouch
                 (reference: gpui-mobile)
```

## Why this approach

- **iOS is not Linux.** No userland, no JIT, no spawning arbitrary binaries — these are
  *kernel-enforced*, so they hold even when sideloaded. That rules out language servers,
  the WASM extension host, the terminal, and the bundled `git` binary.
- **But the editor is portable.** `editor`/`text`/`rope`/`language` run anywhere gpui runs.
- **And the renderer already exists.** Upstream gpui has a Metal renderer in `gpui_macos`;
  iOS Metal is the same API, so we reuse it. Only windowing/input/lifecycle is rewritten
  AppKit -> UIKit.

So the work is: **(A)** a new `crates/gpui_ios` (template: `crates/gpui_macos`), **(B)**
feature-gating the sandbox-forbidden subsystems off, and **(C)** routing git through
libgit2 in-process (desktop Zed shells out to a `git` binary we can't run on iOS).

## Milestones

| # | Milestone | Acceptance |
|---|-----------|------------|
| **M0** | Rebase & build skeleton | `cargo build` green for `aarch64-apple-ios(-sim)`; empty host app launches on simulator |
| **M1** | Pixels on screen | `crates/gpui_ios` renders an animating gpui view on the simulator (Metal + `CADisplayLink` + UIKit) |
| **M2** | Boot the workspace | LSP/extensions/terminal/node/updater feature-gated off; empty Zed workspace renders |
| **M3** | Editor parity | Open/edit/save a real file — rope, tree-sitter, multi-cursor, find/replace — with iOS keyboard, IME, touch selection |
| **M4** | Files & project | `UIDocumentPicker` + security-scoped bookmarks; project panel; multi-buffer tabs |
| **M5** | Git (local) | status / stage / commit / diff via libgit2, entirely on-device |
| **M6** | Git (remote) | clone / fetch / pull / push via libgit2 `https`+`ssh`; credentials in Keychain |

Target for now: **dev/sideload + simulator** (Xcode signing). App Store distribution is a
later concern.

## Credits

- [Zed](https://github.com/zed-industries/zed) by Zed Industries — the editor and gpui.
- [gpui-mobile](https://github.com/itsbalamurali/gpui-mobile) by @itsbalamurali — iOS/Android
  `gpui::Platform` reference implementation.
- [zed-android-port](https://github.com/Dylanmurzello/zed-android-port) by @Dylanmurzello —
  mobile platform module-shape reference.

## License

Inherits Zed's licensing (GPL-3.0 / AGPL-3.0 / Apache-2.0 as applicable per crate). See the
upstream license files retained in this fork.
