# vibe-ui

A UI toolkit designed for AI generation, built on GPU primitives rather than human-oriented widget abstractions.

## Core Thesis

Current UI frameworks (React, Flutter, SwiftUI, etc.) are compression schemes for human cognition — components, props, lifecycle hooks, the cascade in CSS — all exist because humans can't hold the full pixel-level state in working memory. AI doesn't have that limitation. When told to generate UI directly against rendering primitives, AI is more effective because there's no translation layer it doesn't need.

**Question:** Can we design a system that goes back to basic structure (GPU graphics on a touchscreen with events) and is much easier on AI, allowing a cleaner and more expressive way to encode user needs?

## Architecture Overview

```
┌─────────────────────────────────────────┐
│  AI / User intent                        │
│  "show a list of items, tappable,        │
│   with a search filter at top"           │
└──────────────┬───────────────────────────┘
               │ emits
┌──────────────▼───────────────────────────┐
│  Scene buffer (typed array of structs)   │
│  [{id, parent, order, constraints,       │
│    role, style, handler_id, text_ref}]   │
└──────────────┬───────────────────────────┘
               │ uploaded to GPU
┌──────────────▼───────────────────────────┐
│  Compute: layout solver (2 dispatches)   │
│  Output: position + size per node        │
└──────────────┬───────────────────────────┘
               │
┌──────────────▼───────────────────────────┐
│  Render: instanced SDF quads + text      │
└──────────────┬───────────────────────────┘
               │
┌──────────────▼───────────────────────────┐
│  Events: hit-test (GPU or CPU) →         │
│  handler lookup → state mutation →       │
│  dirty flag → next frame                 │
└──────────────────────────────────────────┘
```

---

## 1. GPU Layout

### Constraint Model

The layout system uses compute shaders with a two-pass model:

- **Pass 1 (bottom-up, leaf → root):** Each node computes intrinsic size. Leaves report fixed size or pre-measured text bounds. Parents aggregate children based on distribution axis (sum for stacks, max for layers). Parallel per tree level.
- **Pass 2 (top-down, root → leaf):** Each node receives allocated rect from parent and distributes to children. Root gets viewport. Parallel per tree level.

Two dispatches per level of depth. Typical UI is 8-15 levels → 16-30 dispatches per frame.

### Layout Primitives

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
}

struct LayoutInput {
    size_x: SizePolicy,
    size_y: SizePolicy,
    distribution: Distribution,
    spacing: f32,
    padding: [f32; 4],
    align_x: f32,  // 0.0 = start, 0.5 = center, 1.0 = end
    align_y: f32,
}
```

### What This Covers

| Pattern | Mapping |
|---------|---------|
| Vertical list | `StackY`, children `Fill(1.0)` or `Fit` |
| Horizontal toolbar | `StackX`, items `Fixed` or `Fill` |
| Centered content | `Layer`, child with `align: 0.5, 0.5` |
| Sidebar + main | `StackX`, sidebar `Fixed(250)`, main `Fill(1.0)` |
| Card grid | `Grid { cols: 3, rows: N }`, cells `Fill(1.0)` |
| Header / body / footer | `StackY`, header `Fit`, body `Fill(1.0)`, footer `Fit` |
| Overlay / modal | `Layer`, modal child on top with alignment |

### GPU-Friendly vs. Needs CPU Assist

**Fully GPU-parallel:**
- Fixed/fill/fit sizing (two-pass resolves completely)
- Stack/layer/grid distribution
- Alignment, padding, spacing

**Needs CPU pre-computation:**
- **Wrapping / flow layout:** CPU determines line break indices, GPU then treats each line as a fixed StackX
- **Content-dependent cross-axis sizing:** CPU resolves the dominant axis width, then GPU handles the rest
- **Table auto-columns:** CPU scans column contents for widths, writes fixed column sizes into grid node

**Not supported (by design):**
- Arbitrary cross-tree constraints ("A same width as B in different subtree")
- CSS-style `position: relative` with skip-level containing blocks
- Intrinsic size → wrapping → size feedback loops
- Margin collapse, `float`, specificity cascades

### Compute Shader Structure

```wgsl
struct Node {
    parent_idx: u32,
    first_child: u32,
    child_count: u32,
    level: u32,
    size_policy_x: u32,  // encoded SizePolicy
    size_policy_y: u32,
    distribution: u32,
    spacing: f32,
    padding: vec4<f32>,
    align: vec2<f32>,
    intrinsic: vec2<f32>,  // output of pass 1
    rect: vec4<f32>,       // output of pass 2: x, y, w, h
}

@compute @workgroup_size(64)
fn compute_intrinsic(@builtin(global_invocation_id) id: vec3<u32>) {
    let idx = id.x;
    if nodes[idx].level != params.current_level { return; }
    if nodes[idx].child_count == 0u { return; }

    var total = vec2(0.0);
    for (var i = 0u; i < nodes[idx].child_count; i++) {
        let child = nodes[nodes[idx].first_child + i];
        if nodes[idx].distribution == STACK_X {
            total.x += child.intrinsic.x + nodes[idx].spacing;
            total.y = max(total.y, child.intrinsic.y);
        }
        // ... other distributions
    }
    nodes[idx].intrinsic = total + padding_sum(nodes[idx]);
}

@compute @workgroup_size(64)
fn compute_allocation(@builtin(global_invocation_id) id: vec3<u32>) {
    let idx = id.x;
    if nodes[idx].level != params.current_level { return; }

    let parent = nodes[nodes[idx].parent_idx];
    // Compute this node's rect from parent's allocation,
    // sibling order, distribution, spacing, alignment
    nodes[idx].rect = computed_rect;
}
```

Host dispatches: loop `max_depth→0` for bottom-up, then `0→max_depth` for top-down.

### Why GPU Layout?

For typical UI node counts (200-2000), CPU layout is already sub-millisecond. The GPU advantage is architectural:

1. **No CPU→GPU position upload.** Position data is born on GPU, stays on GPU.
2. **GPU-driven animation.** Springs, transitions, interpolation in compute shaders — no CPU roundtrip per frame.
3. **Simplicity.** One buffer, one pipeline. No split ownership.
4. **Scaling.** For data visualization with 10K+ nodes, parallelism becomes genuine.

---

## 2. Text Pipeline

### The Problem

SDF rendering of glyphs is straightforward. The hard parts are everything *before* the glyph reaches the renderer and everything around *editing*.

```
codepoints → segmentation → shaping → layout → rendering
                                                    ↑ easy (SDF)
←────────── all of this is the hard part ──────────→
```

### Text as Enum

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

### CPU Text Pipeline (Libraries)

| Stage | Library | What it does |
|-------|---------|--------------|
| Line breaking | ICU BreakIterator or UAX #14 | Determines legal break opportunities |
| BiDi | SheenBidi or ICU | Reorders mixed LTR/RTL text for visual display |
| Shaping | HarfBuzz | Converts codepoints → positioned glyph IDs (handles Arabic forms, Devanagari ligatures, Latin kerning) |
| Grapheme clusters | UAX #29 | Determines cursor movement boundaries |

All are CPU-side, well-solved as linkable C libraries. Feed results to GPU.

### IME (Input Method Editor) — Platform Bridge

IME is the one area requiring bidirectional communication with the OS. The platform owns the input method; the app is a client.

**Required protocol per platform:**

| Obligation | What it means |
|-----------|---------------|
| Report cursor rect | OS positions candidate popup near insertion point |
| Accept preedit string | Display uncommitted composition (underlined, highlighted segments) |
| Handle commit | Replace preedit with final text |
| Report surrounding text | OS uses for prediction, reconversion |
| Support reconversion | User selects committed text, re-enters composition |

**Platform APIs:**

| Platform | API |
|----------|-----|
| Windows | TSF (Text Services Framework) |
| macOS | NSTextInputClient |
| Linux | IBus / Fcitx / text-input-v3 (Wayland) |
| Android | InputConnection |
| iOS | UITextInput |

**Design decision:** Text input is the one place where we accept a platform primitive rather than reimplementing. We own the *rendering* (SDF) but delegate the *input protocol* to a thin platform bridge (~5-8 methods per platform).

### Text Architecture

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

---

## 3. Event / Interaction Model

### Hit Testing

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

### Gesture Arena

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

### Node Event Configuration

```rust
struct EventConfig {
    gestures: GestureSet,       // bitflags: TAP | DRAG | LONG_PRESS | SCROLL_X | SCROLL_Y | PINCH
    hit_padding: [f32; 4],      // expand hit area beyond visual bounds (for 44px mobile targets)
    cursor: CursorShape,        // pointer, text, grab, etc.
    focusable: bool,
    opaque: bool,               // stops hit test from reaching nodes behind
}
```

### Pointer Capture

During drag, events route to drag source even when pointer leaves bounds:

```rust
enum Routing {
    HitTest,                    // normal: topmost node at pointer position
    Captured { node_id: u32 },  // locked: all events go here until release
}
```

### Scroll Containers

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

### Focus and Keyboard Routing

```rust
struct FocusState {
    focused: Option<u32>,       // current focused node_id
    focus_order: Vec<u32>,      // tab traversal order (tree order or explicit)
}
```

Keyboard events → focused node's handler. Tab → advance. Shift+Tab → reverse.

### Multi-Touch

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

### Event Types Delivered to Handlers

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

### What's Eliminated vs. Web/DOM

| Web/DOM | This system | Why |
|---------|-------------|-----|
| Event bubbling (capture → target → bubble) | Direct dispatch to handler | AI knows which node handles what |
| `stopPropagation()` / `preventDefault()` | Gesture arena resolves conflicts | Structural, not imperative escape hatches |
| `addEventListener` / `removeEventListener` | Static handler_id in scene buffer | Declared at creation |
| Touch → mouse compatibility | Pointer events only | One unified model |
| `pointer-events: none` CSS | `opaque: false` + no gestures | Explicit, not inherited |

---

## 4. State Model

### Design

State is a flat typed store. The scene buffer has binding slots that reference state. When state changes, a small CPU reconciler patches affected nodes.

```rust
type StateId = u32;

enum Value {
    Bool(bool),
    Float(f64),
    Text(String),
    List(Vec<StateId>),
    Enum { variant: u32, data: Vec<StateId> },
}

