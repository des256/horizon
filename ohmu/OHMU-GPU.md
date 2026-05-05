# ohmu-gpu

Compute shader layout, SDF rendering, and GPU buffer management. Depends on `ohmu-core` for data structures and `wgpu` for the GPU abstraction.

## What lives here

- **Compute shader layout** — WGSL shaders that implement the same two-pass layout algorithm as the CPU reference in `ohmu-core`. Bottom-up intrinsic sizing dispatched per tree level (max_depth → 0), top-down allocation dispatched per level (0 → max_depth). Workgroup size 64.
- **SDF rendering** — instanced quad rendering with signed distance field evaluation for rounded rects, borders, shadows. Fragment shader reads style properties per node.
- **Text rendering** — SDF atlas lookup, instanced glyph quads from pre-shaped glyph buffers (shaping itself is CPU-side via HarfBuzz in `ohmu-platform`).
- **Hit buffer** — optional second render attachment where fragment shader writes `node_id` as uint. Used for GPU-accelerated hit testing of complex/overlapping shapes.
- **Buffer management** — upload scene buffer to GPU storage buffers, manage atlas textures, handle buffer resizing when node count changes.
- **GPU animation** — spring/transition interpolation in compute shaders for scroll physics, layout transitions, opacity/transform animations. No CPU roundtrip per frame.
- **Pipeline management** — compute and render pipeline creation, bind group layouts, shader module compilation.

## Dependencies

- `ohmu-core` — scene buffer types, layout primitives, node struct layout.
- `wgpu` — GPU abstraction over Vulkan, Metal, DX12, WebGPU.
- `naga` (via wgpu) — WGSL shader compilation/validation.

## WGSL shaders

Stored in `shaders/` at the workspace root. Key files:

- `layout_intrinsic.wgsl` — bottom-up pass, computes intrinsic size per node.
- `layout_allocate.wgsl` — top-down pass, assigns rect per node from parent allocation.
- `render_sdf.wgsl` — vertex + fragment shader for SDF quad rendering.
- `render_text.wgsl` — instanced glyph rendering from atlas.
- `hit_buffer.wgsl` — fragment shader writing node IDs.
- `animation.wgsl` — per-frame spring/transition updates.

## Testing tiers

### Tier 1 — Shader validation only

WGSL source files can be parsed and validated by `naga` without a GPU. Tests here catch syntax errors and type mismatches in shader code at `cargo test` time:

- Parse every `.wgsl` file with `naga::front::wgsl::parse_str`.
- Validate module with `naga::valid::Validator`.
- Check that bind group layouts match the Rust-side pipeline expectations.

### Tier 2 — GPU correctness (primary)

This is the main testing tier for this crate. Uses `wgpu` with a software adapter (Mesa llvmpipe) or headless backend — no window, no display needed.

**Compute shader verification:**
1. Build a scene buffer in `ohmu-core`.
2. Run CPU reference layout → expected rects.
3. Upload scene buffer to GPU storage buffer.
4. Dispatch compute shaders (intrinsic pass, then allocation pass, per level).
5. Read back result buffer.
6. Compare GPU rects against CPU reference within floating-point tolerance.

Test cases mirror `ohmu-core` layout tests: same tree structures, same size policies, same expected results. Any CPU/GPU mismatch is a bug in the shader.

**Visual regression:**
1. Build scene buffer with known content.
2. Run layout (GPU compute).
3. Render to offscreen texture (SDF pass + text pass).
4. Read back pixels, save as PNG.
5. Compare against blessed reference image with per-pixel tolerance.

Reference images live in `ohmu-test/references/` organized by test name. Any pixel diff beyond threshold fails the test and produces a visual diff image for inspection.

**Specific areas:**
- SDF rounding: corner radius at various sizes, border rendering, shadow blur.
- Text atlas: glyph positioning matches shaping output, atlas UV coordinates correct.
- Hit buffer: rendered node IDs match expected topmost node at sample points.
- Animation: spring solver converges to target within tolerance after N frames.

**CI setup:** Mesa llvmpipe provides a software Vulkan/GL implementation. `wgpu` selects it when no hardware GPU is available. Tests run in CI without GPU hardware, though slower than on real GPUs.

### Tier 3 — Real GPU validation

Run the same Tier 2 tests on real GPU hardware (different vendors: Intel, AMD, NVIDIA, Apple Silicon, Adreno/Mali) to catch driver-specific bugs. These are nightly/weekly CI jobs, not per-commit.

Differences to watch for:
- Floating-point precision across GPU vendors.
- Workgroup size limits on mobile GPUs.
- Texture format support variations.
- Buffer alignment requirements.
