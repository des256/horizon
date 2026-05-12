# ice

A system for rapid 2D JRPG-style game development with LLMs.

## Concept

An LLM-driven multi-agent system generates playable 2D games from high-level story descriptions. A hierarchy of specialized agents — narrative, level design, character design, encounter design, scripting, dialogue — collaborate to produce engine-ready game data. Image generation models render the visual assets. A Rust engine consumes the output and runs it in the browser via WebAssembly.

The target aesthetic is closer to illustrated/storybook than retro pixel art: flat, layered, tolerant of inter-asset variation, expressive despite structural simplicity. The style is a parameter, not a constant — the structural grammar stays fixed while visual language can vary.

## Design Philosophy

### Why 2D JRPG-style

Not nostalgia. Three structural advantages:

1. **Constrained workload.** Hardware limitations of the era forced a small, well-defined asset vocabulary. Limitations breed creativity, and they make LLM generation tractable.
2. **Structurally processable assets.** Tilesets, sprite sheets, and portrait sets are discrete, small, and describable in text. 3D assets are not.
3. **Presentational, not immersive.** Classic JRPGs had a theatrical quality — the player watched a stage rather than inhabiting a world. The gap between simple presentation and grand narrative was productive; the player's imagination filled it. This quality is preservable without the dated aesthetic.

### Game Mechanics

The games ice produces are **action-exploration games with narrative focus**, not stat-manager simulators.

- **Movement:** Direct character control (WASD/cursor/gamepad) across tile-based maps.
- **Combat:** Real-time, in the overworld (Zelda/Ys-style). No zoom-into-battle mode. Enemies have behavior patterns composed from simple primitives (approach, charge, patrol, projectile). Stats exist but are hidden or semi-hidden — the story LLM describes narrative difficulty, the system translates to numbers.
- **Dialogue:** First-class system, well-integrated into the game view. Characters speak through a structured dialogue system supporting branches, conditions, and expression changes.
- **Play scripts:** Dramatic moments are choreographed sequences of engine primitives — move entity, face direction, show dialogue, screen effects, timing. These carry the emotional weight.
- **Interaction:** Characters can touch/collide with other entities to trigger dialogue, combat, item exchange, or scripted events.

What's deliberately excluded: inventory management, crafting, skill trees, party management, turn-based battles. The mechanical layer stays thin so the story LLM doesn't need to be a game designer.

## Agent Hierarchy

### Tier 1: Direction

#### Story Director

The single source of narrative truth. All other agents work from its output.

**Thinks in:** narrative arcs, character motivations, dramatic beats, pacing, world structure.

**Outputs:** a structured game spec consisting of:
- World definition: regions, their thematic identity, how they connect
- Character definitions: appearance descriptors, personality, role in story, stat profile hints ("fragile but fast," "overwhelming force")
- Scene sequence: ordered list of scenes with location, characters present, events, emotional tone, purpose in the arc
- Per-scene specs: high-level spatial description ("autumn forest, branching paths, hidden shrine at the north end"), encounter descriptions ("guardian spirit, tests the player, not lethal"), dialogue beats, dramatic moments to script

**Constraints enforced:** every scene must decompose into a map + entity list + trigger list + script list. The Story Director knows the engine's capability grammar — if the engine can't do it, the Director can't write it.

**Does not do:** tile placement, sprite design, damage numbers, exact positioning, animation specifics. It specifies *what and why*, never *how*.

### Tier 2: Domain Specialists

Each owns one production domain. They take the Story Director's spec and produce concrete, engine-ready data. They have deep knowledge of their domain's constraints and best practices.

#### Level Designer

**Input:** Story Director's spatial description + atmosphere + narrative purpose of the area.

**Domain knowledge:** tile composition, spatial pacing, navigability, sight lines, chokepoints, secret placement, how exploration rhythm works, how to guide the player without explicit markers.

**Output:** tilemap (tile IDs on a grid), collision layer (walkable/blocked), trigger zones (areas that fire events), spawn points, transition points (doors/exits to other maps), decorative object placement.

**Key responsibility:** translating vague spatial intent ("a maze") into playable space that serves the narrative purpose. Pushes back when the spec doesn't work spatially — "a maze doesn't fit the tile budget, but branching paths with a hidden shortcut achieve the same feel."

#### Character Designer

**Input:** Story Director's character definition — appearance, personality, role.

**Domain knowledge:** sprite sheet structure (idle/walk/attack/hurt/death × N directions), animation frame counts, visual personality (how posture and silhouette convey character), expression vocabulary for portraits.

**Output:** structured character asset spec that the Tier 3 renderers consume — detailed visual descriptions per sprite state, portrait expression list, color palette constraints. After rendering: validated sprite sheet + portrait set conforming to engine format.

**Key responsibility:** visual consistency within a character and across the cast. A village of NPCs should feel like they belong together. The villain should read as a villain from the silhouette.

#### Encounter Designer

**Input:** Story Director's encounter description — narrative purpose, intended difficulty feel, context.

**Domain knowledge:** enemy behavior primitives available in the engine, difficulty curves, attack telegraphing, fairness principles, phase design, how to make encounters feel distinct using limited primitives.

**Output:** enemy definitions — behavior pattern (composed from engine primitives: approach player, charge-and-pause, patrol path, fire projectile in pattern, flee at health threshold), stat block (HP, damage, speed, knockback), phase transitions, drop/reward.

**Key responsibility:** translating narrative intent ("tests the player," "overwhelming horde," "puzzle boss") into mechanically sound encounters using only the engine's behavior vocabulary.

#### Script Director

**Input:** Story Director's dramatic beat descriptions — what happens emotionally, which characters are involved, what the player should feel.

**Domain knowledge:** cinematic pacing using engine primitives, beat timing, how to create tension/release/surprise with entity movement + dialogue + screen effects, when to take and return control from the player.

**Output:** play scripts — ordered sequences of engine commands: `move_entity(who, where, speed)`, `face(who, direction)`, `dialogue(who, text, expression)`, `wait(duration)`, `screen_shake(intensity)`, `fade(color, duration)`, `play_effect(what, where)`, `set_flag(key, value)`.