struct StateStore {
    slots: Vec<Value>,      // indexed by StateId
    dirty: BitVec,          // which slots changed this frame
}
```

### Mutations (what handlers produce)

```rust
enum Mutation {
    Set(StateId, Value),
    Toggle(StateId),
    ListPush(StateId, Value),
    ListRemove(StateId, usize),
    ListReorder(StateId, Vec<usize>),
    Increment(StateId, f64),
}
```

### Bindings (how state reaches the scene)

```rust
enum Binding {
    // Direct: node property = state value
    Property { node: u32, prop: NodeProp, state: StateId },

    // Conditional: which subtree is active
    Conditional { slot: SubtreeSlot, state: StateId, branches: Vec<SubtreeRange> },

    // List: repeat template per item
    List { slot: SubtreeSlot, state: StateId, template: SubtreeTemplate },

    // Derived: property = f(state_a, state_b, ...)
    Derived { node: u32, prop: NodeProp, expr: Expr },
}
```

### Reconciliation Loop

```rust
fn reconcile(state: &StateStore, scene: &mut SceneBuffer, bindings: &[Binding]) {
    if state.dirty.none() { return; }

    for binding in bindings {
        match binding {
            Property { node, prop, state_id } => {
                if state.dirty[*state_id] {
                    scene.set(*node, *prop, state.get(*state_id));
                }
            }
            Conditional { slot, state_id, branches } => {
                if state.dirty[*state_id] {
                    let variant = state.get_variant(*state_id);
                    slot.activate_branch(scene, branches[variant]);
                }
            }
            List { slot, state_id, template } => {
                if state.dirty[*state_id] {
                    slot.sync_instances(scene, state.get_list(*state_id), template);
                }
            }
            Derived { node, prop, expr } => {
                if expr.any_input_dirty(&state.dirty) {
                    scene.set(*node, *prop, expr.evaluate(state));
                }
            }
        }
    }

    state.dirty.clear();
    scene.mark_layout_dirty();
}
```

### Common Scenarios

#### Toggle (show/hide sidebar)

```
State: sidebar_open: bool = true
Handler: on_tap(toggle_btn) → Toggle(sidebar_open)
Binding: Property { node: sidebar, prop: Visible, state: sidebar_open }
Transition: width animates 250 → 0 over 200ms
```

#### Todo list

```
State:
  items: list of { text: Text, done: Bool }

Scene:
  list_container: List binding over items
    template: stack_x
      checkbox: bind(checked → item.done), tap → Toggle(item.done)
      label: bind(text → item.text), bind(strikethrough → item.done)
      delete_btn: tap → ListRemove(items, index)

  add_btn: tap → ListPush(items, {text: new_text, done: false})
```

#### Form with validation

```
State:
  email: Text = ""
  email_error: Enum(Valid | Error(msg))
  submittable: Bool = false

Bindings:
  Property { email_input.value ← email }
  Conditional { error_region ← email_error: [nothing, error_label] }
  Derived { submit_btn.opacity ← if submittable then 1.0 else 0.4 }

Handler:
  on_change(email_input, value) → [
    Set(email, value),
    Set(email_error, validate(value)),
    Set(submittable, is_valid(value)),
  ]
```

#### Navigation

```
State: current_page: Enum(Home | Profile | Settings)

Binding: Conditional {
    page_container ← current_page: [home_subtree, profile_subtree, settings_subtree]
}

Handler: on_tap(nav_profile) → Set(current_page, Profile)
```

#### Async loading

```
State: data: Enum(Loading | Loaded(list) | Error(msg))

Binding: Conditional {
    content ← data: [spinner, data_list, error_display]
}

Handlers:
  on_mount → [Set(data, Loading), AsyncFetch(url, on_ok, on_err)]
  on_ok(response) → Set(data, Loaded(response.items))
  on_err(e) → Set(data, Error(e.message))
```

#### Drag to reorder

```
State:
  items: list
  drag_offset_y: Float = 0.0
  drag_state: Enum(Idle | Dragging(index))

Handler:
  on_drag_start(item_N) → Set(drag_state, Dragging(N))
  on_drag_move(delta_y) → Increment(drag_offset_y, delta_y)
  on_drag_end → [ListReorder(items, new_order), Set(drag_state, Idle)]
```

### What's Eliminated

| Framework concept | Why it existed | Gone because |
|-------------------|---------------|--------------|
| `useState` / `setState` | Track component state ownership | Global flat store, not component-scoped |
| `useEffect` / lifecycle | Side effects on mount/change | Handlers fire on events. Async is explicit. |
| `useMemo` / `React.memo` | Avoid recomputing during re-renders | No re-renders. Bindings are direct writes. |
| Context / prop drilling | Pass state through component tree | Any binding references any slot directly |
| Immutability / reducers | Enable diff detection | Dirty bits on slots. Mutation is fine. |
| Component boundaries | Human code organization | AI doesn't need them. Flat scene. |
| Keys for list reconciliation | Diff algorithm needs identity | List items have state slot IDs — identity is intrinsic |

---

## 5. Minimum Viable Human Semantics

### Accessibility

| Required | Why | Not required |
|----------|-----|--------------|
| Semantic role (actionable / readable / inputtable / navigable / decorative) | Assistive tech needs to know what a region does | Specific widget taxonomy (button vs link vs menuitem) |
| Focus order (linear sequence) | Keyboard/switch users need traversal | Complex focus groups, roving tabindex |
| Text alternative (per non-text node) | Screen readers | aria-describedby chains |
| State flags (disabled, expanded, selected, busy) | Communicates dynamic state | 50+ ARIA states |

### Platform Expectations

| Needed | Delegated to |
|--------|-------------|
| Scroll with inertia | GPU physics (spring/deceleration in compute shader) |
| Text input + IME | Platform bridge (5-8 methods per OS) |
| Clipboard | Platform bridge |
| Cursor shapes | Platform API call |

### Hard Boundary

Text input and IME are where "rawdog GPU" hits a wall. Every OS has a deeply integrated text services framework. Design decision: treat text input as a platform primitive you embed, not something you render yourself. Everything else is GPU-native.

---

## 6. What's Eliminated Globally

### Serialization / Ceremony

| Eliminated | Why it existed | Gone because |
|------------|---------------|--------------|
| HTML/CSS parsing | Human authoring format → runtime | No authoring format. Scene IS the runtime. |
| Style cascade + specificity | Humans need inheritance to avoid repetition | AI emits explicit style per node. Repetition is free for machines. |
| Virtual DOM diffing | Detect minimal changes to expensive DOM | GPU redraws are cheap. Dirty-flag buffer. |
| Component serialization / hydration | Server → client format mismatch | No server rendering. One format everywhere. |
| Event bubbling/capturing | Humans need delegation patterns | Direct handler map. |
| Build/bundle step | Modules, tree-shaking, transpilation | AI emits target format directly. |
| Layout invalidation tracking | CPU layout is expensive | GPU layout is a dispatch. Recompute if dirty. |
| Prop drilling / context | Humans need data flow abstractions | Flat state buffer. Any node reads any slot. |

### AI's Vocabulary

The AI expresses a complete UI with:

**Layout (6 properties):**
```
node.size = Fixed(px) | Fill(ratio) | Fit(min, max)
node.distribution = StackX | StackY | Layer | Grid(c, r) | Flow
node.spacing = f32
node.padding = [f32; 4]
node.align = (f32, f32)
```

**Style:** fill color, border radius, border, shadow, opacity, text properties.

**Events:** gestures declared as bitflags, handler_id reference.

**State:** flat slots, mutations, bindings.

No cascade, no specificity, no `display`/`position`/`float` interactions, no margin collapse, no `box-sizing`. The AI picks a distribution and sizes. The GPU figures out the geometry.

---

## 7. Directive-Based UI (No Design)

### The Idea

Eliminate UI design entirely. Instead of specifying layout and visual properties, express only *intent* — what the user needs to accomplish, what data exists, what actions are possible. A deterministic compiler (or AI-assisted transformer) maps directives to the scene buffer. No human designs anything.

### How It Integrates

```
┌─────────────────────────────────────────┐
│  Layer 1: Directives (intent DSL)        │
│  "user can search, filter, and tap       │
│   items to view details"                 │
└──────────────┬───────────────────────────┘
               │ compiled by
┌──────────────▼───────────────────────────┐
│  Layer 2: Scene buffer (structure)       │
│  Flat node array with layout/style/      │
│  bindings — as described in §1-4         │
└──────────────┬───────────────────────────┘
               │
