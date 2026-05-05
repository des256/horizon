# Multi-Target Compilation

The directive DSL specifies behavior and intent. It says nothing about device form factor, screen size, input modality, or platform conventions. The **compiler** makes all presentation decisions based on a **target profile**.

Same directives → different scene buffers → same behavior, different presentation.

## Target Profile

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

## How Patterns Adapt

### `nav` — most form-factor-sensitive

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
```rust
fn compile_nav(nav: NavPattern, target: &TargetProfile) -> SceneNodes {
    match (target.form, nav.destinations.len()) {
        (Phone, 1..=5) => bottom_tab_bar(nav),
        (Phone, 6..) => hamburger_drawer(nav),
        (Tablet, _) if target.viewport.x > 600 => side_rail(nav),
        (Desktop, _) => sidebar_full(nav),
        (Watch, _) => page_dots(nav),
        (TV, _) => top_horizontal(nav),
        _ => bottom_tab_bar(nav),
    }
}
```

### `list` — adapts to available space

| Target | Presentation |
|--------|-------------|
| Phone | Vertical scroll, full-width cards or rows |
| Tablet portrait | 2-column grid of cards |
| Tablet landscape / Desktop | 3-4 column grid, or table view for data-heavy items |
| Watch | Single-item pager with crown scroll |
| TV | Horizontal carousel with focus scaling |

Key decision: **when does a list become a grid?** Rule: if items have images AND viewport width > 2× minimum card width, use grid. Otherwise vertical list.

### `split` — the classic responsive pattern

| Target | Presentation |
|--------|-------------|
| Wide (desktop, tablet landscape) | Side-by-side: master on left (1/3), detail on right (2/3) |
| Narrow (phone, tablet portrait) | Drill-down: show master. On select, navigate to detail. Back returns. |
| Watch | Not applicable — degrade to list + detail as separate screens |

In narrow mode, `split` implicitly creates a navigation transition between master and detail.

### `form` — adapts to space and input method

| Target | Presentation |
|--------|-------------|
| Phone | Single column, full-width inputs, large tap targets |
| Tablet/Desktop | Two-column for related fields, inline labels |
| Watch | One field at a time (wizard-style), dictation input prioritized |
| TV | Directional-pad navigation between fields, on-screen keyboard |
| Keyboard-primary | Tab order emphasized, no scroll between fields if they fit |

### `dashboard` — information density scaling

| Target | Presentation |
|--------|-------------|
| Phone | KPIs as horizontal scroll cards, sections as vertical scroll |
| Tablet | 2×2 KPI grid, sections below |
| Desktop | Full grid: KPIs top row, sections in 2-3 columns |
| Watch | Single KPI pager, swipe for next metric |
| TV | Full dashboard visible, no scroll, large type |

### `chat` — input modality adaptation

| Target | Presentation |
|--------|-------------|
| Phone | Full-screen thread, keyboard pushes messages up, send button |
| Desktop | Split view (contacts left, thread right), Enter to send |
| Watch | Voice-to-text input, canned quick replies, minimal history |
| TV | Not typical — degrade to notification display only |

## Compilation Pipeline

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

## Compilation Registry

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

New form factors or pattern-to-layout mappings can be added without modifying the DSL grammar or existing directives.

## Runtime Recompilation

The target profile can change at runtime:
- Device rotation (viewport + orientation change)
- Window resize (desktop)
- Fold state change (foldable devices)
- Accessibility settings change (text scale, contrast, motion)
- Display connection (phone → external monitor)

When the target profile changes, the compiler re-runs pattern expansion and layout assignment on the current AST. State is preserved. Only the visual structure updates.

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

Cost: for a typical app (50-200 nodes), this is microseconds on CPU. Happens only on profile change events, not per frame.

## Cross-Target Behavior Guarantees

| Same across all targets | Varies by target |
|------------------------|-----------------|
| State shape and values | Node arrangement |
| Mutation effects | Visual hierarchy |
| Navigation graph (which screens exist) | Navigation animation / mechanism |
| Data flow (bindings, derived state) | Input method (tap vs click vs d-pad) |
| Effect triggers and async behavior | Touch target sizes |
| Validation rules | Information density |
| Event semantics (tap is tap) | Gesture interpretation (scroll vs swipe-to-navigate) |

## Target-Specific Hints (Optional)

For cases where the directive author knows something the compiler can't infer:

```rust
list(&products)
    .item(|p| group([...]))
    .hint(Phone, ListHint::Rows)
    .hint(Desktop, ListHint::Table)
    .hint(Watch, ListHint::MaxVisible(3))
```

Hints are optional overrides. They break the "pure intent" abstraction slightly but provide an escape valve. They're target-keyed, not pixel-keyed — you never say `width > 768px`, you say `Phone`.

## Watch/Embedded — Extreme Constraints

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

## TV — 10-foot UI

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
