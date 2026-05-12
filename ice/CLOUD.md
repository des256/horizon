# Cloud Architecture

The base game runs without any LLM. It is a complete, pre-generated experience — story, characters, dialogue, combat, narration, assets — produced by the generation pipeline and consumed by the engine. The player downloads it and plays in the browser. Nothing else is needed.

The cloud is what makes the game come alive on replay, and what makes it personal.

## Three Tiers

### Tier 1: The Base Game (Free, No Account)

A complete game file. Downloaded once, plays offline in the browser. The Story Director (frontier model) produced it at generation time, along with domain specialists and an asset pipeline. The result is a finished game: maps, NPCs, dialogue trees, combat encounters, play scripts, narrator text, perception modifiers, all visual assets. Every system works. The player has a real, full experience.

The base game is not a demo. It's not truncated. It's the game the Story Director authored. It stands on its own.

What the player doesn't see: the Story Director also produced an **expansion space** — authored potential that the base game doesn't surface. Unexplored character depths, dormant plot branches, latent world regions, NPC relationship arcs that never activate, enemy motivations that stay hidden. The expansion space is narratively coherent with the base game because the same frontier model authored both in the same pass. It ships as part of the game file but is inert without the cloud.

### Tier 2: Cloud-Enhanced Replay (Account Required, Subscription)

The player creates an account and replays the game. The engine now communicates with the cloud service during play. The cloud has access to:

- The full game file including the expansion space.
- The player's complete playthrough history (every choice, every combat outcome, every dialogue path, every location visited and in what order, time spent in each area).
- The current game state in real time.

With this context, the cloud enhances the game live:

**Dialogue.** New options appear that weren't in the base game. The blacksmith who was grumpy but functional in the base game now has something to say about the war — because the expansion space defined his hidden backstory, and the cloud decided this player's behavior pattern (spending time in the ruined district, reading every memorial plaque) warrants revealing it. The pre-written dialogue options appear instantly; cloud-generated options stream in alongside them, visually distinguished by a subtle indicator or presented after a natural beat.

**Narration.** The narrator remembers. "You approach the bridge. You've been here before, though you didn't know it then. The last time, you hesitated. This time your step is different." The narrator develops a relationship with the player as a returning player — not just a character in the story but a person who has played this game before. Dramatic irony becomes real: the narrator can reference what the player knows from previous playthroughs while the character remains ignorant.

**Encounters.** Enemies adapt. The bandit leader who killed the player three times last playthrough taunts differently now. The morally complex enemy whose hidden motivation the player missed has new discoverable information seeded in different locations. Combat difficulty adjusts not through stat scaling but through behavioral adaptation — enemies who noticed the player favors ranged attacks close distance faster.

**Perception.** The perception engine gets deeper modifiers from the cloud. The player who consistently chose aggressive options experiences a world that feels slightly more hostile — not through stat penalties but through subtle rendering shifts, NPC body language, narrator tone. The cloud feeds contextual perception parameters that the rule-based engine alone couldn't derive.

**Environmental Memory.** Locations accumulate memory across replays. The forest the player burned in playthrough 1 is regrowing in playthrough 2. The town square where a dramatic fight happened carries a mood that the narrator acknowledges. Locations develop genuine layered history through repeated play.

**The Expansion Space Activates.** Content the base game kept dormant becomes available based on the player's demonstrated interests. The player who spent time talking to every NPC in the village gets deeper NPC arcs. The player who explored every corner of the map gets access to hidden regions. The player who engaged with the moral dimensions of combat gets more morally complex encounters. The cloud watches what the player gravitates toward and materializes the expansion space accordingly.

### Tier 3: Personalized DLC (Account Required, Per-Generation Purchase)

After finishing the game (base or cloud-enhanced), the player can request new content. This is the most distinctive feature.

#### The "What If" Request

The player describes what they want in natural language:

- "What would have happened if I refused to sell the magic ring?"
- "I really wished they had expanded the barmaid's arc in that one village."
- "I want to play as the villain."
- "What happens to the kingdom twenty years after my ending?"
- "I want to explore the northern mountains that were mentioned but never accessible."
- "Give me a harder version where the bandit faction actually wins early on."

