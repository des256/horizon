# ohmu-platform

Trait definitions for the platform bridge. Defines the interface between Ohmu and the operating system without implementing it for any specific OS.

## What lives here

- **Window trait** — create/resize/close windows, set title, enter/exit fullscreen, get display properties (DPI, safe insets, refresh rate).
- **Input trait** — receive pointer events (down/move/up with pointer ID), keyboard events (key down/up, character input), and translate to `ohmu-core` event types.
- **IME trait** — the ~5-8 methods each platform must implement for text input: `report_cursor_rect`, `accept_preedit`, `handle_commit`, `get_surrounding_text`, `support_reconversion`.
- **Clipboard trait** — read/write text (and optionally rich content) to system clipboard.
- **Cursor trait** — set cursor shape (pointer, text, grab, etc.).
- **Text shaping trait** — interface to HarfBuzz (or platform text shaper) for `Raw → Preprocessed` glyph conversion. Covers the CPU text pipeline: UAX#14 line breaking, UAX#9 BiDi reordering, HarfBuzz shaping, grapheme cluster detection.
- **Accessibility trait** — expose semantic roles, focus order, text alternatives, and state flags to the platform's accessibility tree (AT-SPI on Linux, NSAccessibility on macOS, UIA on Windows, etc.).
- **Lifecycle trait** — app suspend/resume, low-memory warnings, orientation lock.
- **TargetProfile detection** — query the OS for viewport size, pixel density, safe insets, input capabilities, accessibility settings, motion preferences. Populates the `TargetProfile` struct from `ohmu-core`.

## Dependencies

- `ohmu-core` — event types, TargetProfile, cursor shapes, accessibility role enum.
- No platform-specific dependencies. This crate is pure trait definitions.

## Testing tiers

### Tier 1 — Trait contract tests

Define test helpers that verify any implementation of the platform traits satisfies the expected contracts:

- IME: preedit → commit sequence produces expected text.
- Input: pointer event sequences produce expected gesture arena states.
- TargetProfile detection: returned values are within valid ranges.

These are test *utilities* consumed by the platform implementation crates (`ohmu-desktop`, `ohmu-mobile`, `ohmu-web`), not standalone tests.

### Tier 2 — N/A

No GPU interaction.

### Tier 3 — Implemented by downstream crates

Each platform crate (`ohmu-desktop`, `ohmu-mobile`, `ohmu-web`) implements these traits and tests them against real OS APIs. The contract tests from Tier 1 are reused there.