┌──────────────▼───────────────────────────┐
│  Layer 3: GPU (presentation)             │
│  Compute layout → SDF render             │
└──────────────────────────────────────────┘
```

The directive layer sits *above* the scene buffer. It doesn't replace it — it's a higher-level input. The scene buffer remains the execution format.

### Compilation Strategies (Layer 1 → Layer 2)

| Strategy | Determinism | Expressiveness | Tradeoff |
|----------|-------------|----------------|----------|
| Template library | Fully deterministic | Limited to known patterns | Predictable, testable, ceiling on novelty |
| Constraint solver (SUPPLE-style) | Deterministic given inputs | High — optimization explores 10^17+ configurations | Expensive, cost function embeds aesthetic assumptions |
| AI-mediated | Non-deterministic | Unlimited — can produce novel layouts | Bad for reproducibility, hard to test |
| Hybrid: deterministic structure + AI refinement | Structure deterministic, style variable | Practical balance | Most likely production approach |

### Existing Research & Systems

#### Classical Model-Based UI (MBUID, 1990s-2010s)

The W3C's CAMELEON Reference Framework defines four abstraction levels:
1. Tasks and Concepts (what users accomplish)
2. Abstract UI (interaction spaces, modality-independent)
3. Concrete UI (platform-specific widgets)
4. Final UI (running code)

Transformations ("reification") move between levels. ConcurTaskTrees (CTT) models hierarchical tasks with temporal operators, automatically derives Abstract UIs.

**Assessment:** Works but rigid. Models are predefined and static. A systematic review of 96 papers / 30 tools found they don't yet fully exploit model-driven benefits — difficulty of use, lack of design flexibility.

References:
- W3C CAMELEON Reference Framework
- W3C Model-Based UI XG Final Report
- ConcurTaskTrees (Paterno, 1997)
- IFML (OMG standard, 2014) — models interaction flow, NOT presentation

#### SUPPLE (Harvard, 2004)

The strongest proof that specification → UI works without design.

- **Input:** Functional specification of WHAT the interface must expose + device constraints + user activity model + cost function
- **Process:** Branch-and-bound optimization with constraint propagation
- **Output:** Complete personalized interface, generated in <1 second for solution spaces up to 10^17

Won IUI 2019 Most Impact Paper Award. Key insight: specify needed functionality, let optimization decide presentation.

References:
- Gajos & Weld, "Automatically Generating User Interfaces" (IUI 2004)
- "Automatically Generating Personalized UIs with Supple" (Artificial Intelligence Journal, 2010)

#### Modern Protocols (2025-2026)

- **Google A2UI** — Open protocol (Apache 2.0). Agent sends declarative JSON component tree; client renders with native components. No executable code, security-first. Framework-agnostic (Lit, Angular, Flutter renderers).
- **MCP Apps** — MCP extension where tools return interactive UI components. Components bubble up *intents* rather than modifying state directly.
- **Open-JSON-UI (OpenAI)** — Standardized generative UI schema for agents.
- **AG-UI** — Transport layer connecting agents to UI, complementary to A2UI.

#### LLM-Driven Systems (2024-2026)

- **Jelly (CHI 2025)** — LLM analyzes prompt → infers goals → derives sub-tasks → generates "Task-Driven Data Model" (object-relational schema + dependency graph) → composes UI from patterns. Models evolve with user needs.
- **Google Generative UI** — LLMs produce custom interactive UIs from any prompt. Preferred over markdown by users, comparable to human experts in 50% of cases. Deployed as "Dynamic View" in Gemini.
- **v0 (Vercel)** — Natural language → React/Tailwind. ~6M developers, ~$42M ARR.
- **PrototypeFlow** — Divide-and-conquer: NL → component-level DSL + theme module. Produces editable SVG prototypes.
- **SpecifyUI (2025)** — Iterative UI design intent through structured specifications + generative AI.
- **AlignUI (2026)** — Injects user preference data into AI UI generation.
- **Adaptive UI via RL (2024)** — Markov decision processes dynamically adjust layouts from user feedback.

#### Broader Discourse

- "The Next Era of Design is Intent-Driven" (UX Collective)
- "From UI (User Interface) to UI (User Intent)" — 2025 UX trend redefining the acronym
- Headless Business Applications — decouple logic from presentation, generate UI on-demand per context
- Zero UI / Screenless Interfaces — voice, gesture, contextual awareness (Gartner: 70% of customer journeys via conversational AI by 2028)
- Spec-Driven Development (Marc Brooker, 2026) — specifications as primary artifact, AI handles implementation

### Pros and Cons

**Pros:**

| Advantage | Detail |
|-----------|--------|
| Eliminates largest time sink | UI design/iteration is where most dev time goes |
| Perfect fit for AI generation | AI reasons well about structure/constraints, poorly about visual aesthetics — play to strength |
| Adaptive by nature | Same directives → different presentations for different devices, users, a11y needs |
| Composable | Directives are data. Merge, override, version trivially. |
| Testable | Assert on directive correctness without rendering. Spec IS the test. |
| Eliminates design drift | No gap between "what was designed" and "what was built" |

**Cons:**

| Problem | Detail |
|---------|--------|
| Usable but not beautiful | Every system hits this wall. Automatic generation → functional, not emotionally resonant. |
| Loss of spatial reasoning | Designers think spatially ("draw the eye", "visual rhythm"). Directives encode logic, not spatial relationships. |
| Ambiguity in specification | "Show a list" — scrollable? paginated? cards or rows? The DSL needs enough vocabulary to resolve without becoming a layout language. |
| Ceiling problem | Works for CRUD/admin/dashboards. Falls apart for marketing pages, games, creative tools, novel visualization. |
| Convention gap | Humans expect decades of patterns (hamburger menus, pull-to-refresh, cards). Must encode somewhere. |

### The Key Differentiator

Every existing system targets conventional rendering backends (DOM, Flutter widgets, native controls). None targets a GPU-first flat scene buffer. If directives compile directly to the scene buffer format (§1-4), the entire widget-toolkit translation layer disappears. Directives → positioned quads. No intermediate framework.

---

## 8. The Directive DSL

### Design Principles

1. **Intent over layout.** Describe WHAT users accomplish, not WHERE things go.
2. **Data-first.** All UI is a projection of typed data. Declare the data, then how it's presented.
3. **Deterministic semantics.** Same directives → same behavior (presentation may vary by device/theme, but behavior is fixed).
4. **Composable.** Small pieces combine. An `app` is a composition of `screen`s, a `screen` is a composition of `view`s.
5. **No ambiguity in behavior.** Every interaction has an explicit mutation. No implicit side effects.
6. **AI-friendly surface area.** Minimal keyword count, uniform structure, no special cases.

### Grammar (Sketch)

```
program     = declaration*
declaration = data_decl | screen_decl | flow_decl | style_decl | effect_decl

data_decl   = "data" IDENT ":" type ("=" expr)?
screen_decl = "screen" IDENT ("(" params ")")? "{" view* "}"
flow_decl   = "flow" IDENT "{" transition* "}"
style_decl  = "style" IDENT "{" property* "}"
effect_decl = "effect" IDENT "(" trigger ")" "{" mutation* "}"

view        = IDENT ":" view_kind properties?
view_kind   = "text" | "input" | "image" | "list" | "action" | "group"
            | "choice" | "toggle" | "indicator" | "slot"

properties  = "(" property ("," property)* ")"
property    = IDENT ":" expr

type        = "bool" | "int" | "float" | "text" | "enum" "{" variants "}"
            | "list" "<" type ">" | "record" "{" fields "}"
            | "option" "<" type ">" | "ref" "<" IDENT ">"

transition  = IDENT "->" IDENT ("when" expr)?

mutation    = "set" | "toggle" | "push" | "remove" | "reorder" | "increment"
            | "fetch" | "navigate" | "dismiss"

trigger     = "on_mount" | "on_change" "(" IDENT ")" | "on_interval" "(" duration ")"
            | "on_visible" | "on_focus"
```

### Core Concepts

#### Data Declarations

All state is declared at the top level. Typed, with optional defaults.

```
data products: list<record {
  id: text,
  name: text,
  price: float,
  image: text,
  in_stock: bool
}>

data cart: list<record { product: ref<products>, quantity: int }>
data search_query: text = ""
data selected_category: option<text> = none
data checkout_state: enum { idle, processing, success, error(text) } = idle
```

#### Screens

A screen is a named scope containing views. It has no inherent layout — the compiler decides arrangement.

```
screen product_list {
  search: input(
    bind: search_query,
    placeholder: "Search products...",
    clear: true
  )

  categories: choice(
    source: distinct(products, .category),
    selected: selected_category,
    style: chips
  )

  results: list(
    source: products |> filter(matches(search_query, .name))
                     |> filter(category_match(selected_category, .category)),
    empty: "No products found",
    item: |product| group(
      image: image(product.image),
      title: text(product.name),
      price: text(format_currency(product.price)),
      stock: indicator(product.in_stock, label: "In Stock"),
      action: action("Add to Cart",
        enabled: product.in_stock,
        on_tap: push(cart, {product: product, quantity: 1})
      )
    )
  )
}
```

#### Views (the vocabulary)

Views are semantic building blocks — they declare *what* something is, not how it looks.

| View | Meaning | Compiler decides |
|------|---------|-----------------|
| `text(expr)` | Display a value | Font size, weight, color, truncation |
| `input(bind, ...)` | Editable text field | Box style, label placement, clear button |
| `image(src)` | Display an image | Sizing, aspect ratio, placeholder |
| `list(source, item)` | Repeating collection | Scroll, virtualization, layout (cards/rows/grid) |
| `action(label, on_tap)` | Something the user can activate | Button shape, haptics, loading state |
| `choice(source, selected)` | Pick from options | Chips, dropdown, radio, segmented control |
| `toggle(bind, label)` | Boolean switch | Checkbox, switch, chip toggle |
| `indicator(expr, ...)` | Display a status | Badge, dot, bar, color |
| `group(...)` | Logical grouping | Card, row, section, expandable |
| `slot(condition, ...)` | Conditional content | Transition, placeholder |

The compiler maps these to scene buffer nodes. `action` becomes a tappable quad with text. `list` becomes a scroll container with templated children. The directive author never specifies layout.

#### Flows (navigation + transitions)

```
flow app_navigation {
  product_list -> product_detail  when selected_product != none
  product_detail -> product_list  when back
  product_list -> cart_screen     when navigate_cart
  cart_screen -> checkout         when cart.length > 0 && start_checkout
  checkout -> product_list        when checkout_state == success
}
```

Flows declare which screens connect and under what conditions. The compiler decides the transition animation (slide, fade, modal) based on the relationship type.

#### Effects (side effects and async)

```
effect load_products(on_mount) {
  fetch("/api/products", on_ok: set(products, response.items),
                         on_err: set(load_error, error.message))
}

effect validate_email(on_change(email)) {
  set(email_error, if !valid_email(email) { some("Invalid email") } else { none })
}

effect auto_save(on_interval(30s)) {
  when draft != saved_version {
    fetch("/api/save", method: post, body: draft,
          on_ok: set(saved_version, draft))
  }
}
```

Effects are explicit, named, triggered by specific events. No implicit lifecycle. The runtime manages cancellation (navigating away cancels pending effects for that screen).

#### Styles (theming, not layout)

```
style brand {
  primary: #2563eb
  surface: #ffffff
  error: #dc2626
  radius: 8
  font: "Inter"
  density: comfortable  // compact | comfortable | spacious
}
```

Styles are semantic tokens, not CSS. The compiler uses them to fill in visual properties on scene buffer nodes. `density` affects spacing/padding uniformly. `radius` applies to all interactive elements. No per-node styling in the directive layer.

### Full Example: Shopping App

```
// === Data ===

data products: list<record {
  id: text, name: text, price: float,
  image: text, category: text, in_stock: bool
}>

data cart: list<record { product: ref<products>, quantity: int }>
data search: text = ""
data selected_product: option<ref<products>> = none
data checkout_state: enum { idle, processing, success, error(text) } = idle

// === Screens ===

screen catalog {
  search: input(bind: search, placeholder: "Search...", clear: true)

  results: list(
    source: products |> filter(matches(search, .name)),
    item: |p| group(
      image: image(p.image),
      name: text(p.name),
      price: text(format_currency(p.price)),
      tap: action("View", on_tap: set(selected_product, some(p)))
    )
  )

  cart_badge: indicator(cart.length, label: "Cart")
  cart_action: action("Cart", on_tap: navigate(cart_screen))
}

screen product_detail(product: ref<products>) {
  hero: image(product.image)
  name: text(product.name)
  price: text(format_currency(product.price))
  stock: indicator(product.in_stock, label: "Available")

  add: action("Add to Cart",
    enabled: product.in_stock,
    on_tap: [
      push(cart, {product: product, quantity: 1}),
      navigate(catalog)
    ]
  )

  back: action("Back", on_tap: set(selected_product, none))
}

