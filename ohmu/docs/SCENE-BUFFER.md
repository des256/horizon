# Scene Buffer

The scene buffer is the central data structure of the entire system. It's a flat typed array of node structs — no tree pointers, no component hierarchy. Parent-child relationships are encoded via indices. The GPU reads this buffer directly.

## Node Struct

```rust
struct Node {
    // Tree structure
    parent_idx: u32,
    first_child: u32,
    child_count: u32,
    level: u32,             // depth in tree (root = 0)

    // Layout input
    size_policy_x: SizePolicy,
    size_policy_y: SizePolicy,
    distribution: Distribution,
    spacing: f32,
    padding: [f32; 4],     // top, right, bottom, left
    align_x: f32,           // 0.0 = start, 0.5 = center, 1.0 = end
    align_y: f32,

    // Layout output (computed by GPU)
    intrinsic: Vec2,        // output of pass 1 (bottom-up)
    rect: Vec4,             // output of pass 2 (top-down): x, y, w, h
}
```

### WGSL Representation

```wgsl
struct Node {
    parent_idx: u32,
    first_child: u32,
    child_count: u32,
    level: u32,
    size_policy_x: u32,    // encoded SizePolicy
    size_policy_y: u32,
    distribution: u32,
    spacing: f32,
    padding: vec4<f32>,
    align: vec2<f32>,
    intrinsic: vec2<f32>,   // output of pass 1
    rect: vec4<f32>,        // output of pass 2: x, y, w, h
}
```

## Layout Primitives

```rust
enum SizePolicy {
    Fixed(f32),                     // exactly this many px
    Fill(f32),                      // ratio of remaining space (like flex: N)
    Fit { min: f32, max: f32 },     // shrink to content, clamped
}

enum Distribution {
    StackX,                         // children laid out horizontally
    StackY,                         // children laid out vertically
    Layer,                          // children overlaid (z-stack)
    Grid { cols: u32, rows: u32 },  // fixed grid cells
    Flow { break_indices: BufferRange }, // CPU-computed line breaks

    // Game/visualization extensions
    Radial { radius: f32, start_angle: f32, sweep: f32 },
    Polar { rings: u32, segments: u32 },
    Graph { positions: BufferRange },   // explicit or force-directed
    Hex { cols: u32, rows: u32 },
}

struct LayoutInput {
    size_x: SizePolicy,
    size_y: SizePolicy,
    distribution: Distribution,
    spacing: f32,
    padding: [f32; 4],
    align_x: f32,
    align_y: f32,
}
```

### Common Layout Mappings

| Pattern | Mapping |
|---------|---------|
| Vertical list | `StackY`, children `Fill(1.0)` or `Fit` |
| Horizontal toolbar | `StackX`, items `Fixed` or `Fill` |
| Centered content | `Layer`, child with `align: 0.5, 0.5` |
| Sidebar + main | `StackX`, sidebar `Fixed(250)`, main `Fill(1.0)` |
| Card grid | `Grid { cols: 3, rows: N }`, cells `Fill(1.0)` |
| Header / body / footer | `StackY`, header `Fit`, body `Fill(1.0)`, footer `Fit` |
| Overlay / modal | `Layer`, modal child on top with alignment |
| Weapon wheel | `Radial { radius, start_angle: 0, sweep: TAU }` |

## Event Configuration

```rust
struct EventConfig {
    gestures: GestureSet,       // bitflags: TAP | DRAG | LONG_PRESS | SCROLL_X | SCROLL_Y | PINCH
    hit_padding: [f32; 4],      // expand hit area beyond visual bounds (44px mobile targets)
    cursor: CursorShape,        // pointer, text, grab, etc.
    focusable: bool,
    opaque: bool,               // stops hit test from reaching nodes behind
}
```

## Scroll Configuration

```rust
struct ScrollConfig {
    content_size: Vec2,         // total scrollable area
    offset: Vec2,               // current scroll position (mutable)
    axes: ScrollAxes,           // X | Y | BOTH
    inertia: bool,
    overscroll: OverscrollBehavior,
}
```

Layout shader uses `offset` to position children. Hit test translates coordinates into content space. Scroll physics (inertia, spring overscroll) run on GPU as per-frame differential equation updates.

## World-Space Anchoring (Game UI)

```rust
struct WorldAnchor {
    world_position: Vec3,               // 3D point to track
    offset: Vec2,                       // screen-space offset from projected point
    billboard: bool,                    // always face camera?
    distance_fade: Option<(f32, f32)>,  // (start_fade, fully_hidden) distances
    occlusion: bool,                    // hidden when behind geometry?
    scale_with_distance: bool,          // smaller when far?
}
```

Each frame: project through view-projection matrix → screen `(x, y, depth)`. Apply distance fade/scale. Write to node position. One matrix multiply per anchored element.

## Custom Shaders (Game UI)

```rust
enum NodeShader {
    Default,                                        // standard SDF quad
    Hologram { scan_speed: f32, noise: f32 },
    Glitch { intensity: f32, slice_count: u32 },
    Dissolve { progress: f32, edge_color: Color },
    Energy { flow_speed: f32, color: Color },
    Frost { coverage: f32, crystal_scale: f32 },
    Custom { shader_id: u32 },                      // user-registered
}
```

Instanced quad renderer switches variant per-instance via material ID. Each variant has same inputs (rect, UV, time) but different fragment processing.

## Particle Emitters

```rust
struct ParticleEmitter {
    attached_to: u32,           // node_id
    rate: f32,                  // particles per second
    lifetime: (f32, f32),       // min, max seconds
    velocity: ParticleVelocity, // direction + spread + speed
    size: (f32, f32),           // start, end
    color: (Color, Color),      // start, end (lerp)
    sprite: Option<u32>,        // atlas index, or none for dot
    gravity: Vec2,
    target: Option<u32>,        // node_id to fly toward (for toast_particle)
    on_arrive: Option<Mutation>,// trigger when particle reaches target
}
```

Particles live in a separate buffer, updated by a compute pass (physics), rendered as instanced points/quads.

## Audio Triggers

```rust
struct AudioConfig {
    on_tap: Option<SoundId>,
    on_hover: Option<SoundId>,
    on_focus: Option<SoundId>,
    on_state_change: Option<SoundId>,
    ambient: Option<SoundId>,   // loops while visible
}
```

Runtime sends audio events to platform audio system. Declared in scene format, not GPU-rendered.

## Animation

```rust
enum Animation {
    // Triggered by state change
    Transition { from: f32, to: f32, duration: f32, easing: Easing },

    // Runs continuously while node is visible
    Loop { property: NodeProp, keyframes: Vec<Keyframe>, duration: f32 },
    Pulse { property: NodeProp, amplitude: f32, frequency: f32 },
    Wobble { property: NodeProp, amplitude: f32, damping: f32 },

    // Driven by external value
    Driven { property: NodeProp, source: StateId, mapping: fn(f32) -> f32 },

    // Procedural (shader-time-based, zero CPU cost)
    Procedural { shader: NodeShader },
}
```

GPU handles `Loop`, `Pulse`, `Wobble`, and `Procedural` entirely — time-based computations in the animation compute pass. No CPU per-frame cost.
