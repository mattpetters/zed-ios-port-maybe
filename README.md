# iZed? Zios? zedios?

**A clean, fast, stable text editor for iOS — the real [Zed](https://github.com/zed-industries/zed) editor, not a reskin — for editing markdown and code over your iOS Files providers (iCloud, Google Drive, local).**

This branch is a *clean fork of upstream `zed-industries/zed`*. We add an iOS platform layer
to gpui and stub out what the iOS kernel forbids, rather than booting Zed inside a Linux
userland (the approach the [Android port](https://github.com/Dylanmurzello/zed-android-port)
takes — which doesn't translate to iOS). We *learn* from that Android port and lift the
iOS-specific pieces from [`itsbalamurali/gpui-mobile`](https://github.com/itsbalamurali/gpui-mobile).

> Status: planning / early. See [`docs/ios-port-plan.md`](docs/ios-port-plan.md) for the full plan.

## Why

Obsidian mobile's plugin instability makes editing a vault on iOS miserable. The goal here is
simple: **Zed's editing quality on iOS** — open a folder from the Files app, browse it, search
it, and edit markdown/code fast and stably. **Git is out of scope** — handle it with GitSync or
Working Copy. (Dropping git also removes the single biggest technical risk: libgit2 network
transport, which iOS can't do the way desktop Zed does.)

## The idea in one diagram

```
Zed editor stack (editor, text, rope, language, search, markdown_preview)  <- runs unchanged
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
            UIKit view . CADisplayLink . UITextInput . UITouch . UIDocumentPicker
                          (reference: gpui-mobile)
```

## Why this approach

- **iOS is not Linux.** No userland, no JIT, no spawning arbitrary binaries — these are
  *kernel-enforced*, so they hold even when sideloaded. That rules out language servers,
  the WASM extension host, the terminal, and any bundled binary.
- **But the editor is portable.** `editor`/`text`/`rope`/`language`/`markdown_preview` run
  anywhere gpui runs.
- **And the renderer already exists.** Upstream gpui has a Metal renderer in `gpui_macos`;
  iOS Metal is the same API, so we reuse it. Only windowing/input/lifecycle is rewritten
  AppKit -> UIKit.

So the work is: **(A)** a new `crates/gpui_ios` (template: `crates/gpui_macos`), **(B)**
Files-provider folder access (`UIDocumentPicker` + security-scoped bookmarks), and **(C)**
feature-gating the sandbox-forbidden subsystems off.

## Milestones

M0–M1 are prerequisite platform plumbing; **M2 onward are the product milestones you feel.**

| # | Milestone | Acceptance |
|---|-----------|------------|
| **M0** | Build skeleton | `cargo build` green for `aarch64-apple-ios(-sim)`; empty host app launches on simulator |
| **M1** | Pixels (`gpui_ios`) | `crates/gpui_ios` renders an animating gpui view on the simulator (Metal + `CADisplayLink` + UIKit + `UITextInput` IME) |
| **M2** | Open a folder | Pick a folder via iOS **Files** (iCloud / Google Drive / On My iPhone) with `UIDocumentPicker` + security-scoped bookmarks; browse the tree; reopens with access after restart |
| **M3** | Edit files | Open/edit/save markdown + code — rope, multi-cursor, find/replace — with iOS keyboard, IME, touch selection |
| **M4** | Search the project | Fuzzy file finder + full-text project-wide search across the opened folder |
| **M5** | Markdown + preview | Tree-sitter markdown highlighting + Zed `markdown_preview` render pane; `[[wikilink]]` navigation for vault use |
| **M6** | Language features | On-device tree-sitter (symbol outline, folding, structural selection). *Stretch: BYO-host remote LSP — connect to a language server on your own machine, not a hosted service.* |

Target for now: **dev/sideload + simulator** (Xcode signing). App Store distribution is a
later concern.

### Non-goals
- **Git** — use GitSync / Working Copy. No libgit2 network work.
- **Terminal, extension host, task spawning** — kernel-forbidden on iOS.
- **On-device LSP server binaries** — can't spawn subprocesses; real code intelligence is a
  BYO-host remote-LSP stretch goal, not core, and explicitly not a hosted SaaS.

## Credits

- [Zed](https://github.com/zed-industries/zed) by Zed Industries — the editor and gpui.
- [gpui-mobile](https://github.com/itsbalamurali/gpui-mobile) by @itsbalamurali — iOS/Android
  `gpui::Platform` reference implementation.
- [zed-android-port](https://github.com/Dylanmurzello/zed-android-port) by @Dylanmurzello —
  mobile platform module-shape reference.

## License

Inherits Zed's licensing (GPL-3.0 / AGPL-3.0 / Apache-2.0 as applicable per crate). See the
upstream license files retained in this fork.