The cloud receives this request along with:

- The full game file and expansion space.
- The player's complete playthrough history.
- The narrative constraints the Story Director defined (what must stay coherent, what can change, what must not be contradicted).

A frontier model — running asynchronously with no latency constraint — takes this and generates new content.

#### What Gets Generated

A personalized DLC is a complete, self-contained game segment. It downloads as a new game file using the same engine and schema. The player opens it and plays. It typically contains:

- **New maps.** 3-5 new playable areas with tile maps, collision, traversal, NPCs, enemies. These are generated from spatial templates using the game's existing tile palettes and biome definitions, supplemented with new assets where the existing vocabulary doesn't cover the new content.
- **New characters.** NPCs with full dialogue trees, personality profiles, voice parameters. Some are entirely new; some are existing characters in new circumstances (the barmaid who now has a full arc, the villain whose perspective you now inhabit).
- **New story content.** A narrative arc that branches from the player's specified point of interest. Dialogue, play scripts, narrator text, encounter designs. Authored by a frontier model with the full game context, so it feels like a natural extension, not a bolted-on addition.
- **New assets.** Sprites, portraits, tilesets generated by the image pipeline on cloud GPUs. The DLC arrives with all visual assets included — the player doesn't need to generate anything locally.
- **Narrative continuity.** The DLC references the player's specific choices from their playthrough. If the player's request is "what if I refused to sell the ring," the generated content knows everything else the player did — who they befriended, who they fought, what they explored — and weaves it in.

#### Generation Process

The DLC generation is asynchronous. The player submits a request and gets a notification when it's ready — the generation may involve multiple frontier model passes, image generation, validation, and quality checks. The process:

1. **Request Interpretation.** A frontier model reads the player's natural language request, their playthrough history, and the game's expansion space. It produces a structured brief: what narrative branch to explore, which characters are involved, what new locations are needed, what the emotional arc should be, how it connects to the player's existing experience.

2. **Content Generation.** The same agent hierarchy that produced the base game runs again, but scoped to the DLC brief. Story Director produces the narrative spine. Domain specialists generate maps, encounters, dialogue, scripts. The expansion space is the primary source — much of the DLC content may already be latent in the expansion space, needing only realization and personalization rather than invention from scratch.

3. **Asset Generation.** Image generation models produce any new visual assets the DLC requires. The style parameters from the base game carry over — the DLC looks like part of the same game. Where possible, existing assets are reused or recolored.

4. **Validation.** Schema validation ensures the DLC file is structurally valid. Narrative validation checks continuity with the base game and the player's history. The same validators that checked the base game check the DLC.

5. **Delivery.** The DLC downloads as a game file. The engine loads it alongside or in place of the base game. The player plays it in the browser, same as the base game. No LLM needed at runtime — the DLC is fully pre-generated, just personalized.

#### What Makes This Different from Generic DLC

Traditional DLC: "Here's the next chapter. Same for everyone." The studio wrote it. It ships once.

Personalized DLC: "Here's what happens next *in your story*." A frontier model wrote it, using the full context of your playthrough. It exists only for you. Every player who requests DLC from the same base game gets different content — because their playthroughs were different, their requests are different, and the generated narrative responds to both.

The personalization is the product. A pirated DLC is someone else's story — it references choices you didn't make, characters you didn't meet, consequences that don't match your experience. It doesn't work as *your* continuation.

And the player can keep requesting. Finish a DLC, request another. "Now what happens to the barmaid after the events of the first DLC?" Each generation builds on everything before it. There's no content exhaustion because the content is generated from the accumulated context of the player's entire history with the game.

#### The Verbal Request as Game Design

The "what if" request is itself a game design element. The player's imagination becomes an input to the system. They don't choose from a menu of pre-defined expansions — they describe what they're curious about, and the system responds. This is the moment where the player transitions from consumer to collaborator.

But it must be bounded. The player can't request "turn this into a space opera" — the narrative constraints from the Story Director define what the game world can accommodate. The system can say "that would break the world's internal logic — here's what I can do instead" and offer alternatives. The constraint negotiation itself is an interesting interaction: the player proposes, the system responds with what's possible within the authored world, and together they find the DLC that satisfies the player's curiosity while respecting the game's coherence.

