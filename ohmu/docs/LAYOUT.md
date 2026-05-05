# Layout System

GPU compute shader layout using a two-pass model. A CPU reference implementation defines correctness; the shaders are a parallel optimization of the same algorithm.

## Two-Pass Model

- **Pass 1 (bottom-up, leaf → root):** Each node computes intrinsic size. Leaves report fixed size or pre-measured text bounds. Parents aggregate children based on distribution axis (sum for stacks, max for layers). Parallel per tree level.
- **Pass 2 (top-down, root → leaf):** Each node receives allocated rect from parent and distributes to children. Root gets viewport. Parallel per tree level.

Two dispatches per level of depth. Typical UI is 8-15 levels → 16-30 dispatches per frame.

## Compute Shader Structure

```wgsl
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

### Radial Layout (Game Extension)

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

## GPU-Friendly vs. Needs CPU Assist

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

## Why GPU Layout?

For typical UI node counts (200-2000), CPU layout is already sub-millisecond. The GPU advantage is architectural:

1. **No CPU→GPU position upload.** Position data is born on GPU, stays on GPU.
2. **GPU-driven animation.** Springs, transitions, interpolation in compute shaders — no CPU roundtrip per frame.
3. **Simplicity.** One buffer, one pipeline. No split ownership.
4. **Scaling.** For data visualization with 10K+ nodes, parallelism becomes genuine.