screen cart_screen {
  items: list(
    source: cart,
    empty: "Your cart is empty",
    item: |entry| group(
      name: text(entry.product.name),
      qty: input(bind: entry.quantity, kind: stepper, min: 1, max: 99),
      subtotal: text(format_currency(entry.product.price * entry.quantity)),
      remove: action("Remove", on_tap: remove(cart, entry))
    )
  )

  total: text(format_currency(sum(cart, |e| e.product.price * e.quantity)))

  checkout: action("Checkout",
    enabled: cart.length > 0,
    on_tap: set(checkout_state, processing)
  )

  status: slot(
    when checkout_state == processing: indicator(busy, label: "Processing..."),
    when checkout_state == success: text("Order placed!"),
    when checkout_state == error(msg): text(msg)
  )
}

// === Flow ===

flow navigation {
  catalog -> product_detail(selected_product!)  when selected_product != none
  product_detail -> catalog                     when selected_product == none
  catalog -> cart_screen                        when navigate_cart
  cart_screen -> catalog                        when back
}

// === Effects ===

effect load_catalog(on_mount) {
  fetch("/api/products", on_ok: set(products, response))
}

effect process_checkout(on_change(checkout_state)) {
  when checkout_state == processing {
    fetch("/api/checkout", method: post, body: cart,
      on_ok: [set(checkout_state, success), set(cart, [])],
      on_err: set(checkout_state, error(err.message))
    )
  }
}

// === Style ===

style app {
  primary: #2563eb
  surface: #fafafa
  radius: 12
  density: comfortable
}
```

### What the Compiler Decides

Given the above directives, the compiler (deterministic or AI-assisted) decides:

| Decision | Based on |
|----------|----------|
| Whether `results` renders as cards, rows, or grid | Device width, item count, image presence |
| Scroll direction (vertical list vs. horizontal carousel) | Item count, screen orientation |
| Where `search` appears (sticky header, inline, modal) | Screen size, interaction pattern |
| Button style (filled, outlined, text-only) | Primary vs. secondary action, context |
| Navigation animation (slide, modal, fade) | Flow relationship (forward, back, overlay) |
| Spacing, padding, font sizes | `density` token + device DPI |
| Loading skeleton shape | Data structure + view types |
| Error placement (inline, toast, banner) | Error type + screen context |

The directive author controls WHAT. The compiler controls HOW. The scene buffer is the interface between them.

### Properties of this DSL

**What it guarantees:**
- Type safety (data references checked at compile time)
- No dead state (all declared data must be referenced by a view or effect)
- No orphan mutations (every mutation targets declared data)
- Flow completeness (every screen is reachable)
- Effect determinism (same trigger + same state → same mutations)

**What it doesn't specify:**
- Pixel positions, sizes, or spacing
- Colors (beyond semantic tokens)
- Font choices or weights
- Animation curves or durations
- Responsive breakpoints
- Platform-specific conventions

**Compilation target:**
The output is the scene buffer format from §1-4 — flat node array with layout primitives, style, bindings, and handler references. The DSL compiles to the *same* format the AI would emit directly when working at the scene buffer level.

### Two Modes of Use

1. **AI generates directives** → compiler produces scene buffer → GPU renders
   - AI works at highest abstraction (intent). Compiler handles structure. GPU handles pixels.
   - Best for: apps, forms, CRUD, dashboards, standard workflows.

2. **AI generates scene buffer directly** → GPU renders
   - AI works at mid-level (structure). Bypasses directive compilation.
   - Best for: custom visualizations, creative layouts, game-like interfaces, anything the directive vocabulary can't express.

Both target the same runtime. The directive layer is optional — it raises the ceiling for common cases without limiting escape to raw scene manipulation.

### View Vocabulary (Complete)

The vocabulary is organized in tiers:

#### Tier 1: Atomic Views (irreducible)

| View | Meaning |
|------|---------|
| `text(expr)` | Display a value |
| `input(bind, ...)` | Editable value (text, number, date, etc.) |
| `image(src)` | Display visual media |
| `action(label, on_tap)` | Something the user can activate |
| `toggle(bind, label)` | Boolean switch |
| `choice(source, selected)` | Pick from options |
| `indicator(expr, ...)` | Display a status or metric |
| `slot(conditions...)` | Conditional content |

#### Tier 2: Collection Views (structuring data)

| View | Meaning |
|------|---------|
| `list(source, item)` | Linear scrollable collection |
| `tree(source, children, item)` | Hierarchical expandable collection |
| `grid(source, item, cols?)` | 2D matrix of items |
| `table(source, columns)` | Columnar structured data |

#### Tier 3: Patterns (common compositions with compiler intelligence)

| Pattern | Meaning | Possible presentations |
|---------|---------|----------------------|
| `form(fields, submit)` | Structured data entry with validation | Inline, stepped wizard, floating labels, sectioned |
| `detail(subject, ...)` | Expanded view of a single item | Full-screen, modal, slide-over, split pane |
| `search(bind, source, item)` | Input that filters visible content | Sticky header, expandable icon, modal overlay |
| `nav(destinations, active)` | Set of navigable destinations | Tab bar, sidebar, hamburger, bottom sheet, rail |
| `feed(source, item)` | Append-only stream | Infinite scroll, paginated, pull-to-refresh |
| `timeline(source, event)` | Time-ordered events | Vertical thread, horizontal scroll, calendar-linked |
| `chat(source, compose, send)` | Bidirectional message exchange | Bubbles, threads, inline replies |
| `wizard(steps, current)` | Multi-step process with progress | Stepper, paginated cards, accordion |
| `dashboard(kpis, sections)` | KPIs + summary views | Grid of cards, single-scroll, tabbed |
| `settings(sections)` | Key-value configuration | Grouped list, searchable, segmented |
| `profile(identity, fields)` | Identity display + edit | Card, full page, header + content |
| `auth(mode, fields, submit)` | Sign in / sign up / recovery | Single card, split screen, full-page |
| `comparison(items, columns)` | Side-by-side evaluation | Columns, swipeable cards, diff view |
| `checkout(items, total, submit)` | Confirmation + payment flow | Stepped, single-page, slide-up sheet |
| `split(master, detail)` | Master-detail linked pair | Side-by-side (wide), drill-down (narrow) |
| `onboarding(steps)` | First-time guidance | Carousel, overlay tooltips, progressive disclosure |
| `notification(message, ...)` | Transient alert or banner | Toast, snackbar, banner, badge, inline |
| `command(source, on_select)` | Keyboard/voice action search | Palette overlay, omnibar, spotlight |
| `empty(message, action?)` | No data state with guidance | Illustration + CTA, inline hint |
| `error(message, retry?)` | Something went wrong | Banner, full-page, inline, toast |
| `media(source, controls?)` | Playable/viewable content | Lightbox, inline player, gallery carousel |
| `picker(source, selected, kind)` | Select from large/structured set | Modal list, dropdown, typeahead, calendar |

#### Tier 4: Embeds (platform/native — declared in DSL, rendered by registered implementation)

| Embed | Why it needs native support |
|-------|---------------------------|
| `embed("map", ...)` | Tile rendering, geographic gestures, POI layers |
| `embed("chart", ...)` | Axis math, data-to-pixel mapping, interactive tooltips |
| `embed("calendar", ...)` | Date math, locale day/month names, complex overflow grid |
| `embed("rich_editor", ...)` | Formatting toolbar, cursor management, paste handling |
| `embed("video", ...)` | Codec decoding, media session, DRM |
| `embed("canvas", ...)` | Freeform drawing, stroke undo, pressure sensitivity |
| `embed("code_editor", ...)` | Syntax highlighting, language services |

Embed usage:
```
screen location {
  map_view: embed("map", {
    center: user_location,
    markers: nearby_stores,
    on_select: |marker| set(selected_store, marker.store)
  })
  selected: detail(subject: selected_store, ...)
}
```

The runtime has a registry of embed renderers. The scene buffer allocates a viewport rect; the platform renders into it.

#### Coverage Check

| App type | Patterns used |
|----------|--------------|
| E-commerce | list, detail, search, checkout, auth, comparison, nav |
| Social media | feed, profile, chat, media, notification, nav, search |
| Productivity (email) | split, list, detail, search, settings, nav |
| Admin/dashboard | dashboard, table, form, settings, auth, nav |
| Messaging | chat, list, profile, search, notification |
| Finance | dashboard, list, detail, chart(embed), timeline, settings |
| Health/fitness | timeline, dashboard, chart(embed), settings, profile |
| Travel | search, list, detail, map(embed), comparison, checkout |
| Education | list, detail, media, wizard, indicator |
| Food delivery | search, list, detail, map(embed), checkout, timeline |

~24 patterns + ~7 embeds covers the space. Most apps are compositions of the same structural patterns with domain-specific data.

#### Pattern Composition

Patterns nest freely:

```
screen order_tracking {
  layout: split(
    master: timeline(
      source: order.events,
      event: |e| group(
        status: indicator(e.status),
        description: text(e.description),
        time: text(relative_time(e.timestamp))
      )
    ),
    detail: embed("map", {
      route: order.delivery_route,
      current: order.driver_location
    })
  )
}
```

---

## 9. Multi-Target Compilation

### The Principle

The directive DSL specifies behavior and intent. It says nothing about device form factor, screen size, input modality, or platform conventions. The **compiler** makes all presentation decisions based on a **target profile** — a description of the device and context the UI will run on.

Same directives → different scene buffers → same behavior, different presentation.

### Target Profile

```rust
struct TargetProfile {
    // Physical
    viewport: Vec2,             // available pixels
    pixel_density: f32,         // DPI scale factor
    safe_insets: [f32; 4],      // notch, status bar, home indicator

    // Input modality
    input: InputCapabilities,   // touch, pointer, keyboard, stylus, voice
    min_touch_target: f32,      // 44px mobile, 24px desktop, 32px watch

    // Form factor
    form: FormFactor,           // phone, tablet, desktop, watch, tv, embedded
    orientation: Orientation,   // portrait, landscape, any
    foldable: Option<FoldState>, // folded, half-open, flat

    // Platform conventions
    nav_paradigm: NavParadigm,  // bottom_tabs, sidebar, top_nav, gestures
    density: Density,           // compact, comfortable, spacious
    motion: MotionPreference,   // full, reduced, none