The request can also be vague. "Surprise me." "Show me something I missed." "Make it harder." "Tell the story from someone else's perspective." The system has enough context — the player's playthrough, the expansion space, the narrative constraints — to interpret vague requests meaningfully.

## Agent Hierarchy

### Pre-Game (Runs Once Per Base Game)

**Story Director** — frontier model. Produces the base game content AND the expansion space in a single coherent pass. The expansion space is not an afterthought; it's part of the authorial act. The Story Director writes the story that gets told (base game) and the stories that could be told (expansion space), ensuring they don't contradict each other.

The Story Director's output now includes:
- Complete base game narrative (acts, scenes, characters, dialogue, encounters, scripts).
- Character depth profiles — what each character is hiding, what could be revealed, under what conditions.
- Dormant plot branches — decision points in the base game where the story could have gone differently, with enough narrative scaffolding that the cloud can generate the alternate path.
- Latent world content — regions, characters, factions that are referenced or implied but not realized in the base game.
- NPC relationship potential — how relationships could deepen or deteriorate beyond what the base game shows.
- Narrative constraints — invariants that must hold across all DLC and enhancement. "The river kingdom fell because of the drought, not the war — any DLC must respect this." "Character X's motivation is always grief, never revenge, even if the player misreads it."

**Domain Specialists** — frontier or mid-tier models, producing base game content under the Story Director's spec. Level Designer, Character Designer, Encounter Designer, Script Director, Dialogue Writer. Same as the original hierarchy, but their output quality determines the base game's quality ceiling, which is the player's first impression.

**Asset Pipeline** — image generation models. All visual assets for the base game plus a surplus vocabulary of asset variants (alternate color palettes, alternate character expressions, environmental variations) that the cloud can reference during enhancement or DLC generation.

**Validators** — small models or deterministic. Continuity, consistency, pacing checks on the base game.

### Cloud Service

#### Session Manager (Deterministic, No LLM)

The backbone of the cloud service. Handles:

- **Player accounts and playthrough storage.** Every choice, every path, every outcome, timestamped. This is the raw material the LLM agents consume.
- **Context construction.** When an LLM agent needs to generate something, the session manager assembles the right context: relevant slice of the game file, relevant slice of the expansion space, relevant player history, current game state. The LLM never sees the full game — it sees a curated window optimized for the current task.
- **Output validation.** Every LLM response is validated against the schema before reaching the engine. Malformed output is rejected and re-requested or replaced with fallback content.
- **Narrative constraint enforcement.** The Story Director's invariants are checked deterministically. If the LLM generates dialogue that contradicts an invariant, it's caught here, not in the LLM's reasoning.
- **Latency management.** Predictive generation based on the player's likely next actions. While the player is in a dialogue scene, the cloud is pre-generating enhanced content for the next area they'll probably visit.
- **Rate limiting and cost control.** The cloud LLM budget per session must be bounded. Not every moment needs enhancement — the session manager decides when enhancement adds value and when the base game is sufficient.

#### Tier 2 Agents: Real-Time Enhancement

A mid-tier model (30B–70B) running on cloud GPUs with low-latency inference. The session manager prompts it in different roles depending on what the engine requests:

**Dialogue Enhancer.** Receives: NPC profile, current knowledge state, player history summary, current scene context, base game dialogue options. Produces: additional dialogue options and NPC lines that weave in expansion space content and player-specific context. The base game options appear instantly; enhanced options stream in with minimal delay.

**Narrator.** Receives: narrator personality profile, current scene, player's recent actions, cross-playthrough context (what happened here last time). Produces: contextual narrator text that supplements or replaces the base game's pre-written narration. The narrator is the most natural place for cloud enhancement because narrator text has inherent pacing — a slight delay reads as dramatic timing, not latency.

**Encounter Adapter.** Receives: upcoming encounter design, player combat history, player behavior patterns. Produces: modified enemy behaviors, adjusted placement, contextual enemy dialogue, discoverable information placement. This runs predictively — the cloud adapts encounters the player hasn't reached yet based on their trajectory.