**Key responsibility:** the emotional delivery of the story. The Script Director is the difference between "two characters talk" and "a dramatic confrontation." Works within tight constraints (no custom animations, no camera angles the engine doesn't support) but uses timing, positioning, and pacing to create impact.

#### Dialogue Writer

**Input:** Story Director's scene context, character profiles, emotional beats, player choice points.

**Domain knowledge:** voice consistency per character, exposition economy (show don't tell), meaningful choice design (no false choices), tone management, how to convey information through subtext, how text pacing works without voice acting.

**Output:** dialogue trees — structured text with branches, conditions (based on game flags/state), expression tags per line, speaker tags, and optional choice nodes with consequences.

**Key responsibility:** every character sounds like themselves across the entire game. Exposition is woven into conversation, not dumped. Player choices have visible consequences.

### Tier 2: Validators

These agents don't generate content — they audit it.

#### Continuity Tracker

**Maintains:** a structured record of all established facts — which characters have met, what the player knows, what items they hold, what flags are set, what has been said.

**Validates:** every agent's output against the fact store. Catches: dialogue referencing events that haven't happened on this path, characters appearing in locations they can't reach, flags checked but never set, items required but never given.

**Why it exists:** LLMs are bad at long-range consistency. Externalizing state into a structured store and checking mechanically is more reliable than hoping any single agent remembers everything.

#### Pacing Analyst

**Monitors:** the structural shape of the game as a sequence — scene types, combat density, dialogue-to-exploration ratio, time since last player choice, time since last dramatic beat.

**Feeds back to:** the Story Director, before domain agents begin work. "This section has three dialogue-heavy scenes in a row — consider a breather or exploration scene." "The player hasn't had a meaningful choice in four scenes." "Combat density is front-loaded; the mid-game has no encounters for a long stretch."

**Why it exists:** individual scenes can be excellent while the overall rhythm is terrible. Pacing is a structural property that's hard to evaluate from inside any single scene.

#### Atmosphere Director

**Input:** Story Director's scene descriptions and emotional context.

**Output:** per-scene atmosphere descriptors consumed by the Level Designer (color palette, lighting mood, particle effects like rain/dust/fireflies), the Character Designer (expression defaults), and the Tier 3 renderers (style intensity, palette constraints).

**Why it exists as a separate agent:** atmosphere is cross-cutting. It affects tiles, sprites, effects, and dialogue tone simultaneously. Having each domain agent independently interpret "melancholy" produces incoherent results. A single agent making atmosphere decisions that all others follow produces a unified mood.

### Tier 3: Asset Renderers

Image generation models (diffusion, etc.) that produce actual pixels. Not LLM agents — they're generation models with strict output format constraints.

**Input:** structured visual descriptions from Tier 2 agents — detailed enough to generate from, constrained to the engine's format requirements.

**Output formats:**
- Sprite sheets: fixed grid dimensions, N animations × M frames × D directions, consistent scale
- Portraits: fixed dimensions, K expressions per character, consistent style within character
- Tilesets: fixed tile size, organized by type (ground, wall, obstacle, interactive, decorative)
- Effects: sprite-based particle/animation sheets, fixed frame count

**Key constraint:** every character the Story Director invents triggers a fixed-size asset request. "New character" = one sprite sheet template + one portrait template. The pipeline enforces this — no unbounded asset generation.

## Agent Coordination

### Information Flow

```
                    Story Director
                         │
                    Pacing Analyst ←──── (structural feedback)
                         │
                    Scene Specs
                         │
              ┌──────────┼──────────────┬─────────────┐
              │          │              │             │
         Level Designer  │    Encounter Designer    Script Director
              │     Character Designer  │             │
              │          │              │        Dialogue Writer
              │          │              │             │
              └──────────┼──────────────┴─────────────┘
                         │
                  Atmosphere Director ──→ (palette/mood to all Tier 2)
                         │
                  Continuity Tracker ──→ (validation pass on all output)
                         │
                  Consistency Check ──→ (cross-domain: do the outputs fit together?)
                         │
                   Asset Renderers (Tier 3)
                         │
                  Engine Schema Validation
                         │
                    Playable Game
```

### Coordination Protocol

**Phase 1 — Story pass.** The Story Director produces the full game spec. The Pacing Analyst reviews it and feeds back structural suggestions. The Director revises. This loop runs until the Pacing Analyst passes.

**Phase 2 — Atmosphere pass.** The Atmosphere Director reads the approved spec and emits per-scene atmosphere descriptors. These are distributed to all domain agents before they begin.

**Phase 3 — Domain production.** Domain agents work in parallel where independent, sequentially where dependent:
- Level Designer and Character Designer can work in parallel (maps and characters are independent).
- Encounter Designer depends on Level Designer output (needs to know the map layout to place enemies and define patrol paths).
- Script Director depends on Level Designer (needs walkable positions) and Character Designer (needs to know available animations/expressions).
- Dialogue Writer depends on Script Director (dialogue is placed within the scripted sequences) or can work in parallel if dialogue is standalone.

**Phase 4 — Validation.** The Continuity Tracker checks all outputs against the fact store. A Consistency Checker verifies cross-domain coherence: scripts don't move characters to unwalkable tiles, dialogues don't reference locations absent from the map, encounters don't use behaviors the enemy's sprite can't animate. Failures route back to the relevant domain agent — not to the Story Director, unless the conflict is narrative.

**Phase 5 — Asset rendering.** Tier 3 renderers produce pixels from validated Tier 2 specs. Output is validated against engine format requirements (correct dimensions, valid tile IDs, frame counts).

**Phase 6 — Engine assembly.** The Rust engine's build step consumes all validated data, runs schema validation, and produces the playable game.

### Feedback and Conflict Resolution

- **Domain → Story Director:** only for narrative-level conflicts ("this scene is impossible within engine constraints"). Domain agents solve their own domain problems.
- **Validator → Domain agent:** for factual/consistency errors. Agent revises and resubmits.
- **Cross-domain conflicts** (e.g., map too small for the scripted sequence): resolved by the agent whose output is cheaper to change. Usually the Script Director adjusts choreography to fit the map, not the other way around.

## Engine Requirements

The Rust engine is not just a renderer. It is the **runtime + schema definition + validator**. Its game data format is the interface contract for the entire agent hierarchy.

### Schema as Contract

The engine defines the complete vocabulary of what a game can contain:
- Tile types and their properties (walkable, blocking, interactive, animated)
- Entity types (player, NPC, enemy, object) and their required components
- Behavior primitives (movement patterns, attack patterns, triggers)
- Script commands (the complete set of play script operations)
- Dialogue structures (nodes, branches, conditions, expression tags)
- Effect types (particles, screen effects, transitions)

Every agent's output must validate against this schema. The schema is the single source of truth for "what can this game do." Changing engine capabilities propagates as a schema change that all agents learn about.

### Runtime Capabilities

- Tile-based map rendering with layering (ground, objects, overhead)
- Sprite animation system (state machine driven by entity state)
- Real-time collision and physics (simple: AABB, no complex physics)
- Dialogue rendering with expression support and branching
- Play script execution (sequence player for dramatic moments)
- Enemy AI from behavior primitives (state machine composed from defined patterns)
- Game state management (flags, variables, inventory if minimal)
- Scene transitions (fade, slide, cut between maps)
- Input handling (keyboard, gamepad)

### Build Target

WebAssembly. The output is a self-contained web page: Rust compiled to WASM, assets bundled, playable in a browser with no installation.

## Game Generation Flow

From the user's perspective:

1. **User provides a story premise.** Could be a sentence ("a wandering healer discovers the plague is manufactured") or a detailed outline. Optionally specifies style preferences.

2. **Story Director iterates with the user.** Expands the premise into a structured game spec — world, characters, scenes, arc. The user reviews and refines. This is the main creative loop.

3. **Automated production pipeline runs.** Once the spec is approved, Tier 2 agents and Tier 3 renderers produce all game data. Validators ensure coherence. Failures are resolved automatically where possible, flagged to the user when not.

4. **Engine assembles the game.** Schema validation, asset bundling, WASM compilation.

5. **User plays and iterates.** Plays the generated game, identifies what to change, returns to step 2. The system re-generates only the affected portions — changing a scene's dialogue doesn't require re-rendering all tilesets.

## Research Directions

These exploit a shared structural property: LLM generation is cheap per-variation, so game elements that were traditionally static (because authoring each variation was expensive) can become **continuous functions of game state**. The game state space is continuous and high-dimensional; the LLM generates content for arbitrary points in that space. Content becomes a function, not a table.

### Subjective Rendering

The world renders through the character's perception, not as objective space. Any internal state — psychological condition, skill, emotion, relationship, knowledge, physical condition, worldview — can be projected outward onto every engine subsystem. The core function is `render(base_state, perception_vector) → perceived_state`, where the perception vector is high-dimensional and continuous.

#### Psychological Conditions

A character's DSM-V-style traits distort specific engine subsystems:

| Trait | Engine subsystem affected | Example |
|---|---|---|
| Agoraphobia (0.0–1.0) | Map geometry, tile scale, crowd density | Market square at 0.9: vast, cold, shouting crowds blocking the way. At 0.2: cozy, sunlit, manageable. |
| Social anxiety | NPC behavior, ambient dialogue, gaze direction | NPCs look at you, whisper when you pass, conversations stop when you approach. |
| Narcissism | Dialogue option filtering | Only egotistical dialogue options are available. The character literally cannot choose humility. |
| Depression | Color palette, animation speed, music tempo | Colors desaturate, warm interiors feel empty, the world moves slowly. |
| Paranoia | Fog of war, enemy spawn perception, NPC trust signals | Are those enemies real? Is that NPC actually following you? The player doesn't know either. |
| ADHD | UI focus, quest marker reliability, distraction events | Quest markers drift, side details compete for attention, focus mechanic required for key moments. |
| PTSD | Trigger zones, map warping, sound layering | Specific stimuli flashback-warp to a different map. The present and past overlay. |
| Body dysmorphia | NPC rendering | Everyone else renders as the opposite of the character's self-perception. |

**Therapeutic arc as difficulty curve.** The character's condition IS the difficulty. Early-game agoraphobia makes crossing the market square genuinely hard through spatial distortion. As the trait improves, the space becomes easier. The difficulty curve is embedded in the narrative arc. The player feels progress because the game literally changes.

**Subjectivity as information asymmetry.** The player doesn't know what's real. Is that NPC actually whispering about you (social anxiety), or is the character projecting? The game plays with this — sometimes perception is accurate (someone IS plotting), sometimes not. The player must learn when to trust their own character's senses. The LLM decides per-instance whether perception matches reality based on story context.

**Empathy through forced perspective.** The player doesn't read about agoraphobia — they experience spatial distortion that makes open areas genuinely unpleasant to navigate. Games can do this where no other medium can: make the audience experience a subjective state through mechanics.

#### Skill and Proficiency

Character competence reshapes the environment. Skill progression feels visceral because the world itself changes, not a number on a stat screen:

- **Combat mastery** → enemies telegraph more obviously, move slower/clumsier from the character's perspective. The same enemy at skill 0.3 is fast and unpredictable; at skill 0.9 their patterns are visible and their movements feel sluggish.
- **Stealth** → more cover objects (boxes, hedges, shadows) appear in the maps. Sight lines become visible. The world literally provides more options as the character learns to see them.
- **Perception/awareness** → hidden things become visible as actual tile changes. Low perception: a plain wall. High perception: the crack in the mortar, the draft under the door, the scratches from a moving bookshelf. The secret passage was always in the base map; perception reveals it.
- **Crafting/trade knowledge** → a blacksmith sees ore quality in rocks, grain in wood, structural weakness in a bridge. A herbalist sees medicinal plants the fighter walks past. Different tile variants for the same base map, selected by expertise.
- **Social/charisma** → NPCs orient toward you, body language opens, crowds part. At low charisma, backs are turned, conversations are hard to initiate, doors are closed. The same town renders as welcoming or indifferent.
- **Magical attunement** → ley lines, energy flows, enchantments become visible. An entire world layer that non-magical characters cannot see. Two characters in the same space see different worlds.

#### Physical and Bodily States

- **Exhaustion** → distances stretch. The path renders as longer (more tiles, repeated sections). Rest spots glow invitingly. The environment expands rather than the character slowing.
- **Hunger** → food-related objects become more saturated and prominent. An NPC's market stall that was background decoration becomes the loudest thing in the scene. The character's attention is hijacked.
- **Injury/pain** → the world narrows. Peripheral tiles darken or lose detail. The narrator becomes muffled. Focus collapses to the immediate — ambushes are more dangerous because you can't see as much of the map.
- **Sickness/poison** → color distortion, tile wavering, unreliable NPC rendering (one guard or two?). The player must act while information is degraded.

#### Knowledge and Cognition

- **Ignorance as rendered assumption.** Locations not yet visited render based on what you've been told. An NPC says "the eastern forest is full of monsters" — looking east shows a dark, threatening forest. Visit it: pleasant meadow. Preconceptions are the visible world until direct experience overwrites them.
- **Memory as degraded rendering.** A location visited long ago re-renders from memory when referenced — slightly wrong, simpler, warmer or colder than reality, combining elements from multiple places. Revisiting reveals the contrast between memory and reality.
- **Obsession** → the object of obsession appears where it shouldn't. A character searching for someone sees that person's face in crowds, their silhouette in doorways, their handwriting on unrelated signs. Perception-generated false positives.
- **Deja vu** → a new location briefly flickers to render as somewhere you've been. Could be a narrative clue (these places ARE connected) or a red herring (pattern-matching misfire).

#### Relationships

- **Love/infatuation** → one character renders with more detail, warmer palette, slight glow, more vivid environment. Everyone else is comparatively flat. The world literally revolves around them.
- **Rivalry** → a rival renders larger, more imposing, takes more visual space. Their achievements are more visible — bigger banners, more followers — even if "objectively" they're equal.
- **Trust** → trusted NPCs face you, open body language, stand in lit areas. Distrusted NPCs are in shadow, face away, near exits. Shifts dynamically with the relationship.
- **Loneliness** → crowds feel distant. Visual gap around the character. NPC conversations happen in clusters the character isn't part of. Sounds feel far away.

#### Moral and Worldview

- **Guilt** → locations of past wrongs render darker. The spot persists as a stain. NPCs you wronged linger at the edge of perception.
- **Innocence/naivety** → the world is simpler and more colorful. Dangers are less visible — thugs in the alley, poison in the cup go unnoticed. The player can't see threats until the character can.
- **Cynicism** → NPC motivations render as transparent and selfish. Beautiful places have visible rot. Desaturated palette. Whether this is accurate or projection is the gameplay question.
- **Corruption/moral decay** → the world degrades around the character's choices. Not punishment — perception. A character who's done terrible things sees a harsher world because that's how they interpret it.

#### Multi-Character Perspective

Subjective rendering makes multi-character replay structurally different, not cosmetically. Two characters with different perception stacks inhabit different versions of the same world:

- **Same scene, different experience.** Character A (paranoid) and Character B (naive) enter the same room. A sees a threatening interrogation chamber; B sees a friendly office. A cutscene can render both perspectives, showing the player how each character experienced the identical moment.
- **Replay as new game.** Playing the same story as a different character isn't "different dialogue options" — it's different maps (perception-modified), different NPC behavior, different narrator voice, different information available. The base scene spec is identical; the perception stack transforms everything.
- **Communication unreliability.** In multiplayer or multi-character narrative: "watch out for the guards by the gate" / "what guards?" — because subjective rendering means they're playing different games on the same base map.

#### Contrast as Storytelling

The most powerful moment isn't the distortion — it's when it lifts. Hours of anxiety-warped navigation, then a story event triggers a moment of clarity. The same map renders without modifiers. The player sees the space as it "actually" is for the first time. Emotional impact delivered through the rendering system alone, no dialogue needed.

#### Engine Architecture

**Perception modifier stack.** An ordered list of functions that take base game state and output modified render/behavior state. Each trait is a modifier. They compose: hungry + paranoid + skilled-in-stealth applies all three simultaneously.

**Independence by default.** Each modifier targets a different subsystem (tiles, NPC behavior, dialogue, palette, scale, visibility). Explicit interaction rules only where traits conflict or amplify each other.

**Combinatorial space.** No one explicitly authors "hungry + paranoid + stealth-skilled." The modifiers compose cleanly because they operate on orthogonal subsystems. The LLM pre-generates the perception rules per trait dimension; the engine applies them at runtime from the current perception vector.

**Pre-generation vs. runtime.** The LLM generates perception rules ("at anxiety > 0.7, NPC idle animations include glancing at player") ahead of time. The engine applies rules at runtime based on current state values. The LLM does not run during gameplay — it authored the rules; the engine executes them.

### Infinite Dialog

The LLM generates contextual dialogue options per conversation, per character state, per game state — not from a fixed tree. The player always chooses from presented options, never free-form input. This is deliberate: the constraint IS the character. The player experiences the character's limitations by being forced to choose between options that reflect those limitations. Free-form input breaks this because the player can always be smarter than their character.

#### Core Principle: Options as Character Expression

The dialogue options themselves are the primary tool for expressing who the character is:

- **Low social skill** → only rough, blunt utterances available. The diplomatic option doesn't exist.
- **Autistic character** → options include getting stuck elaborating on intricate details an NPC got wrong. The conversation derails not by player choice but because the available options all pull toward precision.
- **Narcissist** → two Homelander-style options. The character literally cannot choose humility.
- **Shy character** → options are tentative, indirect, self-deprecating. Assertiveness isn't available.
- **Scholarly character** → options reference obscure knowledge, cite precedent, use complex vocabulary. The "simple" explanation isn't in their repertoire.

The player never sees a locked option with a stat requirement — they just don't know a different option could have existed. The character cannot conceive of the approach they lack the trait for.

#### Dialog as Skill Check Replacement

Traditional RPGs gate conversations behind visible stat checks ("Charisma 15 required"). With LLM-generated options, the check is invisible and continuous. Charisma 0.3 produces three blunt options. Charisma 0.8 produces three options including a tactful one. This extends to every trait:

- **Intelligence** → determines whether the character can articulate a complex idea or only grasp at it vaguely.
- **Perception** → determines whether the character notices the NPC's tell when they're lying (the option to call it out only appears if perception is high enough).
- **Courage** → determines whether "confront them directly" appears at all, or whether only indirect approaches are available.

#### Conversational Momentum and Commitment

Earlier choices constrain later ones within a conversation. An aggressive opening generates follow-up options reflecting that the character is now in an aggressive posture. The player can't smoothly pivot to charm after insulting someone. The NPC's responses shift, and the player's subsequent options narrow or widen based on where the conversation has gone.

This creates **conversational regret** — the player picks option A, the conversation goes somewhere they didn't want, and the options get worse. They committed to a tone and are stuck with it. This is how real conversations work and was never possible with pre-authored trees because the combinatorics are too expensive.

#### Eavesdropping as Intelligence Gathering

NPCs have generated conversations with each other. The player can overhear them, but what you hear depends on where you stand, how long you wait, and the current game state. You catch a fragment between two guards — enough to know something is wrong, not enough to know what. Follow one guard to hear more, or find another NPC and cross-reference.

Dialogue becomes a **spatial and temporal mechanic**. Information exists in the world as NPC conversations happening in real time. The player must be in the right place at the right time, and pieces together partial information from multiple sources.

#### NPC Information Network

- **NPC lives:** NPCs have ongoing conversations with each other, generated contextually. Walk past two guards and hear them arguing about the supply shortage your actions caused three scenes ago.
- **Gossip propagation:** actions in one location produce contextually appropriate NPC reactions in other locations — each character responding according to personality and relationship to the event. Not flag-triggered, but generated from game state + character profiles.
- **Lying and unreliability:** NPCs lie based on their motivations. The player cross-references what different characters say. Detective mechanics emerge naturally from personality + knowledge + motivation without hand-authoring every deception and its corresponding truth.

#### Conversation as Combat

Some encounters are talked through, not fought. The LLM generates conversations where stakes are as high as combat: convince the guard, talk the villain down, negotiate a hostage release. Mechanical structure mirrors combat:

- The NPC has a **stance** (defensive, aggressive, open).
- The player's choices shift that stance.
- There's a failure state (NPC walks away, calls guards, attacks).
- Character traits determine what arguments are available: a scholar cites precedent, a streetwise character appeals to self-interest, a charismatic character makes an emotional appeal. Same encounter, completely different option sets.

"Damage" is credibility, trust, patience. Say the wrong thing and the NPC's willingness to listen drops.

#### Misunderstanding as Mechanic

Characters with low social skills, language barriers, or specific conditions don't just get worse options — they get options that **mean something different to the character than to the NPC**. The character thinks they're being helpful; the NPC hears condescension. The character tries to explain precisely; the NPC hears pedantry. The player sees both interpretations (dialogue text vs. NPC reaction) and navigates the gap.

This is a puzzle mechanic: the player understands what needs to be communicated, sees that their character's available options don't quite say it right, and picks the least-wrong one. The frustration is the point — it's the character's frustration made mechanical.

#### Information Asymmetry Between Player and Character

- **Character knows, player doesn't.** An option reads "Mention what happened at the bridge" — the player doesn't know what happened (it's backstory). Picking it reveals history through the dialogue itself. The player learns their character by choosing to deploy knowledge they don't yet understand.
- **Player suspects, character doesn't.** The player has pieced together that the innkeeper is the spy. But the character has no reason to suspect them, so "I know you're the spy" isn't an option. The player maneuvers using friendly, trusting options — trying to extract a confession through available means. The character's ignorance is the constraint.

#### Tone Drift Across the Game

Dialogue options gradually shift in vocabulary, confidence, and complexity as the character develops. Early game: short, uncertain, reactive. Late game: the same character speaks with authority, uses precise language, initiates rather than responds. The shift is incremental and invisible moment-to-moment, but comparing early and late dialogue reveals a character who has grown — delivered through the mechanical texture of options, not cutscene monologues.

#### NPC Relationship Memory

Every conversation with a recurring NPC carries forward — not just flags ("told blacksmith about quest") but tone, rapport, in-jokes, callbacks. The blacksmith remembers you were rude last time and opens guardedly. Remembers you brought good news and greets you warmly with a reference to last conversation. Every NPC relationship feels like an ongoing relationship rather than a state machine flipping between "neutral" and "friendly."

#### Silence as an Option

One of the generated options can be silence — saying nothing, or leaving. Silence is contextual: standing mute during an accusation means something different from walking away during small talk. The NPC reacts to the silence specifically. Character traits determine what silence means — a shy character's silence reads as discomfort; a menacing character's silence reads as threat.

#### Connection to Subjective Rendering

The dialogue system is another subsystem the perception vector modulates. The same conversation with the same NPC produces different available options, different NPC reactions, and different information flow depending on who the player character is and what state they're in. Dialogue is not separate from the world — it's rendered through the same perceptual lens.

### Animated Narrator

#### Fundamental Position

Every other system in the game speaks to the character or exists within the game world. NPCs talk to the character. The map is the character's environment. Dialogue options are the character's voice. Subjective rendering is the character's perception.

The narrator is the only entity that speaks **to the player, about the experience of being in this game**. It exists in the liminal space between the game world and the person holding the controller. No other system has access to that channel. In film, the score occupies a similar position — it tells the audience how to feel about what they're seeing without existing inside the scene. The animated narrator is the textual equivalent of a dynamic film score, except it can also convey information, lie, withhold, and develop a relationship.

The fundamental added value: **a direct, persistent, responsive communication channel between the game and the player that is neither in-world nor UI**. Traditional games don't have this because responsive narration is too expensive to author. An LLM makes it free.

#### Techniques

- **Unreliable narrator:** lies early, gradually reveals truth, or distorts reality in ways the player learns to detect.
- **Emotional tracking:** clinical during calm, breathless during crisis, the narrator's tone follows the story's arc.
- **Player-aware:** notices behavior patterns ("You've searched every bookshelf in this library. You won't find it here.") and comments on them through the narrative voice.
- **Contradiction as tool:** during psychological horror or altered-state moments, the narrator describes something different from what's on screen.

These are specific techniques. The position the narrator occupies — between game world and player — is what generates them and what enables the deeper exploits below.

#### Contextualizing Subjective Rendering

When the perception stack distorts the world, the player might not understand why. The narrator can confirm the distortion ("the market square stretches endlessly, every face turned toward you") or undercut it ("but of course, it's just a square — twenty paces across at most").

This creates a second axis: **do the narration and the visuals agree or disagree?** When they agree, the player is immersed in the character's subjective experience. When they disagree, the narrator pulls the player out — creating critical distance, hinting that the perception is unreliable. The push-pull between immersion and awareness is a storytelling tool that doesn't exist in any current game.

#### Dramatic Irony

The narrator can tell the player things the character doesn't know. "She smiles, and you smile back. You don't notice the letter she slips into her pocket." The character continues obliviously. The player carries knowledge the character doesn't — and the dialogue options still reflect the character's ignorance.

The tension between what the player knows and what the character can do is classic dramatic irony. The narrator is the only system that can create it without breaking the game's internal consistency, because the narrator is already outside the game world.

#### Teaching Without UI

Instead of tutorial popups: "The shadows here are deeper — welcoming, almost. Like they'd swallow you whole if you stepped into them." The player learns stealth is available through narration that feels like atmosphere, not instruction.

The teaching adapts — a player who already uses stealth doesn't get the hint; a player who never has gets a more explicit nudge. The narrator teaches mechanics through description, and because it's LLM-generated, it responds to the specific player's behavior history.

#### Narrator-Player Relationship

The narrator develops a relationship with the player over time based on how they play:

- A cautious player gets a narrator that becomes protective, worried, invested.
- An aggressive player gets a narrator that becomes wary, darkly amused, or disgusted.
- A thorough explorer gets a narrator that becomes a co-conspirator, pointing out details, sharing enthusiasm.
- A player who ignores NPCs gets a narrator that becomes lonely, commenting on silence.

This isn't a binary state — it's a continuous tone shift across the whole game. The player feels it accumulating without any single moment being dramatic. By the endgame, the narrator's attitude toward the player is itself a commentary on how they played.

#### Information Pacing

Film has editing. Literature has chapter structure. Games have almost nothing — information arrives when the player walks to it. The narrator can withhold, delay, foreshadow, and reveal:

- "There's something about this room. You'll understand later." The player files it away.
- Three hours later, the narrator delivers: "Now you see it."
- Foreshadowing that only makes sense in retrospect. Seeds planted early that pay off late.

The narrator has memory and intent — it can orchestrate information revelation across the entire game. No other system in the engine can do this because no other system has a persistent voice that spans the whole experience.

#### Self-Revision

Different from lying. The narrator describes what it believes is happening, and later events prove it wrong:

- "He was your friend. That much was certain."
- Ten scenes later: "It was not, as it turns out, certain."

The narrator revising its own earlier statements creates a sense that the story is being *told* by someone who is also discovering it — which mirrors the player's experience. This is the literary technique of a non-omniscient narrator, applied dynamically based on actual game events.

#### Emotional Amplification

Before a boss fight, the narrator builds tension — not describing stats, but the silence, the character's breathing, the weight of what's about to happen. After a loss, grief. After a triumph, restraint (which is more powerful than celebration). This is what a film score does, but because it's text and LLM-generated, it responds to the specific narrative context rather than being a generic "tense music" loop.

#### Per-Character Narrator on Replay

If the game supports multi-character perspective replay, the narrator's entire voice changes. The same events narrated for a naive character have warmth and gentle foreshadowing. The same events narrated for the villain are cold, analytical, maybe sympathetic in unexpected places. The narrator is a character in its own right, and its relationship to the viewpoint character defines the entire tone of the playthrough.

#### The Deepest Exploit: Three-Way Tension

The single most powerful capability: **the narrator can disagree with the game.** The visuals show one thing (subjective rendering). Characters say another (dialogue). The narrator offers a third interpretation (narration). None of these are guaranteed to agree.

In traditional media, the audience receives one authoritative version of events. Here, the player triangulates between three independent, responsive, contextual layers of information:

1. **What they see** — the world as rendered through the perception stack.
2. **What characters say** — dialogue filtered through NPC personality, knowledge, and motivation.
3. **What the narrator tells them** — a third perspective that may confirm, contradict, or recontextualize the other two.

The player is never passively receiving the story. They're always actively interpreting, weighing sources, forming their own understanding. This level of narrative engagement requires three independent responsive layers of information — which is exactly what the LLM agent hierarchy produces. No other game architecture can generate this because no other architecture has cheap, contextual, independent generation across all three channels simultaneously.

### Narrative Enemies

#### Fundamental Shift

In traditional games, combat is where the story pauses and mechanics take over. Cutscene, fight, cutscene. Story and combat are separate systems that alternate.

Here, **combat IS story**. Every enemy is a character with generated context. Every encounter is a narrative decision point — not through a menu prompt, but through the information available about who you're fighting and why. Every outcome (kill, spare, flee, negotiate) ripples through the game's narrative state and changes what comes next. The traditional game's budget for enemy characterization is one boss cutscene and three grunt voice lines. This system's budget is unlimited contextual generation from game state.

#### Adaptive Difficulty Through Story

Difficulty adapts through narrative, not stat multipliers:

- **Narrative easing:** when the player struggles, the Story Director introduces story reasons for relief — a mentor appears, the enemy faction fractures internally, an NPC opens a shortcut because they trust you now.
- **Narrative escalation:** a dominant player triggers the villain adapting strategy, deploying different enemy types, changing terrain. Feels authored, not algorithmic.

#### Enemy Personality

Bosses taunt based on the player's actual behavior, acknowledge the player's fighting style, reference story events during combat. Generated per-encounter from the game state.

During combat, taunts aren't generic — they're generated from game state and serve as an **information delivery channel**:

- "I saw what you did in the forest. Did you think no one was watching?"
- "Your friend the blacksmith talks too much. He told my agent everything."
- "You fight like someone who's afraid to lose. Interesting."

The player learns that the villain has spies, that the blacksmith is compromised, that their fighting style reveals psychological state. Combat dialogue means the player is fighting and learning simultaneously.

#### Enemies with Discoverable Motivations

Traditional enemies exist to be defeated — HP and a pattern. An LLM-generated enemy has a *reason*. The guard is protecting his family's home. The bandit is desperate and starving. The monster is territorial and scared.

The player can discover this before, during, or after combat. The discovery changes what the encounter means. You killed the guard, then found the letter from his daughter. You defeated the bandit, then an NPC mentions the famine in the eastern villages. Combat was mechanically identical — but narrative weight is completely different.

The LLM generates **biography, motivation, and context** that exists in the game world as discoverable information. The Encounter Designer creates the fight; the Story Director and Dialogue Writer seed the world with context that reframes it.

#### Combat as Moral Choice Without a Menu

The moral dimension doesn't come from a "spare/kill" dialogue prompt after combat. It comes from information the player encountered (or didn't) before the fight. You overheard that the forest hermit is actually an exiled healer — now the "clear the forest" quest feels different. But you might not have overheard that conversation.

This ties directly into eavesdropping and NPC information networks from Infinite Dialog. Information about enemies exists in the world as NPC gossip, discovered documents, environmental clues. A thorough player knows more about who they're fighting. A rushed player treats every enemy as an obstacle. Both play the same game with different narrative experiences.

#### Enemies That Aren't Enemies

The LLM can generate encounters where what presents as an enemy is something else entirely:

- The "monster" blocking the path is a mother protecting a nest.
- The "bandits" are refugees defending their camp.
- The aggressive NPC is panicking, not hostile.

The combat system engages — the player can fight. But the right response might be to stop, back off, or find another approach. Clues must be available before the encounter: tracks showing a nest, NPC mentions of refugees, body language that reads as fear not threat. Subjective rendering feeds in — a perceptive character might see the fear; a paranoid character sees only aggression.

#### The Villain as Narrative Mirror

The main antagonist's arc responds to the player's arc — not as difficulty scaling, but as character development:

- **Cruel player** → a villain who genuinely believes they're the righteous one, with evidence from the player's own behavior. The villain quotes the player's actions back at them.
- **Merciful player** → a villain who sees mercy as exploitable weakness and designs encounters to punish compassion, forcing the player to question their approach.
- **Chaotic player** → a villain who is terrifyingly ordered and principled. The contrast highlights both.

The villain's final speech isn't generic — it's a response to the specific player's specific choices. Impossible to hand-author because the combinatorial space of player behavior is too large, but straightforward for an LLM with access to the game's event log.

#### Recurring Enemies with Actual Arcs

Not "same enemy, more HP." A recurring antagonist evolves through a character arc parallel to the player's, built from real shared encounter history:

- **First encounter:** confident, dismissive, underestimates you.
- **After defeat:** humiliated, angry, references the specific way you beat them. Changes tactics.
- **After second defeat:** desperate or obsessive. Taunts reveal personal stakes — why this fight matters beyond the plot.
- **Final encounter:** either broken and pitiable, or evolved into a genuine threat who has genuinely adapted to you.

The player feels the weight of the rivalry because it's built from real shared events, not scripted beats.

#### Enemy Relationships to Each Other

Enemies aren't interchangeable units. The enemy group has internal dynamics:

- **Loyalty structures.** Troops who serve out of loyalty fight differently from mercenaries. Kill the person they're loyal to: some surrender, mercenaries recalculate odds.
- **Friendship.** Two guards are friends — kill one and the other fights recklessly harder, or breaks down and flees.
- **Internal dissent.** Not every enemy agrees with the villain. Moderates within the enemy faction can become potential allies or information sources. The enemy faction has politics the player can exploit or ignore.
- **Grief.** Enemies grieve their fallen. The player encounters a patrol mourning a comrade killed two scenes ago. Not a cutscene — generated NPC behavior based on game events.
- **Command dynamics.** A commander's death causes disorganization (leaderless troops scatter) or rage (loyalists become more dangerous).

#### Boss Fights as Narrative Climaxes

Each phase of a boss fight corresponds to a narrative beat, not just a mechanical escalation:

- **Phase 1:** the boss is testing you. Dialogue is curious or dismissive. Behavior patterns probe.
- **Phase 2:** triggered not just by HP but by a story beat — the boss reveals something that changes the fight's context. Motivation becomes clear. Behavior shifts to match emotional state.
- **Phase 3:** the climax is emotional, not mechanical. The boss's final phase might be desperate (losing something they care about), resigned (accepted the outcome), or transcendent (moved beyond caring about the fight).

Phase transitions can involve dialogue choices — the player might end a boss fight with words rather than damage, if they've gathered enough information and their character has the right traits.

#### The Player's Relationship to Violence

How the player handles enemies feeds back into the character's psychological state, which feeds into subjective rendering and narration:

- **Desensitization:** a character who kills frequently — combat renders flatter, less intense, almost routine. The narrator's tone during fights becomes detached. Not a reward; a character statement.
- **Trauma:** alternatively, frequent killing — combat renders as increasingly distorted, disturbing. The narrator dwells on details the player would rather not notice.
- **Reputation:** a character who spares enemies develops a reputation. Some future enemies surrender earlier. Others see mercy as weakness and fight harder.

The kill/spare pattern isn't a binary morality meter. It's a continuous influence on the perception vector, narrator voice, NPC reactions, and enemy behavior — all generated contextually from the player's actual history.

### Replay and Multiplayer

#### Fundamental Insight

The boundary between single-player replay and multiplayer dissolves. The game is generated from state. Any player's state can feed into any other player's generation pipeline. "Multiplayer" is just: whose state feeds into whose generation? Replaying your own game with a different character is structurally identical to playing in a world shaped by another player's character. The system doesn't distinguish between "your past self" and "another person" — both are prior state that feeds current generation.

There is no "multiplayer mode" as a separate thing. There's a story generation pipeline that accepts state from one player or many, synchronously or asynchronously, symmetrically or asymmetrically.

#### Single-Player Replay

Second (and subsequent) playthroughs are structurally different, not just cosmetically:

- **Hidden layers revealed:** information withheld in the first playthrough surfaces — the same scenes carry different meaning with new context.
- **Perspective shifts:** first playthrough is the hero's arc; second is the same events from the villain's or a bystander's perspective, using the same maps with different interpretation.
- **Meta-narrative awareness:** the game acknowledges repetition. NPCs behave differently. The narrator's tone shifts. Story beats subvert first-playthrough expectations.

Generation cost per playthrough variant is low because the structural grammar (maps, entities, scripts) stays constant while the narrative content layered on top changes.

#### Asynchronous Narrative Influence

Player A completes a story. Player B starts the same premise later, but the world bears traces of Player A's playthrough — not as explicit "Player A was here" markers, but as generated consequences woven into the world:

- NPCs remember "a traveler who came through before" and reference Player A's actions in contextual dialogue. The blacksmith mentions someone bought all the iron. The innkeeper warns about trouble on the road — trouble Player A caused or failed to prevent.
- Environmental state reflects Player A's choices. The bridge Player A burned is still gone. The village Player A saved is thriving. Not "your save file" — world state inherited by the next player.
- Faction dynamics carry forward. Player A sided with the rebels — Player B enters a world where the rebellion is further along, with different power structures.

Player B doesn't know which elements of their world were shaped by another player and which are baseline. The world feels like it has history because it actually does — someone else's history.

#### Ghost Traces

Like Dark Souls messages but narrative rather than mechanical. Player A's actions create ripples that Player B encounters as world texture:

- An NPC tells a story about "someone who did something remarkable at the mountain pass." The someone was Player A. The story is generated from Player A's actual actions. Player B hears it as lore.
- A grave marker in the forest — generated from where Player A's character died, or where Player A buried an enemy they felt guilty about killing.
- A path through the wilderness that's slightly more worn, because Player A took it. Not a marker, just a subtle tile variant generated from traffic data.

The traces are **diegetic** — they exist in the world as world-appropriate artifacts, not as multiplayer UI. Player B can believe they're authored content or suspect they're another player's footprint. The ambiguity is the point.

#### Parallel Perspective Multiplayer

Two players play the same story simultaneously as different characters. Each experiences the same events through their own subjective rendering and perception stack:

- Player A is the hero entering the town. Player B is a townsperson watching the hero arrive. Same scene, two completely different experiences — different actions, different dialogue, different narrator voice.
- Player A fights the bandit camp. Player B, playing a character in the town, experiences the consequences: refugees arriving, supply shortages, political shifts. Player A's combat is Player B's story context.
- At key moments their storylines intersect — a conversation between their characters where each player gets dialogue options from their own character's perception and traits. Player A sees a tense negotiation; Player B sees an intimidating stranger making demands.

Not co-op. They're experiencing the same narrative from different positions in it, and each player's actions generate consequences the other encounters.

#### The Villain Is a Player

One player is the hero. Another plays the antagonist — not in direct combat, but in story:

- The villain player makes strategic decisions: where to deploy forces, which NPCs to corrupt, what traps to set, how to respond to the hero's actions. Their choices become the Encounter Designer's input for the hero player's game.
- The hero player experiences these as narrative events — a town suddenly hostile, a trusted NPC turned traitor, a new enemy type appearing. They don't know which elements are the other player's choices and which are generated.
- The villain player sees the hero's progress through spy reports, NPC informants, and their own subjective rendering — perhaps a paranoid surveillance view where the hero is always a threatening presence.
- Each sees the other through their perception stack. The hero's narrator frames the villain's actions one way; the villain's narrator frames them completely differently.

Asymmetric multiplayer where the asymmetry is narrative, not mechanical. One player generates the other's story.

#### Generational Play

Player A's completed playthrough becomes the **history** of Player B's world:

- Player A was the legendary hero (or villain) whose actions shaped the world. NPCs tell stories about them — generated from the actual playthrough, not generic legend text.
- Statues, monuments, or scars in the world correspond to Player A's major decisions. The statue in the town square depicts what Player A actually did. The scar on the landscape is where Player A's final battle happened.
- Player B's character might be a descendant, a successor, or someone living in the consequences. Their story is a response to Player A's — dealing with what Player A left behind.
- The narrator references the previous era with the weight of actual events: "They say the old hero showed mercy at the bridge. You can see how that turned out."

Replay as inheritance. Each generation of play adds a layer of history that the next generation inherits as world state.

#### Asymmetric Information Multiplayer

Two players in the same story know different things. Each sees partial information through their character's perception:

- Player A's character overheard a conspiracy but didn't see the conspirators' faces. Player B's character saw them but doesn't know what they're planning.
- They need to communicate — in-game through their characters meeting, or out-of-game as players. But can you trust the other player's report? Their character might be lying, or their subjective rendering might be distorting what they saw.
- A mystery that requires two perspectives to solve. Neither player alone has enough information. The full picture only emerges from combining two subjective, potentially distorted viewpoints.

#### The Witness / Audience Mode

One player plays. Others exist in the story as NPCs with limited agency:

- A "witness" player controls a character in the background — a bartender, a bystander, a minor court official. They have dialogue options when the main player interacts with them, and can make small choices (what information to share, whether to help or hinder) that nudge the story.
- Multiple witnesses create a living world where every NPC the main player talks to is potentially another person making choices. The main player doesn't know which NPCs are players and which are generated.
- Witnesses see the story from the margins — watching someone else's adventure happen around them. A fundamentally different kind of gameplay, closer to improvisational theater than to action gaming.

#### Dungeon Master Mode

One player is the Story Director, working alongside the LLM rather than replacing it:

- The DM player sees game state, story structure, upcoming beats. They can modify NPC behavior in real time, inject events, change atmosphere, reshape encounters.
- The LLM handles mechanical generation (dialogue text, tilemap adjustments, enemy behavior) from the DM's high-level directions. The DM says "make the innkeeper suspicious" — the LLM generates dialogue, NPC behavior changes, and environmental cues.
- Tabletop RPG with the engine handling rendering and the LLM handling detail generation. The DM provides creative direction; the system produces a playable game in real time.

### Environmental Memory

#### Passive Memory

Locations accumulate history based on what happened there:

- **Event residue:** a room where something bad happened re-renders with subtle differences on return — colder lighting, a stain, heavier descriptive text. Generated by the Atmosphere Director from the location's event history.
- **World aging:** early-game locations evolve as the story progresses. A saved village shows scaffolding, new NPCs, warmer palette. A neglected one declines — broken tiles, fewer NPCs, different mood. Not expensive hand-authored tile swaps but Level Designer variants driven by game state.
- **Cumulative atmosphere:** a location visited many times by the player becomes familiar to the narrator and the character — descriptions shorten, tone warms or sours depending on the associations built up.

#### The Environment as Agent

The deeper exploit: locations that accumulate enough memory develop **agency**. They don't just look different — they behave differently. They act on their history.

- A forest where violence has repeatedly occurred starts working against the player: paths rearrange between visits, safe clearings shrink, the canopy thickens and darkens, travel takes longer because routes become less direct. The forest itself IS the obstacle — not spawning enemies, but reshaping space.
- A place where the player has acted with care actively helps: paths open, shortcuts reveal themselves, resources appear in convenient locations, rendering becomes warmer and more navigable. The garden you've tended responds to your presence.

The LLM generates environmental behavior rules from accumulated event history, treating the location as an NPC with a personality formed through experience. The engine applies those rules at runtime. Mechanically identical to NPC behavior — but the "NPC" is the room, the forest, the city district.

The result: **the player has relationships with places, not just people.** You can be on good terms with the forest and bad terms with the city. Each location responds to your history with it. "Going back to the forest" becomes a meaningful decision because the forest has an opinion about you.

#### Temporal Echoes

Environmental memory becomes navigable. At certain moments or under certain conditions, the environment replays fragments of what happened there — not as cutscenes, but as ghostly overlays in the live game world:

- Translucent figures walking through a corridor, following the path a character took during a past event.
- The sound of a conversation that happened here, fragmented, key phrases audible.
- The flicker of a fire that was extinguished three scenes ago.
- During dramatic moments: the full scene replays around the player — they stand in the present while the past happens around them.

The player character's perception stats determine how much they see. High perception or magical sensitivity: rich temporal echoes. Without: the same room, empty and quiet. The echoes are a perception layer, not an objective event — tied directly to subjective rendering.

**Gameplay mechanic:** temporal echoes contain **information**. A murder happened in this room. The echo shows who did it — if you can perceive it, if you're here at the right time, if you know to look. Environmental memory becomes a detective tool. The world is a witness that can be interrogated by being present in its space.

#### Emotional Archaeology

Locations accumulate emotional layers that coexist rather than overwrite:

- First visit: the town square is neutral.
- After a celebration there: a layer of warmth.
- After a betrayal in the same spot: the warmth curdles — both layers exist. The square feels like a happy place where something went wrong. The dissonance is the emotion.
- After a reconciliation: a new layer. The square now has depth — warmth, then pain, then healing. All three inform how it renders and how the narrator describes it.

A perceptive character reads more layers than others. A naive character just sees a square. A perceptive character sees accumulated emotional history: "This square has known joy and betrayal in equal measure. You can feel both in the cobblestones."

Places become **emotionally complex** over the course of the game — not because a designer authored that complexity, but because events accumulated there. A boring location at the start can become the most emotionally charged space by the end, purely through what happened during play.

#### Spatial Contagion

Events don't stay where they happened. Environmental memory spreads:

- A murder in the alley affects the whole street. Over time, the whole district. The darkness seeps outward tile by tile across visits.
- A corrupted forest slowly contaminates neighboring farmland. Crops render as withered. Farmer NPCs become anxious.
- Conversely, a healed location pushes back. Cleansing the forest source point gradually restores surrounding areas. The healing spreads at the same rate the corruption did.

This creates **geographic emotional weather** — macro-scale mood patterns across the game map that emerge from play history rather than authored palette assignments. A region where war happened has a different atmospheric baseline than one at peace. The player can see the boundary between affected and unaffected territory, and that boundary is a meaningful game element — pushing it back is tangible progress.

#### The World as Unreliable Historian

Environmental memory isn't objective. The location "remembers" through its own lens:

- A church remembers a battle inside it as desecration — rendering emphasizes violation, broken sacred objects, cracked stained glass. The soldiers saw a necessary tactical position.
- A prison remembers its inmates as dangerous — cells render as barely containing something. The inmates experienced the same space as suffocating and unjust.
- A home remembers the family fondly — warm tones, worn but comfortable details. An outsider might see a shabby hovel.

The environment's memory is as subjective as any character's perception. Which version the player sees depends on their relationship to the location, their knowledge, their perception traits. The narrator may tell a different story than the environment shows — three-way tension applied to place memory.

#### Place Identity Evolution

A location develops a character over time — a generated personality emerging from accumulated events:

- An inn where many warm events occurred becomes known. NPCs reference "the good inn." Its tiles brighten, NPC behavior inside becomes more relaxed, the narrator describes it with affection. It earns its reputation through play.
- The same inn, after a tragedy: reputation sours. NPCs say "it hasn't been the same since..." The environmental personality changes — not a flag flip, but a gradual transformation driven by event accumulation.

The inn didn't have a personality at the start. It acquired one. The player participated in creating it without intending to. Player actions don't just affect the plot — they shape the **identity of the world itself**. Two players starting from the same premise end up with differently characterized locations because different things happened there.

#### Cross-Player Environmental Memory

In multiplayer and generational modes, environmental memory creates the most tangible form of cross-player influence:

- Player A nurtured the forest. Player B enters a world where the forest is welcoming — paths open, light filters through, the narrator speaks of it warmly. Player B doesn't know why the forest is kind; it just is, because of someone else's history.
- Player A devastated a city. Player B encounters the ruins. The environmental memory is grief and anger — the city resists Player B even though Player B did nothing wrong. Player B has to heal a wound they didn't cause.
- The world accumulates memory across players. After ten players, a location has rich, layered emotional history that no single player authored. It has become a genuinely complex place through accumulated human decisions.

The game world develops **genuine history through play** — not procedurally generated backstory, but actual accumulated decisions by actual people, reinterpreted by the LLM into environmental character.

## Model Tier Analysis: Frontier vs Small

The system has two fundamentally different execution contexts — generation time (building the game) and runtime (playing the game) — with different model requirements.

### The Core Insight: Authors vs Performers

Frontier models are **authors**. They create the narrative spine, character voices, dramatic structure, and the quality ceiling. They run once during generation.

Small models are **performers**. They execute within the authored space — interpolating, adapting, filling contextual gaps. They run continuously during play.

This means the generation pipeline's job isn't just "make a game" — it's **"make a game AND the fine-tuning data for the runtime models."** The Dialogue Writer doesn't just produce dialogue trees; it produces example dialogues in each character's voice that become training signal for the runtime dialogue generator. The Story Director doesn't just write narrator text; it produces the narrator's voice profile that constrains the runtime narrator's generation.

### Generation-Time Agents

These run once when producing the game. Cost and latency matter but aren't existential constraints.

**Frontier required (GPT-4+, Claude, Gemini):**

- **Story Director.** The hardest job in the system. Must hold an entire narrative arc in context, balance six research dimensions simultaneously (subjective rendering, narrator voice, enemy motivations, replay layers, environmental memory seeds, dialogue architecture), and make creative decisions under ambiguity. Every other agent inherits its quality ceiling from here.
- **Script Director.** Dramatic timing, emotional beats, the play-script system. Orchestrating when the narrator disagrees with perception, when a boss fight shifts phases on a revelation, when silence is more powerful than dialogue. Compositional creativity with long-range dependencies.
- **Dialogue Writer.** Character voice differentiation, information asymmetry (what options exist given what the character knows vs. doesn't), conversation-as-combat design, constrained-options-as-character-expression. The dialogue trees are the player's primary mode of expression — quality here is non-negotiable.
- **Character Designer.** Personality as a high-dimensional vector, motivation networks, behavioral parameters that cascade into every other system (how perception distorts them, how the narrator describes them, how they fight). Foundation other agents build on.

**Mid-tier sufficient (30B–70B):**

- **Level Designer.** Spatial reasoning is more structured than narrative reasoning. Translating "the forest should feel oppressive and get worse as you go deeper" into region definitions, tile palettes, traversal costs, and enemy placement zones is systematic. The Story Director already defined what the level means.
- **Encounter Designer.** Combat design is behavioral engineering — behavior trees, phase transitions, difficulty curves, patrol patterns. Narrative weight comes from the Character Designer's enemy biographies, not from the encounter designer inventing it.
- **Atmosphere Director.** Tonal consistency requires aesthetic judgment but within defined parameters. The Story Director said "this act feels like dread." The Atmosphere Director translates that to color palettes, music cues, weather, lighting. Creative but constrained.

**Small fine-tuned sufficient (7B–13B):**

- **Pacing Analyst.** Pattern analysis against tension models. Input: scene sequence with emotional tags. Output: pacing violations and structural imbalances. Analytical, not generative.
- **Continuity Tracker.** Fact retrieval and contradiction detection. Structured comparison against a facts database, not open-ended reasoning. Partially rule-based with an LLM for ambiguous cases.
- **Consistency Checker.** Same logic as continuity tracker but across agents rather than across time. Structural cross-referencing.

### Runtime Agents (In the Shipped Game)

These run during play, potentially every scene transition or interaction, in the player's browser or on their machine. The constraint is real.

**Mostly rules + small LLM (3B–7B) for edge cases:**

- **Perception Engine.** The perception stack is mathematically defined: `render(base_state, perception_vector) → perceived_state`. 90% of this is deterministic — fear darkens shadows, paranoia makes NPCs seem hostile, exhaustion mutes colors. Parameterized transformations, not generation. The small LLM handles novel combinations of perception dimensions and transitional states the pre-generation didn't anticipate.
- **Environmental Memory updater.** Event-to-emotional-layer transformation is almost entirely rule-based. "Combat event at location + enemy had known motivation = violence layer + moral weight modifier." Natural language descriptions of location mood changes are template expansions from typed emotional tags.

**Small-to-mid fine-tuned (7B–13B) on the game's own voice:**

- **Narrator (runtime).** The hardest runtime job. Must maintain consistent voice, reference game history, contextually adapt. But the key: the narrator's personality, arc, and major beats are pre-generated by the frontier model. The runtime narrator interpolates — filling gaps between pre-written anchor points, adjusting tone based on perception, selecting which pre-generated foreshadowing to surface. Fine-tuning on the specific narrator voice established at generation time is the play.
- **Dialogue Generator (runtime).** Same principle. The Dialogue Writer pre-generated the conversation architecture — branching structures, key lines, character voice parameters. The runtime generator fills in contextual variations: the same refusal but colder because the player attacked her friend two scenes ago. Constrained generation within a well-defined envelope.

**No LLM:**

- **Multiplayer State Bridge.** Pure serialization and state management. Deterministic transformation of game state into exportable format. This is code, not language generation.

### Implications for the Generation Pipeline

The split sharpens the three hardest unresolved questions from INTERFACES.md:

1. **Pre-gen vs runtime** → Pre-generate the structure and quality ceiling. Runtime handles contextual adaptation within that ceiling. The frontier model's output includes both game data AND the constraints/examples that define the small model's operating envelope.
2. **Perception modifier format** → Must be evaluable by rules (no LLM needed for 90% of cases) with a typed fallback path to the small LLM for edge cases.
3. **Game state schema** → Must be dual-purpose: readable by the frontier model that authors the game AND parseable by the small model that performs at runtime.

## Development Plan

### Phase 0: The Schema (Foundation)

Everything depends on the engine schema. Agents can't generate valid output without knowing what the engine accepts. The engine can't consume anything without knowing what agents produce. The schema is the contract between the two worlds.

Phase 0 defines the schema as Rust types — structs, enums, and validation logic — that serialize to a well-defined JSON format. This is the single source of truth. Every downstream component reads from it. The schema doesn't need to be complete to be useful; it needs to be extensible and versioned so that early subsystems can start working against a subset while later subsystems add their vocabulary.

What the schema must cover to be minimally useful: tile map definition (regions, layers, tile types, traversal properties), entity definition (NPCs, enemies, objects — position, sprite reference, behavior reference), player definition (position, perception vector, hidden stats), dialogue tree structure (nodes, choices, conditions, consequences), play script primitives (move, face, speak, wait, screen effect), and asset manifest (sprite sheet references, portrait references, tile palette references).

What the schema adds later as subsystems mature: perception modifier stack, narrator command vocabulary, environmental memory model, enemy biography and faction structures, multiplayer state serialization format, temporal echo definitions.

The schema crate also includes a validator — a program that takes a JSON game file and reports whether it's structurally valid, with clear error messages aimed at the generating agents (since they'll be the ones producing malformed output).

### Phase 1: Minimal Playable Engine

A Rust/WebAssembly engine that can load a schema-valid game file and run it in the browser. The first version is deliberately minimal:

- Tile-based map rendering (single layer, simple palette).
- Player entity with WASD movement and collision.
- Static NPCs with touch-to-talk trigger.
- Basic dialogue display (linear text, no branching yet).
- A single play script that runs on game start (camera pan, NPC walks, speaks).

No perception engine, no narrator, no combat, no environmental memory. The point is to prove the schema-to-browser pipeline works end to end. A handwritten JSON file (not LLM-generated) exercises the engine. If this works, the hardest infrastructure question is answered: can we go from structured data to a running game in the browser?

### Phase 2: Core Game Systems

Build out the engine's actual game systems, still consuming handwritten test data:

- **Dialogue system.** Branching, conditions (flag-based), consequence triggers, expression changes. This is the player's primary interaction mode.
- **Combat system.** Real-time overworld combat. Enemy behavior trees from simple primitives. Hit detection, damage (hidden stats), death/defeat handling.
- **Play script system.** The full choreography language: entity movement, camera control, dialogue interjection, screen effects, timing, parallel sequences. This carries dramatic weight.
- **Map transitions.** Multi-map games, door/exit triggers, spawn points, map-to-map state persistence.

Each system is developed against the schema — the schema grows as systems need new vocabulary. The test data files grow correspondingly. At the end of Phase 2, you can handwrite a JSON file that describes a small but complete game: walk around, talk to people, fight enemies, experience a scripted dramatic moment, move between maps.

### Phase 3: Perception and Narrator

The novel runtime systems that distinguish ice from a generic 2D engine:

- **Perception engine.** The `render(base_state, perception_vector) → perceived_state` pipeline. Parameterized visual modifiers (color grading, shadow intensity, sprite swaps), auditory modifiers (music tempo, ambient sound selection), behavioral modifiers (NPC dialogue tone shifts, movement pattern changes). Mostly rule-based; the small LLM integration point is defined but can initially be stubbed with fallback rules.
- **Narrator system.** A text channel distinct from dialogue, addressed to the player. Narrator state model (personality, relationship to player, current stance). Pre-generated narrator text with runtime selection and interpolation. The narrator reacts to player actions, perception state, and game events.
- **Environmental memory.** Event log → emotional layer transformation. Location mood tracking. Narrator and perception engine consume location mood to modify rendering and narration. Temporal echo rendering (ghostly overlays of past events).

These systems are where the small runtime LLM plugs in. Phase 3 defines the integration interface — the contract between the deterministic engine and the probabilistic language model — but can function entirely on rules and pre-generated content without the LLM present.

### Phase 4: Agent Orchestration (Generation Pipeline)

With the engine able to run rich games from schema-valid JSON, build the generation side:

- **Orchestrator.** The coordination layer that manages the multi-agent pipeline. Takes a story premise, runs it through the agent hierarchy, collects outputs, validates against schema, assembles the final game file. Likely Python.
- **Story Director agent.** The frontier model integration. Prompt engineering, context management, output parsing. This is the most critical prompt work in the system.
- **Domain specialist agents.** Level Designer, Character Designer, Encounter Designer, Script Director, Dialogue Writer. Each has a prompt template, receives the Story Director's spec, and produces schema-valid output for its domain.
- **Validator agents.** Continuity Tracker, Consistency Checker, Pacing Analyst. Run after domain specialists, flag issues, trigger revision loops.
- **Asset pipeline.** Image generation API integration. Sprite sheet assembly from individual asset generations. Style consistency enforcement across assets.

Phase 4 is where the system becomes self-serving — you describe a story and get a playable game. But it depends entirely on Phases 0–3 being solid. A buggy engine or underspecified schema means every generated game is broken in ways the agents can't diagnose.

### Phase 5: Multiplayer and Replay

The most architecturally complex features, built on everything below:

- **State serialization.** Exporting a complete playthrough state for consumption by another player's world.
- **Ghost traces.** Diegetic artifacts from other players' playthroughs rendered as world elements.
- **Perspective-variant generation.** Same base game, different perception vectors and narrator voices per character.
- **Generational play.** Successor games that inherit world state as backstory.

Phase 5 is speculative — it may reshape Phases 3 and 4 significantly once the real constraints become clear. It's listed for completeness but shouldn't constrain earlier phases.

### Cross-Cutting: Runtime LLM Integration

Not a phase but a thread that runs through Phases 3–5. The shipped game optionally includes a small local LLM (7B–13B) for runtime adaptation. The integration has a strict contract:

- The engine sends a typed request (narrator text needed, dialogue variation needed, perception edge case).
- The LLM responds within the schema's type vocabulary.
- If the LLM is unavailable or too slow, the engine falls back to pre-generated content or rule-based defaults.
- The generation pipeline (Phase 4) produces fine-tuning data for the runtime LLM as a byproduct of game generation.

The LLM is always optional. Every game must be fully playable without it. The LLM makes it *better* — more contextually responsive, more surprising — but never *necessary*.

## Repo Structure (Sketch)

The repo is a Rust workspace with multiple crates, plus a Python package for agent orchestration. The structure reflects the schema-engine-agents separation.

```
ice/
├── Cargo.toml                  # Workspace root
├── README.md
├── IDEA.md                     # This file
├── INTERFACES.md               # Interface definitions
│
├── crates/
│   ├── ice-schema/             # The contract (Phase 0)
│   │   ├── src/
│   │   │   ├── lib.rs          # Re-exports
│   │   │   ├── map.rs          # Tile map types (regions, layers, tiles, traversal)
│   │   │   ├── entity.rs       # Entity types (NPC, enemy, object, player)
│   │   │   ├── dialogue.rs     # Dialogue tree types (nodes, choices, conditions)
│   │   │   ├── script.rs       # Play script types (choreography primitives)
│   │   │   ├── combat.rs       # Combat types (behavior trees, damage, phases)
│   │   │   ├── perception.rs   # Perception vector, modifier stack, render rules
│   │   │   ├── narrator.rs     # Narrator types (commands, state, voice profile)
│   │   │   ├── memory.rs       # Environmental memory (event log, emotional layers)
│   │   │   ├── asset.rs        # Asset manifest (sprite refs, palettes, portraits)
│   │   │   └── validate.rs     # Schema validation (JSON → Result<Game, Vec<Error>>)
│   │   └── Cargo.toml
│   │
│   ├── ice-engine/             # The runtime (Phases 1–3)
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── game.rs         # Game loop, state management, event dispatch
│   │   │   ├── render.rs       # Tile and sprite rendering (2D, layered)
│   │   │   ├── input.rs        # Player input handling (keyboard, gamepad)
│   │   │   ├── movement.rs     # Entity movement, collision, pathfinding
│   │   │   ├── dialogue.rs     # Dialogue display and interaction
│   │   │   ├── combat.rs       # Real-time combat system
│   │   │   ├── script.rs       # Play script executor
│   │   │   ├── perception.rs   # Perception engine (rule evaluation + LLM fallback)
│   │   │   ├── narrator.rs     # Narrator display and state
│   │   │   ├── memory.rs       # Environmental memory runtime
│   │   │   └── llm.rs          # Runtime LLM integration interface
│   │   └── Cargo.toml          # Depends on ice-schema
│   │
│   ├── ice-web/                # WebAssembly shell (Phase 1)
│   │   ├── src/
│   │   │   └── lib.rs          # wasm-bindgen entry point, canvas setup, event loop
│   │   ├── index.html          # Minimal HTML host
│   │   └── Cargo.toml          # Depends on ice-engine, wasm target
│   │
│   ├── ice-validator/          # CLI validation tool (Phase 0)
│   │   ├── src/
│   │   │   └── main.rs         # Takes a JSON game file, reports schema errors
│   │   └── Cargo.toml          # Depends on ice-schema
│   │
│   └── ice-export/             # Multiplayer state serialization (Phase 5)
│       ├── src/
│       │   └── lib.rs          # State diff, ghost trace generation, perspective transforms
│       └── Cargo.toml          # Depends on ice-schema
│
├── agents/                     # Generation pipeline (Phase 4, Python)
│   ├── pyproject.toml
│   ├── ice_agents/
│   │   ├── __init__.py
│   │   ├── orchestrator.py     # Multi-agent coordination, pipeline runner
│   │   ├── director.py         # Story Director agent (frontier model)
│   │   ├── level.py            # Level Designer agent
│   │   ├── character.py        # Character Designer agent
│   │   ├── encounter.py        # Encounter Designer agent
│   │   ├── script.py           # Script Director agent
│   │   ├── dialogue.py         # Dialogue Writer agent
│   │   ├── validators/
│   │   │   ├── continuity.py   # Continuity Tracker
│   │   │   ├── consistency.py  # Consistency Checker
│   │   │   └── pacing.py       # Pacing Analyst
│   │   ├── atmosphere.py       # Atmosphere Director
│   │   ├── assets.py           # Image generation pipeline
│   │   └── prompts/            # Prompt templates per agent
│   │       ├── director.md
│   │       ├── level.md
│   │       └── ...
│   └── tests/
│
├── games/                      # Test game data (handwritten JSON, then generated)
│   ├── minimal/                # Phase 1 test: one map, one NPC, one script
│   │   ├── game.json
│   │   └── assets/
│   ├── dialogue-test/          # Phase 2 test: branching dialogue
│   ├── combat-test/            # Phase 2 test: enemy encounters
│   └── perception-test/        # Phase 3 test: perception vector effects
│
└── tools/
    ├── schema-doc/             # Auto-generate schema documentation from Rust types
    └── game-inspector/         # Visual inspector for game.json files (debug tool)
```

### Key Structural Decisions

**ice-schema is the keystone.** Everything depends on it, nothing else has circular dependencies. The schema crate has zero runtime dependencies beyond serde — it's pure data types and validation. This means agents (Python) can read the JSON schema specification generated from the Rust types, and the engine (Rust) consumes the same types natively.

**ice-engine is a library, not a binary.** The engine is consumed by ice-web (for the browser), but could also be consumed by a native runner, a headless test harness, or a server-side renderer. Keeping it as a library preserves this flexibility.

**agents/ is Python, not Rust.** LLM API integration, prompt management, and orchestration logic are better served by the Python ecosystem (LangChain/alternatives, API client libraries, rapid iteration). The schema boundary is JSON — agents produce it, the engine consumes it. No FFI needed.

**games/ is test data, not code.** Handwritten game files exercise the engine before the generation pipeline exists. They're also the reference implementation — if a handwritten game works but a generated one doesn't, the bug is in the agent, not the engine.

**The LLM integration boundary in ice-engine is a trait.** The engine defines `trait RuntimeLlm { fn request(&self, req: LlmRequest) -> Option<LlmResponse>; }`. The implementation can be a local model, an API call, or a stub that always returns `None` (forcing the fallback path). The engine never depends on a specific LLM.

### What This Structure Doesn't Decide Yet

- **Rendering backend.** The engine needs a 2D renderer that works in WASM. Candidates: wgpu (powerful but complex), raw canvas2d via web-sys (simple but limited), a lightweight 2D framework like macroquad or comfy. Decision deferred to Phase 1 when we actually build the renderer.
- **Audio.** Not in the schema yet. Music and sound effects matter enormously for atmosphere but the integration is relatively straightforward once the engine exists. The schema will need audio cue types eventually.
- **Asset format.** Sprites, tile palettes, portraits — what format? PNG sheets with metadata? Individual images assembled at build time? The Atmosphere Director needs to influence palette selection, which implies some level of programmatic asset manipulation. Deferred to Phase 2.
- **LLM hosting.** How does the runtime LLM actually run in the browser? ONNX runtime in WASM? WebGPU inference? External API call? This is a Phase 3 problem and the technology landscape may shift by then. The trait boundary in the engine means this decision doesn't affect anything else.
- **Agent framework.** The orchestration layer in `agents/` needs multi-agent coordination, retry logic, context windowing. Build from scratch? Use an existing framework? Depends on what the prompt engineering looks like when we get there.

## Constraints

- All output-side code is **Rust** (engine, schema, validator, build pipeline).
- Agent orchestration can be any language (likely Python or Rust).
- Image generation models are external services called via API.
- The engine's schema is the single source of truth for all agent output formats.
