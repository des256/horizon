# ohmu-test

Shared test utilities, fixtures, reference images, and test infrastructure. Not a runtime crate — exists only for `#[cfg(test)]` and test binaries.

## What lives here

- **Scene buffer builders** — ergonomic helpers for constructing test scene buffers without manually specifying every field. Fluent API for common patterns:
  ```rust
  SceneBuilder::new()
      .node(StackY, Fill(1.0), Fill(1.0))
          .child(StackX, Fixed(250.0), Fill(1.0))  // sidebar
          .child(StackX, Fill(1.0), Fill(1.0))      // main
      .build()
  ```

- **TargetProfile presets** — pre-defined profiles for common test scenarios:
  ```rust
  profiles::IPHONE_15       // 393×852, 3x, touch, phone portrait
  profiles::IPAD_LANDSCAPE  // 1194×834, 2x, touch+keyboard, tablet landscape
  profiles::DESKTOP_1080P   // 1920×1080, 1x, pointer+keyboard, desktop
  profiles::WATCH_44MM      // 184×224, 2x, touch+crown, watch
  profiles::TV_4K           // 3840×2160, 1x, dpad, TV
  ```

- **Reference images** — blessed PNG screenshots organized by test name and target profile:
  ```
  references/
  ├── layout/
  │   ├── sidebar-main/
  │   │   ├── desktop-1080p.png
  │   │   ├── phone-portrait.png
  │   │   └── tablet-landscape.png
  │   └── header-body-footer/
  │       └── ...
  ├── text/
  │   ├── latin-basic.png
  │   ├── cjk-mixed.png
  │   └── rtl-arabic.png
  └── patterns/
      ├── nav-bottom-tabs.png
      ├── nav-sidebar.png
      └── ...
  ```

- **Image comparison** — pixel-diff utility with configurable tolerance. Produces diff images highlighting mismatches. Supports both exact comparison and perceptual comparison (accounting for anti-aliasing differences across GPU vendors).

- **DSL fixtures** — sample directive source files covering all view types, patterns, and edge cases. Used by `ohmu-dsl` parser and compiler tests.

- **State mutation sequences** — pre-defined sequences of mutations for testing reconciliation behavior (toggle N times, add/remove list items, navigate between screens).

- **Event replay** — recorded input event sequences (touch gestures, keyboard input, IME composition) that can be replayed against any platform implementation to verify consistent behavior.

## Dependencies

- `ohmu-core` — scene buffer types (for builders).
- `image` or `png` — reference image loading and comparison.
- No platform or GPU dependencies. Builders produce data structures; rendering is done by consuming crates.

## Testing tiers

This crate *supports* testing in other crates. It has its own minimal tests:

### Tier 1 — Self-tests

- Scene builders produce valid scene buffers (parent/child indices consistent, level values correct).
- TargetProfile presets have reasonable values (viewport > 0, density > 0, etc.).
- Image comparison utility correctly identifies identical images as matching and different images as mismatched.
- DSL fixtures parse without errors.

### Updating references

When a visual change is intentional (new SDF rendering approach, style tweak, layout algorithm fix), reference images need updating:

```bash
# Run visual regression tests with UPDATE flag to overwrite references
OHMU_UPDATE_REFS=1 cargo test -p ohmu-gpu -- visual_regression
```

Updated references are committed alongside the code change. Review the diff visually before committing.