**Perception Advisor.** Receives: current perception vector, recent player choices, cross-session behavior patterns. Produces: additional perception modifier suggestions that the rule engine evaluates and applies. This is a light role — most perception is rule-based, and this role only fires when the session manager identifies a moment where cloud-driven perception would add value.

**Memory Narrator.** Receives: location memory state, player's history at this location, other players' history at this location (if cross-player features are enabled). Produces: natural language descriptions that the narrator role and the perception advisor consume. "This clearing remembers fire. Not yours — someone else's, from before."

#### Tier 3 Agents: DLC Generation (Asynchronous)

Frontier models running without latency constraints. Triggered by the player's "what if" request.

**Request Interpreter.** Receives: the player's natural language request, their full playthrough history, the game's expansion space, and the narrative constraints. Produces: a structured DLC brief — the narrative branch to explore, characters involved, new locations needed, emotional arc, connection points to the player's history, constraint boundaries.

If the request conflicts with narrative constraints, the interpreter produces alternatives: "The ring can't be un-sold because it triggered the merchant guild arc — but I can generate a story where you steal it back. Or a story where you find a second ring with different properties. Which interests you more?" This negotiation is itself cloud-generated and can go back and forth.

**Story Director (DLC mode).** The same frontier model that authored the base game (or one with the same prompt lineage and full context). Takes the DLC brief and produces a scoped narrative: 3-5 scenes, new characters or character developments, encounter designs, dialogue architecture, narrator arc for the DLC segment. Respects all constraints. References the player's history.

**Domain Specialists (DLC mode).** The same specialist roles as base game generation, scoped to the DLC brief. Level Designer generates new maps from the game's spatial template library. Character Designer realizes new NPCs or deepens existing ones. Encounter Designer creates new combat encounters. Dialogue Writer writes new dialogue trees. Script Director creates new play scripts for dramatic moments.

**Asset Pipeline (DLC mode).** Image generation for any new visual content. New tilesets if the DLC visits a new biome. New character sprites and portraits. The style parameters from the base game carry over — consistency is automatic because the same style prompts are reused.

**Validators (DLC mode).** Schema validation, narrative continuity with the base game and player history, pacing checks on the DLC as a standalone segment and as a continuation.

#### Cross-Player Services

These operate across all players of the same base game. They're the infrastructure for the multiplayer and environmental memory research directions.

**Ghost Trace Generator.** When a player completes a notable action at a location, the cloud generates a diegetic trace — a world-appropriate artifact that other players might encounter. A grave marker, a worn path, an NPC who tells a story about "a traveler who came before." Ghost traces are pre-generated and seeded into other players' enhanced replays.

**Environmental Memory Aggregator.** Accumulates location memory across all players. The forest that Player A nurtured, Player B burned, and Player C ignored has three layers of emotional history. The cloud composes these into a rich location personality that informs the narrator and perception engine for subsequent players.

**Community World State.** The aggregate of all player actions on a base game. Not real-time multiplayer — asynchronous influence. The cloud tracks faction outcomes, environmental changes, NPC fate statistics ("73% of players saved the barmaid"). This feeds into enhanced replay: the narrator might remark, "Most who came before you chose differently here."

**Multiplayer State Bridge.** For players who opt into synchronous or near-synchronous multiplayer: one player's real-time actions influence another's world through the cloud. Player A's combat in the forest generates smoke visible in Player B's sky. This is the most architecturally complex service and depends on everything else working first.

## Research Topics in the Cloud Model

### Subjective Rendering

**Base game:** Rule-based perception modifiers. Fear darkens, exhaustion mutes, paranoia shifts NPC behavior. Parameterized, deterministic, runs in the engine.

**Cloud replay:** The cloud feeds deeper perception parameters based on the player's accumulated behavior. A player who consistently avoids conflict experiences a world that feels calmer — not because the danger is lower, but because the rendering softens edges, the narrator speaks more gently, NPCs approach more openly. A player who hoards items sees the world as slightly more scarce. These are behavioral inferences the rule engine can't make alone.

