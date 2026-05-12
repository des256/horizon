# ice — Interface Definitions

Every interface in the system that must be fully defined before building. Each chapter is one boundary where data crosses between components. The three hardest unresolved questions — pre-generation vs. runtime LLM, the perception modifier format, and the unified game state schema — cut across multiple interfaces and are called out where relevant.

---

## 1. User ↔ Story Director

The primary creative loop. The user provides narrative intent; the Story Director expands it into a structured game spec; the user reviews and refines.

**What crosses the boundary:**

- **User → Director:** story premise (natural language, from a sentence to a detailed outline), style preferences (visual, tonal, mechanical), iterative feedback on the game spec (scene-level or global)
- **Director → User:** the game spec in a reviewable format — world map, character roster, scene sequence, arc structure

**Open questions:**

- What format is the game spec presented in for human review? A structured document? A visual outline with a world map and character cards? An interactive tool?
- How does the user give scene-level feedback? ("Make this scene more tense," "this character feels flat," "I want a choice point here")
- How much does the user see of the internal structure (trigger lists, flag names) vs. a narrative-only view?

**Subjective Rendering Impact:**

The user must be able to define and preview subjective rendering parameters during the creative loop. This means:

- **Character creation includes perception profiles.** When the user defines a character, they specify (or the Story Director proposes) psychological conditions, skill affinities, and worldview traits with continuous values. The interface must present these as narrative descriptors ("crippling agoraphobia," "novice swordsman," "deeply cynical") that map to perception vector values, not raw floats.
- **Preview of perception effects.** The user should be able to see how a scene looks through different characters' perception stacks. "Show me the market square through the agoraphobic character's eyes vs. the confident character's eyes." This requires the review format to support side-by-side or switchable perception previews — a significant UI requirement.
- **Therapeutic/progression arcs as authoring targets.** The user doesn't just set a trait value — they define a trajectory ("agoraphobia starts at 0.9, reaches 0.4 by midgame, 0.1 at the end"). The interface must support arc-over-time definitions for perception traits, not just static values.

**Infinite Dialog Impact:**

The user's creative loop must expose dialogue philosophy — not individual lines, but the systems that generate them.

