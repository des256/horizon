# ohmu-desktop

Platform bridge implementation for Windows, macOS, and Linux. Wires `ohmu-core` + `ohmu-gpu` + `ohmu-dsl` together with OS windowing and input.

## What lives here

- **Window management** — `winit` integration for window creation, resize, DPI change, fullscreen. Implements the window trait from `ohmu-platform`.
- **Input translation** — converts `winit` events (keyboard, mouse, touch, trackpad) to `ohmu-core` pointer/keyboard events.
- **IME implementation** — per-platform text input:
  - Windows: TSF (Text Services Framework)
  - macOS: NSTextInputClient
  - Linux: IBus / Fcitx / text-input-v3 (Wayland), XIM (X11 fallback)
- **Clipboard** — platform clipboard via `arboard` or direct OS APIs.
- **Cursor** — set cursor shape through `winit`.
- **Text shaping** — links HarfBuzz (via `harfbuzz-rs` or `rustybuzz`) for the CPU text pipeline. Integrates ICU/UAX libraries for line breaking, BiDi, grapheme clusters.
- **Accessibility** — bridges to AT-SPI (Linux), NSAccessibility (macOS), UIA (Windows) via `accesskit`.
- **Main loop** — event loop that drives the frame cycle: receive OS events → update state → reconcile → GPU layout → render → present.
- **TargetProfile detection** — queries OS for monitor DPI, window size, safe insets, input devices present, accessibility settings, reduced-motion preference.

## Dependencies

- `ohmu-core`, `ohmu-gpu`, `ohmu-dsl`, `ohmu-platform`.
- `winit` — cross-platform windowing.
- `harfbuzz-rs` or `rustybuzz` — text shaping.
- `accesskit` — accessibility tree bridge.
- Platform-specific system libraries (linked conditionally).

## Testing tiers

### Tier 1 — Input translation and event mapping

Unit tests for the translation layer between `winit` events and `ohmu-core` events. No window needed:

- Keyboard event mapping (scancode → key enum, modifier tracking).
- Mouse/touch event translation (position, pointer ID assignment).
- DPI scaling calculations.
- TargetProfile construction from mock OS values.

### Tier 2 — Headless integration

Create a `wgpu` surface without presenting to a display. Run the full pipeline:

1. Parse DSL source or construct scene buffer directly.
2. Simulate input events.
3. Run state mutations + reconciliation.
4. Execute GPU layout + render.
5. Read back framebuffer, compare to reference.

Tests the wiring between crates — does the main loop actually connect everything correctly?

### Tier 3 — Real window, real input (primary)

This is where Tier 3 testing happens for desktop targets. Tests require an actual display (or virtual framebuffer like Xvfb on Linux).

**Per-platform verification:**
- Window opens at correct size and DPI.
- Mouse/keyboard input produces expected scene state changes.
- IME composition works (CJK input, emoji picker, dead keys for accents).
- Clipboard round-trip (copy text from app, paste back).
- Accessibility tree exposes correct roles and labels (verify with AT-SPI/VoiceOver/Narrator).
- Window resize triggers TargetProfile change → recompilation → correct new layout.
- Multi-monitor with different DPIs.

**Running on remote machines:**
For cross-platform CI (test macOS from a Linux dev machine, or vice versa):

1. Cross-compile: `cross build --target x86_64-apple-darwin --release`
2. Push binary: `rsync` to remote machine.
3. Execute: `ssh remote "./ohmu-desktop-test"` — runs headless tests or captures screenshots.
4. Pull results: exit code + screenshots + test output back to local.

Linux CI can use Xvfb for windowless GPU rendering. macOS CI runners have real displays. Windows CI uses virtual desktop.