**DLC:** The DLC's entire tone is perception-tuned to the requesting player. A player whose playthrough was marked by fear gets a DLC that leans into horror. A player who engaged with the political systems gets a DLC that leans into intrigue. The perception vector for the DLC is set by the player's history, not by a generic default.

### Infinite Dialog

**Base game:** Pre-written dialogue trees. Good, complete, finite. The constrained-options-as-character-expression model — 3-4 options per node, each revealing the player character's personality.

**Cloud replay:** New options appear. The cloud adds 1-2 options that reference the player's history or activate expansion space content. "You could ask about the war" is a base game option. "You could ask about his son — you noticed the portrait in the back room" is a cloud option that only appears if the player explored the room on a previous playthrough.

**DLC:** Fully generated dialogue trees for new characters and expanded arcs for existing ones. The barmaid who had 10 lines in the base game now has full conversation arcs across multiple scenes. The "what if" request is itself a dialogue — the player talks to the system in natural language, and the system responds with generated content.

### Animated Narrator

**Base game:** Pre-written narrator text, one consistent voice. Reacts to game events through pre-authored passages.

**Cloud replay:** The narrator becomes truly contextual. It references the player's previous playthroughs explicitly. It develops a relationship with the player across sessions. It might become warmer if the player explores thoroughly, or more sardonic if the player rushes through. The narrator knows things the player doesn't yet know (because the cloud knows the expansion space), creating real dramatic irony.

**DLC:** The narrator's voice carries over from the base game but adjusts to the DLC's tone. In a "what if" DLC, the narrator might be wistful ("In the world you remember, you sold the ring. Here, you held onto it. The weight of it changes everything."). The narrator bridges the player's lived experience and the hypothetical, functioning as a guide between realities.

### Narrative Enemies

**Base game:** Fixed enemy encounters with pre-designed behavior trees, discoverable motivations, faction dynamics. Boss fights with narrative phases.

**Cloud replay:** Enemies adapt to the player's combat history. New discoverable motivations are seeded in different locations. Recurring enemies reference previous encounters. The enemy who defeated the player multiple times last playthrough has new dialogue that acknowledges the rivalry. Faction dynamics shift based on how the player interacted with faction members.

**DLC:** The "play as the villain" request generates the entire game from the other side. The player's actions from their original playthrough become the environment the villain navigates. The DLC's enemies are the base game's heroes, complete with their own motivations and discoverable reasons for opposing you. The system generates new combat encounters where the player's previous character is the boss.

### Replay and Multiplayer

**Cloud replay:** Every replay is genuinely different because the cloud enhances differently based on accumulated history. The narrator voice shifts. New dialogue appears. The expansion space activates different content. This is replay as a first-class game design feature, not "play the same thing again."

**Cross-player:** Ghost traces from other players appear in enhanced replays. Environmental memory accumulates across the community. The narrator references collective player behavior. The game world develops genuine shared history.

**DLC and cross-player:** The most ambitious combination. Player A's DLC generates a character who is a ghost trace in Player B's world. Player B's "what if" request generates a DLC that takes place in a world shaped by Player A's original playthrough. The cloud is the connective tissue between all players' stories.

### Environmental Memory

**Base game:** Location memory within one playthrough, rule-based event-to-emotion transformation.

**Cloud replay:** Location memory persists across replays. The forest the player burned in playthrough 1 shows regrowth in playthrough 2 — not through pre-authored content but through cloud-generated descriptions and perception modifiers. Locations accumulate emotional layers that the narrator acknowledges.

**Cross-player:** Locations accumulate memory from all players. The community's actions compose into rich, layered location personalities. A location visited by hundreds of players has history that no individual player authored — it emerged from the aggregate of their decisions, interpreted by the cloud into environmental character.

**DLC:** New locations in the DLC inherit the emotional context of adjacent base-game locations through spatial contagion. The forest near the town the player saved feels different from the forest near the town they abandoned. Environmental memory is the substrate that makes DLC feel like it belongs in the same world.

## Latency Architecture

### Tier 2: Real-Time Enhancement

