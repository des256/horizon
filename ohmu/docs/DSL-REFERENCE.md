# DSL Reference

The view and pattern vocabulary for describing UI intent. The primary authoring surface is Rust (builder-pattern functions implementing the `View` trait). An `.ohmu` interpreter provides the same vocabulary as a serialization format for hot-reload and AI generation.

## Pattern Count

- **Tier 1 (atomic):** 8 views
- **Tier 2 (collections):** 4 views
- **Tier 3 (app patterns):** 24 patterns
- **Tier 3 (game patterns):** 13 patterns
- **Tier 4 (embeds):** 7 embed types
- **FX sublanguage:** continuous animation, particles, shaders, audio

Total: ~56 named concepts covering both traditional apps and game interfaces, all compiling to the same GPU scene buffer.

---

## Tier 1: Atomic Views (irreducible)

| View | Meaning | Compiler decides |
|------|---------|-----------------|
| `text(expr)` | Display a value | Font size, weight, color, truncation |
| `input(bind, ...)` | Editable value (text, number, date, etc.) | Box style, label placement, clear button |
| `image(src)` | Display visual media | Sizing, aspect ratio, placeholder |
| `action(label, on_tap)` | Something the user can activate | Button shape, haptics, loading state |
| `toggle(bind, label)` | Boolean switch | Checkbox, switch, chip toggle |
| `choice(source, selected)` | Pick from options | Chips, dropdown, radio, segmented control |
| `indicator(expr, ...)` | Display a status or metric | Badge, dot, bar, color |
| `slot(conditions...)` | Conditional content | Transition, placeholder |

## Tier 2: Collection Views

| View | Meaning |
|------|---------|
| `list(source, item)` | Linear scrollable collection |
| `tree(source, children, item)` | Hierarchical expandable collection |
| `grid(source, item, cols?)` | 2D matrix of items |
| `table(source, columns)` | Columnar structured data |

## Tier 3: App Patterns

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
| `group(...)` | Logical grouping | Card, row, section, expandable |
| `for_each(source, item)` | Iteration over collection in world-space contexts | Per-item content |

## Tier 3: Game Patterns

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

## Tier 4: Embeds

Platform/native components — declared in DSL, rendered by registered implementation.

| Embed | Why it needs native support |
|-------|---------------------------|
| `embed("map", ...)` | Tile rendering, geographic gestures, POI layers |
| `embed("chart", ...)` | Axis math, data-to-pixel mapping, interactive tooltips |
| `embed("calendar", ...)` | Date math, locale day/month names, complex overflow grid |
| `embed("rich_editor", ...)` | Formatting toolbar, cursor management, paste handling |
| `embed("video", ...)` | Codec decoding, media session, DRM |
| `embed("canvas", ...)` | Freeform drawing, stroke undo, pressure sensitivity |
| `embed("code_editor", ...)` | Syntax highlighting, language services |

The runtime has a registry of embed renderers. The scene buffer allocates a viewport rect; the platform renders into it.

---

## The `fx` Sublanguage (Effects & Feedback)

Views gain an `fx` property for continuous visual/audio feedback:

```rust
meter(state.player.hp, state.player.max_hp)
    .fx(|fx| fx
        .when(|v| v < 0.25, pulse(Color::RED, hz(2.0)))
        .when(|v| v < 0.10, screen_vignette(Color::RED, 0.3))
        .on_decrease(flash(Color::WHITE, secs(0.1)))
        .on_increase(particle(HealSparkle, 5))
    )
```

`fx` declarations compile to combinations of:
- Continuous animations (pulse, glow, wobble)
- Particle emitters (burst, stream, toast)
- Shader variants (vignette, dissolve, energy)
- Audio triggers (sound IDs mapped to events)
- Screen-space effects (shake, flash — applied to root node)

---

## Core Trait

```rust
/// Anything that can become UI
trait View {
    fn build(&self, ctx: &mut BuildContext) -> ViewNode;
}

/// Every pattern implements View
impl<T> View for List<T> { ... }
impl View for Form { ... }
impl View for Nav { ... }
// ...etc

/// Tuples and arrays compose automatically
impl<V: View, const N: usize> View for [V; N] { ... }

/// Runtime entry point
fn run(root: impl View, target: TargetProfile, style: Style) {
    let intent_tree = root.build(&mut BuildContext::new());
    let scene = compile(intent_tree, &target, &style);
    gpu::run_loop(scene);
}
```

## Proc Macros

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

## Screens as Functions

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
```

## Flows and Effects

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
```

## Styles (Theming, Not Layout)

```rust
Style::new()
    .primary(0x2563eb)
    .surface(0xfafafa)
    .radius(12.0)
    .density(Density::Comfortable)
    .font("Inter")
```

Styles are semantic tokens, not CSS. The compiler uses them to fill in visual properties on scene buffer nodes. `density` affects spacing/padding uniformly. `radius` applies to all interactive elements. No per-node styling in the directive layer.

## Entry Point

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

## App Type Coverage

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

## Game Genre Coverage

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
