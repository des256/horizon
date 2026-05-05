# Open Questions

Unresolved design decisions from the original discussion. Annotated with current status where later sections provided partial answers.

1. **Compiler architecture:** How does the directive compiler decide layout? Rule-based templates? Constraint solver? Trained model? Hybrid?
   - *Partially addressed:* the compilation registry (see [MULTI-TARGET.md](MULTI-TARGET.md)) supports all three via `LayoutStrategy` (Rule, Template, Solver). The practical default is rule-based with template fallbacks. Solver reserved for complex cases.

2. **Escape hatch:** When directives can't express something, how does the author drop to scene-buffer level without breaking the abstraction?
   - *Partially addressed:* the Rust API allows importing `ohmu::scene::*` to build raw scene buffer nodes alongside directive-generated ones. Tier 4 embeds (`embed("map", ...)`) handle known escape cases. The general mechanism for mixing directive and raw nodes in the same tree needs design.

3. **Async model:** In-flight fetches, cancellation, race conditions in the effect system.
   - *Partially addressed:* effects are `async fn` with cancellation on screen navigation. Detailed concurrency semantics (multiple effects mutating same state, debounce, throttle) need specification.

4. **Undo/redo:** Flat state store → cheap snapshots. Expose as first-class directive?
   - *Open.* The flat `Vec<Value>` store is snapshot-friendly. Whether undo is a runtime capability, a directive declaration, or both is undecided.

5. **Optimistic updates:** Tentative mutations with rollback on server rejection.
   - *Open.* Could be an effect modifier (`optimistic: true` on `fetch`) or a state-level concept (shadow slots that revert on error).

6. **Animations:** GPU-side transitions for property changes. Enter/exit animations for conditional nodes (slots, list items).
   - *Partially addressed:* the `Animation` enum (see [SCENE-BUFFER.md](SCENE-BUFFER.md)) covers state-triggered transitions, continuous loops, driven animations, and procedural shaders. Enter/exit animations for conditional content still need design — when a `slot` switches branches, how does the outgoing branch animate out while the incoming one animates in?

7. **Theming:** Semantic tokens work for colors/spacing. What about layout density, information density, motion preferences?
   - *Addressed:* `TargetProfile` carries `density`, `motion`, `text_scale`, and `contrast`. The `Style` struct handles color/radius/font tokens. The compiler uses both when applying styles.

8. **Responsiveness:** Directive compiler adapts per device, or explicit responsive hints in DSL?
   - *Addressed:* compiler adapts via `TargetProfile`. Optional `hint()` per-pattern for overrides when the compiler's default is wrong. See [MULTI-TARGET.md](MULTI-TARGET.md).

9. **Custom views:** Can users define new view kinds (like "calendar", "map", "chart") and register compilers for them?
   - *Addressed:* Tier 4 embeds with a runtime registry of embed renderers. The `PatternCompiler` registry allows adding new pattern-to-layout strategies. Custom `View` trait implementations in Rust add arbitrary new views.

10. **Collaboration:** Multiple directive files composed together? Import/export?
    - *Open.* Rust module system handles composition for the Rust API path. For `.ohmu` files, an import mechanism is undecided.

11. **Testing:** How do you test directives without a visual renderer? Property-based testing of compiler output?
    - *Addressed:* see the crate documentation (OHMU-CORE.md through OHMU-TEST.md). Tier 1 tests validate compiler output as struct assertions. Property-based tests verify cross-target behavior invariants. Visual regression via Tier 2 headless GPU rendering.

12. **Versioning:** When the compiler improves, does the same directive produce a different (better) UI? Is that acceptable?
    - *Open.* The intent is yes — directives are stable, compilation improves. But this means visual regression tests need a "blessed update" workflow, and users may want to pin compiler versions for production stability. No mechanism designed yet.
