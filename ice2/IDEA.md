# Ice

A basic 2D game engine (direct characters over a 2D map, real-time combat, dialogue system, play scripts, simple interactions) runs an action-exploration game with narrative focus as described in a Rust schema with assets. The schema is developed entirely automatically from a prompt using a hierarchy of LLMs and procedural scripts. This allows for unbounded content commoditization and customization, which can be exploited in various ways — one of which is fully automated custom DLCs on request.

## Schema

The schema is a bunch of Rust structs, enums, etc. This is essentially the game data structure in the engine. It's the ground truth for the game. It's what the authoring tools ultimately produce that can be played by the user using the engine. The schema should cover tile map definitions (regions, layers, tile types, traversal properties), entity definitions (NPCs, enemies, objects with position, sprite reference, behavior reference), player definition (position, perception vector, hidden stats), dialogue tree structure (nodes, choices, conditions, consequences), play script primitives (move, face, speak, wait, screen effect), and asset references (sprite sheets, portraits, tile patterns). As the project grows (see phases), the schema refines, more types are added, different content becomes possible.

## Engine

The engine starts as a Rust/WebAssembly engine that can load schema-valid game files and run in the browser. The first version is very minimal and only contains tile-based map rendering, player entity with WASD movement, static NPCs with touch-to-talk trigger, basic dialogue display and a single play script that runs on game start (camera pan, NPC walks, speaks). As the project grows (see phases), the engine refines, more features are added.

## Authoring

A Director (frontier model) sits at the top. It receives the user prompt, interprets creative intent, and decomposes the game into structured work packages. Below it, specialized agents handle domains they're suited for. The authoring system develops in phases, each adding agents, steps, and capabilities.

### Phase 1 — Core Pipeline

Agents: Director, Narrative Agent, World Agent, Asset Agent.

The Director takes the prompt and produces a game design document (world concept, narrative arc, region list, character roster, tone/aesthetic direction). The Narrative Agent takes the design doc and produces story structure: dialogue trees, character arcs, branching consequences. The World Agent takes the design doc + narrative structure and produces spatial layout: region definitions, tile maps, entity placements, traversal properties. The Asset Agent takes character/world descriptions and produces visual assets: sprite sheets, tile patterns, portraits (via image generation models or procedural pixel art scripts).

Steps:
1. Prompt intake — Director parses user prompt, makes reasonable defaults.
2. Design document — Director produces setting, tone, scope, narrative premise, character concepts.
3. Narrative pass — Narrative Agent builds story structure and dialogue trees. Director reviews for coherence.
4. World layout — World Agent builds regions and tile maps, places entities. Director checks that every narrative-referenced location exists and is reachable.
5. Asset generation — Asset Agent produces sprites, tiles, portraits. Can run in parallel with world layout for characters/items that don't depend on placement.
6. Schema assembly — All outputs merged into a single valid schema. Structural validation against the Rust types (deterministic, not LLM).
7. Continuity validation — Director reviews the assembled schema for cross-cutting issues: are all entity references valid? Do dialogue conditions reference stats that exist?

Output: a valid, playable schema with a static world, NPCs, dialogue, and assets.

### Phase 2 — Scripting and Balance

Agents added: Script Agent, Balance Agent.

The Script Agent takes narrative events and world layout and produces play scripts: cutscene choreography (move, face, speak, wait, screen effect sequences). The Balance Agent takes entity definitions, combat parameters, player stats, and narrative pacing, and tunes numbers: enemy difficulty curves, item distributions, stat progression.

Steps added (between asset generation and schema assembly):
- Script assembly — Script Agent translates narrative beats into play script primitives, referencing actual entity IDs and map positions.
- Balance pass — Balance Agent tunes combat/progression numbers against narrative pacing.

Output: a schema with scripted cutscenes, tuned combat, and progression curves.

### Phase 3 — Audio

Agent added: Music/Audio Agent.

Takes tone/setting descriptions per region and produces background music and sound cues. Can run in parallel with scripting and balance.

Step added:
- Audio pass — Music Agent generates per-region audio and sound cues.

Output: a complete schema with audio.

### Phase 4 — Playtesting

No new agents — the Director and a dedicated playtest harness use frontier models to validate specific aspects of the authored game.

Playtest targets:
- Critical path walkability — A model walks the main quest route through tile maps, checking traversal. Catches blocked paths and missing region transitions that no static check can find.
- Dialogue coherence — A model plays through key dialogue trees, checking that branches make narrative sense, conditions are reachable, and no branch leads to a dead end or contradictory state.
- Play script timing — A model watches cutscenes play out to verify no overlapping movements, characters speaking while off-screen, or broken pacing.
- Difficulty spikes — A model plays through combat encounters in sequence and flags dramatic difficulty jumps without narrative justification.
- First 5 minutes — A model plays the opening to optimize the player's introduction to the game.

The Director sends revision requests back to the relevant agents when playtesting surfaces issues. This loops until the schema passes all checks.

## Ecosystem