The challenge is that the player is playing in real time. Cloud requests add network latency (50-200ms round trip) plus inference latency (100-500ms for short generations on a mid-tier model).

**Dialogue.** Base game options appear instantly from the pre-generated game file. Cloud options stream in alongside — the UI presents base options immediately and adds cloud options as they arrive. A subtle animation or visual cue indicates additional options are incoming. If the player selects a base option before cloud options arrive, that's fine — the game continues. The cloud enhancement is additive, never blocking.

**Narration.** Narrator text has natural pacing. A brief pause before narration reads as dramatic timing, not latency. The cloud generates narrator text during natural pauses — scene transitions, map loads, combat resolution. The engine requests narrator text for the next likely moment while the current moment plays.

**Encounters.** Predictive generation. The cloud adapts upcoming encounters while the player is in the current scene. By the time the player reaches the next combat, the adapted encounter is already prepared. Only sudden, unpredicted encounters (ambushes triggered by unexpected player behavior) have real-time constraints.

**Perception.** Perception modifiers update on scene transitions, not per-frame. The cloud sends updated perception parameters when the player enters a new area or when a significant game event occurs. The engine interpolates between updates.

### Tier 3: DLC Generation

No real-time constraint. The player submits a request and gets a notification when the DLC is ready. The generation process can take minutes to hours depending on scope:

- Request interpretation: seconds (single frontier model call).
- Narrative generation: minutes (multiple frontier model passes, revision loops).
- Map and encounter generation: minutes (domain specialists in parallel).
- Asset generation: minutes to hours (image generation models, the slowest step).
- Validation: seconds (schema checks, continuity checks).
- Packaging: seconds (assemble game file).

The player can request a DLC, close the browser, and come back later. Push notification when it's ready. This is the familiar "your video is processing" pattern.

## Business Model

**Tier 1 (base game):** Free. The distribution mechanism. Anyone can play. The game is complete and satisfying on its own. The purpose is to give the player a taste of the world and make them want more.

**Tier 2 (cloud replay):** Subscription. Monthly fee for access to cloud-enhanced replay across all games on the platform. The player gets richer replays, narrator personalization, cross-player features, environmental memory persistence. The value proposition: "your game remembers you."

**Tier 3 (personalized DLC):** Per-generation purchase. The player pays for each DLC request. Pricing could vary by scope — "explore a character" is cheaper than "generate a sequel chapter." The value proposition: "your curiosity becomes content."

**Platform economics.** ice is not one game — it's a platform for generating games. Each base game is a seed that can produce unlimited personalized content. The generation cost (cloud LLM inference + image generation) is per-request, and the player pays per-request. The margin is the difference between generation cost and player willingness to pay for personalized content.

**Content moat.** Personalized content can't be redistributed meaningfully. A DLC generated for one player's playthrough doesn't work as another player's continuation. The personalization is the product, and it requires the cloud to produce.

## What This Architecture Doesn't Decide Yet

- **Cloud infrastructure.** Which cloud, which inference framework (vLLM, TGI, etc.), how to scale. Deferred until the engine and base game generation are working.
- **Account system specifics.** Auth, billing, playthrough storage format, data retention. Standard web infrastructure, nothing novel.
- **Cross-player consent model.** How do players opt into having their actions influence others' worlds? Privacy implications of playthrough data. Needs design.
- **DLC scope boundaries.** How much content per DLC request? What prevents "generate me a 100-hour sequel" requests? Pricing and generation budget caps, but the specifics need design.
- **Content moderation.** The player's natural language request could ask for anything. The system needs to handle requests that conflict with the game's tone, that are inappropriate, or that are simply impossible to generate well. Constraint negotiation handles the latter; content policy handles the former.
- **Model selection.** Which specific models for which roles. The architecture is model-agnostic — the session manager constructs prompts, the LLM generates, the validator checks output. The specific model can change as the landscape evolves.
- **Offline DLC play.** Once a DLC is generated and downloaded, it should play fully offline (same as the base game). This is straightforward — the DLC is a complete game file — but the engine needs to handle multiple game files (base + DLCs) and maintain continuity across them.