- **Character voice authoring.** The user defines (or reviews the Story Director's proposal for) each character's speaking style: vocabulary level, verbal habits, emotional range, formality. The interface must present this as readable character descriptions ("speaks in short, blunt sentences; avoids metaphor; swears when frustrated") not parameterized models.
- **Choice point philosophy.** The user defines what kinds of choices the player has: "always three options, one kind/one pragmatic/one aggressive" vs. "options reflect character limitations, some conversations have no good option." This is a game-level design decision that shapes every conversation.
- **NPC relationship preview.** The user should see how NPC relationships evolve across the story — how the blacksmith's dialogue changes from first meeting through repeated visits. The spec review must show relationship arcs, not just individual conversations.
- **Conversation-as-combat preview.** When a scene includes a high-stakes dialogue encounter (negotiate a hostage release, convince the council), the user should see the NPC stance model, the failure states, and the range of arguments available at different character trait levels.

---

## 2. Story Director → Domain Specialists (the Game Spec)

The most important interface in the system. Every downstream agent works from this spec. It must be complete enough that domain agents can work independently, and structured enough that it's machine-parseable.

**What crosses the boundary:**

- **World definition:** regions, thematic identity, connections between regions, world-level rules and lore
- **Character definitions:** appearance descriptors, personality, role in story, stat profile hints ("fragile but fast"), perception traits (the full perception vector — psychological conditions, skills, physical states, knowledge, relationships, moral stance)
- **Scene sequence:** ordered list of scenes with location, characters present, events, emotional tone, narrative purpose in the arc
- **Per-scene specs:** spatial description ("autumn forest, branching paths, hidden shrine at the north end"), encounter descriptions ("guardian spirit, tests the player, not lethal"), dialogue beats, dramatic moments to script, atmosphere intent
- **Perception rules:** how each character trait should modulate each engine subsystem — these are authored by the Story Director as narrative intent ("the agoraphobic character experiences open spaces as threatening") and translated into engine-consumable rules downstream
- **Narrator profile:** personality, reliability level, relationship trajectory with player, per-character voice variants for replay, foreshadowing seeds
- **Enemy biographies:** motivation, faction membership, relationships to other enemies, role in story, narrative difficulty feel
- **Environmental memory seeds:** which locations should accumulate memory, their personality bias, contagion rules (does this location's mood spread to neighbors?)

**Open questions:**

- What is the concrete schema for a scene spec? This is the contract that all domain agents parse.
- How are perception rules expressed at the narrative level before they're compiled into engine rules?
- How does the spec handle branching? A scene may have multiple variants depending on prior player choices. Is the spec a tree, a DAG, or a linear sequence with conditional annotations?

**Subjective Rendering Impact:**

Subjective rendering massively expands what the game spec must contain. The spec is no longer "describe each scene once" — it's "describe each scene once, plus describe how every perception dimension transforms it."

- **Perception vector definition becomes a first-class spec component.** Every player character needs a full perception profile: psychological conditions (agoraphobia, social anxiety, paranoia, etc.), skill levels (combat, stealth, perception, charisma, crafting, magic), physical states (hunger, exhaustion, injury — these may be dynamic rather than authored), relationship initial values, moral/worldview starting position. Each as a named float with a progression arc over the game timeline.
- **Per-scene perception annotations.** Each scene spec must include which perception dimensions are *narratively active* in that scene. The market square scene activates agoraphobia and social anxiety modifiers. The forest dungeon activates stealth and perception modifiers. Not every modifier fires everywhere — the spec must say which ones matter where, or the combinatorial space is unmanageable.
- **Multi-character specs.** If the game supports replay as a different character, the spec must define multiple perception profiles and the Story Director must consider how each major scene reads through each character's stack. The spec becomes a matrix: scenes × characters × active perception dimensions.
- **Knowledge and cognition state tracking.** The spec must define what each character knows at each point in the story — for "ignorance as rendered assumption" (locations render based on hearsay until visited) and "memory as degraded rendering" (revisited locations render from memory). This is a per-character, per-scene knowledge manifest.
- **Perception progression events.** The spec must mark story events that change the perception vector: "after the mentor scene, agoraphobia drops by 0.2," "after the betrayal, trust_default drops by 0.3, paranoia increases by 0.2." These are authored narrative beats that drive mechanical state changes.

**Infinite Dialog Impact:**

The game spec expands significantly to accommodate dialogue as a first-class system rather than a scripted accessory.

- **NPC knowledge graphs.** Every NPC needs a knowledge profile: what they know, what they believe (possibly incorrectly), what they want to hide, and what they want to learn. This is not a simple flag list — it's a structured knowledge model that the Dialogue Writer and runtime dialogue generator both consume. The spec must define these per-NPC.
- **Character voice profiles.** Each character (player and NPC) needs a voice specification: vocabulary level, sentence structure patterns, verbal tics, emotional range, formality default, humor style. Sufficient for the Dialogue Writer to maintain consistency across hundreds of dialogue nodes and for the runtime generator (if hybrid) to produce on-voice options.
- **Conversation-as-combat encounter definitions.** High-stakes dialogue encounters need spec-level definitions alongside physical encounters: NPC stance model (starting stance, stance transitions, breaking points), available argument types per character trait level, failure states and consequences, success states and rewards. These are encounter definitions that go to the Dialogue Writer, not the Encounter Designer.
- **NPC relationship network.** The spec must define the initial relationship graph: who knows whom, who trusts whom, who is lying to whom, faction affiliations, gossip channels (which NPCs talk to which other NPCs). This drives gossip propagation and information network behavior.
- **Gossip propagation rules.** How fast does information travel? Which NPCs are gossip hubs? What information gets distorted in retelling? The spec must define the propagation model at least at the narrative level: "news from the forest reaches town within one scene transition, but arrives distorted — the NPC heard there was a fight but doesn't know who won."
- **Player choice point map.** Where in the story does the player have meaningful dialogue choices, and what are the consequence branches? The spec must mark these explicitly so the Dialogue Writer knows which conversations are linear delivery and which are branching decision points.
- **Tone drift trajectory per character.** The spec must define how the player character's dialogue voice evolves: early game vocabulary, mid game vocabulary, late game vocabulary. Not individual lines but the parameters that shift — confidence level, sentence complexity, initiative (reactive → proactive).

**Animated Narrator Impact:**

The game spec must define the narrator as a first-class character with its own profile and arc.

- **Narrator personality profile.** The spec must define the narrator's voice (tone, vocabulary, reliability level, humor style), initial relationship stance toward the player, and how that stance evolves based on play style categories (cautious, aggressive, thorough, dismissive). This is a character definition as rich as any NPC's.
- **Per-scene narrator stance.** Each scene spec must include whether the narrator agrees or disagrees with the subjective rendering — immersing the player in the character's perception or pulling them out with critical distance. This is the agree/disagree axis that creates the push-pull between immersion and awareness.
- **Foreshadowing seeds and payoff markers.** The spec must define information the narrator plants early and pays off later: which scenes carry foreshadowing, which carry payoffs, and what the narrator knows but withholds at each point. This is an information pacing plan that spans the entire game.
- **Dramatic irony annotations.** The spec must mark moments where the narrator tells the player things the character doesn't know. These annotations must specify what the character is ignorant of, what the narrator reveals, and how dialogue options remain constrained by the character's ignorance despite the player's knowledge.
- **Teaching cues.** The spec must mark mechanics the narrator should teach through description rather than UI, along with player behavior triggers that determine when hints are delivered and when they're unnecessary.
- **Per-character narrator voice for replay.** If the game supports multi-character replay, the spec must define how the narrator's entire voice changes per viewpoint character — warm and protective for the naive character, cold and analytical for the villain, conspiratorial for the rogue.

**Narrative Enemies Impact:**

The game spec expands to treat every enemy as a narrative entity, not just a mechanical obstacle.

- **Enemy biography and motivation.** Every enemy (not just bosses) needs a spec-level biography: why they fight, what they protect, who they care about, and what discoverable information exists in the world about them. The spec must define where this context is seeded — NPC gossip, documents, environmental clues.
- **Combat-as-moral-choice information map.** The spec must map which encounters have moral dimensions and what information paths exist for the player to discover them before, during, or after the fight. A rushed player misses context; a thorough player fights with narrative weight. The spec must define both experiences.
- **Villain mirror arc.** The main antagonist's spec must define adaptive responses to player behavior categories: how the villain evolves in response to a cruel player, a merciful player, a chaotic player. The villain's final speech is parameterized by the player's event log.
- **Recurring enemy progression.** Named recurring enemies need arc definitions: encounter-by-encounter evolution of behavior, dialogue, and emotional state, built from shared combat history rather than fixed scripted beats.
- **Enemy faction dynamics.** The spec must define internal faction structures: loyalty graphs, command hierarchies, potential dissidents, and behavioral rules for group responses to leader death, comrade death, and morale breaks.
- **Boss fight narrative phases.** Boss encounters must be specified as narrative climaxes with per-phase emotional beats, phase transitions triggered by story revelations (not just HP thresholds), and optional dialogue choice windows between phases.

**Replay and Multiplayer Impact:**

The game spec must be structured for multi-perspective generation, not single-playthrough authoring.

- **Multi-perspective scene matrix.** The spec must define how each major scene reads from multiple character perspectives. The base scene is one structure; the spec annotates how perception, dialogue, narrator voice, and available information change per viewpoint character. This is the foundation for both replay and parallel perspective multiplayer.
- **Hidden layer annotations.** The spec must mark information withheld on first playthrough that surfaces on subsequent plays — the same scene carrying different meaning with new context. These are layered reveals, not branching paths.
- **Asynchronous inheritance points.** The spec must define which world state elements are exportable to other players' worlds: faction outcomes, environmental changes, NPC relationship states, character reputation summaries. These are the seams where one player's story feeds another's generation pipeline.
- **Ghost trace seeds.** The spec must define what kinds of diegetic traces a playthrough leaves: grave markers, worn paths, NPC stories about past events. These must be world-appropriate artifacts, not multiplayer UI.
- **Generational play contracts.** For successor-generation stories, the spec must define what the previous player's playthrough becomes in the new world — statues, legends, political consequences, environmental scars — and how the narrator references them.

**Environmental Memory Impact:**

The game spec must define locations as entities with memory, personality, and agency potential.

- **Per-location memory profiles.** The spec must define which locations accumulate memory, their personality bias (a church remembers differently from a battlefield), and their contagion susceptibility (does mood spread to neighboring locations).
- **Environmental agency thresholds.** The spec must define at what point a location's accumulated history grants it agency — when the forest starts rearranging paths, when the garden starts helping the player. These are narrative beats tied to event accumulation, not fixed triggers.
- **Temporal echo conditions.** The spec must define which locations support temporal echoes (ghostly replays of past events), what perception thresholds are required to see them, and what information the echoes contain that serves as gameplay (detective mechanics, hidden history).
- **Emotional archaeology layers.** The spec must define how emotional layers coexist at a location without overwriting — warmth from a celebration, pain from a betrayal, and healing from a reconciliation all present simultaneously, informing rendering and narration.
- **Cross-player memory inheritance.** For multiplayer modes, the spec must define how one player's location history persists for the next player — the forest that Player A nurtured is welcoming to Player B, who doesn't know why.

---

## 3. Story Director ↔ Pacing Analyst

A feedback loop that runs before domain agents begin work. The Pacing Analyst evaluates the structural shape of the game and suggests revisions.

**What crosses the boundary:**

- **Director → Analyst:** the full scene sequence with per-scene metadata: scene type (exploration, combat, dialogue, mixed), estimated duration, combat density, dialogue density, choice point presence, dramatic beat intensity
- **Analyst → Director:** structural feedback — pacing problems, rhythm issues, density imbalances. "Three dialogue-heavy scenes in a row." "No combat for a long stretch in the mid-game." "The player hasn't had a choice point in four scenes."

**Open questions:**

- What is the vocabulary for scene types and pacing metrics? What makes a scene "dialogue-heavy" quantitatively?
- What are the pacing heuristics? Are they genre-specific (JRPG pacing vs. horror pacing)?
- How many feedback iterations are allowed before the loop terminates?

**Infinite Dialog Impact:**

Dialogue density is a pacing dimension the Pacing Analyst must now evaluate with more nuance.

- **Conversation-as-combat counts as both.** A high-stakes dialogue encounter (negotiation, interrogation) is simultaneously a dialogue scene and a tension/combat-equivalent scene. The Pacing Analyst must categorize these correctly — three "dialogue scenes" in a row where one is a tense hostage negotiation isn't the same as three exposition dumps.
- **Choice point density.** The Pacing Analyst must track meaningful player choice frequency. Infinite dialog means more conversations could have choices, but too many choices causes decision fatigue. The Analyst needs a metric: "meaningful choices per play-hour" with genre-appropriate targets.
- **Eavesdropping as pacing texture.** Ambient NPC conversations (eavesdropping content) provide information without requiring player engagement. The Pacing Analyst should account for these as passive information delivery — they can substitute for exposition-heavy dialogue scenes and improve rhythm.

---

## 4. Atmosphere Director → All Domain Agents

A cross-cutting pass that runs after the story spec is approved and before domain agents begin. Ensures unified mood across all outputs for a given scene.

**What crosses the boundary:**

- **Atmosphere Director → Level Designer:** color palette, lighting mood (warm/cold/neutral, intensity), time of day, weather, particle effects (rain, dust, fireflies, fog), environmental sound mood
- **Atmosphere Director → Character Designer:** expression defaults for the scene (NPCs generally tense, relaxed, fearful), body language bias
- **Atmosphere Director → Encounter Designer:** combat mood (desperate, playful, oppressive), environmental combat modifiers
- **Atmosphere Director → Script Director:** dramatic tone (intimate, epic, eerie), timing feel (urgent, languid)
- **Atmosphere Director → Dialogue Writer:** conversational tone (guarded, warm, tense), vocabulary register
- **Atmosphere Director → Tier 3 Renderers:** style intensity, palette constraints, texture mood

**Open questions:**

- What is the format for atmosphere descriptors? A flat key-value map per scene? A structured object with typed fields?
- Does atmosphere include perception-variant descriptors? (Atmosphere at anxiety 0.3 vs. 0.8 for the same scene.) If so, the Atmosphere Director needs the perception rules as input.
- How granular is atmosphere — per scene, per map region, per room?

**Subjective Rendering Impact:**

The Atmosphere Director can no longer emit a single atmosphere descriptor per scene. Subjective rendering means the atmosphere is perception-dependent — the same scene has different mood depending on who's experiencing it.

- **Perception-variant atmosphere descriptors.** The Atmosphere Director must emit atmosphere as a function of perception state, not a flat value. For the market square: at agoraphobia < 0.3, palette is warm and inviting, particle effects are gentle sunlight; at agoraphobia > 0.7, palette shifts cold, particle effects become oppressive crowd dust, lighting flattens. This means atmosphere descriptors are keyed to perception thresholds or expressed as continuous interpolation rules.
- **The Atmosphere Director needs perception rules as input.** It must receive the Story Director's perception annotations for the scene to know which dimensions are active and what the narrative intent is per dimension. The information flow changes: Story Director → Atmosphere Director is no longer just "scene description + emotional context" but also "perception dimensions active in this scene + narrative intent per dimension."
- **Cross-cutting mood interactions.** Depression desaturates the palette. Guilt darkens specific locations. Hunger saturates food objects. These can conflict — depression desaturating while hunger saturates. The Atmosphere Director must define resolution rules for palette conflicts across perception dimensions. This is a concrete authoring burden that didn't exist without subjective rendering.
- **Relationship-driven atmosphere.** Love/infatuation brightens the environment around one specific NPC. Rivalry makes a rival's territory feel more imposing. These are per-entity atmosphere modifiers, not per-scene — a finer granularity than the base interface assumed.

**Infinite Dialog Impact:**

- **Conversational tone as atmosphere component.** The Atmosphere Director must emit conversational tone descriptors per scene: guarded, warm, tense, conspiratorial, formal, casual. These constrain the Dialogue Writer's register and the runtime generator's prompt. In a funeral scene, NPC ambient dialogue is hushed and subdued; in a festival, it's boisterous. This is atmosphere applied to the dialogue subsystem.
- **Eavesdropping atmosphere.** The density, volume, and content-type of ambient NPC conversations is an atmosphere property. A busy market has overlapping fragments. A tense military camp has clipped, functional exchanges. A deserted village has silence (which is itself meaningful). The Atmosphere Director must specify ambient dialogue mood alongside visual/audio mood.

---

## 5. Story Director → Level Designer

Spatial intent translated into playable space.

**What crosses the boundary:**

- **Spatial description:** natural-language description of the area ("autumn forest, branching paths, hidden shrine at the north end")
- **Narrative purpose:** why this space exists in the story (exploration, ambush site, safe haven, puzzle area, dramatic stage)
- **Spatial constraints:** required features (must have a clearing for the boss fight, must have a hidden path to the shrine, must connect to the mountain pass exit)
- **Atmosphere descriptors:** from the Atmosphere Director
- **Perception modifier rules:** how this location should change under different perception states (at agoraphobia > 0.7, open areas expand; at stealth skill > 0.5, cover objects appear)
- **Environmental memory seeds:** initial personality bias, contagion susceptibility, what kinds of events this location should remember

**Open questions:**

- What is the vocabulary for spatial intent? How does "a maze" differ from "branching paths" differ from "open field with scattered cover" in a way the Level Designer can parse?
- How are perception-variant maps represented? Multiple tilemap layers keyed to trait thresholds? A base map plus modifier overlays?
- What are the size/budget constraints per map? Tile count limits, entity count limits?

**Subjective Rendering Impact:**

The Level Designer's job fundamentally changes. It no longer produces one map per scene — it produces a **base map plus a set of perception-variant overlays** or multiple map variants keyed to perception state.

- **Agoraphobia / spatial distortion.** Open areas must have variant tilemaps at different perception thresholds. The market square at agoraphobia 0.2 is 20×20 tiles. At 0.7 it's 40×40 with repeated tile patterns and denser crowd entities. The Level Designer must author (or the LLM must generate) these expanded variants while maintaining navigability — the player must still be able to cross the space, just with greater difficulty.
- **Stealth skill → cover generation.** The base map has a certain layout. At stealth > 0.5, additional cover objects (boxes, hedges, shadow tiles) appear. The Level Designer must define cover insertion points — positions where cover can be added without breaking navigation or visual coherence. This is a new map annotation type: "potential cover slot."
- **Perception/awareness → hidden feature revelation.** Secret passages, hidden items, environmental clues exist in the base map data but are invisible below a perception threshold. The Level Designer must author two tile layers for these features: the concealed version (plain wall) and the revealed version (cracked wall with visible seam). The perception engine swaps between them.
- **Exhaustion → distance stretching.** The base map is the "real" size. Under exhaustion, the engine inserts repeated tile sections to make paths feel longer. The Level Designer must mark which path segments are stretchable — "this corridor can be extended by repeating tiles 5-12" — without creating visual artifacts or breaking trigger/entity placement.
- **Ignorance → assumption rendering.** Locations the player hasn't visited render based on hearsay. The Level Designer must generate two versions of unvisited-but-referenced locations: the "assumed" version (dark and threatening, if that's what NPCs said) and the "real" version. The assumed version isn't a gameplay space — it's a visual for the map edge or a cutaway view.
- **Memory → degraded revisit rendering.** When a location is remembered (referenced in dialogue, thought about), it renders from memory. The Level Designer must define how maps simplify under memory degradation: which details drop out, which elements blur or shift, what the "emotional essence" tileset is for each map.
- **Guilt / moral worldview → location-specific darkening.** Locations where morally significant events occurred need a darkening/staining overlay keyed to the guilt value. The Level Designer must flag tiles as "moral event site" so the perception engine knows where to apply the overlay.

The Level Designer's output format changes from `tilemap` to `tilemap + perception_overlays[]`, where each overlay specifies: perception dimension, threshold or interpolation curve, tile substitutions or insertions, and spatial transformation rules.

**Infinite Dialog Impact:**

Dialogue becomes a spatial mechanic, which means the Level Designer must account for it.

- **Eavesdropping zones.** NPCs have conversations at specific positions on the map. The Level Designer must define NPC conversation positions and audibility ranges — the area within which the player can overhear. These are a new map annotation type: `conversation_zone(npc_pair, position, audibility_radius)`. Placement matters narratively: a conversation near a pillar the player can hide behind is a designed eavesdropping opportunity.
- **Conversation spatial design.** Meaningful dialogue encounters happen in space. A negotiation in a throne room has different spatial dynamics than one in a narrow alley. The Level Designer must consider dialogue staging when placing named positions — where the player character and NPC stand, whether other NPCs can observe, whether there's an exit the player can retreat to if the conversation fails.
- **NPC movement and conversation routes.** If NPCs have ongoing lives with conversations, they need movement paths that bring them into contact. The Level Designer must define NPC patrol/routine paths that include conversation meeting points — "guard A and guard B converge at the gate at scene-time T and hold a conversation for N beats."
- **Information-gated spatial design.** Some paths or areas might only become apparent through dialogue (an NPC tells you about a shortcut). The Level Designer must mark these as dialogue-revealed features, coordinated with the Dialogue Writer's output.

**Replay and Multiplayer Impact:**

Maps must support multi-perspective use and cross-player inheritance.

- **Perspective-variant spatial emphasis.** The same map may need different named positions, staging areas, and focal points depending on which character experiences it. The hero enters from the gate; the townsperson watches from the market stall. The Level Designer must annotate maps with per-character spatial entry points and significant positions.
- **Inherited environmental state.** Maps must define how cross-player inheritance manifests spatially: the bridge Player A destroyed stays gone for Player B, the village Player A saved shows scaffolding and growth. The Level Designer must define which map features are inheritable and what their variant states are.
- **Ghost trace spatial anchors.** Maps must define positions where ghost traces can manifest — graves, worn paths, memorial sites — as conditional spatial elements that appear only when prior-player state includes relevant events.

**Environmental Memory Impact:**

Maps are no longer static — they accumulate history and develop personality between visits.

- **Mutable map regions.** The Level Designer must define which parts of a map can change between visits: paths that rearrange, clearings that shrink or expand, cover that appears or disappears. These are annotated as `mutable_region(type, change_rules, memory_threshold)`.
- **Spatial contagion boundaries.** Maps must define how environmental mood spreads to neighboring maps. The Level Designer must mark contagion edges — tile regions at map boundaries where darkness or healing can seep through from adjacent locations.
- **Temporal echo staging.** Maps must define positions and conditions for temporal echo overlays — where ghostly replays of past events appear, what tile variants represent the echoes, and how they layer over the live map without breaking navigation.
- **Decay and growth tile variants.** The Level Designer must produce variant tiles for environmental aging: broken versions of intact tiles, overgrown versions of maintained tiles, warmer versions of cold tiles. These are the visual vocabulary for a location's personality evolving through accumulated events.
- **Environmental agency spatial rules.** For locations that develop agency, the Level Designer must define how the space changes when the location acts: path rearrangement rules, cover generation/removal patterns, shortcut revelation conditions. These are spatial behavior patterns analogous to NPC behavior patterns.

---

## 6. Story Director → Character Designer

Character identity translated into visual specification.

**What crosses the boundary:**

- **Appearance descriptors:** physical description, clothing, distinguishing features, silhouette profile (should be recognizable from silhouette alone)
- **Personality:** traits that affect visual expression — confident posture vs. hunched, open gestures vs. closed, energetic movement vs. slow
- **Role in story:** protagonist, antagonist, mentor, trickster, etc. — informs visual archetype expectations
- **Perception traits:** how this character should render to others (a feared character renders as larger to anxious viewers), how the world renders to this character (their perception vector)
- **Cast context:** other characters in the game, for visual differentiation and thematic grouping (villagers should look like they belong together; the villain should contrast)

**Open questions:**

- What is the format for appearance descriptors that serves both the Character Designer LLM and the Tier 3 image generators? Natural language? Structured fields (hair color, build, clothing type)?
- How is "visual personality" (posture, energy, gesture vocabulary) encoded for sprite generation?
- How many sprite states and expressions are in the standard character template? This must be fixed across the system.

**Subjective Rendering Impact:**

Characters don't have one fixed appearance — they have a base appearance plus perception-variant renderings.

- **Body dysmorphia → NPC rendering inversion.** If the player character has body dysmorphia, other characters render as the opposite of the player's self-perception. The Character Designer must produce variant sprite descriptions keyed to this trait: the same NPC looks different through a dysmorphic lens. This may mean variant sprite sheets or palette/scale modifiers the engine applies at runtime.
- **Love/infatuation → enhanced rendering of one character.** A loved character renders with more detail, warmer palette, slight glow. The Character Designer must define what "enhanced" means per character — which visual features intensify, what the glow/warmth effect looks like. This could be a sprite variant or a render-time shader effect.
- **Rivalry → scale distortion.** A rival renders larger and more imposing. The Character Designer must ensure sprites scale cleanly — pixel art that looks correct at 1× may not at 1.3×. This constrains the art style or requires the Character Designer to produce explicitly oversized variants.
- **Social anxiety / trust → NPC body language variants.** Trusted NPCs face the player with open posture. Distrusted NPCs turn away, face exits. Under social anxiety, all NPCs exhibit gaze behavior (looking at the player, whispering). The Character Designer must define these behavioral sprite variants — not just "idle" but "idle_facing_player," "idle_turned_away," "idle_whispering." This expands the standard sprite state vocabulary.
- **Cynicism → transparent motivations.** Under high cynicism, NPCs render with visual cues that expose their motivations (greedy eyes on a merchant, clenched fists on a jealous NPC). The Character Designer must define "exposed motivation" sprite variants or overlays per NPC archetype.
- **Innocence/naivety → simplified NPC rendering.** A naive character sees simpler, more colorful versions of NPCs — dangers are less visible in their appearance. Thugs look like ordinary people. The Character Designer must produce "naive-view" variants where threatening visual cues are absent.

The character asset template expands from a fixed set of states to: **base states + perception-variant states**. The number of variants per character is bounded by which perception dimensions the Story Director marks as active for scenes containing that character.

---

## 7. Story Director → Encounter Designer

Narrative combat intent translated into mechanical encounter definitions.

**What crosses the boundary:**

- **Encounter description:** narrative purpose ("tests the player," "overwhelming horde," "puzzle boss"), intended difficulty feel, story context
- **Enemy biography:** motivation, faction, relationships to other enemies, personal history relevant to behavior
- **Phase narrative beats:** for boss fights — what each phase should feel like emotionally, what story information is revealed at phase transitions
- **Difficulty feel vocabulary:** "not lethal," "punishing," "desperate," "playful," "unfair-on-purpose" — mapped to mechanical constraints
- **Faction dynamics:** internal loyalties, dissent, command structure — affects group behavior on leader death, morale breaks, surrender conditions

**Open questions:**

- What is the vocabulary for difficulty feel? How does "tests the player" translate to HP/damage/speed constraints?
- How are enemy relationships represented? A graph? Per-enemy metadata referencing other enemies?
- How does the encounter definition specify adaptive behavior (the villain-as-mirror mechanic, taunts from game state)?

**Subjective Rendering Impact:**

Subjective rendering transforms encounters from objective mechanical events into perception-dependent experiences.

- **Combat mastery → enemy behavior perception.** At high combat skill, enemy telegraphing becomes more visible and their movements appear slower. The Encounter Designer must define enemy behavior at multiple skill perception levels — not changing the actual speed/damage (that's balancing), but specifying how the enemy's movements are *presented*. At skill 0.3, a charge attack has a 200ms telegraph. At skill 0.9, the same attack has a 500ms telegraph (the character reads the windup earlier). This is a perception modifier on encounter timing, not on encounter mechanics.
- **Paranoia → false enemies.** Paranoid perception spawns enemies that aren't real — shadows that look like threats, movement in peripheral tiles, hostile-looking NPCs that are actually neutral. The Encounter Designer must define "false encounter" templates that the perception engine can inject when paranoia is above threshold. These must be visually convincing but mechanically inert (no damage, no collision) until the player attacks first — then they either vanish (hallucination) or become real (the paranoia was justified). The Encounter Designer needs a new entity type: `phantom_enemy`.
- **Innocence/naivety → danger concealment.** A naive character doesn't perceive enemies as threatening until they attack. The Encounter Designer must define how enemies present under naivety: a wolf looks like a friendly dog until it lunges, a bandit looks like a traveler until they draw a weapon. This requires paired entity states: `perceived_neutral` and `actual_hostile`, with the transition triggered by proximity or aggression.
- **Perception/awareness → encounter telegraphing.** High perception reveals enemy positions before visual contact (footprints, sounds, disturbed environment). The Encounter Designer must define per-encounter environmental cues that appear at perception thresholds — cues that the Level Designer places in the map as conditional elements.

**Infinite Dialog Impact:**

Conversation-as-combat creates a new encounter type that the Encounter Designer must handle.

- **Dialogue encounters as encounter definitions.** A negotiation, interrogation, or persuasion scene has the same structural needs as a combat encounter: NPC stance model (defensive/aggressive/open), transition triggers (what shifts the stance), failure states (NPC walks away, calls guards, attacks), success conditions. The Encounter Designer must produce these definitions using a dialogue-combat vocabulary: `stance_model(initial, transitions[], break_points[])`, `argument_types(available_per_trait_level)`, `failure_consequence(type, severity)`.
- **Combat-to-dialogue transitions.** An enemy encounter can transition to dialogue mid-fight (the enemy surrenders, offers information, pleads). The Encounter Designer must define surrender/negotiation triggers: HP thresholds, morale breaks, relationship conditions. These triggers hand control from the combat system to the dialogue system with a defined NPC emotional state.
- **Enemies that talk during combat.** Taunts, revelations, and pleas during combat are dialogue events within encounter definitions. The Encounter Designer must specify taunt triggers, taunt content parameters (what game state to reference), and whether taunts are interruptible (can the player respond mid-combat?).

**Narrative Enemies Impact:**

Every encounter becomes a narrative event, not just a mechanical challenge.

- **Discoverable motivation integration.** The Encounter Designer must coordinate with the Dialogue Writer and Level Designer to place discoverable context — letters, NPC gossip, environmental clues — that reframes the encounter's meaning. The encounter definition must reference these information sources and define how combat changes if the player knows the enemy's story.
- **Non-enemy encounter templates.** The Encounter Designer must define encounters where the correct response is not combat: the mother protecting a nest, the refugees defending their camp, the panicking NPC. These use the combat system (the player can fight) but include behavioral cues (fear, desperation, protection) and alternative resolution paths (back off, calm down, find another route).
- **Recurring enemy state tracking.** Named recurring enemies accumulate encounter history. The Encounter Designer must define how behavior, tactics, dialogue, and emotional state evolve based on prior encounters — not scripted escalation but adaptation generated from shared combat history.
- **Enemy relationship behavior rules.** The Encounter Designer must define group behavior driven by internal relationships: loyalty-based troops who surrender on leader death vs. rage, friends who fight harder or flee when a companion falls, command dynamics where leaderless groups scatter or reorganize.
- **Boss phase-narrative correspondence.** Each boss phase must map to a narrative beat with defined emotional intent. Phase transitions are triggered by story revelations or dialogue choices, not just HP thresholds. The Encounter Designer must define per-phase behavior, dialogue, and transition conditions as a narrative arc.
- **Violence feedback loop.** The player's combat history (kill/spare patterns, fighting style, frequency of violence) must feed back into encounter definitions. The Encounter Designer must define how the player's relationship to violence affects future encounter presentation — desensitization flattening combat, trauma distorting it, reputation causing enemies to surrender earlier or fight harder.

---

## 8. Story Director → Script Director

Dramatic intent translated into choreographed engine sequences.

**What crosses the boundary:**

- **Dramatic beat descriptions:** what happens emotionally, which characters are involved, what the player should feel, narrative stakes
- **Available script commands:** the complete set of engine primitives the Script Director can use (the engine capability schema)
- **Spatial context:** which map this plays on (from Level Designer, but the Story Director specifies which scene uses which map)
- **Character context:** which characters are present, their current emotional state, their relationship to each other at this point

**Open questions:**

- What is the formal, closed vocabulary of script commands? Currently listed informally: `move_entity`, `face`, `dialogue`, `wait`, `screen_shake`, `fade`, `play_effect`, `set_flag`. What's the complete set?
- How does the Script Director handle player agency within scripts? Can the player interrupt a script? Are there branching scripts (script forks based on player action mid-sequence)?
- How are timing and pacing encoded? Absolute milliseconds? Relative beats? Tempo markers?

**Subjective Rendering Impact:**

Play scripts must be perception-aware — the same dramatic beat may play differently depending on the character's state.

- **Contrast as storytelling → perception clearing.** The most powerful subjective rendering moment is when distortion lifts. The Script Director needs a new command: `clear_perception(trait, duration)` or `set_perception(trait, value, duration)` — temporarily overriding the perception vector during a scripted sequence. "After the mentor scene, the market square renders without agoraphobia for the first time." This is a script-authored emotional beat that uses the perception engine as its instrument.
- **PTSD → flashback scripting.** PTSD trigger zones warp the player to a different map (a memory). The Script Director must be able to author flashback sequences that interrupt normal play scripts: `trigger_flashback(memory_map, duration, exit_condition)`. The flashback is itself a play script running on a different map with different entities, then returning control to the present.
- **Depression → timing distortion.** Under depression, script timing stretches — waits feel longer, movements are slower, pauses between dialogue lines extend. The Script Director can either author perception-variant timing (explicit slow versions) or define a timing modifier that the perception engine applies: `timing_scale = lerp(1.0, 1.5, depression)`. The latter is cleaner but requires the engine to support timing modification on active scripts.
- **Obsession → entity injection.** During scripts, obsession can cause the object of obsession to appear briefly — a face in the crowd during a scripted procession, a silhouette in a doorway during a dramatic scene. The Script Director must be able to define conditional entity appearances: `if obsession > 0.6, inject phantom_entity(obsession_target, position, duration)`.
- **Multi-character scripts.** If a scripted scene is experienced by different characters on replay, the Script Director may need to define per-character script variants — same events, different framing, different entity emphasis. Or the perception engine handles this automatically through the modifier stack. This is an architectural choice: explicit script variants vs. perception-modulated script playback.

**Infinite Dialog Impact:**

Scripts and dialogue interleave — dramatic sequences contain dialogue, and dialogue can trigger scripts.

- **Dialogue within scripts.** Play scripts already include `dialogue(who, text, expression)` commands. Infinite dialog means these aren't fixed text — they're dialogue nodes that may offer the player choices mid-script. The Script Director must distinguish between `dialogue_fixed(who, text, expression)` (non-interactive, part of the choreography) and `dialogue_interactive(conversation_id)` (hands control to the dialogue system, script pauses until the conversation resolves, then resumes based on outcome).
- **Script branching on dialogue outcome.** A scripted dramatic sequence may fork based on what the player said: "If the player convinced the guard, the script proceeds to the gate opening; if not, the script proceeds to the alarm." The Script Director must define conditional continuations keyed to dialogue outcome flags.
- **Silence as scripted beat.** The Script Director can use silence (the player choosing to say nothing) as a dramatic moment within a choreographed sequence: `dialogue_interactive(conversation_id)` where one option is silence, and the script's continuation for the silence path is specifically authored as a dramatic beat — the NPC reacts to the silence, the pause holds, the narrator comments.
- **Eavesdropping as script trigger.** An NPC conversation overheard by the player can trigger a script: the player hears something alarming, a scripted reaction plays (camera pan to the player's face, narrator commentary, flag set). The Script Director must define eavesdropping-triggered scripts keyed to the player entering a conversation zone.

**Animated Narrator Impact:**

The Script Director must choreograph narrator beats alongside entity movement and dialogue.

- **Narrator as script participant.** The script command vocabulary must include narrator-specific commands: `narrator(text, tone, stance)` for narration during choreographed sequences. The narrator can agree with, contradict, or recontextualize what's happening on screen — the Script Director must specify which.
- **Dramatic irony scripting.** The Script Director must be able to script moments where the narrator reveals information the character doesn't have: "You don't notice the letter she slips into her pocket." The script must coordinate the narrator's revelation with character behavior that demonstrates ignorance.
- **Emotional amplification beats.** Before boss fights, after losses, during quiet moments — the Script Director must script narrator beats that function as the textual equivalent of a film score: building tension, expressing grief, or using restraint. These are timing-sensitive narrator commands interleaved with the choreography.
- **Three-way tension scripting.** The Script Director must be able to script moments where visuals, dialogue, and narration disagree simultaneously. This requires coordinating subjective rendering state, NPC dialogue, and narrator text as three independent channels within a single scripted sequence.

**Narrative Enemies Impact:**

Combat encounters become scripted narrative events, not just mechanical fights.

- **Boss fight scripted transitions.** Phase transitions in boss fights are dramatic beats the Script Director must choreograph: the boss pauses, reveals something, the narrator comments, the player may have a dialogue choice. The Script Director must define these transition sequences as interruptible play scripts between combat phases.
- **Combat-to-dialogue transition scripting.** The Script Director must define choreographed transitions from combat to dialogue: an enemy drops their weapon, raises their hands, the combat system yields to the dialogue system. The reverse too — a failed negotiation where the NPC attacks and the script hands control to the combat system.
- **Taunt choreography.** Enemy taunts during combat are scripted dialogue events. The Script Director must coordinate taunt timing with combat behavior — a taunt during a charge pause, a revelation during a phase transition — so that information delivery and combat flow reinforce each other.

**Replay and Multiplayer Impact:**

Scripts must support multi-perspective playback and cross-player variation.

- **Per-character script variants.** The same scripted event experienced by different characters on replay needs different framing: different entity emphasis, different narrator voice, different information revealed. The Script Director must define per-character variant layers on top of the base choreography, or the perception engine must handle this automatically.
- **Inherited event references in scripts.** In multiplayer/generational modes, scripted sequences may reference prior-player events: a ceremony at a statue of the previous hero, a narrator callback to inherited history. The Script Director must define conditional script segments triggered by inherited world state.

---

## 9. Story Director → Dialogue Writer

Narrative and character context translated into dialogue content.

**What crosses the boundary:**

- **Scene context:** where, when, what just happened, what's about to happen, narrative stakes
- **Character profiles:** personality, speaking style, vocabulary, verbal tics, knowledge (what this character knows and doesn't), motivation (what they want from this conversation), perception traits that filter available player options
- **Emotional beats:** the intended emotional arc of the conversation — starts casual, turns tense, ends with a revelation
- **Player choice points:** where the player should have agency, what the meaningful choices are, what consequences they trigger (flag sets, relationship changes, branch selection)
- **Conversation-as-combat parameters:** if applicable — NPC stance model, credibility/patience mechanics, failure states
- **NPC knowledge and motivation:** for lying/unreliability — what the NPC knows, what they want to hide, how skilled they are at deception
- **Character voice reference:** enough information to maintain voice consistency across the whole game — vocabulary level, sentence structure patterns, emotional range, verbal habits

**Open questions:**

- What is the format for "character voice" that ensures consistency? A style guide per character? Example dialogue? A parameterized voice model (formality: 0.7, verbosity: 0.3, warmth: 0.8)?
- How are dialogue options tagged for perception filtering? If charisma < 0.5 suppresses a tactful option, how is that rule attached to the option?
- How are NPC conversation stances (defensive, aggressive, open) represented and transitioned?

**Subjective Rendering Impact:**

Dialogue is one of the subsystems most heavily modulated by the perception vector. The Dialogue Writer's output must account for perception-based filtering, distortion, and character limitation.

- **Trait-based option filtering is the core mechanic.** Narcissism limits options to egotistical choices. Low social skill produces only blunt utterances. Autism generates options that fixate on precision. Shyness produces tentative, indirect options. The Dialogue Writer must generate a **superset of options per conversation node**, each tagged with perception trait requirements: `{text: "Perhaps we could discuss this calmly", requires: {charisma: >0.6, courage: >0.4}}`. At runtime, options that don't meet the current perception vector's thresholds are pruned. The player only sees what their character can conceive of.
- **Courage gates confrontation.** The option to directly confront someone only appears above a courage threshold. Below it, only indirect approaches are available. The Dialogue Writer must author both the direct and indirect option sets for the same narrative beat, tagged with the threshold.
- **Perception gates observation.** The option to call out an NPC's lie ("I noticed you flinched when you said that") only appears at high perception. The Dialogue Writer must author lie-detection options and tag them with perception requirements. Below threshold, the player has no way to challenge the NPC's deception — the character doesn't see it.
- **Intelligence gates articulation.** At low intelligence, complex ideas are expressed vaguely or incorrectly. The same intended meaning has multiple authored versions at different intelligence levels — from "Something's wrong with the water" to "The contamination pattern suggests a point source upstream of the eastern district."
- **Knowledge gates content.** A character who hasn't learned a fact can't reference it in dialogue. The Dialogue Writer must tag knowledge-dependent options with fact-store prerequisites: `{text: "The hermit told me about the ritual", requires: {knows: "hermit_ritual_conversation"}}`.
- **Misunderstanding generation.** At low social skill or with specific conditions (autism, social anxiety), options must include the misunderstanding mechanic — the player's chosen text means something different to the NPC than to the character. The Dialogue Writer must author both the character's intended meaning and the NPC's received meaning, with the gap determined by the relevant perception traits.

**Infinite Dialog Impact:**

The Dialogue Writer becomes the most complex domain agent. Infinite dialog transforms its output from static trees to a multi-layered generation system.

- **Conversation momentum model.** Each conversation must define a momentum/commitment state: the current tone (aggressive, friendly, formal, desperate), how each option shifts the tone, and how the NPC's available responses narrow based on accumulated tone. The output format expands from `{option_text, npc_response}` to `{option_text, tone_shift, npc_response, npc_stance_change, new_available_tones}`. This is the mechanical backbone of conversational regret.
- **NPC voice consistency contract.** The Dialogue Writer receives a voice profile per NPC and must produce dialogue that's verifiably on-voice across all conversations in the game. For pre-generated content, this is auditable. For runtime generation, the voice profile becomes part of the LLM prompt — and the system needs a verification pass that checks generated dialogue against the voice profile.
- **Conversation-as-combat dialogue trees.** High-stakes conversations need a different tree structure: each node has an NPC stance, the player's options shift the stance, there's a "health bar" equivalent (patience, trust, credibility) that depletes on bad choices. The Dialogue Writer must produce these as structured encounter-dialogues with explicit failure/success states, not open-ended trees.
- **NPC-to-NPC ambient dialogue.** The Dialogue Writer must produce dialogue that the player never directly participates in — guard conversations, market chatter, tavern gossip. These must be contextual (reference game state, recent events) and positional (appropriate for where the NPCs are). This is a new output type: `ambient_dialogue(npc_pair, location, game_state_context, fragments[])`.
- **Gossip-transformed information.** When information propagates through the NPC network, it changes. The Dialogue Writer must produce gossip variants: the original fact ("the hero killed the bandit leader") and its transformations as it passes through NPCs with different personalities ("I heard there was a massacre in the forest" from a dramatic NPC, "some trouble on the road, I gather" from a reserved one).
- **Silence as authored option.** For each conversation where silence is meaningful, the Dialogue Writer must produce the NPC's specific reaction to silence — not a generic "..." but a contextual response that reflects the NPC's personality and the conversation's stakes. The silence reaction is as authored as any spoken option.
- **Information asymmetry options.** Options where the character knows something the player doesn't ("Mention what happened at the bridge") must be authored with two layers: the option text (vague to the player) and the dialogue content that unfolds when selected (revealing the backstory). The player learns their character by choosing to speak.
- **Tone drift corpus.** The Dialogue Writer must produce options across the entire game that exhibit gradual tone drift — early-game options are short, uncertain, reactive; late-game options are confident, precise, initiative-taking. This requires the Dialogue Writer to receive the character's current point on the tone drift trajectory and adjust option generation accordingly.

**Narrative Enemies Impact:**

Enemies are characters the player can talk to, learn about, and form relationships with through dialogue.

- **Enemy context seeding through NPC dialogue.** The Dialogue Writer must produce NPC conversations that seed enemy backstories: a villager mentioning the guard captain's family, a merchant describing the bandits' desperation. This information reframes combat encounters and must be coordinated with the Encounter Designer's enemy biographies.
- **Combat taunt authoring.** Taunts delivered during combat are dialogue events. The Dialogue Writer must produce taunt content parameterized by game state — the villain referencing the player's actual choices, an enemy acknowledging a prior defeat. These are context-sensitive dialogue fragments, not generic bark lines.
- **Surrender and negotiation dialogue.** When combat transitions to dialogue (an enemy surrenders, offers information, pleads), the Dialogue Writer must produce the conversation tree for that transition. The NPC's emotional state is set by the combat context — desperate, relieved, defiant — and the player's options reflect their character's perception of the enemy.
- **Post-combat narrative dialogue.** After encounters with discoverable motivations, NPCs in the world react. The Dialogue Writer must produce dialogue that reflects the moral weight of the player's combat choices — the village mourning the guard the player killed, or the refugees grateful the player found another way.

**Replay and Multiplayer Impact:**

Dialogue must support multi-perspective generation and cross-player context.

- **Per-character dialogue voice.** When the same conversation is experienced by different characters on replay, the player's available options change entirely — not just perception filtering, but different vocabulary, different knowledge, different relationship to the NPC. The Dialogue Writer must produce per-character option sets for key conversations, or the runtime generator must produce them from character voice profiles.
- **Inherited context dialogue.** In multiplayer/generational modes, NPCs reference prior-player events in dialogue. The Dialogue Writer must produce dialogue templates that incorporate inherited world state: "A traveler came through before — [generated from prior player's event log]." These are parameterized references, not fixed text.
- **Asymmetric information dialogue.** In parallel perspective multiplayer, two players in the same conversation have different information and different options. The Dialogue Writer must produce dialogue trees that support asymmetric knowledge — Player A's character knows about the conspiracy; Player B's character doesn't — within the same NPC conversation.

---

## 10. Level Designer → Encounter Designer

Spatial layout consumed by encounter placement.

**What crosses the boundary:**

- **Tilemap:** the complete map grid with tile IDs
- **Collision layer:** walkable/blocked per tile
- **Walkable regions:** connected walkable areas, chokepoints, open areas — pre-analyzed spatial features
- **Trigger zones:** already-placed trigger areas (so encounters don't overlap)
- **Transition points:** exits and entrances (enemy patrol shouldn't block required paths)
- **Spatial features:** cover positions, elevation changes, line-of-sight properties

**Dependency:** the Encounter Designer cannot begin work on a scene until the Level Designer has produced the map for that scene.

**Open questions:**

- Does the Encounter Designer receive the raw tilemap or a spatial analysis summary? (Probably both — raw for exact placement, summary for strategic decisions.)
- How are spatial features (chokepoints, cover, open areas) represented as structured data?

---

## 11. Level Designer → Script Director

Spatial layout consumed by dramatic choreography.

**What crosses the boundary:**

- **Walkable positions:** where entities can stand and move
- **Map dimensions:** total size, screen boundaries
- **Entity placement:** where characters and objects are at script start
- **Named positions:** key locations within the map (door, throne, altar, window) that scripts can reference by name rather than coordinate
- **Spatial constraints:** areas that would break visual framing if used for dramatic staging

**Dependency:** the Script Director cannot choreograph movement without knowing the space.

**Open questions:**

- How are named positions defined? By the Level Designer during map creation? By the Story Director in the scene spec?
- Does the Script Director have access to the visual layout (what's on screen at a given camera position) or only the logical layout (walkable tiles)?

---

## 12. Character Designer → Script Director

Character capabilities consumed by choreography.

**What crosses the boundary:**

- **Available animations per character:** idle, walk, run, attack variants, hurt, death, special (per character type)
- **Available expressions per character:** the set of portrait expressions that can be referenced in `dialogue()` commands
- **Animation timing:** frame counts and durations for each animation (so scripts can time waits correctly)
- **Character scale and dimensions:** for spatial choreography (how much space a character occupies)

**Dependency:** the Script Director can only choreograph what the character's sprite sheet supports.

**Open questions:**

- Is the animation vocabulary fixed across all characters (every character has the same states) or variable (some characters have unique animations)?
- How are animation names standardized? An enum? A capability list per character?

---

## 13. Character Designer → Dialogue Writer

Expression vocabulary consumed by dialogue authoring.

**What crosses the boundary:**

- **Expression set per character:** the list of valid expression tags (neutral, happy, sad, angry, surprised, fearful, disgusted, pensive, etc.) that the Dialogue Writer can attach to dialogue lines
- **Expression constraints:** if a character has a limited expression range (a stoic character might only have neutral, slight_frown, slight_smile), the Dialogue Writer must work within that range

**Open questions:**

- Is the expression vocabulary standardized (all characters share the same expression enum) or per-character?
- Can expressions blend or transition, or are they discrete states?

---

## 14. All Domain Agents → Continuity Tracker

Generated content validated against the accumulated fact store.

**What crosses the boundary:**

- **Domain agent → Tracker:** every generated artifact — dialogue trees, scripts, encounter definitions, NPC placements — for fact-checking
- **Tracker → Domain agent:** validation results — passed, or specific violations ("dialogue line references event X which hasn't occurred on this branch," "character Y is placed in location Z but has no path there," "flag A is checked but never set")

**Open questions:**

- What is the fact store schema? How are facts represented?
  - **Knowledge states:** what the player character knows, what each NPC knows
  - **Flags/variables:** boolean flags, integer counters, string states
  - **Relationship states:** per-NPC relationship values (trust, rapport, hostility)
  - **Item possession:** who has what
  - **Location reachability:** which characters can be where based on story progression
  - **Event history:** what has happened, in what order, on which branch
- How does the fact store handle branching? Multiple parallel fact states for different player paths?
- Is validation synchronous (blocking) or asynchronous (report after the fact)?

---

## 15. All Domain Agents → Consistency Checker

Cross-domain coherence validation — do the outputs from different agents fit together?

**What crosses the boundary:**

- **All domain outputs simultaneously** for cross-referencing:
  - Scripts reference walkable tile positions (Script Director output vs. Level Designer output)
  - Dialogue references locations that exist on the map (Dialogue Writer vs. Level Designer)
  - Encounters use behavior patterns the enemy's sprite can animate (Encounter Designer vs. Character Designer)
  - Script commands reference animations/expressions that exist (Script Director vs. Character Designer)
  - Trigger zones don't overlap with encounter spawn zones (Level Designer vs. Encounter Designer)
  - NPC placements in scripts match NPC placements on the map (Script Director vs. Level Designer)

**Open questions:**

- What are the complete validation rules? This is essentially a type-checking pass across all outputs. Each rule is: "output A from agent X references concept B which must exist in output C from agent Y."
- Is this a separate agent or a deterministic validation tool? (Likely deterministic — the rules are mechanical, not requiring LLM judgment.)
- How are cross-domain references expressed? By ID? By name? The ID/naming scheme must be consistent across all agents.

**Subjective Rendering Impact:**

The Continuity Tracker must validate not just narrative facts but **perceptual consistency** — does the player character's perception state justify what they can see, say, and interact with?

- **Knowledge-gated perception.** If "ignorance as rendered assumption" is active, the Continuity Tracker must verify that the assumed version of an unvisited location is consistent with what NPCs have said about it. If NPC A said "the forest is dangerous" but NPC B said "the forest is peaceful," the assumption renderer has contradictory inputs — the Tracker flags this as a continuity issue that the Story Director must resolve.
- **Perception-gated dialogue.** Every dialogue option tagged with a perception requirement must be validated: does the fact store confirm the player can have the required trait value at this point in the story? If an option requires `perception > 0.7` but the character's perception can never exceed 0.5 in the current playthrough, that option is dead code — the Tracker flags it.
- **Memory consistency.** If "memory as degraded rendering" replays a location from memory, the Tracker must verify that the memory version is consistent with what actually happened there — the degradation should lose details, not invent them. The memory version is a subset of the real version, never a contradiction.
- **Fact store expansion.** The fact store schema must now include per-character perception vectors (current values and progression history), per-character knowledge states (what has been seen, heard, told), and per-location visit history (for memory rendering). This is a significant expansion of Interface 14's data model.

**For the Consistency Checker (Interface 15):**

New cross-domain validation rules specific to subjective rendering:

- Perception overlay tile IDs must exist in the tileset (Level Designer overlays vs. Tier 3 rendered tilesets).
- Character perception-variant sprites must exist in the sprite sheet (Character Designer variants vs. Tier 3 rendered sprites).
- Dialogue perception-trait thresholds must reference traits that exist in the perception vector schema.
- Script perception commands (`clear_perception`, `set_perception`) must reference valid trait names and value ranges.
- Phantom enemies (paranoia-spawned) must not have collision or damage in their base state — mechanical validation that hallucinated enemies can't hurt the player unless the story intends it.

**Infinite Dialog Impact on Continuity Tracker (Interface 14):**

The fact store expands massively to support dialogue's information requirements.

- **NPC knowledge tracking.** The Tracker must maintain per-NPC knowledge states: what each NPC knows, believes, suspects, and is hiding. Every dialogue event that reveals information to an NPC updates their knowledge. The Tracker validates that NPCs never reference information they couldn't have learned — either through direct conversation, gossip propagation, or witnessing events.
- **Gossip consistency.** When NPC A tells the player something they heard from NPC B, the Tracker must verify: (1) NPC B actually knew this, (2) NPC B and NPC A have a gossip connection, (3) sufficient in-game time elapsed for the gossip to propagate. Gossip that violates the propagation model is a continuity error.
- **Lie tracking.** When an NPC lies, the Tracker must record both the lie and the truth, linked to the NPC's motivation for lying. If the player later catches the lie (through cross-referencing or high perception), the Tracker validates that the truth was discoverable through available information paths.
- **Conversation history persistence.** The Tracker must store per-NPC conversation history — not just flags but tone, key topics, promises made, rapport level. When the Dialogue Writer generates a follow-up conversation with a recurring NPC, the Tracker validates that callbacks reference actual prior exchanges.
- **Information asymmetry validation.** The Tracker must verify that dialogue options tagged with character knowledge prerequisites are gated correctly — the player character can't reference information they haven't acquired on this playthrough path.

**Infinite Dialog Impact on Consistency Checker (Interface 15):**

- Conversation zones reference valid NPC pairs and map positions (Dialogue Writer × Level Designer).
- NPC conversation routes align with NPC patrol paths (Dialogue Writer × Level Designer).
- Conversation-as-combat stance models reference valid stance types from the engine schema.
- Ambient dialogue fragments reference valid game state conditions.
- Gossip propagation references valid NPC network connections defined in the spec.

**Animated Narrator Impact:**

The Continuity Tracker must validate narrator consistency across the game.

- **Narrator self-revision tracking.** When the narrator states something as fact and later revises it, the Tracker must record both the original statement and the revision. Revisions must be consistent with actual game events — the narrator can be wrong about predictions, but its revisions must reference events that actually occurred.
- **Foreshadowing fulfillment validation.** The Tracker must verify that every foreshadowing seed planted by the narrator has a corresponding payoff, and that payoffs reference seeds that were actually delivered in the player's path through the game. Unfulfilled foreshadowing on a reachable path is a continuity error.
- **Dramatic irony consistency.** When the narrator tells the player something the character doesn't know, the Tracker must verify that subsequent dialogue options remain constrained by the character's ignorance — the player's knowledge from narration must not leak into character behavior.

**Narrative Enemies Impact:**

The Continuity Tracker must validate enemy narrative consistency.

- **Enemy biography accessibility.** The Tracker must verify that discoverable enemy motivations are actually discoverable — the information exists in the world (NPC dialogue, documents, environmental clues) on at least one reachable path before or during the encounter.
- **Recurring enemy history consistency.** When a recurring enemy references prior encounters, the Tracker must verify the references match what actually happened — the enemy's taunts must reflect the player's actual combat choices, not generic escalation.
- **Faction dynamics consistency.** The Tracker must verify that enemy group behavior follows the defined loyalty/command structures — troops loyal to a killed commander must behave according to the loyalty rules, not continue fighting as if nothing changed.
- **Combat-to-dialogue state continuity.** When combat transitions to dialogue (surrender, negotiation), the Tracker must verify that the NPC's emotional state in dialogue matches the combat context — a defeated enemy at low HP enters dialogue as desperate, not confident.

**Environmental Memory Impact:**

The Continuity Tracker must validate that environmental memory reflects actual events.

- **Location memory accuracy.** The Tracker must verify that a location's accumulated memory corresponds to events that actually occurred there — the throne room remembers the betrayal only if the betrayal happened in the throne room, on the player's path.
- **Temporal echo content validation.** When temporal echoes replay past events, the Tracker must verify the echoes show what actually happened, not invented history. Echo content must be a subset of the event log for that location.
- **Spatial contagion consistency.** The Tracker must verify that environmental mood contagion follows the defined rules — darkness spreading from a murder site to adjacent locations must follow the contagion model, not skip locations or exceed defined spread rates.
- **Environmental agency rule validation.** When a location acts on its accumulated memory (paths rearranging, cover changing), the Tracker must verify the behavior follows the defined agency rules and that the accumulated event count exceeds the agency threshold.

---

## 16. Domain Agents → Tier 3 Asset Renderers

Structured visual descriptions translated into actual pixels.

**What crosses the boundary:**

- **Character Designer → image gen:**
  - Per-sprite-state visual description (idle facing south: "elderly man, gray robe, staff in right hand, slight hunch, warm expression")
  - Portrait expression descriptions ("happy: eyes crinkled, warm smile, slight head tilt")
  - Palette constraints (primary colors, accent colors, forbidden colors)
  - Style reference (if applicable — art style parameters)
  - Output format requirements (grid dimensions, frame count, pixel dimensions per frame)

- **Level Designer → image gen:**
  - Per-tile-type visual description ("ground_grass_autumn: warm orange-brown grass, scattered fallen leaves, soft texture")
  - Tileset organization (which tiles connect to which, edge rules for seamless tiling)
  - Palette constraints from Atmosphere Director
  - Output format requirements (tile size in pixels, tileset grid layout)

- **Encounter Designer → image gen:**
  - Enemy visual descriptions per sprite state (same format as character sprites)
  - Effect descriptions (attack effects, projectile appearance)

**Open questions:**

- What is the prompt format for image generation? This is the bridge between LLM-generated text descriptions and diffusion model input. Is it natural language? Structured prompts with specific parameters?
- How is visual consistency enforced across multiple generation calls? (Same character across many sprite states, same tileset across many tile types.) Style transfer? LoRA? Seed locking?
- How are generation failures handled? (Wrong dimensions, inconsistent style, artifacts.) Automated retry? Human review?
- What is the standard character template? (N animations × M frames × D directions — exact numbers needed.)

**Subjective Rendering Impact:**

The asset generation pipeline multiplies in scope. Every perception-variant visual requires actual pixels.

- **Perception-variant tilesets.** The Level Designer's perception overlays reference tile variants that don't exist in the base tileset. For each perception dimension active in a scene, the Tier 3 renderers must produce variant tiles: the market square's warm ground tiles at agoraphobia > 0.7 become cold, cracked, expansive tiles. The prompt to the image generator must include both the base description and the perception modification: "Same ground tile, but colder palette, more worn texture, slightly larger scale impression."
- **Character perception variants.** Body language variants (idle_facing_player, idle_turned_away, idle_whispering for social anxiety), scale variants (rival at 1.3×), enhanced rendering (loved character with warmer palette) — each requires sprite generation. The number of render calls per character scales with the number of active perception dimensions that affect character rendering.
- **Naive/cynical NPC variants.** Under naivety, a thug looks like an ordinary person. Under cynicism, a friendly merchant looks greedy. These are distinct sprite variants for the same NPC, generated from different prompts. The Character Designer must provide both prompt variants; the Tier 3 renderer must produce visually related but tonally different sprites.
- **Visual consistency across variants.** The same character rendered at trust 0.2 (shadowed, turned away) and trust 0.8 (lit, facing player) must be recognizably the same character. This is a hard constraint on the image generation pipeline — style consistency across perception-variant prompts. May require reference images, LoRA fine-tuning per character, or post-processing to enforce consistency.
- **Asset budget explosion.** Without constraints, subjective rendering can multiply the asset count by the number of perception dimensions × threshold steps per dimension. The system must enforce a budget: maximum N perception-variant tilesets per map, maximum M variant sprite states per character. The Story Director's "active perception dimensions per scene" annotation is the primary budget control.

---

## 17. All Outputs → Engine Schema Validator

The master contract. Every generated artifact must validate here before entering the build.

**What crosses the boundary:**

- **Tilemaps:** valid tile IDs, correct dimensions, at least one player spawn, no unreachable trigger zones, all transition points connect to existing maps
- **Entity definitions:** required components present (sprite reference, position, collision box), valid entity type, referenced sprite sheet exists
- **Behavior patterns:** only use engine-defined behavior primitives, valid state transitions, referenced animations exist
- **Scripts:** only use engine-defined script commands, reference valid entity IDs, reference walkable positions, valid timing values
- **Dialogue trees:** valid structure (no orphan nodes, no unreachable branches), valid expression tags, valid condition references (flags exist), valid consequence actions
- **Perception rules:** valid trait references, valid subsystem targets, valid threshold ranges, composable (no contradictory rules on the same subsystem)
- **Asset manifests:** all referenced assets exist, correct formats, correct dimensions

**Open questions:**

- Is the schema defined in Rust types (serde-deserializable structs) that are the canonical source of truth?
- How are schema changes versioned and propagated to agents?
- What is the error reporting format? Agents need actionable error messages to fix violations.

**Subjective Rendering Impact:**

The schema validator gains an entire new validation domain — perception rules and their cross-references.

- **Perception rule validation.** Every perception modifier rule must reference a valid trait name (from the perception vector schema), a valid subsystem target (tile_variant, npc_behavior, dialogue_filter, palette, scale, visibility, collision), and a valid threshold range (0.0–1.0). Rules that reference non-existent traits or target subsystems the engine doesn't support must be rejected.
- **Composability validation.** Two perception rules targeting the same subsystem with contradictory effects must be flagged. If agoraphobia > 0.7 sets palette to "cold_blue" and depression > 0.5 sets palette to "desaturated_gray," the validator must either flag the conflict or verify that a resolution rule exists (blending, priority, layering).
- **Perception overlay asset validation.** Every tile ID referenced in a perception overlay must exist in a tileset. Every sprite variant referenced by a perception rule must exist in a sprite sheet. This is the same validation as base assets but applied to the perception-variant asset layer — and it catches the case where the Level Designer authored an overlay referencing tiles the Tier 3 renderers never produced.
- **Phantom entity validation.** Entities marked as `phantom_enemy` (paranoia hallucinations) must have no collision and no damage in their default state. If they can transition to real enemies, the transition trigger must be defined and validated.
- **Perception budget validation.** If the system enforces an asset budget (max N overlays per map, max M variants per character), the validator must check that generated content stays within budget.

**Infinite Dialog Impact:**

The schema validator gains dialogue-specific validation rules.

- **Dialogue tree structural validation.** No orphan nodes, no unreachable branches, all condition checks reference valid flags/variables, all consequence actions are valid engine operations. This exists in the base spec but scales up: infinite dialog means much larger trees with more conditional branches.
- **Conversation-as-combat validation.** Stance models must have valid initial states, defined transitions, and at least one reachable success and one reachable failure state. No deadlocked conversations (NPC stance can't transition and player has no options that change it).
- **Perception-tagged option validation.** Every dialogue option with perception trait requirements must reference valid traits and value ranges. The superset of options per node must include at least one option reachable at every possible perception state — no conversation node where all options are gated above a threshold the player might not meet.
- **Ambient dialogue validation.** NPC-to-NPC conversation fragments must reference valid NPC IDs, valid location IDs, valid game state conditions, and the NPC pair must have a defined relationship in the NPC network.
- **Momentum state validation.** Conversation momentum models must have valid tone transitions — no state from which all options lead to the same tone (no momentum), no state from which no options are available (conversation deadlock).

---

## 18. Engine Schema → All Agents (Capability Advertisement)

The engine tells agents what's possible. This flows in the opposite direction from all other interfaces — the engine defines, agents consume.

**What crosses the boundary:**

- **Tile types:** enumeration with properties (walkable, blocking, interactive, animated, destructible)
- **Entity types:** player, NPC, enemy, object — with required and optional components per type
- **Behavior primitives:** the complete vocabulary of enemy/NPC behaviors (approach_player, patrol_path, charge_pause, fire_projectile, flee_threshold, idle_wander, etc.)
- **Script commands:** the complete set with parameter signatures (`move_entity(entity_id, position, speed)`, `fade(color, duration)`, etc.)
- **Dialogue structures:** node types (text, choice, condition, consequence), valid fields per node type
- **Effect types:** particle systems, screen effects, transitions — enumerated with parameters
- **Perception modifier types:** the subsystems that can be modulated (tile_variant, npc_behavior, dialogue_filter, palette, scale, visibility, collision) and the rule format

**Open questions:**

- How do agents receive and internalize this schema? Options:
  - Part of the agent's system prompt (static, simple, but long)
  - A reference document the agent can query (dynamic, but adds latency)
  - A formal grammar/DSL that the agent generates code in (most precise, highest learning curve)
- How are new capabilities added? If the engine adds a new script command, how does every agent learn about it?
- Should the schema be human-readable (for the Story Director user review) and machine-readable (for agents and validators) simultaneously?

**Subjective Rendering Impact:**

The engine schema must advertise a new capability category — the perception modifier system — with enough specificity that every agent can author rules for it.

- **Perception vector schema.** The schema must define the complete list of supported perception traits, their value ranges, and their update semantics (event-driven, continuous, story-directed). This is the shared vocabulary that the Story Director, Level Designer, Character Designer, Encounter Designer, Dialogue Writer, and Script Director all reference when authoring perception-dependent content.
- **Modifier subsystem enumeration.** The schema must enumerate every engine subsystem that perception can modulate: tile_variant, tile_scale, tile_insertion (for exhaustion stretching), npc_behavior_overlay, npc_scale, npc_palette, dialogue_option_filter, dialogue_tone_modifier, narrator_tone, palette_global, palette_per_entity, visibility_layer, collision_modifier, timing_scale, particle_effects, ambient_sound. Each with its parameter signature — what values can be set, what types they accept.
- **Modifier rule format.** The schema must define the syntax for perception rules that agents author. This is the DSL or data format that appears in the game spec, gets validated by the schema validator, and gets evaluated by the perception engine. All agents must agree on this format. It's the most novel part of the engine schema — nothing like it exists in standard game engine schemas.
- **Composability rules.** The schema must define how modifiers on the same subsystem interact: priority ordering, blending modes, override vs. additive semantics. Agents need to know this when authoring rules to avoid conflicts.

**Infinite Dialog Impact:**

The engine schema must advertise an expanded dialogue capability set.

- **Dialogue node types.** Beyond text/choice/condition/consequence, the schema must define: `stance_node` (conversation-as-combat with NPC stance tracking), `ambient_node` (NPC-to-NPC, player not participating), `silence_option` (explicit silence as a choice with NPC-specific reaction), `momentum_node` (options constrained by prior tone choices within the conversation).
- **Conversation state model.** The schema must define conversation-level state that persists within a conversation: current tone, NPC stance, accumulated credibility/patience (for conversation-as-combat), prior option choices (for momentum). This is distinct from game-level state (flags, variables).
- **NPC knowledge model.** The schema must define how NPC knowledge is represented, updated, and queried: knowledge entries (fact ID, confidence, source, timestamp), knowledge queries (can NPC X know fact Y at this point?), knowledge update events (NPC X learns fact Y through gossip/observation/dialogue).
- **Eavesdropping mechanics.** The schema must define conversation zones as engine entities: position, radius, NPC participants, trigger conditions, audibility rules (full text within inner radius, fragments within outer radius).
- **Dialogue-combat integration.** The schema must define how dialogue and combat systems hand control to each other: `enter_dialogue(conversation_id, from_combat=true)`, `exit_dialogue(outcome, resume_combat=bool)`, surrender triggers, mid-combat taunt events.

**Animated Narrator Impact:**

The engine schema must advertise the narrator as a first-class engine subsystem with its own command vocabulary.

- **Narrator command set.** The schema must define narrator-specific commands: `narrator_text(text, tone, stance)`, `narrator_teach(mechanic, hint_level)`, `narrator_foreshadow(seed_id)`, `narrator_payoff(seed_id)`, `narrator_revise(prior_statement_id, new_text)`. These are distinct from dialogue commands — narrator text appears in a different channel and is addressed to the player, not the character.
- **Narrator state model.** The schema must define the narrator's persistent state: personality parameters, current relationship to the player (continuous values for protectiveness, amusement, wariness, investment), statement history (for self-revision), and per-scene stance (inside/outside the character's perception).
- **Narrator-perception integration.** The schema must define how the narrator consumes the perception vector: which perception dimensions trigger narrator commentary, how the narrator's tone adjusts based on the character's emotional state, and how the agree/disagree axis is represented as an engine-evaluable rule.
- **Player behavior pattern types.** The schema must enumerate recognizable play style patterns (cautious, aggressive, thorough, dismissive, exploratory) that the narrator tracks and responds to. These are engine-level classifications derived from player action history.

**Narrative Enemies Impact:**

The engine schema must advertise enemy narrative capabilities alongside mechanical ones.

- **Enemy biography model.** The schema must define how enemy biographical data is structured and queryable: motivation, faction, relationships to other enemies, discoverable information references. This data is consumed by the dialogue generator (for taunts), the narrator (for commentary), and the combat system (for behavior modifiers).
- **Combat-dialogue transition protocol.** The schema must define the handoff between combat and dialogue systems with full state preservation: `surrender_trigger(hp_threshold, morale_threshold, relationship_condition)`, `enter_negotiation(enemy_id, emotional_state)`, `resume_combat(from_dialogue_failure)`. Both directions must be defined.
- **Enemy relationship types.** The schema must enumerate relationship types between enemies (loyalty, friendship, command, rivalry, fear) and define how each affects group behavior: morale propagation rules, death-response behaviors, command succession.
- **Taunt event model.** The schema must define taunts as a dialogue event type within combat: `taunt(enemy_id, trigger_condition, content_parameters, interruptible)`. Content parameters reference game state for contextual generation.

**Replay and Multiplayer Impact:**

The engine schema must advertise multiplayer and replay capabilities as first-class features.

- **Player state serialization schema.** The schema must define the canonical format for exporting a player's state for consumption by another player's world: event log summary, world state delta, environmental memory, character reputation. This is the contract between the multiplayer state bridge and the generation pipeline.
- **Ghost trace entity type.** The schema must define ghost traces as engine entities: `ghost_trace(type, source_player_event, position, visual, trigger_condition)`. These are diegetic world elements that appear based on inherited prior-player state.
- **Perspective mode flags.** The schema must define how the engine handles parallel perspective multiplayer: simultaneous generation from the same base state with different perception vectors, dialogue routing for shared conversations with asymmetric options, and event synchronization between perspectives.
- **Meta-narrative awareness flags.** The schema must define how the engine signals replay to the narrator and dialogue systems: `is_replay`, `prior_playthrough_summary`, `character_perspective_id`. These enable NPCs and the narrator to behave differently on subsequent playthroughs.

**Environmental Memory Impact:**

The engine schema must advertise environmental memory as a subsystem with defined data types and operations.

- **Location memory model.** The schema must define the per-location memory structure: event list, emotional layers (with coexistence, not overwrite semantics), personality values, agency threshold, contagion susceptibility. Each field must have typed access patterns for consumers (perception engine, narrator, level regeneration).
- **Environmental agency behavior primitives.** The schema must enumerate behaviors available to locations with agency: `rearrange_paths(region, intensity)`, `modify_cover(add|remove, positions)`, `adjust_traversal_cost(region, multiplier)`, `reveal_shortcut(path_id)`. These parallel the NPC behavior primitives but operate on spatial geometry.
- **Temporal echo entity type.** The schema must define temporal echoes as engine entities: `temporal_echo(event_reference, position, visual_style, perception_threshold, information_content)`. These are perception-gated overlays that replay past events as ghostly figures.
- **Spatial contagion rules.** The schema must define how environmental mood spreads: `contagion(source_location, target_locations, mood_type, spread_rate, decay_rate)`. The engine evaluates these between scene transitions.

---

## 19. Game State → Perception Engine (Runtime)

The runtime heart of subjective rendering. Every frame, the engine evaluates the perception stack to determine what the player sees.

**What crosses the boundary:**

- **Input:** the current perception vector (all trait values as continuous floats: agoraphobia 0.7, stealth_skill 0.4, hunger 0.2, trust_for_npc_12 0.8, ...) + the base game state (map, entities, NPC states)
- **Output:** the perceived state — modified tilemap (variant selection or overlay), modified NPC behaviors (gaze, body language, movement), modified visibility (what's shown/hidden), modified palette, modified scale/geometry, modified ambient elements

**The perception modifier stack:**

Each trait maps to one or more modifiers. Each modifier targets a specific engine subsystem. Modifiers compose by default (independent subsystems) with explicit interaction rules where traits conflict or amplify.

**Open questions — this is one of the three hardest problems:**

- What is the concrete data format for a perception modifier rule? Candidates:
  - Threshold rules: `if anxiety > 0.7 then npc_behavior.idle = glance_at_player`
  - Continuous interpolation: `tile_scale = lerp(base_scale, 2.0 * base_scale, agoraphobia)`
  - Variant selection: `at fear > 0.5, use tilemap_variant "market_square_threatening"`
  - Behavior overlays: `add npc_behavior_layer(ambient_whisper) when social_anxiety > 0.6`
- How are multiple modifiers on the same subsystem resolved? Priority? Blending? Strongest-wins?
- How expensive is perception evaluation per frame? Must be O(active_modifiers), not O(all_possible_modifiers).
- Are perception modifiers pre-generated (LLM authors rules at build time, engine evaluates at runtime) or runtime-generated? The doc says pre-generated. This constrains the expressiveness but guarantees performance.

**Subjective Rendering Impact:**

This is the interface where every subjective rendering concept converges at runtime. The full taxonomy of perception modifiers must be evaluable here.

- **Psychological conditions** modulate: tile geometry/scale (agoraphobia), NPC behavior (social anxiety → gaze/whisper), dialogue option availability (narcissism → egotistical-only), palette/speed (depression), fog of war + phantom entities (paranoia), UI reliability + distraction events (ADHD), map warping + flashback triggers (PTSD), NPC rendering inversion (body dysmorphia). Each is a different modifier type targeting a different subsystem, but all evaluated from the same perception vector.
- **Skills** modulate: enemy telegraph timing (combat mastery), cover tile insertion (stealth), hidden feature visibility (perception/awareness), object annotation overlays (crafting/trade), NPC orientation/body language (charisma), visibility layer addition (magical attunement). These are progressive — they make the world reveal more as skill increases, not distort more.
- **Physical states** modulate: tile insertion/path stretching (exhaustion), object saturation (hunger), viewport narrowing + detail reduction (injury/pain), color distortion + entity duplication (sickness). These are temporary and fluctuate during gameplay, unlike psychological conditions which trend over the game arc.
- **Knowledge/cognition** modulates: tile variant selection based on knowledge state, not perception float (ignorance renders assumptions; knowledge reveals reality). This is a different modifier type — it's not threshold-based on a continuous value but binary on a fact-store check. The perception engine must support both continuous-float modifiers and discrete-knowledge modifiers.
- **Relationships** modulate: per-entity rendering (not global) — one specific NPC renders differently based on the relationship value. This requires entity-scoped modifiers, not scene-scoped. The modifier references a specific entity ID and a relationship value.
- **Moral/worldview** modulates: location-specific palette (guilt darkens event sites — requires event log cross-reference), global detail level (innocence simplifies the world), NPC motivation transparency (cynicism exposes selfishness), environmental degradation (corruption). These are slow-moving and cumulative, changing over the arc of the game.
- **Modifier interaction examples that the engine must handle:**
  - Hunger saturates food objects + depression desaturates the palette = food objects are relatively more visible even in a desaturated world (additive on the same subsystem with different scopes).
  - Paranoia spawns phantom enemies + high combat skill makes enemies telegraph obviously = phantom enemies also telegraph, which is wrong (they should be ambiguous). Needs an interaction rule: phantom entities are exempt from combat skill perception modifiers.
  - Agoraphobia expands open spaces + stealth adds cover to spaces = the expanded space has cover. Modifiers compose cleanly (different subsystems) but the Level Designer's cover insertion points must scale with spatial expansion.

**Animated Narrator Impact:**

The perception engine feeds the narrator and the narrator contextualizes the perception engine — a bidirectional relationship.

- **Narrator tone as perception output.** The perception engine must evaluate narrator tone modifiers alongside visual modifiers. The narrator's emotional register shifts with the perception vector: clinical during calm, breathless during crisis, muted during depression. Narrator tone is a perception subsystem target like palette or tile_variant.
- **Agree/disagree axis evaluation.** The perception engine must evaluate the narrator's per-scene stance (confirming or undercutting the distortion) and pass this to the narrator system. When the narrator disagrees with the rendering, it describes objective reality while the screen shows subjective perception — the engine must coordinate these two outputs.
- **Teaching trigger evaluation.** The perception engine must detect when a perception modifier creates a teaching opportunity (stealth cover appearing for the first time, a hidden passage revealed by perception skill) and fire a narrator teaching event with the mechanic and the player's behavior history as context.

**Narrative Enemies Impact:**

The player's combat history becomes a perception input that modulates how combat renders.

- **Violence history as perception dimension.** The player's kill/spare record and combat frequency feed into the perception vector as continuous values (desensitization, trauma, reputation). The perception engine must evaluate these against combat rendering modifiers: desensitization flattens combat (less intensity, less detail), trauma distorts it (more disturbing, harder to read), and reputation changes enemy pre-combat behavior.
- **Enemy perception through character lens.** The perception engine must apply character perception modifiers to enemy rendering: a paranoid character sees neutral NPCs as threats, a naive character sees threats as friendly, a perceptive character reads enemy telegraphs earlier. The same enemy presents differently through different perception stacks.
- **Moral weight rendering.** When the player has discovered an enemy's backstory, the perception engine must apply a "moral knowledge" modifier that changes how the encounter renders — the guard protecting his family looks different when you know about the family.

**Replay and Multiplayer Impact:**

The perception engine must support multiple simultaneous or sequential perception stacks.

- **Multi-character perception isolation.** On replay with a different character, the entire perception stack changes. The perception engine must cleanly swap the full modifier set — not just adjust values but load a different character's complete perception profile. The same base game state produces a fundamentally different rendered experience.
- **Inherited perception context.** In generational play, the prior player's perception trajectory influences the current world. The perception engine must support inherited environmental state modifiers — locations that carry emotional residue from a prior player's experience, evaluated as environmental memory modifiers rather than character perception modifiers.

**Environmental Memory Impact:**

The perception engine must evaluate location memory alongside character perception.

- **Environmental memory as modifier source.** Location memory generates perception modifiers at runtime — a location where violence occurred applies a darkening modifier, a nurtured location applies a warming modifier. These are location-scoped modifiers distinct from character-scoped ones, and they compose with character perception (a depressed character in a dark location experiences both).
- **Temporal echo gating.** The perception engine must evaluate temporal echo visibility based on character perception traits (perception skill, magical attunement) crossed with location memory intensity. High perception + rich location history = visible echoes. Low perception = empty room. This is a cross-reference between character state and environmental state.
- **Emotional archaeology rendering.** Multiple emotional layers at a location must be rendered simultaneously, not as overrides. The perception engine must compose coexisting emotional modifiers — warmth and pain and healing — into a unified but complex rendering that reflects the location's accumulated history.
- **Spatial contagion modifier propagation.** The perception engine must apply environmental mood modifiers not just to the source location but to neighboring locations at reduced intensity, following the contagion rules. This requires the engine to query neighboring location memory states during rendering.

---

## 20. Game State → Narrator (Runtime or Pre-generated)

The narrator's text generation pipeline. Responds to current game state, player behavior history, and narrative context.

**What crosses the boundary:**

- **Input:** current location, recent events, character emotional state, player behavior history (play style patterns), narrator personality profile, narrator relationship state (how the narrator feels about the player), foreshadowing seeds (information to plant or pay off)
- **Output:** narrator text — scene descriptions, commentary, foreshadowing, emotional amplification, teaching hints, self-revisions

**The fundamental architectural question — one of the three hardest problems:**

The doc states "the LLM does not run during gameplay." But the narrator's responsiveness to player behavior (player-awareness, relationship development, self-revision, contextualizing subjective rendering in real time) seems to require either:

- **Runtime LLM calls** during gameplay (responsive but latency/cost concerns)
- **Extremely large pre-generation pass** covering all reachable game states × perception states × player behavior patterns (combinatorial explosion)
- **Hybrid:** pre-generated narrator text for planned story beats, runtime generation for reactive commentary (player-awareness, contextualizing perception). The engine caches and batches runtime calls.
- **Template + interpolation:** pre-generated templates with runtime variable substitution ("The {location_adjective} square {perception_verb} before you" where variables are selected from pre-generated pools based on game state)

**Open questions:**

- Which architecture? This decision cascades to every system that consumes or produces narrator text.
- If runtime: what's the latency budget? Can narrator text be generated asynchronously (appears a moment after the player enters a space)?
- If pre-generated: how is the combinatorial space managed? Scene × perception state × player behavior pattern × narrator relationship state is enormous.
- How does the narrator track its own prior statements for self-revision? A narrator memory store?

**Subjective Rendering Impact:**

The narrator is the primary system for **contextualizing** subjective rendering — explaining, confirming, or contradicting what the perception engine shows.

- **Perception-variant narration.** The narrator's text must reflect the current perception state. Entering the market square at agoraphobia 0.8: "The square opens before you like a wound. Every step forward feels like a step into nothing." At agoraphobia 0.2: "The market square is busy today. Manageable." The narrator needs the current perception vector as input to select or generate the appropriate text.
- **Agree/disagree axis.** The narrator can confirm the distortion (immersion) or undercut it (critical distance). This is an authored choice per scene, not automatic: the Story Director specifies whether the narrator is "inside" or "outside" the character's perception for each major beat. The narrator's input must include a per-scene "narrator stance on perception" flag.
- **Perception change narration.** When a perception trait changes (the mentor scene reduces agoraphobia by 0.2), the narrator should acknowledge the shift: "The square seems... smaller today. Less empty." This requires the narrator to detect perception delta events and generate transitional text. The input to the narrator expands to include "perception changes since last narration."
- **Therapeutic arc commentary.** Over the game, the narrator tracks the trajectory of perception traits and comments on the character's growth or decline. "You walked through the square today without hesitating. You didn't even notice." This requires narrator access to the perception history, not just the current value.
- **Contrast narration.** When a script clears perception (`clear_perception`), the narrator has the opportunity for the most impactful text in the game: describing the world as it "actually is" for the first time. This must be specifically authored or pre-generated for contrast moments — it's too important for generic template substitution.

**Infinite Dialog Impact:**

The narrator and dialogue system interleave — the narrator comments on conversations, contradicts NPCs, and provides the player with interpretive context.

- **Post-dialogue narrator commentary.** After significant conversations, the narrator can comment: "He was lying. You could see it, even if your character couldn't." Or: "That went worse than it needed to." The narrator's input must include recent dialogue outcomes and the gap between what happened and what could have happened (the pruned options the player never saw).
- **Narrator contradicting NPC statements.** The three-way tension (visual/dialogue/narrator) applies directly: an NPC says one thing, the narrator says another. "She says the road is safe. The way her hand tightens on the door frame says otherwise." The narrator needs NPC deception data from the fact store — when an NPC lies, the narrator can expose or hint at the lie, depending on the narrator's current reliability and the Story Director's intent.
- **Narrator referencing past conversations.** The narrator tracks NPC relationship memory and can callback to prior exchanges: "The last time you spoke to the blacksmith, he wouldn't meet your eyes. Today he does." This requires the narrator to have access to conversation history, not just game flags.

**Animated Narrator Impact:**

This is the narrator's core interface — where all narrator capabilities converge at runtime.

- **Narrator relationship state as persistent input.** The narrator's relationship with the player (protectiveness, amusement, wariness, investment) evolves based on play style and must persist across the full game. The narrator input must include the full relationship trajectory, not just current values, so the narrator can reference its own development ("I used to worry about you. Now I just watch.").
- **Self-revision memory.** The narrator must track its own prior statements and be able to reference, contradict, or revise them. The input must include a narrator statement log keyed to game events, enabling self-revision triggered by new information: "It was not, as it turns out, certain."
- **Information pacing control.** The narrator must have access to a foreshadowing schedule — seeds to plant, payoffs to deliver, information to withhold. The narrator's generation pipeline must track what has been hinted, what has been revealed, and what remains hidden, producing text that advances the information pacing plan.
- **Emotional amplification timing.** The narrator must receive dramatic context — is this a pre-battle moment, a post-loss moment, a quiet discovery? — and generate text that functions as emotional amplification. The input must include dramatic beat type and intensity level so the narrator matches the moment's weight.
- **Player behavior commentary.** The narrator must receive player behavior history (cautious, aggressive, thorough, dismissive) and generate text that reflects the accumulated relationship: a protective narrator for a cautious player, a darkly amused narrator for a reckless one. This requires the play style classification as continuous input, not a one-time snapshot.

**Replay and Multiplayer Impact:**

The narrator is one of the most replay-sensitive systems — its voice defines the entire tone of a playthrough.

- **Per-character narrator voice.** On replay with a different character, the narrator's personality changes entirely. The same events narrated for the naive hero have warmth and gentle foreshadowing; narrated for the villain they're cold, analytical, sympathetic in unexpected places. The narrator's input must include the viewpoint character's identity, and the narrator personality profile must have per-character variants.
- **Meta-narrative awareness.** On replay, the narrator can acknowledge repetition — hinting that this has happened before, subverting first-playthrough expectations, commenting on the player's prior knowledge. The narrator input must include a replay flag and optionally a summary of prior playthroughs, so the narrator can layer meta-narrative without breaking diegesis.
- **Inherited history narration.** In generational play, the narrator references prior-player events with the weight of actual history. The narrator input must include inherited world state summaries so it can say "They say the old hero showed mercy at the bridge" with specificity generated from Player A's actual choices.

**Environmental Memory Impact:**

The narrator is the primary voice for environmental memory — describing what locations remember and how they've changed.

- **Place description from memory.** The narrator must receive per-location emotional history and generate place descriptions that reflect accumulated layers — not just current mood but the trajectory: "This square has known joy and betrayal in equal measure." The narrator input must include the location's full emotional archaeology, not just its current state.
- **Environmental agency narration.** When a location acts on its memory (the forest rearranging paths, the garden helping the player), the narrator must describe the agency in diegetically appropriate terms — the forest "seems to close behind you," the garden "offers up a shortcut that wasn't there before." The narrator must receive environmental agency events as they occur.
- **Spatial contagion commentary.** When environmental mood spreads to neighboring locations, the narrator can comment on the spreading: "The darkness from the alley has seeped into the street. You can feel it at the corner." The narrator must receive contagion state updates.
- **World-as-unreliable-historian narration.** The narrator can tell a different story about a location than the environment shows — the three-way tension applied to place memory. A church "remembers" desecration; the narrator might offer the soldiers' perspective. The narrator must receive both the location's subjective memory and the objective event log.

---

## 21. Game State → Dialogue Generator (Runtime or Pre-generated)

Dialogue option generation pipeline. Produces contextual options per conversation based on character state, NPC state, game state, and conversation history.

**What crosses the boundary:**

- **Input:** NPC profile (personality, knowledge, motivation, deception intent), player character traits (perception vector — determines which options are available), game flags and variables, conversation history (for momentum/commitment), NPC stance (defensive/aggressive/open), relationship history with this NPC
- **Output:** a set of dialogue options (typically 2–4) with: display text, internal effect (flag changes, relationship changes, stance shifts), expression tag, NPC response (text + expression + stance change + potential consequences)

**Same architectural question as narrator:**

Infinite dialog implies contextual generation that can't be fully pre-authored. Options:

- **Runtime LLM:** most expressive, highest cost/latency
- **Pre-generated dialogue trees with contextual pruning:** the Dialogue Writer generates large trees at build time; runtime selects branches based on game state. Less "infinite" but performant.
- **Hybrid:** pre-generated trees for main story beats, runtime generation for side conversations, NPC ambient dialogue, and eavesdropping content
- **Parameterized generation:** pre-generated option templates with runtime variable substitution based on character traits and game state

**Open questions:**

- Which architecture? This is coupled to the narrator decision — likely the same approach for both.
- How are dialogue options filtered by perception traits at runtime? Is filtering a simple threshold check on pre-generated options, or does the LLM generate different options from scratch?
- How is conversational momentum tracked? A conversation state object that accumulates tone/stance/commitment?
- How is NPC-to-NPC ambient dialogue (eavesdropping) generated? This is lower-stakes than player dialogue and could use a simpler generation approach.

**Subjective Rendering Impact:**

Dialogue generation is already identified as a perception-modulated subsystem (Interface 9), but the runtime implications are specific to this interface.

- **Runtime perception filtering.** If dialogue options are pre-generated with perception tags (the superset approach from Interface 9), the dialogue generator at runtime must evaluate the current perception vector against each option's requirements and prune. This is a lightweight operation — threshold checks on floats — and doesn't require runtime LLM calls.
- **If dialogue options are runtime-generated,** the perception vector is a critical input to the LLM prompt: "Generate 3 dialogue options for a character with charisma 0.3, courage 0.6, social anxiety 0.7 speaking to a merchant they distrust." The LLM must reliably produce options that reflect the constraints. This is where the pre-gen vs. runtime decision matters most — pre-generated with pruning is predictable; runtime generation is more expressive but less controllable.
- **Conversational misunderstanding rendering.** When the misunderstanding mechanic is active (low social skill, autism), the dialogue generator must output two text layers per option: what the character means and what the NPC hears. The UI must render both — likely the player sees their option text, chooses it, and then sees the NPC react to a different interpretation. This is a UI rendering concern that flows from the dialogue generator's output format.
- **Perception state changes during conversation.** A conversation that makes the character anxious should shift the perception vector mid-dialogue, affecting subsequent option generation. If pre-generated, this means dialogue trees must branch on perception state, not just narrative flags. If runtime, the updated perception vector is simply part of the next generation prompt.

**Infinite Dialog Impact:**

This is the core runtime interface for infinite dialog. Every dialogue concept converges here.

- **The pre-gen vs. runtime decision is sharpest here.** Truly infinite dialog (contextual option generation per conversation, per character state, per game state) requires runtime LLM calls. Pre-generated trees with pruning approximate it but can't produce genuinely novel options. The likely resolution: **pre-generated for main story conversations** (quality-controlled, voice-verified, fully authored trees with perception-based pruning), **runtime for ambient/side content** (NPC gossip, eavesdropping, optional NPC conversations, repeat visits to NPCs after main story events).
- **Conversation state as generator input.** The runtime generator needs: NPC profile, current NPC stance, conversation history (this conversation), relationship history (all conversations with this NPC), current perception vector, current game flags, current NPC knowledge state, atmosphere tone. This is a large context window per generation call.
- **Option format.** Each generated option must include: display text, perception requirements (for filtering), tone tag (for momentum tracking), NPC response text, NPC expression tag, stance shift, consequence actions (flag sets, relationship changes), and optionally a misunderstanding layer (intended meaning vs. received meaning).
- **Ambient dialogue generation.** NPC-to-NPC conversations generated for eavesdropping can use a lighter pipeline — no player options, no perception filtering, just contextual NPC dialogue based on location + game state + NPC knowledge. Could be pre-generated in bulk at scene transitions and cached.
- **Gossip generation.** When information propagates through the NPC network, the generator must transform facts through each NPC's personality filter. This is a batch operation triggered by game events: event occurs → gossip engine propagates to connected NPCs → each NPC's version generated → stored for next player conversation.

**Narrative Enemies Impact:**

The dialogue generator must produce combat-integrated dialogue — taunts, surrenders, and mid-fight revelations.

- **Contextual taunt generation.** During combat, the dialogue generator must produce taunts parameterized by game state: the enemy references the player's actual choices, fighting style, and prior encounters. These are lightweight dialogue events — no player options, but contextually specific text generated from the enemy's biography and the player's event log.
- **Surrender dialogue generation.** When an enemy triggers surrender, the dialogue generator must produce a full conversation tree with the enemy in a defined emotional state (desperate, defiant, relieved). The player's options depend on their perception traits and the information they've gathered about the enemy.
- **Enemy gossip integration.** NPC dialogue about enemies must reflect discovered and undiscovered enemy motivations. The dialogue generator must produce contextually appropriate NPC reactions to the player's combat choices — a village mourning a killed guard, a faction acknowledging a spared rival.

---

## 22. Game Event Log → Environmental Memory Store

Every significant game event feeds into per-location memory.

**What crosses the boundary:**

- **Input:** game events tagged with location, timestamp, type, participants, and outcome:
  - Combat events: who fought, who won, casualties, method
  - Dialogue events: who spoke, tone, key topics, lies told, information revealed
  - Player presence: duration, actions taken, emotional state during visit
  - NPC actions: NPC-to-NPC interactions, movements, state changes
  - Story events: quest completions, betrayals, discoveries, flag changes
- **Output:** per-location memory records consumed by the perception engine (for environmental rendering), the narrator (for place descriptions), and the Level Designer (for map variant regeneration)

**Open questions:**

- What constitutes a "significant event"? The store can't record everything. Filtering options:
  - Threshold: only events above a significance score
  - Type-based: all combat and story events, sampled dialogue and presence events
  - Impact-based: events that change game state (flags, relationships, faction status)
- What is the memory record format? Structured events? Compressed summaries? Emotional tags?
- How does memory decay? Do old events fade, or is all history equally weighted?
- How is the contagion model (spatial spread of environmental mood) evaluated? Per-visit? Per-scene-transition? Continuously?

**Subjective Rendering Impact:**

The event log must capture perception-relevant events, not just narrative events, because the character's perceptual experience of a location contributes to environmental memory.

- **Perception state at time of event.** The event log must record not just "combat happened at location X" but "combat happened at location X while the character was at anxiety 0.8 and guilt 0.6." The perception state during an event affects how the location "remembers" it — a fight experienced in terror leaves a different environmental residue than a fight experienced in confidence. This doubles the event record size but enables richer environmental memory.
- **Subjective experience as event type.** Beyond objective events (combat, dialogue, quest), the log should capture subjective experiences: "the character experienced intense agoraphobia at the market square," "the character saw a phantom enemy in the forest (paranoia hallucination)." These aren't objective game events — they're perception events — but they contribute to the character's relationship with the location and feed environmental memory.
- **Perception-driven event significance.** A combat event at high fear is more significant (to the character and to environmental memory) than the same combat at low fear. The significance weighting function must account for the perception state, not just the event type. This addresses the open question of "what constitutes a significant event" — significance is partly objective (event type) and partly subjective (perception state during event).

**Infinite Dialog Impact:**

Dialogue events are a primary source of environmental memory — conversations shape how locations are remembered.

- **Dialogue events as location memory.** The event log must capture conversations as location-bound events: where the conversation happened, who spoke, the emotional tone, key revelations, lies told, relationship changes. A betrayal conversation in the throne room marks the throne room as a betrayal site — environmental memory responds.
- **NPC knowledge state changes as events.** When an NPC learns something (through gossip, observation, or conversation), this is a state change event that the log must capture. The environmental memory store uses this to track information flow through locations — "this tavern is where the rumor started spreading."
- **Conversation emotional weight.** A heated argument contributes more to a location's emotional memory than casual small talk. The event log must tag dialogue events with emotional intensity, derived from the conversation's tone trajectory and outcome.

**Animated Narrator Impact:**

Narrator events are a source of environmental memory — the narrator's words about a place shape how it's remembered.

- **Narrator statement as location event.** When the narrator delivers significant commentary at a location (foreshadowing, revelation, emotional amplification), the event log must capture this as a location-bound event. The narrator's description of a place contributes to how the place remembers itself — a location the narrator described as "hungry for violence" accumulates that characterization.
- **Narrator self-revision as memory update.** When the narrator revises a prior statement about a location, the event log must record both the original and the revision as distinct emotional layers — the location was first described one way, then recontextualized. Both descriptions coexist in the location's emotional archaeology.

**Narrative Enemies Impact:**

Combat events are primary contributors to environmental memory with narrative weight beyond mechanical outcomes.

- **Combat moral weight in event log.** The event log must capture not just combat outcomes (who won, casualties) but the moral context: was the enemy's motivation known? Did the player have alternatives? Was the enemy protecting something? The moral weight of combat — derived from discovered information — determines how heavily the event marks the location's memory.
- **Enemy death as location scar.** When a narratively significant enemy dies at a location, the event log must record the enemy's biography alongside the combat outcome. The location "remembers" not just that a fight happened but who died and why they were fighting — informing temporal echoes and narrator descriptions on revisit.
- **Recurring enemy encounter sites.** The event log must track which locations hosted encounters with recurring enemies, enabling the location to accumulate the rivalry's emotional history. A battlefield where the same adversary was fought three times has richer memory than one hosting three unrelated skirmishes.

**Replay and Multiplayer Impact:**

The event log is the primary export channel for cross-player environmental inheritance.

- **Event log as multiplayer export source.** The environmental memory store's event log is the data that crosses the multiplayer state bridge (Interface 24). The log format must support serialization for cross-player inheritance: which events are exportable, how they compress into inherited world state, and how significance weighting determines which events persist for the next player.
- **Per-player event attribution.** In multiplayer modes, the event log must tag events with player identity so the environmental memory store can attribute location character to specific players. Player B encounters a forest shaped by Player A's actions — the attribution enables the narrator to reference "someone who came before" with specificity.
- **Generational event layering.** In generational play, the event log accumulates across player generations. The environmental memory store must compose multi-generation event histories into layered location personalities — the town square remembers Player A's celebration and Player B's mourning as distinct but coexisting emotional layers.

**Environmental Memory Impact:**

This is environmental memory's core interface — where game events become location personality.

- **Event-to-memory transformation rules.** The event log feeds raw events; the memory store transforms them into emotional layers. The interface must define how event types map to emotional tags: combat events produce fear/violence layers, celebrations produce warmth layers, betrayals produce bitterness layers. These mappings are defined per-location-personality-type (a church processes a fight differently from a battlefield).
- **Memory decay model.** The interface must define how events age in the memory store: do they decay over time (fading residue), persist indefinitely (permanent scar), or intensify through repetition (cumulative atmosphere)? The decay model may vary per event type and per location personality.
- **Contagion propagation triggers.** The memory store must define when and how location mood spreads to neighbors. The interface must specify: trigger threshold (minimum emotional intensity before contagion begins), spread rate (how fast mood propagates per scene transition), and boundary rules (what stops contagion — rivers, walls, healed locations).
- **Agency accumulation tracking.** The memory store must track cumulative event impact per location to determine when the location crosses the agency threshold — when it shifts from passively reflecting history to actively acting on it. The event log must provide enough structure for the store to compute this running total.

---

## 23. Environmental Memory Store → Level Designer / Atmosphere Director (Regeneration)

Accumulated location history triggers regeneration of map variants and atmosphere updates.

**What crosses the boundary:**

- **Memory Store → Level Designer:** location event history + current environmental personality → request for map variant (paths rearranged, cover added/removed, visual decay or growth, spatial expansion/contraction based on perception rules)
- **Memory Store → Atmosphere Director:** location emotional layers → updated atmosphere descriptors (palette shift, particle effects, lighting mood change)
- **Memory Store → Narrator:** location emotional history → narrator text context for place descriptions, callbacks, emotional resonance

**Open questions:**

- When does regeneration happen? Options:
  - **Between scenes:** regenerate affected maps during scene transitions (loading screen)
  - **On location re-entry:** regenerate when the player returns to a previously visited location
  - **Batch:** regenerate all affected locations periodically (between game chapters)
  - **Pre-computed variants:** generate multiple variants at build time, select at runtime based on memory state (fastest, least expressive)
- Is this a build-time or runtime system? If the environment develops agency through play, some regeneration must happen during gameplay.
- How much can change between visits? Constraints needed to prevent player disorientation (the map shouldn't be unrecognizably different).
- How does this interact with the pre-generation vs. runtime question? Environmental memory seems to require runtime generation — the LLM regenerates map/atmosphere variants based on accumulated play history.

**Replay and Multiplayer Impact:**

Environmental regeneration must account for inherited cross-player memory.

- **Inherited memory as regeneration input.** When Player B enters a location shaped by Player A's history, the Level Designer and Atmosphere Director must regenerate the location from inherited environmental memory — not Player B's own event history. The regeneration pipeline must support external memory sources alongside local ones.
- **Multi-generation accumulation.** In generational play, each generation adds to the location's memory. Regeneration must compose memories from multiple players into a single coherent visual and atmospheric result — the town square reflects three generations of decisions, not just the most recent.
- **Ghost trace generation.** The Level Designer must generate spatial elements from inherited player events: grave markers, worn paths, memorial sites, scars in the landscape. The regeneration pipeline must treat inherited event summaries as spatial feature generation inputs.

**Environmental Memory Impact:**

This is environmental memory's output interface — where accumulated history becomes visible, navigable space.

- **Emotional layer composition in regeneration.** The Level Designer must regenerate map variants that reflect coexisting emotional layers, not just the dominant mood. A location with warmth, pain, and healing layers needs visual elements from all three — not a simple palette swap but a complex tonal composition that shows emotional depth.
- **Environmental agency as regeneration trigger.** When a location crosses its agency threshold, regeneration must produce spatial changes that reflect the location's active behavior: paths rearranged, cover added or removed, traversal costs modified. These changes must be navigable and must not break game progression.
- **Temporal echo placement.** Regeneration must place temporal echo entities at positions corresponding to past events — ghostly figures walking a corridor, the flicker of an extinguished fire. The Level Designer must receive event positions from the memory store and generate echo overlays at those coordinates.
- **Contagion-driven regeneration.** Neighboring locations affected by spatial contagion must be partially regenerated: the corruption from the forest seeping into farmland as withered crop tiles, the healing from a cleansed shrine spreading as brightened tiles at the district boundary. Regeneration must be able to affect map edges without regenerating the entire map.

---

## 24. Player State → Multiplayer State Bridge

A player's game state serialized for consumption by another player's generation pipeline.

**What crosses the boundary:**

- **Input:** a completed or in-progress playthrough's state:
  - Event log: major decisions, combat outcomes, dialogue choices, quest completions
  - World state: faction standings, NPC relationship states, location conditions (destroyed, thriving, corrupted)
  - Environmental memory: per-location accumulated history
  - Character state: the player character's final perception vector, reputation, kill/spare record
- **Output:** "prior world state" consumed by another player's Story Director and domain agents as inherited context:
  - World history facts (for NPC dialogue generation: "a traveler came through before and...")
  - Environmental state inheritance (for Level Designer: the bridge is destroyed, the village is thriving)
  - Ghost trace seeds (for ambient world texture: grave markers, worn paths, NPC stories about past events)
  - Faction/political state (for Story Director: the rebellion succeeded, the empire is fractured)

**Open questions:**

- What is the serialization format? Must be sufficient for another player's world to inherit consequences without inheriting the full game save.
- How much is inherited? A full event log produces a world dominated by the prior player. A compressed summary produces subtle traces. This is a design knob, not a technical constraint.
- How are contradictions resolved? Player A destroyed the bridge; Player B's story requires the bridge. Does the generation pipeline route around inherited state, or is inherited state advisory?
- How does generational play (Player B is a successor generation to Player A) differ from parallel play (Player B is a contemporary in the same world)? Different inheritance rules for different multiplayer modes.
- How is privacy handled? Does Player A consent to their playthrough influencing others?

**Subjective Rendering Impact:**

The multiplayer state bridge must serialize perception-related state, not just narrative state.

- **Perception vector inheritance.** Player A's final perception vector (or its trajectory) becomes context for Player B's world. If Player A ended with high corruption, Player B's world inherits an environmentally darker starting state. The serialization format must include the perception vector trajectory — not just final values but the arc (how values changed over time) — because the trajectory affects environmental memory and NPC stories about the previous player.
- **Subjective experience export.** Player A's perception-mediated experiences (hallucinated enemies, perceived hostility, memory distortions) are part of their story but not part of objective world state. When Player B inherits the world, should these subjective experiences leave traces? If Player A saw phantom enemies in the forest (paranoia), does the forest carry that memory for Player B? Probably not — phantom enemies weren't real. But Player A's behavioral response to them (fleeing, attacking nothing) WAS real and could leave traces (NPCs saw Player A acting strangely). The serialization must distinguish objective events from perception-mediated experiences and export only the objective consequences.
- **Perception profile as world-shaping context.** In generational play, Player A's perception profile shapes how NPCs remember them. "The old hero was paranoid — saw threats everywhere, trusted no one." This is generated from Player A's perception trajectory and becomes NPC dialogue context for Player B. The serialization must include a summarized perception profile suitable for NPC story generation.
- **Multi-character perception state.** If Player A played multiple characters (replay), the state bridge must track per-character perception vectors and their world impacts separately. The inherited world state is the union of all characters' objective impacts, but the perception-mediated aspects remain per-character.

**Infinite Dialog Impact:**

Dialogue history is a primary export for multiplayer inheritance — conversations shape the world other players inherit.

- **NPC relationship history export.** Player A's conversation history with each NPC becomes inherited context for Player B's world. The blacksmith remembers "the previous traveler" and references specific conversational details — generated from Player A's actual dialogue choices. The serialization must include per-NPC relationship summaries: rapport level, key conversation topics, promises made, lies told, tone pattern.
- **Gossip network state export.** The state of information propagation across the NPC network — who knows what, what's been distorted — carries forward. Player B enters a world where NPCs already have opinions about events from Player A's playthrough, informed by how gossip transformed the information.
- **Conversation outcomes as world facts.** Key dialogue outcomes (the player convinced the council, failed to negotiate the treaty, extracted a confession) become world history facts that NPCs in Player B's world reference. The serialization must distinguish between outcomes that reshape the world (treaty signed) and outcomes that are interpersonal (rapport with the blacksmith).

**Replay and Multiplayer Impact:**

This is the core multiplayer interface — every cross-player concept converges here.

- **Asymmetric inheritance modes.** The state bridge must support multiple inheritance patterns: asynchronous influence (Player A's completed world shapes Player B's), parallel perspective (two players in the same story sharing synchronized state), generational play (Player A's playthrough becomes Player B's world history), and villain-as-player (one player's strategic decisions become the other's encounter definitions). Each mode requires different serialization depth and different consumption patterns.
- **Ghost trace serialization.** The state bridge must export player events in a format suitable for diegetic trace generation: where the player died, where they buried an enemy, where they took a specific path. These must be tagged with narrative significance so the receiving world generates appropriate artifacts (a grave marker, a worn path, an NPC story) rather than flooding the world with traces.
- **Environmental memory export.** The state bridge must export per-location accumulated memory for environmental inheritance. The format must include emotional layers, event summaries, agency state, and contagion patterns — sufficient for the receiving world to regenerate locations that feel shaped by a prior player's history.
- **Dungeon master state bridge.** In dungeon master mode, one player's real-time directions must be serialized and consumed by the other player's generation pipeline as Story Director input. The state bridge must support synchronous, low-latency state exchange — not just post-game export.
- **Privacy and consent model.** The serialization format must support selective export: which events the player consents to share, which are private. The state bridge must define a privacy schema that lets players control how much of their playthrough influences others.

---

## Three Hardest Unresolved Questions

These cut across multiple interfaces and must be resolved before detailed design of any subsystem.

### 1. Pre-generation vs. Runtime LLM

**Interfaces affected:** 19, 20, 21, 22, 23

The doc states "the LLM does not run during gameplay." But infinite dialog (Interface 21), the animated narrator (Interface 20), and environmental memory regeneration (Interface 23) all require contextual generation that cannot be fully pre-authored. The combinatorial space of game state × perception vector × player behavior history × conversation context is too large for exhaustive pre-generation.

**Likely resolution:** hybrid. Pre-generated content for the main story spine (planned scenes, key dialogue, narrator beats for story events). Runtime LLM for reactive content (ambient NPC dialogue, narrator commentary on player behavior, dialogue option generation for unplanned conversations, environmental memory responses). The engine manages a generation budget and caches aggressively.

This decision determines whether the output is a **static game artifact** (WASM bundle, play offline) or a **live system** (requires LLM API access during play). Both modes may need to be supported.

### 2. The Perception Modifier Format

**Interfaces affected:** 2, 5, 18, 19

Subjective rendering touches every engine subsystem. The rules for how character traits modulate tiles, NPC behavior, dialogue options, narrator voice, collision, visibility, scale, and geometry need a concrete representation the engine evaluates at runtime.

This is a new data type that doesn't exist in any standard game engine. It needs to be:
- **Expressive enough** for the range described in the research (agoraphobia expanding space, stealth adding cover, paranoia spawning false enemies)
- **Composable** (hungry + paranoid + stealthy applies all three simultaneously without conflicts)
- **Performant** (evaluated per-frame for rendering modifiers, per-interaction for behavior/dialogue modifiers)
- **Authorable** by LLMs (the Story Director and domain agents generate these rules)

### 3. The Game State Schema

**Interfaces affected:** 14, 19, 20, 21, 22, 24

The fact store, perception vector, event log, environmental memory store, NPC relationship graph, and player behavior history are all "game state" but serve different consumers with different access patterns:

| State component | Written by | Read by | Update frequency |
|---|---|---|---|
| Perception vector | Game events, story progression | Perception engine, dialogue generator, narrator | Per-event |
| Fact store | All agents (build time), game events (runtime) | Continuity tracker, dialogue generator | Per-event |
| Event log | Engine (runtime) | Environmental memory, narrator, multiplayer bridge | Per-event |
| Environmental memory | Event log processor | Perception engine, narrator, level regeneration | Per-location-visit |
| NPC relationships | Dialogue outcomes, story events | Dialogue generator, NPC behavior, narrator | Per-conversation |
| Player behavior history | Engine (runtime) | Narrator, difficulty adaptation, enemy behavior | Continuous |

A unified schema must accommodate all of these with consistent naming, typed fields, and clear ownership semantics.
