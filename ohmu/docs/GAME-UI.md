# Game UI & 3D Integration

## What Makes Game UI Different

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

## Four Spatial Categories

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

## Game Patterns

See [DSL-REFERENCE.md](DSL-REFERENCE.md) for the full Tier 3 game pattern table. Key patterns:

`hud`, `meter`, `radial`, `inventory`, `dialog`, `graph`, `scoreboard`, `context_prompt`, `cooldown`, `world_anchor`, `stack_mode`, `toast_particle`, `minimap`.

## Examples

### HUD with contextual visibility

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
```

### Radial weapon wheel

```rust
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
```

### Skill tree

```rust
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

### Inventory with drag

```rust
fn inventory(state: &GameState) -> impl View {
    screen([
        inventory_panel(InventoryConfig {
            slots: vec![
                Slot::new("head", ArmorHead),
                Slot::new("chest", ArmorChest),
                Slot::new("legs", ArmorLegs),
                Slot::new("weapon_1", Weapons),
                Slot::new("weapon_2", Weapons),
                Slot::new("accessory", Accessories),
            ],
            equipped: &state.player.equipment,
            on_equip: |slot, item| equip(&state.player, slot, item),
            on_unequip: |slot| unequip(&state.player, slot),
        }),

        inventory_grid(6, 4, &state.player.inventory)
            .drag(true)
            .item(|item| group([
                image(&item.icon),
                when(item.stackable, || text(item.quantity)),
                indicator(&item.rarity).fx(border_glow),
            ]))
            .context_menu(|item| vec![
                action("Use").enabled(item.usable).on_tap(|| use_item(item)),
                action("Drop").on_tap(|| drop_item(item)),
                action("Inspect").on_tap(|| state.inspected_item.set(Some(item))),
            ]),

        slot(state.inspected_item.as_ref(), |item| {
            detail(item, [
                text(&item.name).title(),
                text(&item.description),
            ])
        }),
    ])
}
```

### Dialog with branching

```rust
fn conversation(state: &GameState, npc: &NPC) -> impl View {
    dialog(npc)
        .portrait(image(&npc.portrait))
        .lines(&state.current_dialog_node.text)
        .typewriter(true)
        .choices(&state.current_dialog_node.options, |option| group([
            text(&option.text),
            when(option.requires.is_some() && !meets_requirement(&option.requires), || {
                indicator(Locked);
                text(&option.requires.description)
            }),
        ]))
        .on_choice(|option| advance_dialog(npc, option.next_node))
        .on_end(|| navigate(Route::Gameplay))
}
```

## 3D Integration Architecture

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

### World-space and diegetic declarations

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