    // Accessibility
    text_scale: f32,            // user's preferred text scaling
    contrast: ContrastLevel,    // normal, high
    screen_reader: bool,        // whether a11y tree needs extra detail
}
```

### How Patterns Adapt

Each pattern has a **layout strategy** — a set of rules for how it maps to scene buffer structure given the target profile.

#### `nav` — the most form-factor-sensitive pattern

| Target | Presentation |
|--------|-------------|
| Phone portrait | Bottom tab bar (≤5 items) or hamburger drawer (>5) |
| Phone landscape | Bottom tab bar (compact) |
| Tablet portrait | Side rail (icons + labels on tap) |
| Tablet landscape | Full sidebar (icons + labels always visible) |
| Desktop | Top nav bar or persistent sidebar |
| Watch | Single-screen with swipe gestures between destinations |
| TV | Top horizontal nav with focus ring |

Compiler rule (pseudocode):
```
fn compile_nav(nav: NavPattern, target: &TargetProfile) -> SceneNodes {
    match (target.form, nav.destinations.len()) {
        (Phone, 1..=5) => bottom_tab_bar(nav),
        (Phone, 6..) => hamburger_drawer(nav),
        (Tablet, _) if target.viewport.x > 600 => side_rail(nav),
        (Desktop, _) => sidebar_full(nav),
        (Watch, _) => page_dots(nav),
        (TV, _) => top_horizontal(nav),
        _ => bottom_tab_bar(nav),  // fallback
    }
}
```

#### `list` — adapts to available space

| Target | Presentation |
|--------|-------------|
| Phone | Vertical scroll, full-width cards or rows |
| Tablet portrait | 2-column grid of cards |
| Tablet landscape / Desktop | 3-4 column grid, or table view for data-heavy items |
| Watch | Single-item pager with swipe |
| TV | Horizontal carousel with focus scaling |

Key decision: **when does a list become a grid?** Rule: if items have images AND viewport width > 2× minimum card width, use grid. Otherwise vertical list.

#### `split` — the classic responsive pattern

| Target | Presentation |
|--------|-------------|
| Wide (desktop, tablet landscape) | Side-by-side: master on left (1/3), detail on right (2/3) |
| Narrow (phone, tablet portrait) | Drill-down: show master. On select, navigate to detail. Back returns. |
| Watch | Not applicable — degrade to list + detail as separate screens |

This is where the flow system interacts with the compiler. In narrow mode, `split` implicitly creates a navigation transition between master and detail.

#### `form` — adapts to space and input method

| Target | Presentation |
|--------|-------------|
| Phone | Single column, full-width inputs, large tap targets |
| Tablet/Desktop | Two-column for related fields, inline labels |
| Watch | One field at a time (wizard-style), dictation input prioritized |
| TV | Directional-pad navigation between fields, on-screen keyboard |
| Keyboard-primary | Tab order emphasized, no scroll between fields if they fit |

#### `dashboard` — information density scaling

| Target | Presentation |
|--------|-------------|
| Phone | KPIs as horizontal scroll cards, sections as vertical scroll |
| Tablet | 2×2 KPI grid, sections below |
| Desktop | Full grid: KPIs top row, sections in 2-3 columns |
| Watch | Single KPI pager, swipe for next metric |
| TV | Full dashboard visible, no scroll, large type |

#### `chat` — input modality adaptation

| Target | Presentation |
|--------|-------------|
| Phone | Full-screen thread, keyboard pushes messages up, send button |
| Desktop | Split view (contacts left, thread right), Enter to send |
| Watch | Voice-to-text input, canned quick replies, minimal history |
| TV | Not typical — degrade to notification display only |

### Compilation Pipeline

```
┌─────────────────────┐
│  Directive AST       │  (parsed from DSL source)
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│  Semantic Analysis   │  Type checking, flow validation, dead-state detection
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│  Pattern Expansion   │  Each pattern → abstract layout tree (not yet sized)
│                      │  Informed by TargetProfile.form + input modality
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│  Layout Assignment   │  Abstract tree → concrete scene buffer nodes
│                      │  Assigns: distribution, size policy, spacing, alignment
│                      │  Informed by TargetProfile.viewport + density
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│  Style Application   │  Applies theme tokens → fill, radius, shadow, text props
│                      │  Informed by TargetProfile.contrast + text_scale
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│  Binding Generation  │  Creates state bindings, handler mappings, transitions
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│  Scene Buffer        │  Final flat node array ready for GPU
└─────────────────────┘
```

### Runtime Recompilation

The target profile can change at runtime:
- Device rotation (viewport + orientation change)
- Window resize (desktop)
- Fold state change (foldable devices)
- Accessibility settings change (text scale, contrast, motion)
- Display connection (phone → external monitor)

When the target profile changes, the compiler **re-runs pattern expansion and layout assignment** on the current directive state. The result is a new scene buffer. State is preserved (the state store doesn't change). Only the visual structure updates.

This is NOT responsive CSS breakpoints. It's a full recompilation of the layout strategy. A `split` that was side-by-side becomes drill-down navigation. A `nav` that was a sidebar becomes a bottom tab bar. Structural changes, not just size adjustments.

```rust
fn on_target_change(directives: &AST, state: &StateStore, new_target: &TargetProfile) -> SceneBuffer {
    let abstract_tree = expand_patterns(directives, new_target);
    let scene = assign_layout(abstract_tree, new_target);
    let scene = apply_style(scene, &directives.style, new_target);
    let scene = generate_bindings(scene, state, directives);
    scene
}
```

Cost: full recompilation of pattern → scene mapping. For a typical app (50-200 nodes), this is microseconds on CPU. Happens only on profile change events (rotation, resize), not per frame.

### Cross-Target Behavior Guarantees

The key contract: **behavior is identical across targets, only presentation varies.**

| Same across all targets | Varies by target |
|------------------------|-----------------|
| State shape and values | Node arrangement |
| Mutation effects | Visual hierarchy |
| Navigation graph (which screens exist) | Navigation animation / mechanism |
| Data flow (bindings, derived state) | Input method (tap vs click vs d-pad) |
| Effect triggers and async behavior | Touch target sizes |
| Validation rules | Information density |
| Event semantics (tap is tap) | Gesture interpretation (scroll vs swipe-to-navigate) |

### Target-Specific Hints (Optional)

For cases where the directive author knows something the compiler can't infer:

```
screen product_list {
  results: list(
    source: products,
    item: |p| group(...),
    hint: {
      phone: { style: rows },        // force rows on phone (not cards)
      desktop: { style: table },      // force table on desktop
      watch: { max_visible: 3 }       // only show 3 at a time
    }
  )
}
```

Hints are optional overrides. They break the "pure intent" abstraction slightly but provide an escape valve for when the compiler's default choice is wrong. They're target-keyed, not pixel-keyed — you never say `width > 768px`, you say `tablet`.

### Watch/Embedded Targets — Extreme Constraints

Watches and embedded displays (car dashboards, appliances) have severe constraints:

| Constraint | Impact |
|-----------|--------|
| ~40mm viewport | Only 1-2 pieces of information visible at once |
| No keyboard | Voice input, canned replies, crown/bezel rotation |
| Glanceable | Must convey status in <2 seconds |
| Limited touch | Large targets, simple gestures only |
| Battery | Minimize GPU work, reduce animation |

The compiler for watch targets:
- Converts `list` → single-item pager with crown scroll
- Converts `form` → one-field-at-a-time wizard
- Converts `dashboard` → cycling single-KPI display
- Eliminates `split` entirely (no room for two panes)
- Reduces `nav` → swipe between pages with dot indicators
- Converts `action` → fills entire width (large tap target)

This is a genuine structural transformation — not just "make it smaller."

### TV Targets — 10-foot UI

TVs have different constraints: large display but distant viewer, directional pad input.

| Constraint | Impact |
|-----------|--------|
| 3-4 meter viewing distance | Very large text, high contrast |
| D-pad navigation | Must have focus system, no pointer |
| No text input | Voice search, on-screen keyboard (rare) |
| Horizontal orientation | Landscape-only layouts |
| Content-focused | Minimal chrome, media prominence |

The compiler for TV:
- Converts `list` → horizontal carousel with focus scaling (selected item grows)
- Converts `nav` → top horizontal bar with focus ring
- Converts `input` → voice input trigger or on-screen keyboard launch
- Ensures every interactive element is reachable by d-pad (auto-generates focus graph)
- Increases all spacing (density: spacious) and text size
- Converts `chat` → read-only notification display

### Compilation Registry

The compiler is extensible. Each pattern has a registered set of layout strategies per form factor:

```rust
struct PatternCompiler {
    pattern: PatternKind,
    strategies: HashMap<FormFactor, LayoutStrategy>,
}

enum LayoutStrategy {
    Rule(fn(&PatternData, &TargetProfile) -> Vec<SceneNode>),
    Template(SceneTemplate),   // pre-built node tree with holes
    Solver(ConstraintSet),     // SUPPLE-style optimization for complex cases
}
```

New form factors or pattern-to-layout mappings can be added without modifying the DSL grammar or existing directives. The directive is stable; the compilation improves.

### Testing Multi-Target

```
// Property: same directives compile for all supported targets
#[test]
fn all_targets_produce_valid_scene() {
    let directives = parse("shopping_app.vui");
    for target in ALL_TARGET_PROFILES {
        let scene = compile(&directives, &target);
        assert!(scene.is_valid());          // no overlapping nodes, all refs resolve
        assert!(scene.all_reachable());     // every interactive node is reachable via input modality
        assert!(scene.fits_viewport(target.viewport)); // nothing overflows
    }
}

