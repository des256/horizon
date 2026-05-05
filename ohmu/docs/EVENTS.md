# Event & Interaction Model

## Hit Testing

**Primary approach:** Hit buffer (ID buffer) rendered as second attachment. Fragment shader writes `node_id` as uint. On pointer event, read pixel at `(x, y)` → topmost node directly.

**Advantages:**
- O(1) hit test, perfectly shape-accurate
- Z-order correct automatically
- Handles clip/scroll/transforms

**Readback latency solutions:**

| Approach | Tradeoff |
|----------|----------|
| Async readback (1-2 frame delay) | Fine for touch (human latency >> 16ms) |
| Compute shader point test | Test touch point against all rects in one dispatch, read one u32 |
| CPU rect scan | Copy rects to CPU after layout. Linear scan of hundreds is nanoseconds. |

Practical recommendation: CPU rect scan for normal frames + GPU hit buffer for overlapping/complex shapes.

## Gesture Arena

User touches a button inside a scroll container — tap or scroll start? Can't know at `pointer_down`, must observe movement.

```rust
enum GestureState {
    Undecided { candidates: Vec<GestureRecognizer> },
    Claimed { winner: GestureRecognizer },
}

// pointer_down: ancestors that want gestures enter arena
// pointer_move: candidates that can't match drop out
// threshold crossed: winner claims, losers get cancel
// pointer_up with no movement: tap wins
```

**Gesture types:**
- Tap: down + up within N ms, movement < threshold
- Scroll: down + movement > threshold in one axis
- Long press: down + held > N ms without movement
- Pinch/rotate: two pointers, distance/angle change

## Pointer Capture

During drag, events route to drag source even when pointer leaves bounds:

```rust
enum Routing {
    HitTest,                    // normal: topmost node at pointer position
    Captured { node_id: u32 },  // locked: all events go here until release
}
```

## Focus and Keyboard Routing

```rust
struct FocusState {
    focused: Option<u32>,       // current focused node_id
    focus_order: Vec<u32>,      // tab traversal order (tree order or explicit)
}
```

Keyboard events → focused node's handler. Tab → advance. Shift+Tab → reverse.

## Multi-Touch

```rust
struct PointerState {
    id: u32,                    // OS-assigned pointer ID
    position: Vec2,
    node_at: u32,               // from hit test
    captured_by: Option<u32>,
    gesture_arena: GestureState,
}
```

System maintains `Vec<PointerState>` for all active pointers. Pinch/rotate observe two simultaneously.

## Event Types

```rust
enum Event {
    Tap { position: Vec2 },
    DragStart { position: Vec2 },
    DragMove { delta: Vec2, position: Vec2 },
    DragEnd { velocity: Vec2 },
    LongPress { position: Vec2 },
    Scroll { delta: Vec2 },
    Pinch { scale: f32, center: Vec2 },
    KeyDown { key: Key, modifiers: Modifiers },
    Character { ch: char },
    FocusIn,
    FocusOut,
    HoverEnter,
    HoverLeave,
}
```

## What's Eliminated vs. Web/DOM

| Web/DOM | This system | Why |
|---------|-------------|-----|
| Event bubbling (capture → target → bubble) | Direct dispatch to handler | AI knows which node handles what |
| `stopPropagation()` / `preventDefault()` | Gesture arena resolves conflicts | Structural, not imperative escape hatches |
| `addEventListener` / `removeEventListener` | Static handler_id in scene buffer | Declared at creation |
| Touch → mouse compatibility | Pointer events only | One unified model |
| `pointer-events: none` CSS | `opaque: false` + no gestures | Explicit, not inherited |
