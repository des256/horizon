# State Model

State is a flat typed store. The scene buffer has binding slots that reference state. When state changes, a small CPU reconciler patches affected nodes.

## Store

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

## Mutations

What handlers produce:

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

## Bindings

How state reaches the scene:

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

## Reconciliation Loop

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

## Common Scenarios

### Toggle (show/hide sidebar)

```
State: sidebar_open: bool = true
Handler: on_tap(toggle_btn) → Toggle(sidebar_open)
Binding: Property { node: sidebar, prop: Visible, state: sidebar_open }
Transition: width animates 250 → 0 over 200ms
```

### Todo list

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

### Form with validation

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

### Navigation

```
State: current_page: Enum(Home | Profile | Settings)

Binding: Conditional {
    page_container ← current_page: [home_subtree, profile_subtree, settings_subtree]
}

Handler: on_tap(nav_profile) → Set(current_page, Profile)
```

### Async loading

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

### Drag to reorder

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

## What's Eliminated vs. Frameworks

| Framework concept | Why it existed | Gone because |
|-------------------|---------------|--------------|
| `useState` / `setState` | Track component state ownership | Global flat store, not component-scoped |
| `useEffect` / lifecycle | Side effects on mount/change | Handlers fire on events. Async is explicit. |
| `useMemo` / `React.memo` | Avoid recomputing during re-renders | No re-renders. Bindings are direct writes. |
| Context / prop drilling | Pass state through component tree | Any binding references any slot directly |
| Immutability / reducers | Enable diff detection | Dirty bits on slots. Mutation is fine. |
| Component boundaries | Human code organization | AI doesn't need them. Flat scene. |
| Keys for list reconciliation | Diff algorithm needs identity | List items have state slot IDs — identity is intrinsic |
