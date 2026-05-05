# ohmu-dsl

Parser, semantic analyzer, and compiler for the directive DSL. Transforms intent-level declarations into scene buffers. Depends on `ohmu-core` for scene buffer types and TargetProfile.

## What lives here

- **Lexer + parser** — tokenizes and parses the directive DSL grammar into an AST. Handles data declarations, screens, flows, effects, styles.
- **Type system** — type checking for data declarations (bool, int, float, text, enum, list, record, option, ref), expression type inference, binding type compatibility.
- **Semantic analysis** — dead-state detection (declared data not referenced by any view/effect), flow completeness (every screen reachable), orphan mutation detection (mutations targeting undeclared data), ref validity.
- **Pattern expansion** — maps view vocabulary (text, input, list, action, choice, toggle, indicator, group, slot) and higher-level patterns (form, split, nav, dashboard, chat, feed, etc.) into abstract layout trees. This is where TargetProfile influences structure: `nav` becomes bottom tabs on phone, sidebar on desktop. `split` becomes side-by-side or drill-down.
- **Layout assignment** — converts abstract layout trees into concrete scene buffer nodes with distribution, size policy, spacing, alignment. Informed by viewport size and density.
- **Style application** — maps semantic style tokens (primary color, radius, density) to per-node visual properties (fill, border radius, shadow, text size/weight).
- **Binding generation** — creates state bindings, handler mappings, and transition configurations from the directive declarations.
- **Runtime recompilation** — when TargetProfile changes (rotation, resize, fold state), re-runs pattern expansion and layout assignment on the current AST. State is preserved; only visual structure updates.

## Dependencies

- `ohmu-core` — scene buffer types, TargetProfile, layout primitives, state/binding types.
- No platform dependencies. Compiles everywhere.

## Compilation pipeline

```
DSL source text
  → Lexer → tokens
  → Parser → AST
  → Semantic analysis → validated AST (errors reported)
  → Pattern expansion (+ TargetProfile) → abstract layout tree
  → Layout assignment (+ TargetProfile) → scene buffer nodes
  → Style application (+ style tokens + TargetProfile) → styled nodes
  → Binding generation → scene buffer with bindings + handlers
  → Output: complete SceneBuffer ready for GPU
```

## Testing tiers

### Tier 1 — Pure logic (primary)

Everything in this crate is testable with `cargo test`. The DSL compiler is a pure function: `(source_text, TargetProfile) → SceneBuffer`.

**Parser:**
- Round-trip tests: parse → pretty-print → re-parse → compare ASTs.
- Error recovery: malformed input produces useful error messages with source locations.
- Edge cases: empty screens, nested patterns, complex pipe expressions, all view types.

**Semantic analysis:**
- Dead state detection: declared but unreferenced data triggers warning.
- Type errors: binding a bool state to a text view, ref to nonexistent data.
- Flow validation: unreachable screens, missing transitions.
- Orphan mutations: handler mutates undeclared state slot.

**Pattern expansion (parametric across TargetProfiles):**
- Same directives compiled with different TargetProfiles produce structurally different scene buffers.
- `nav` → bottom tabs (phone) vs sidebar (desktop) vs page dots (watch).
- `list` → vertical scroll (phone) vs grid (tablet) vs horizontal carousel (TV).
- `split` → side-by-side (wide) vs drill-down navigation (narrow).
- `form` → single column (phone) vs two-column (desktop) vs wizard (watch).
- These are pure struct-in/struct-out tests. Hundreds of profile combinations run in milliseconds.

**Full compilation:**
- Known directive sources compiled to scene buffers, assert on node count, tree structure, binding count, handler assignments.
- The shopping app example from IDEA.md is a comprehensive integration test.

**Recompilation:**
- Compile with portrait profile, then recompile with landscape profile. State store unchanged, scene buffer structure differs.
- Verify no state loss across recompilation.

### Tier 2 — Visual regression (secondary)

Feed compiled scene buffers to `ohmu-gpu` for rendering. Compare rendered output for the same directive across different TargetProfiles:

- Phone portrait vs tablet landscape for the same app.
- Light vs dark style tokens.
- Normal vs high contrast.
- Default vs 2x text scale.

These tests verify that the compiler's layout decisions look correct when actually rendered, not just structurally correct.

### Tier 3 — N/A

No platform integration needed. The compiler is a pure transformation.
