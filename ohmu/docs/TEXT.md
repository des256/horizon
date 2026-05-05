# Text Pipeline

SDF rendering of glyphs is straightforward. The hard parts are everything *before* the glyph reaches the renderer and everything around *editing*.

```
codepoints → segmentation → shaping → layout → rendering
                                                    ↑ easy (SDF)
←────────── all of this is the hard part ──────────→
```

## Text as Enum

```rust
enum TextContent {
    Raw(String),                        // needs CPU processing
    Preprocessed(Vec<PositionedGlyph>), // ready for GPU
}

struct PositionedGlyph {
    glyph_id: u32,
    x: f32,
    y: f32,
    font_id: u16,
    size: f16,
}
```

The CPU edits text nodes in-place: `Raw → Preprocessed` when text content or constraints change.

## CPU Text Pipeline (Libraries)

| Stage | Library | What it does |
|-------|---------|--------------|
| Line breaking | ICU BreakIterator or UAX #14 | Determines legal break opportunities |
| BiDi | SheenBidi or ICU | Reorders mixed LTR/RTL text for visual display |
| Shaping | HarfBuzz | Converts codepoints → positioned glyph IDs (handles Arabic forms, Devanagari ligatures, Latin kerning) |
| Grapheme clusters | UAX #29 | Determines cursor movement boundaries |

All are CPU-side, well-solved as linkable C libraries. Feed results to GPU.

## IME (Input Method Editor) — Platform Bridge

IME is the one area requiring bidirectional communication with the OS. The platform owns the input method; the app is a client.

### Required Protocol Per Platform

| Obligation | What it means |
|-----------|---------------|
| Report cursor rect | OS positions candidate popup near insertion point |
| Accept preedit string | Display uncommitted composition (underlined, highlighted segments) |
| Handle commit | Replace preedit with final text |
| Report surrounding text | OS uses for prediction, reconversion |
| Support reconversion | User selects committed text, re-enters composition |

### Platform APIs

| Platform | API |
|----------|-----|
| Windows | TSF (Text Services Framework) |
| macOS | NSTextInputClient |
| Linux | IBus / Fcitx / text-input-v3 (Wayland) |
| Android | InputConnection |
| iOS | UITextInput |

**Design decision:** Text input is the one place where we accept a platform primitive rather than reimplementing. We own the *rendering* (SDF) but delegate the *input protocol* to a thin platform bridge (~5-8 methods per platform).

## Architecture

```
┌──────────────────────────────────────┐
│  Scene buffer: text node =           │
│  { string_ref, font_id, size,        │
│    max_width, alignment, role }       │
└──────────────┬───────────────────────┘
               │ CPU text pipeline
┌──────────────▼───────────────────────┐
│  1. UAX#14 line break opportunities  │
│  2. UAX#9 BiDi reordering           │
│  3. HarfBuzz shaping per run         │
│  4. Line layout (greedy/optimal)     │
│  Output: glyph buffer               │
│  [{glyph_id, x, y, font_id}...]     │
└──────────────┬───────────────────────┘
               │ uploaded to GPU
┌──────────────▼───────────────────────┐
│  SDF atlas lookup + instanced render │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│  Platform bridge (thin, per-OS):     │
│  - on_preedit(text, attrs)           │
│  - on_commit(text)                   │
│  - get_cursor_rect() → rect          │
│  - get_surrounding() → (text, pos)   │
│  - clipboard_read/write              │
└──────────────────────────────────────┘
```