// Property: state mutations produce same next-state regardless of target
#[test]
fn behavior_invariant_across_targets() {
    let directives = parse("shopping_app.vui");
    for target_a in ALL_TARGET_PROFILES {
        for target_b in ALL_TARGET_PROFILES {
            let scene_a = compile(&directives, &target_a);
            let scene_b = compile(&directives, &target_b);
            // Apply same sequence of mutations
            let state_a = apply_mutations(&scene_a, &MUTATION_SEQUENCE);
            let state_b = apply_mutations(&scene_b, &MUTATION_SEQUENCE);
            assert_eq!(state_a, state_b);  // identical state regardless of presentation
        }
    }
}
```

---

## 10. Game UI & 3D Integration

### What Makes Game UI Different

| Property | App UI | Game UI |
|----------|--------|---------|
| Visual language | System conventions (Material, HIG) | Unique per title, deeply stylized |
| Idle state | Static | Constantly animated (pulses, glows, particles) |
| Spatial model | 2D screen plane only | Mixed 2D overlay + 3D world-anchored |
| Shape | Rectangles | Radials, arcs, hexagons, organic curves |
| Performance budget | Owns the frame | Shares frame with 3D scene rendering |
| Feedback | State change → visual update | Continuous: particles, shake, flash, audio |
| Visibility | Always visible or toggled | Contextual: fades in/out based on game state |
| Layering | Modals/sheets | Stacked modes (gameplay → pause → settings → keybinds) |
| Input | Pointer/touch/keyboard | Gamepad, mouse+KB, VR controllers, motion |

### Four Spatial Categories of Game UI

**1. Diegetic** — exists within the fictional world. Characters can see it.
- Dead Space: health meter on Isaac's suit spine
- Racing games: car dashboard, rear-view mirror
- Sci-fi terminals: in-world screens the player walks up to

**2. Spatial** — exists in 3D space but isn't part of the fiction.
- Health bars floating over enemies
- Waypoint markers in the distance
- Name tags over players
- Damage numbers rising from hit location

**3. Meta** — overlaid on screen, reacts to game world but isn't spatial.
- Blood vignette when damaged
- Speed lines when boosting
- Screen crack when near death
- Frost/rain on "camera lens"

**4. Non-diegetic** — traditional 2D HUD overlay, no world relationship.
- Minimap, ability bar, quest tracker, inventory screens

### Game UI Patterns (Tier 3 additions)

| Pattern | Meaning | Examples |
|---------|---------|---------|
| `hud(elements)` | Persistent overlay, contextually visible | Health, ammo, minimap, compass |
| `meter(value, max, ...)` | Animated resource bar with urgency states | Health flashes red at low, shield recharges with glow |
| `radial(items, selected)` | Circular/arc selection menu | Weapon wheel, emote wheel, spell ring |
| `inventory(slots, items)` | Grid/list with drag-between-containers | Equipment, storage, crafting ingredients |
| `dialog(speaker, lines, choices)` | Conversation with branching choices | RPG dialog, visual novel |
| `graph(nodes, edges, ...)` | Node-and-edge structure with traversal | Skill tree, tech tree, crafting recipes |
| `scoreboard(players, sort)` | Live-updating ranked list | Multiplayer standings, leaderboard |
| `context_prompt(trigger, label)` | Appears near world object when conditions met | "Press E to interact", "Hold X to revive" |
| `cooldown(action, duration)` | Circular sweep timer on an ability | Ability cooldown, reload timer |
| `world_anchor(target, content)` | UI positioned relative to 3D point | Enemy health bar, waypoint, name tag |
| `stack_mode(layers)` | Layered UI states, each dims/blurs previous | Gameplay → pause → options → keybinds |
| `toast_particle(from, to, value)` | Animated token flying between UI elements | XP flying to counter, gold flying to wallet |
| `minimap(world, viewport, markers)` | Spatial overview with tracking | Corner radar, full map overlay |

### Game UI DSL Examples

#### HUD with contextual visibility

```
screen gameplay {
  overlay: hud(
    visibility: game_state,  // visible during: combat, exploration. hidden during: cutscene, menu
    elements: [
      health: meter(player.hp, player.max_hp,
        urgency: [
          {threshold: 0.25, effect: pulse_red},
          {threshold: 0.1, effect: screen_vignette}
        ],
        regen: player.regen_rate
      ),

      stamina: meter(player.stamina, player.max_stamina,
        drain: smooth,
        regen: delayed(1.5s)
      ),

      abilities: group(
        slot_1: cooldown(ability_1, on_tap: activate(ability_1)),
        slot_2: cooldown(ability_2, on_tap: activate(ability_2)),
        slot_3: cooldown(ability_3, on_tap: activate(ability_3)),
        ultimate: cooldown(ultimate, charge: ultimate_energy, max: 100)
      ),

      minimap: minimap(
        world: level_map,
        viewport: camera.position,
        markers: [
          {source: enemies, icon: "hostile", color: red, visible: detected},
          {source: objectives, icon: "waypoint", color: gold},
          {source: teammates, icon: "friendly", color: blue},
        ],
        radius: 50
      ),

      compass: indicator(camera.heading,
        markers: objectives |> map(|o| bearing_to(camera, o))
      ),

      interact: context_prompt(
        trigger: nearby_interactable != none,
        label: nearby_interactable.prompt,
        input: interact_key
      )
    ]
  )
}
```

#### Radial weapon wheel

```
screen weapon_select {
  wheel: radial(
    items: player.weapons,
    selected: active_weapon,
    trigger: hold(weapon_button),  // appears while held, commits on release
    item: |weapon| group(
      icon: image(weapon.icon),
      name: text(weapon.name),
      ammo: text(weapon.ammo ++ "/" ++ weapon.max_ammo),
      empty: indicator(weapon.ammo == 0, effect: grayed_out)
    ),
    center: group(
      icon: image(active_weapon.icon),
      name: text(active_weapon.name)
    ),
    on_select: set(active_weapon, selection)
  )
}
```

#### Skill tree

```
screen skill_tree {
  tree: graph(
    nodes: skills,
    edges: |skill| skill.prerequisites,
    node: |skill| group(
      icon: image(skill.icon),
      name: text(skill.name),
      state: indicator(skill.state),  // locked, available, acquired
      cost: text(skill.cost ++ " pts"),
      on_tap: when skill.state == available && points >= skill.cost {
        [set(skill.state, acquired), increment(points, -skill.cost)]
      }
    ),
    layout: top_to_bottom,  // or: radial_outward, left_to_right
    camera: pannable(zoom: true, bounds: auto)
  )

  points_remaining: text("Skill Points: " ++ points)
  reset: action("Respec", on_tap: confirm("Reset all skills?", on_confirm: reset_skills()))
}
```

#### Inventory with drag

```
screen inventory {
  equipment: inventory(
    slots: [
      {id: "head", accepts: armor_head},
      {id: "chest", accepts: armor_chest},
      {id: "legs", accepts: armor_legs},
      {id: "weapon_1", accepts: weapons},
      {id: "weapon_2", accepts: weapons},
      {id: "accessory", accepts: accessories},
    ],
    equipped: player.equipment,
    on_equip: |slot, item| equip(player, slot, item),
    on_unequip: |slot| unequip(player, slot)
  )

  backpack: inventory(
    slots: grid(cols: 6, rows: 4),
    items: player.inventory,
    item: |item| group(
      icon: image(item.icon),
      count: when item.stackable { text(item.quantity) },
      rarity: indicator(item.rarity, effect: border_glow)
    ),
    drag: true,
    on_drop_external: |item, target| equip(player, target, item),
    context_menu: |item| [
      action("Use", enabled: item.usable, on_tap: use_item(item)),
      action("Drop", on_tap: drop_item(item)),
      action("Inspect", on_tap: set(inspected_item, item)),
    ]
  )

  item_detail: slot(
    when inspected_item != none: detail(
      subject: inspected_item,
      title: text(inspected_item.name),
      metadata: inspected_item.stats,
      body: text(inspected_item.description)
    )
  )
}
```

#### Dialog with branching

```
screen conversation(npc: ref<npcs>) {
  dialog: dialog(
    speaker: npc,
    portrait: image(npc.portrait),
    lines: current_dialog_node.text,
    typewriter: true,  // characters appear one at a time
    choices: current_dialog_node.options,
    choice: |option| group(
      label: text(option.text),
      locked: indicator(option.requires && !meets_requirement(option.requires)),
      requirement: when option.requires { text(option.requires.description) }
    ),
    on_choice: |option| advance_dialog(npc, option.next_node),
    on_end: navigate(gameplay)
  )
}
```

### Rendering Extensions for Games

#### Non-rectangular layouts

```rust
enum Distribution {
    // ... existing (StackX, StackY, Layer, Grid, Flow)

    // Game additions
    Radial { radius: f32, start_angle: f32, sweep: f32 },
    Polar { rings: u32, segments: u32 },
    Graph { positions: BufferRange },   // explicit or force-directed
    Hex { cols: u32, rows: u32 },
}
```

Compute shader for `Radial`:
```wgsl
fn layout_radial(node: Node, params: RadialParams) {
    let angle_step = params.sweep / f32(node.child_count);
    for (var i = 0u; i < node.child_count; i++) {
        let child_idx = node.first_child + i;
        let angle = params.start_angle + angle_step * (f32(i) + 0.5);
        nodes[child_idx].rect.x = node.center.x + params.radius * cos(angle) - nodes[child_idx].size.x / 2.0;
        nodes[child_idx].rect.y = node.center.y + params.radius * sin(angle) - nodes[child_idx].size.y / 2.0;
    }
}
```

#### World-space anchoring

Nodes anchored to 3D world positions. CPU or compute pass projects each frame:

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

Each frame: project through view-projection matrix → screen `(x, y, depth)`. Apply distance fade/scale. Write to node position. One matrix multiply per anchored element — cheap.

#### Custom shaders per node

Game UIs have unique materials. The SDF quad renderer needs shader variants:

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

Implementation: instanced quad renderer switches variant per-instance via material ID. Each variant has same inputs (rect, UV, time) but different fragment processing.

#### Particle systems

Lightweight GPU particle emitters attached to UI nodes:

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

#### Audio triggers

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

#### Continuous animation

```rust
enum Animation {
    // Triggered by state change (existing)
    Transition { from: f32, to: f32, duration: f32, easing: Easing },

