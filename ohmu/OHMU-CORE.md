# ohmu-core

Scene buffer, state management, layout algorithm, and event system. The foundation crate with zero platform dependencies — compiles to every target.

## What lives here

- **Scene buffer** — flat typed array of node structs (`Node { parent_idx, first_child, child_count, level, size_policy, distribution, spacing, padding, align, intrinsic, rect, ... }`). The single data structure the entire system operates on.
- **Layout algorithm (CPU reference)** — Rust implementation of the two-pass layout solver (bottom-up intrinsic sizing, top-down allocation). This is the *specification* of what the GPU compute shaders must produce. All layout correctness is defined here.
- **Layout primitives** — `SizePolicy` (Fixed/Fill/Fit), `Distribution` (StackX/StackY/Layer/Grid/Flow), alignment, spacing, padding.
- **State store** — flat `Vec<Value>` indexed by `StateId`, dirty bitset, mutation types (Set/Toggle/ListPush/ListRemove/ListReorder/Increment).
- **Bindings** — Property, Conditional, List, and Derived bindings that connect state slots to scene buffer nodes.
- **Reconciliation** — walks dirty state, patches affected nodes, marks layout dirty.
- **Event types** — Tap, DragStart/Move/End, LongPress, Scroll, Pinch, KeyDown, Character, Focus, Hover.
- **Gesture arena** — resolves ambiguous input (tap vs scroll start) by observing pointer movement against candidate recognizers.
- **Hit testing (CPU path)** — rect scan over layout output for pointer-to-node resolution.
- **Focus management** — focus order traversal, keyboard routing.
- **TargetProfile** — the struct describing viewport, input capabilities, form factor, platform conventions, accessibility settings. Used by the DSL compiler but defined here since layout and event behavior depend on it.

## Dependencies

None beyond `std`. This crate must stay dependency-free to guarantee it compiles everywhere (native, WASM, embedded).

## Testing tiers

### Tier 1 — Pure logic (primary)

This is where the bulk of TDD happens. Everything in this crate is testable with plain `cargo test` — no GPU, no windowing, no platform APIs.

**Layout algorithm:**
- Property-based tests over random tree shapes, size policies, and distributions (using `proptest` or `bolero`).
- Known-answer tests: specific layouts (sidebar+main, header/body/footer, nested grids) with exact expected rects.
- Edge cases: zero-size nodes, deeply nested trees (15+ levels), single-child chains, empty containers.
- The CPU reference impl is the source of truth. Every layout test is a contract the GPU shaders must also satisfy.

**State store + reconciliation:**
- Mutation application and dirty tracking.
- Binding evaluation: property bindings update nodes, conditional bindings switch subtrees, list bindings sync instances.
- Derived bindings with multiple inputs.
- Dirty clear after reconciliation.

**Gesture arena:**
- State machine transitions for tap, scroll, long press, drag.
- Timeout behavior (long press threshold, tap timeout).
- Multi-candidate resolution (scroll wins over tap when movement exceeds threshold).
- Pointer capture during drag.

**Hit testing:**
- Rect scan returns topmost node at point.
- Hit padding expands touch targets.
- Opaque nodes block hits to nodes behind.
- Scroll offset translation.

**Event routing:**
- Events dispatch to correct handler based on hit test + focus state.
- Keyboard events route to focused node.
- Tab traversal follows focus order.

### Tier 2 — GPU verification (secondary)

Not directly tested here, but this crate provides the **reference values** that `ohmu-gpu` tests compare against. The pattern:

1. Build a scene buffer in `ohmu-core`.
2. Run CPU layout → expected rects.
3. Pass same scene buffer to GPU compute shaders in `ohmu-gpu`.
4. Compare GPU output against CPU reference.

### Tier 3 — N/A

No platform integration needed. This crate never touches a window, display, or OS API.
