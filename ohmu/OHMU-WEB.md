# ohmu-web

WASM + WebGPU platform bridge. Compiles the Ohmu runtime to run in web browsers.

## What lives here

- **WASM entry point** — `wasm-bindgen` exports that initialize the Ohmu runtime from JavaScript.
- **Canvas integration** — acquires a WebGPU surface from an HTML `<canvas>` element. Handles resize via `ResizeObserver`.
- **Input translation** — converts DOM pointer/keyboard/touch events to `ohmu-core` event types via `web-sys` bindings. Handles browser-specific quirks (passive touch listeners, pointer lock, wheel event normalization).
- **IME** — delegates to the browser's built-in text input. Uses a hidden `<input>` or `<textarea>` for composition events, positioned at the cursor rect reported by the scene.
- **Clipboard** — `navigator.clipboard` API with fallback to `execCommand` for older browsers.
- **Text shaping** — two options:
  1. Compile `rustybuzz` (pure Rust HarfBuzz port) to WASM — portable, no dependencies.
  2. Delegate to browser's `CanvasRenderingContext2D.measureText` for metrics — simpler, less control.
- **Asset loading** — fetch fonts and images over HTTP, decode in WASM or via browser APIs (`createImageBitmap`).
- **Main loop** — `requestAnimationFrame`-driven frame cycle. Async GPU operations use WASM futures.
- **TargetProfile detection** — reads `window.innerWidth/Height`, `devicePixelRatio`, `matchMedia` for `prefers-reduced-motion` and `prefers-contrast`, touch capability detection.

## Dependencies

- `ohmu-core`, `ohmu-gpu`, `ohmu-dsl`, `ohmu-platform`.
- `wasm-bindgen`, `web-sys`, `js-sys` — browser API bindings.
- `wgpu` — compiled to target WebGPU (browser-native) or WebGL2 fallback.

## Build

```bash
wasm-pack build crates/ohmu-web --target web
# produces pkg/ with .wasm + .js glue
```

Output is a JS module that can be loaded with `<script type="module">` or bundled with any JS bundler.

## Testing tiers

### Tier 1 — Pure logic (inherited)

`ohmu-core` and `ohmu-dsl` can be compiled to `wasm32-unknown-unknown` and their tests run via `wasm-pack test`:

```bash
wasm-pack test --headless --chrome -- -p ohmu-core
wasm-pack test --headless --chrome -- -p ohmu-dsl
```

This verifies that the pure-logic crates work correctly when compiled to WASM (catches issues with WASM-incompatible operations, memory layout differences, missing `std` features).

### Tier 2 — WebGPU headless

Run GPU tests in a headless Chrome instance with WebGPU enabled:

```bash
wasm-pack test --headless --chrome -- -p ohmu-gpu --features web
```

Chrome's headless mode supports WebGPU (with `--enable-features=Vulkan,UseSkiaRenderer`). This gives real GPU execution in CI without a display server. Same test pattern as native Tier 2: compute shader output compared against CPU reference, visual regression against reference images.

This is the lowest-friction way to get GPU integration testing in CI — no special hardware, just a Chrome binary.

### Tier 3 — Real browser

Tests that exercise the browser-specific integration: DOM event handling, IME, clipboard, canvas resize, asset loading.

**Automated:**
- Playwright or `wasm-bindgen-test` with a real browser instance.
- Navigate to a test page hosting the WASM module.
- Simulate user interactions (click, type, resize).
- Assert on rendered canvas content (screenshot comparison) and DOM state.

**Manual/visual:**
- Open test page in multiple browsers (Chrome, Firefox, Safari).
- Verify input handling, text rendering, scroll behavior, IME.
- Mobile browser testing via device emulation or real devices.

**Browser compatibility matrix:**
- Chrome (WebGPU native).
- Firefox (WebGPU behind flag, WebGL2 fallback).
- Safari (WebGPU partial, WebGL2 fallback).
- Mobile Chrome/Safari.
