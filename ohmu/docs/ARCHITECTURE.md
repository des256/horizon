# Architecture

A UI toolkit designed for AI generation, built on GPU primitives rather than human-oriented widget abstractions.

## Core Thesis

Current UI frameworks (React, Flutter, SwiftUI, etc.) are compression schemes for human cognition — components, props, lifecycle hooks, the cascade in CSS — all exist because humans can't hold the full pixel-level state in working memory. AI doesn't have that limitation. When told to generate UI directly against rendering primitives, AI is more effective because there's no translation layer it doesn't need.

Can we design a system that goes back to basic structure (GPU graphics on a touchscreen with events) and is much easier on AI, allowing a cleaner and more expressive way to encode user needs?

## System Overview

```
┌─────────────────────────────────────────────────────────┐
│  Authoring (choose one or both)                          │
│                                                          │
│  Rust API                      .ohmu interpreter         │
│  fn catalog() -> impl View     parse("catalog.ohmu")     │
│       │                             │                    │
│       └───────────┬─────────────────┘                    │
│                   │                                      │
│           Intent Tree (in-memory)                        │
│           ViewNode::List { source, filter, item }        │
│           ViewNode::Form { fields, submit }              │
│           ViewNode::Nav { destinations }                 │
└─────────────────┬────────────────────────────────────────┘
                  │
┌─────────────────▼────────────────────────────────────────┐
│  Target Compiler (pattern expansion + layout)             │
│  Intent Tree + TargetProfile → Scene Buffer               │
└─────────────────┬────────────────────────────────────────┘
                  │
┌─────────────────▼────────────────────────────────────────┐
│  GPU Runtime                                              │
│  Compute (layout) → Render (SDF quads + text)             │
└──────────────────────────────────────────────────────────┘
```

### Layer breakdown

```
┌──────────────────────────────────────────┐
│  AI / User intent                         │
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

## Two Modes of Use

1. **AI generates directives** → compiler produces scene buffer → GPU renders
   - AI works at highest abstraction (intent). Compiler handles structure. GPU handles pixels.
   - Best for: apps, forms, CRUD, dashboards, standard workflows.

2. **AI generates scene buffer directly** → GPU renders
   - AI works at mid-level (structure). Bypasses directive compilation.
   - Best for: custom visualizations, creative layouts, game-like interfaces, anything the directive vocabulary can't express.

Both target the same runtime. The directive layer is optional — it raises the ceiling for common cases without limiting escape to raw scene manipulation.

## Hybrid Authoring: Rust API + `.ohmu` Interpreter

```
Production path:
  Rust source → cargo build → native binary with compiled views
  (fastest, type-safe, optimized)

Development path:
  .ohmu file → runtime interpreter → intent tree → scene buffer
  (hot reload, no compile wait, rapid iteration)

AI generation path:
  LLM → generates either Rust or .ohmu depending on context
  (both target the same intent tree → same scene buffer)
```

The `.ohmu` format is a serialization format for the intent tree, not a separate language. The Rust API builds the same tree in-memory.

## Accessibility

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

Text input and IME are where "rawdog GPU" hits a wall. Every OS has a deeply integrated text services framework. Design decision: treat text input as a platform primitive you embed, not something you render yourself. Everything else is GPU-native.

## What's Eliminated Globally

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
