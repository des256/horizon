# Prior Art & Related Research

## GPU-First Rendering

- **Immediate-mode GUIs** (Dear ImGui, egui) — no retained widget tree, "draw this frame." AI-friendly but historically limited to tooling/games.
- **Flutter Impeller / Rive** — owns the pixel pipeline, no platform widgets. Closer to "GPU primitives + events."
- **Vello/Xilem** (Linebender, Rust) — GPU-first 2D renderer with minimal retained scene graph.
- **Pathfinder / Vello** — GPU path rendering research.
- **SDF text rendering** (Valve 2007, multi-channel SDF) — efficient GPU text at any scale.

## Layout & Constraint Systems

- **Cassowary** (Auto Layout) — declarative spatial relationships via linear constraint solving.
- **Penrose** (CMU) — declarative diagramming with constraint solving.
- **Yoga** (Meta) — cross-platform flexbox implementation (used in React Native).

## Specification-Driven UI

- **SUPPLE** (Harvard, 2004) — UI as optimization from functional specification. Solution spaces 10^17+, <1s generation. Won IUI 2019 Most Impact Paper Award. Key insight: specify needed functionality, let optimization decide presentation.
  - Gajos & Weld, "Automatically Generating User Interfaces" (IUI 2004)
  - "Automatically Generating Personalized UIs with Supple" (Artificial Intelligence Journal, 2010)
- **CAMELEON / MBUID** (W3C) — four-level abstraction: Tasks/Concepts → Abstract UI → Concrete UI → Final UI. Systematic review of 96 papers / 30 tools found rigid models and limited design flexibility.
  - W3C CAMELEON Reference Framework
  - W3C Model-Based UI XG Final Report
- **ConcurTaskTrees** (Paterno, 1997) — task modeling with automatic UI derivation.
- **IFML** (OMG, 2014) — interaction flow modeling standard. Models interaction flow, NOT presentation.
- **USIXML / MARIA** — platform-agnostic UI description languages.
- **teleportHQ UIDL** — JSON-based UI definition with multi-framework code generation.

## AI-Driven UI Generation

- **Google A2UI** (2025) — open protocol (Apache 2.0). Agent sends declarative JSON component tree; client renders with native components. No executable code, security-first. Framework-agnostic (Lit, Angular, Flutter renderers).
- **Google Generative UI** — LLMs produce custom interactive UIs from any prompt. Preferred over markdown by users, comparable to human experts in 50% of cases. Deployed as "Dynamic View" in Gemini.
- **Jelly** (CHI 2025) — LLM analyzes prompt → infers goals → derives sub-tasks → generates "Task-Driven Data Model" (object-relational schema + dependency graph) → composes UI from patterns. Models evolve with user needs.
- **MCP Apps / Open-JSON-UI** — protocols for agents returning interactive UI components. Components bubble up *intents* rather than modifying state directly.
- **AG-UI** — transport layer connecting agents to UI, complementary to A2UI.
- **v0 (Vercel)** — natural language → React/Tailwind. ~6M developers, ~$42M ARR.
- **PrototypeFlow** — divide-and-conquer: NL → component-level DSL + theme module. Produces editable SVG prototypes.
- **SpecifyUI** (2025) — iterative UI design intent through structured specifications + generative AI.
- **AlignUI** (2026) — injects user preference data into AI UI generation.
- **UICoder** — finetuning LLMs for SwiftUI generation from NL.
- **Adaptive UI via RL** (2024) — Markov decision processes dynamically adjust layouts from user feedback.

## Broader Discourse

- "The Next Era of Design is Intent-Driven" (UX Collective)
- "From UI (User Interface) to UI (User Intent)" — 2025 UX trend redefining the acronym
- Headless Business Applications — decouple logic from presentation, generate UI on-demand per context
- Zero UI / Screenless Interfaces — voice, gesture, contextual awareness (Gartner: 70% of customer journeys via conversational AI by 2028)
- Spec-Driven Development (Marc Brooker, 2026) — specifications as primary artifact, AI handles implementation

## Research Gaps

Two gaps remain underexplored:
1. **What representation minimizes generation error for AI while remaining performant and accessible?** Optimizing the *target* for AI generation quality.
2. **Directive-to-GPU-primitive compilation without intermediate widget frameworks.** All existing systems target DOM/Flutter/native. None targets a flat GPU scene buffer directly.