    // Runs continuously while node is visible (game addition)
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

### The `fx` Sublanguage (Effects & Feedback)

Directives gain an `fx` property for continuous visual/audio feedback:

```
screen gameplay {
  health: meter(player.hp, player.max_hp,
    fx: [
      when value < 0.25: pulse(color, red, frequency: 2hz),
      when value < 0.1: [screen_vignette(red, 0.3), shake(2px, 0.5s)],
      on_decrease: flash(white, 0.1s),
      on_increase: particle(heal_sparkle, count: 5),
    ]
  )

  xp_gain: toast_particle(
    from: defeated_enemy.position,
    to: xp_counter,
    value: xp_amount,
    on_arrive: increment(player.xp, xp_amount)
  )

  ability_1: cooldown(ability_1,
    fx: [
      when ready: glow(ability_1.color, pulse: true),
      on_activate: [burst_particle(ability_1.color), sound("ability_activate")],
      when cooling_down: sweep_reveal(clockwise),
    ]
  )
}
```

`fx` declarations compile to combinations of:
- Continuous animations (pulse, glow, wobble)
- Particle emitters (burst, stream, toast)
- Shader variants (vignette, dissolve, energy)
- Audio triggers (sound IDs mapped to events)
- Screen-space effects (shake, flash — applied to root node)

### 3D Integration Architecture

```
┌────────────────────────────────────────────┐
│  3D Scene Render Pass                       │
│  (game engine: meshes, lighting, etc.)      │
│  Output: color buffer + depth buffer        │
└──────────────┬─────────────────────────────┘
               │
┌──────────────▼─────────────────────────────┐
│  UI Compute Pass (layout + animation)       │
│  Reads: depth buffer (for occlusion)        │
│  Reads: camera matrices (for world anchors) │
│  Output: positioned scene buffer            │
└──────────────┬─────────────────────────────┘
               │
┌──────────────▼─────────────────────────────┐
│  UI Render Pass                             │
│  Composites UI on top of 3D color buffer    │
│  Spatial/diegetic: depth-tested             │
│  HUD/overlay: depth-ignored                 │
└────────────────────────────────────────────┘
```

For diegetic UI (screens in the game world), the UI render pass writes to a texture that the 3D scene samples as a material on a mesh.

#### DSL spatial declarations

```
screen diegetic_terminal(terminal: ref<world_objects>) {
  render_to: world_surface(terminal.screen_mesh)

  content: group(
    header: text("SYSTEM ACCESS"),
    files: list(source: terminal.files, item: |f| ...),
    status: indicator(terminal.network_status)
  )

  fx: [shader(crt_scanline), shader(screen_flicker, intensity: 0.02)]
}

screen spatial_markers {
  render_mode: world_space  // depth-tested, distance-scaled

  enemies: for enemy in visible_enemies {
    world_anchor(enemy.head_position, offset: (0, -20)) {
      health: meter(enemy.hp, enemy.max_hp, style: minimal),
      name: text(enemy.name),
      level: text("Lv." ++ enemy.level)
    }
  }

  waypoint: world_anchor(objective.position) {
    icon: image("waypoint"),
    distance: text(format_distance(distance_to(objective))),
    direction: when !on_screen { indicator(bearing_to(objective), style: edge_arrow) }
  }
}
```

### Game Genre Coverage

| Game genre | Patterns used |
|-----------|--------------|
| FPS/TPS | hud, meter, cooldown, minimap, radial(weapon wheel), context_prompt, scoreboard |
| RPG | inventory, graph(skill tree), dialog, timeline(quest), meter, detail(character sheet) |
| Strategy/RTS | minimap, grid(unit selection), dashboard(resources), command, tree(tech tree) |
| Racing | hud(speedometer as meter), minimap(track), scoreboard, world_anchor(positions) |
| Puzzle | grid, indicator(score/moves), toast_particle(score fly), cooldown(hints) |
| Fighting | meter(health+super), cooldown, scoreboard(round tracker), toast_particle(combo) |
| Survival | inventory, meter(hunger/thirst/health), graph(crafting), minimap, context_prompt |
| MOBA | hud, minimap, scoreboard, cooldown(abilities), graph(item shop), chat |

### Unified Pattern Count

With game additions, the total vocabulary:

- **Tier 1 (atomic):** 8 views
- **Tier 2 (collections):** 4 views
- **Tier 3 (app patterns):** 24 patterns
- **Tier 3 (game patterns):** 13 patterns
- **Tier 4 (embeds):** 7 embed types
- **FX sublanguage:** continuous animation, particles, shaders, audio

Total: ~56 named concepts covering both traditional apps and game interfaces, all compiling to the same GPU scene buffer.

---

## 11. DSL Implementation: Rust as the Language

### Decision

The DSL should be Rust itself — not a custom syntax with a custom parser, but a well-designed Rust API with proc macros for ergonomics. Developers write Rust functions that describe intent using the pattern vocabulary; the runtime compiles these to scene buffers.

### Why Rust Works

The DSL constructs map directly to Rust functions, types, and traits:

| DSL concept | Rust equivalent |
|-------------|----------------|
| `data` declarations | `#[derive(State)]` structs |
| `screen` definitions | `fn name(state: &S) -> impl View` |
| `flow` transitions | `fn routes(state: &S) -> impl Flow` |
| `effect` triggers | `#[effect(on_mount)]` async functions |
| `style` tokens | `const THEME: Style = Style::new()...` |
| Pattern views | Builder-pattern function calls (`list()`, `form()`, etc.) |
| Closures / lambdas | Rust closures |
| Conditionals | `match`, `if`, `when()` |
| Data transforms | `.filter()`, `.map()`, iterators |

### What You Gain

| Benefit | Detail |
|---------|--------|
| No parser to build | `rustc` parses. You write a library, not a language. |
| Type safety | Misspell a field? Compiler error. Wrong type in mutation? Compiler error. |
| IDE support | rust-analyzer: completion, go-to-def, inline errors, refactoring. Day one. |
| Arbitrary logic | Filters, maps, conditionals, arithmetic — just Rust. No restricted expression sublanguage. |
| Async native | Effects are `async fn`. Cancellation, timeouts, concurrency from tokio/smol. |
| Testing | `#[test]` functions. Mock state, call view function, assert on intent tree. |
| Ecosystem | `serde`, `tracing`, `criterion` — all available. |
| AI-friendly | LLMs are well-trained on Rust. Builder-pattern APIs are straightforward to generate. |
| Natural escape hatch | Drop from `impl View` to raw scene buffer by importing `vibe::scene::*`. Same language. |

### What You Lose (and Mitigations)

| Loss | Mitigation |
|------|-----------|
| Hot reload | `cargo watch` + incremental. Or `hot-lib-reloader` for dylib swap. Or `.vui` interpreter for dev (see Hybrid below). |
| Compile times | View functions in separate crate. Incremental means only changed views recompile. |
| Non-developer accessibility | AI generates Rust. For non-devs, provide `.vui` interpreter as on-ramp. |
| Verbosity vs custom DSL | Builder pattern + good defaults. Roughly same line count as custom DSL (see examples). |

### Core Trait Design

```rust
/// Anything that can become UI
trait View {
    fn build(&self, ctx: &mut BuildContext) -> ViewNode;
}

/// Every pattern implements View
impl<T> View for List<T> { ... }
impl View for Form { ... }
impl View for Nav { ... }
impl View for Dashboard { ... }
impl View for Meter { ... }
impl<T> View for Radial<T> { ... }
impl View for Graph { ... }

/// Tuples and arrays compose automatically
impl<V: View, const N: usize> View for [V; N] { ... }

/// Runtime entry point
fn run(root: impl View, target: TargetProfile, style: Style) {
    let intent_tree = root.build(&mut BuildContext::new());
    let scene = compile(intent_tree, &target, &style);
    gpu::run_loop(scene);
}
```

### Proc Macros (Optional Ergonomics)

```rust
// State derive: generates dirty tracking + mutation methods
#[derive(State)]
struct AppState {
    products: Vec<Product>,  // generates: state.products.push(), .remove(), .set()
    search: String,          // generates: state.search.set(), with dirty flag
}

// Effect attribute: registers trigger + cancellation
#[effect(on_mount)]
async fn load_data(state: &mut AppState) { ... }

#[effect(on_change(state.search), debounce(ms(300)))]
fn filter_products(state: &mut AppState) { ... }

// Flow derive: validates reachability at compile time
#[derive(Flow)]
enum Route {
    Catalog,
    #[to(Catalog, when = "selected.is_none()")]
    ProductDetail(ProductId),
    CartScreen,
}
```

### Hybrid Approach: Rust API + `.vui` Interpreter

Best of both worlds for different contexts:

```
Production path:
  Rust source → cargo build → native binary with compiled views
  (fastest, type-safe, optimized)

Development path:
  .vui file → runtime interpreter → intent tree → scene buffer
  (hot reload, no compile wait, rapid iteration)

AI generation path:
  LLM → generates either Rust or .vui depending on context
  (both target the same intent tree → same scene buffer)
```

The `.vui` format from §8 becomes a serialization format for the intent tree, not a separate language. The Rust API builds the same tree in-memory.

```rust
// These produce the identical intent tree:

// Rust API:
let view = list(&products).filter(|p| p.in_stock).item(|p| text(&p.name));

// .vui interpreted:
// list(source: products |> filter(.in_stock), item: |p| text(p.name))

// Both compile to the same scene buffer via the same target compiler.
```

### Architecture

```
┌──────────────────────────────────────────────────────┐
│  Authoring (choose one or both)                       │
│                                                       │
│  Rust API                    .vui interpreter          │
│  fn catalog() -> impl View   parse("catalog.vui")     │
│       │                           │                   │
│       └──────────┬────────────────┘                   │
│                  │                                    │
│          Intent Tree (in-memory)                      │
│          ViewNode::List { source, filter, item }      │
│          ViewNode::Form { fields, submit }            │
│          ViewNode::Nav { destinations }               │
└────────────────┬─────────────────────────────────────┘
                 │
┌────────────────▼─────────────────────────────────────┐
│  Target Compiler (pattern expansion + layout)         │
│  Intent Tree + TargetProfile → Scene Buffer           │
└────────────────┬─────────────────────────────────────┘
                 │
┌────────────────▼─────────────────────────────────────┐
│  GPU Runtime                                          │
│  Compute (layout) → Render (SDF quads + text)         │
└──────────────────────────────────────────────────────┘
```

### Data Layer in Rust

```rust
#[derive(State)]
struct AppState {
    products: Vec<Product>,
    cart: Vec<CartItem>,
    search: String,
    selected: Option<ProductId>,
    checkout: CheckoutState,
}

#[derive(Record)]
struct Product {
    id: String,
    name: String,
    price: f64,
    image: String,
    category: String,
    in_stock: bool,
}

#[derive(Record)]
struct CartItem {
    product: Ref<Product>,
    quantity: i32,
}

#[derive(StateEnum)]
enum CheckoutState { Idle, Processing, Success, Error(String) }
```

### Screens as Functions

```rust
fn catalog(state: &AppState) -> impl View {
    screen([
        input(&state.search)
            .placeholder("Search...")
            .clearable(),

        list(&state.products)
            .filter(|p| p.name.contains(&state.search))
            .item(|p| group([
                image(&p.image),
                text(&p.name),
                text(format_currency(p.price)),
                indicator(p.in_stock).label("In Stock"),
                action("Add to Cart")
                    .enabled(p.in_stock)
                    .on_tap(state.cart.push(CartItem::new(p, 1))),
            ])),

        indicator(state.cart.len()).label("Cart"),
        action("Cart").on_tap(navigate(Route::CartScreen)),
    ])
}

fn product_detail(state: &AppState, id: &ProductId) -> impl View {
    let product = state.products.get(id);
    detail(product, [
        image(&product.image).hero(),
        text(&product.name).title(),
        text(format_currency(product.price)),
        indicator(product.in_stock).label("Available"),
        action("Add to Cart")
            .primary()
            .enabled(product.in_stock)
            .on_tap(|| {
                state.cart.push(CartItem::new(product, 1));
                navigate(Route::Catalog);
            }),
    ])
}

fn cart_screen(state: &AppState) -> impl View {
    screen([
        list(&state.cart)
            .empty("Your cart is empty")
            .item(|entry| group([
                text(&entry.product.name),
                input(&entry.quantity)
                    .kind(InputKind::Stepper)
                    .range(1..=99),
                text(format_currency(entry.product.price * entry.quantity as f64)),
                action("Remove").on_tap(state.cart.remove(entry)),
            ])),

        text(format_currency(state.cart.iter().map(|e| e.subtotal()).sum())),

        action("Checkout")
            .primary()
            .enabled(!state.cart.is_empty())
            .on_tap(state.checkout.set(CheckoutState::Processing)),

        slot(|s| match &state.checkout {
            CheckoutState::Processing => s.show(indicator(Busy).label("Processing...")),
            CheckoutState::Success => s.show(text("Order placed!")),
            CheckoutState::Error(msg) => s.show(text(msg)),
            _ => {}
        }),
    ])
}
```

### Flows and Effects

```rust
fn navigation(state: &AppState) -> impl Flow {
    flow([
        route(catalog)
            .to(product_detail)
            .when(|| state.selected.is_some()),
        route(product_detail)
            .to(catalog)
            .when(|| state.selected.is_none()),
        route(catalog)
            .to(cart_screen)
            .on(Navigate::Cart),
        route(cart_screen)
            .to(catalog)
            .on(Navigate::Back),
    ])
}

#[effect(on_mount)]
async fn load_catalog(state: &mut AppState) {
    match fetch("/api/products").await {
        Ok(products) => state.products.set(products),
        Err(e) => log::error!("Failed to load: {e}"),
    }
}

#[effect(on_change(state.checkout))]
async fn process_checkout(state: &mut AppState) {
    if matches!(state.checkout, CheckoutState::Processing) {
        match post("/api/checkout", &state.cart).await {
            Ok(_) => {
                state.checkout.set(CheckoutState::Success);
                state.cart.clear();
            }
            Err(e) => state.checkout.set(CheckoutState::Error(e.to_string())),
        }
    }
}
```

### Game UI in Rust

Game patterns translate naturally — Rust's expressiveness handles the complexity:

```rust
fn gameplay_hud(state: &GameState) -> impl View {
    hud(state.game_phase.allows_hud(), [
        meter(state.player.hp, state.player.max_hp)
            .fx(|fx| fx
                .when(|v| v < 0.25, pulse(Color::RED, hz(2.0)))
                .when(|v| v < 0.10, screen_vignette(Color::RED, 0.3))
                .on_decrease(flash(Color::WHITE, secs(0.1)))
                .on_increase(particle(HealSparkle, 5))
            ),

        meter(state.player.stamina, state.player.max_stamina)
            .drain(Curve::Smooth)
            .regen(Regen::Delayed(secs(1.5))),

        group([
            cooldown(&state.ability_1).on_tap(|| activate(&state.ability_1)),
            cooldown(&state.ability_2).on_tap(|| activate(&state.ability_2)),
            cooldown(&state.ultimate).charge(state.ultimate_energy, 100.0),
        ]),

        minimap(&state.level_map)
            .viewport(state.camera.position)
            .radius(50.0)
            .markers(&state.enemies, MarkerStyle::hostile())
            .markers(&state.objectives, MarkerStyle::waypoint())
            .markers(&state.teammates, MarkerStyle::friendly()),

        context_prompt(state.nearby_interactable.as_ref())
            .input(state.keybinds.interact),
    ])
}

fn weapon_wheel(state: &GameState) -> impl View {
    radial(&state.player.weapons)
        .trigger(Hold(state.keybinds.weapon_select))
        .selected(&state.active_weapon)
        .item(|weapon| group([
            image(&weapon.icon),
            text(&weapon.name),
            text(format!("{}/{}", weapon.ammo, weapon.max_ammo)),
        ]))
        .center(|selected| group([
            image(&selected.icon),
            text(&selected.name),
        ]))
        .on_select(|w| state.active_weapon.set(w))
}

fn skill_tree(state: &GameState) -> impl View {
    graph(&state.skills)
        .edges(|skill| &skill.prerequisites)
        .layout(GraphLayout::TopToBottom)
        .camera(Camera::Pannable { zoom: true })
        .node(|skill| group([
            image(&skill.icon),
            text(&skill.name),
            indicator(&skill.state),
            text(format!("{} pts", skill.cost)),
        ])
        .on_tap(|| when(
            skill.state == SkillState::Available && state.points >= skill.cost,
            || {
                skill.state.set(SkillState::Acquired);
                state.points.decrement(skill.cost);
            }
        )))
}
```

### World-Space and Diegetic UI in Rust

```rust
fn spatial_markers(state: &GameState) -> impl View {
    world_space([
        for_each(&state.visible_enemies, |enemy| {
            world_anchor(enemy.head_position)
                .offset(vec2(0.0, -20.0))
                .distance_fade(50.0, 100.0)
                .content([
                    meter(enemy.hp, enemy.max_hp).style(MeterStyle::Minimal),
                    text(&enemy.name),
                    text(format!("Lv.{}", enemy.level)),
                ])
        }),

        world_anchor(state.objective.position)
            .content([
                image("waypoint"),
                text(format_distance(state.objective.distance)),
            ])
            .off_screen(edge_arrow(bearing_to(&state.objective))),
    ])
}

fn terminal_screen(terminal: &Terminal) -> impl View {
    render_to(terminal.screen_mesh, [
        text("SYSTEM ACCESS").title(),
        list(&terminal.files).item(|f| text(&f.name)),
        indicator(&terminal.network_status),
    ])
    .shader(CrtScanline)
    .shader(ScreenFlicker { intensity: 0.02 })
}
```

### Entry Point

```rust
fn main() {
    vibe::run(App {
        root: catalog,
        flow: navigation,
        style: Style::new()
            .primary(0x2563eb)
            .surface(0xfafafa)
            .radius(12.0)
            .density(Density::Comfortable),
    });
}
```

---

## 12. Prior Art & Related Research

### GPU-First Rendering

- **Immediate-mode GUIs** (Dear ImGui, egui) — no retained widget tree, "draw this frame." AI-friendly but historically limited to tooling/games.
- **Flutter Impeller / Rive** — owns the pixel pipeline, no platform widgets. Closer to "GPU primitives + events."
- **Vello/Xilem** (Linebender, Rust) — GPU-first 2D renderer with minimal retained scene graph.
- **Pathfinder / Vello** — GPU path rendering research.
- **SDF text rendering** (Valve 2007, multi-channel SDF) — efficient GPU text at any scale.

### Layout & Constraint Systems

- **Cassowary** (Auto Layout) — declarative spatial relationships via linear constraint solving.
- **Penrose** (CMU) — declarative diagramming with constraint solving.
- **Yoga** (Meta) — cross-platform flexbox implementation (used in React Native).

### Specification-Driven UI

- **SUPPLE** (Harvard, 2004) — UI as optimization from functional specification. Solution spaces 10^17+, <1s generation.
- **CAMELEON / MBUID** (W3C) — four-level abstraction from tasks to final UI.
- **ConcurTaskTrees** (Paterno, 1997) — task modeling with automatic UI derivation.
- **IFML** (OMG, 2014) — interaction flow modeling standard.
- **USIXML / MARIA** — platform-agnostic UI description languages.
- **teleportHQ UIDL** — JSON-based UI definition with multi-framework code generation.

### AI-Driven UI Generation

- **Google A2UI** (2025) — open protocol for agent-driven declarative UI. Security-first, framework-agnostic.
- **Google Generative UI** — LLM-produced custom UIs. PAGEN benchmark. Deployed in Gemini.
- **Jelly** (CHI 2025) — task-driven malleable UIs via LLM-inferred data models.
- **MCP Apps / Open-JSON-UI** — protocols for agents returning interactive UI components.
- **v0 (Vercel)** — NL → React/Tailwind. ~6M developers.
- **PrototypeFlow** — NL → component DSL → editable SVG prototypes.
- **SpecifyUI / AlignUI** — iterative intent expression and user-preference injection for generative UI.
- **UICoder** — finetuning LLMs for SwiftUI generation from NL.
- **Adaptive UI via RL** (2024) — Markov decision processes for layout adjustment.

### Discourse

- "The Next Era of Design is Intent-Driven" (UX Collective)
- "From UI (User Interface) to UI (User Intent)" — 2025 UX trend
- Headless Business Applications — decouple logic from presentation
- Zero UI / Screenless Interfaces — voice, gesture, contextual AI
- Spec-Driven Development (Marc Brooker, 2026) — specifications as primary artifact

### Research Gap

Two gaps remain underexplored:
1. **What representation minimizes generation error for AI while remaining performant and accessible?** Optimizing the *target* for AI generation quality.
2. **Directive-to-GPU-primitive compilation without intermediate widget frameworks.** All existing systems target DOM/Flutter/native. None targets a flat GPU scene buffer directly.

---

## Open Questions

1. **Compiler architecture:** How does the directive compiler decide layout? Rule-based templates? Constraint solver? Trained model? Hybrid?
2. **Escape hatch:** When directives can't express something, how does the author drop to scene-buffer level without breaking the abstraction?
3. **Async model:** In-flight fetches, cancellation, race conditions in the effect system.
4. **Undo/redo:** Flat state store → cheap snapshots. Expose as first-class directive?
5. **Optimistic updates:** Tentative mutations with rollback on server rejection.
6. **Animations:** GPU-side transitions for property changes. Enter/exit animations for conditional nodes (slots, list items).
7. **Theming:** Semantic tokens work for colors/spacing. What about layout density, information density, motion preferences?
8. **Responsiveness:** Directive compiler adapts per device, or explicit responsive hints in DSL?
9. **Custom views:** Can users define new view kinds (like "calendar", "map", "chart") and register compilers for them?
10. **Collaboration:** Multiple directive files composed together? Import/export?
11. **Testing:** How do you test directives without a visual renderer? Property-based testing of compiler output?
12. **Versioning:** When the compiler improves, does the same directive produce a different (better) UI? Is that acceptable?
