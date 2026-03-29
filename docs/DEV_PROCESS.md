# LLMUD — Development Process

How we build this project. These are living guidelines, not rigid rules.

---

## Core Principle: Document-First Development

**NO CODING until all design documents are polished.** This is not a suggestion — it's a hard rule. The design documents must be complete, consistent, and reviewed before any implementation begins.

Every significant feature follows: **Design → Document → Implement → Test → Document**

Why:
- This project is a research platform, not a startup MVP. Design quality matters more than speed.
- Scaffolding is the core innovation — sloppy design of the scaffold system undermines the whole project.
- Documentation serves double duty: it guides implementation AND serves as research documentation.
- Future Claude sessions need context — docs provide it.
- Premature coding locks in decisions that haven't been fully thought through.

### What "Document First" Means in Practice

1. Before writing a new module, check that its design is documented in AI_SYSTEM_DESIGN.md, ARCHITECTURE.md, or WORLD_DESIGN.md
2. If the design doesn't exist or is vague, enter plan mode and flesh it out
3. If implementation reveals the design was wrong, update the doc BEFORE moving on
4. Don't defer doc updates — stale docs are worse than no docs

---

## Implementation Phases

### Phase 1: Core Loop
**Goal:** Connect to Evennia, parse output, load scaffolds, run the Assess → Plan → Act loop with live ConceptMRI visualization.

Subsystems:
- Telnet connection to Evennia (backend/mud/connection.py)
- MUD parser — Evennia format (backend/mud/parser.py, evennia_parser.py)
- Agent loop: assess → plan → act (backend/agent/)
- Context assembly (backend/agent/context.py)
- Scaffold loader and registry (backend/scaffolds/)
- Bootstrap scaffolds — meta_assess, meta_plan, meta_goals, meta_scene, meta_needs, meta_learning (backend/scaffolds/defaults/)
- Goal document — read/write with file locking (backend/goals/)
- LLM tool definitions (backend/tools/)
- **ConceptMRI inference** — Em via PyTorch hooks, harmony parsing, coordinate generation (backend/conceptmri/)
- **Event bus** — central async pub/sub (backend/events/) — build from Phase 1
- **LLM call logging** — full prompt, harmony channels, coordinates, scaffold versions (FR-INT-01)
- **FastAPI + WebSocket** — streams MUD output, analysis channel, coordinates to React frontend (backend/api/, backend/streaming/)
- **React frontend** — MUD panel (xterm.js), analysis panel, agent controls, scaffold browser, ConceptMRI viz (frontend/)
- Evennia test world — Meadow micro-world, Scenario 001 (Dandelion Seeds) (evennia_testworld/)

**Exit criteria:** Agent connects to Evennia, runs Scenario 001, analysis channel and coordinates stream to React frontend in real time. ConceptMRI visualizations update live. All events flow through the event bus.

### Phase 2: Memory & Reflection
**Goal:** The agent remembers things between sessions and learns from experience.

Subsystems:
- Episodic memory — aiosqlite storage (backend/memory/episodic.py)
- Semantic memory — knowledge base (backend/memory/semantic.py)
- remember() tool (backend/tools/memory_tools.py)
- Session journaling (backend/reflection/journal.py)
- Reflection scaffolds — narrative, analytical, strategic, meta-review (backend/reflection/)
- Memory consolidation during REM
- Session export for Claude Code (backend/reflection/exporter.py)
- **Claude Code offline reflection** — `claude -p` reads exports from disk, writes proposals to proposals/
- **Proposal review** — frontend ProposalReview component, approve/reject/edit via WebSocket
- **Institute mode** — room entry triggers viz context switch, fitted manifold download/cache

**Exit criteria:** After a play session, run offline reflection via Claude Code. The agent extracts insights, proposes scaffold updates staged in proposals/. Human reviews in the React frontend. Next session starts with accumulated knowledge.

### Phase 3: Knowledge & World
**Goal:** The agent builds a rich understanding of the game world.

Subsystems:
- Auto-mapping (backend/world/map_graph.py)
- NPC profiles (backend/social/profiles.py)
- Command reference data scaffolds
- Pathfinding on room graph (backend/world/)
- analyze_events tool (backend/tools/)
- Insight extraction pipeline
- Scaffold auto-creation from insights (data scaffolds only; cognitive scaffolds require human review)

**Exit criteria:** After 5 sessions, the agent has a map, knows NPC personalities, has a price database, and has created its own data scaffolds from experience.

### Phase 4: Social & Personality
**Goal:** The agent handles social situations with depth.

Subsystems:
- Personality scaffolding (backend/social/)
- Conversation strategy selection (backend/social/perception.py)
- Empathy modeling
- Groklet formation
- Multi-perspective review (already in REM, validates multi-viewpoint reasoning)

**Exit criteria:** The agent can navigate Scenario 004 (The Distressed NPC) — reading emotional cues, distinguishing genuine distress from manipulation, choosing appropriate social actions.

### Phase 5: Autonomy & Advancement
**Goal:** The agent operates autonomously with the user supervising.

Subsystems:
- Full autonomy mode
- Advanced scaffolding — scaffold meta-notes, self-critique
- REM scheduling (automated offline analysis)
- Eudaimonic motivation (to the extent designed)
- Swarm / IFS — validate with multi-perspective prompting first, then build infrastructure
- Conceptual exploration operator integration
- MUD creator panel (host-only, frontend/src/components/mud/MUDCreator.tsx)

**Exit criteria:** The agent can play for 30 minutes autonomously, managing goals, surviving, completing quests, and learning — with the user able to observe and intervene at any time.

---

## Working with Claude Code

See [CLAUDE_CODE_GUIDE.md](CLAUDE_CODE_GUIDE.md) for detailed usage.

Summary:
- Use `/plan` for design work
- Let Claude create task lists for multi-step implementation
- Commit frequently with descriptive messages
- Update docs when implementation reveals design changes
- Review agents can check code against design docs

---

## Decision Tracking

Every significant design decision gets recorded in `docs/INDEX.md` under "Key Decisions Made" with:
- What was decided
- Why (rationale)
- When
- What alternatives were considered (briefly)

This prevents re-litigating decisions in future sessions while preserving the reasoning for review.

---

## Testing Approach

### Code Tests
Standard pytest. Write tests alongside code, not after.
- Unit tests for parsers, memory, scaffold loading
- Integration tests for the engine loop
- Mock LLM responses for deterministic testing

### Scaffold Tests
A scaffold test is: run the same scenario with different scaffolds, compare outcomes.
- Use the Evennia test world for reproducible scenarios
- Metrics: quest completion, deaths, time, HP remaining, knowledge gained
- Document results in scaffold meta-notes

### Scenario Replay
The Evennia world should support:
- Resetting quest state (for re-running the same scenario)
- Scripted NPC behavior (deterministic for testing)
- Logging game events for analysis

---

## Collaboration Style

### Emily's Preferences
- Wants the best design, not the simplest
- Values theoretical depth — don't hand-wave over hard problems
- Thinks in terms of gradients, exploration operators, virtue ethics
- Prefers to discuss before deciding, then decide firmly
- Cross-pollinates between LLMUD, ConceptMRI, and creativity research

### How to Engage
- Challenge ideas, don't just agree
- Present the ambitious option first, simple fallback second
- Walk through scenarios to stress-test designs
- Be honest about confidence levels
- Flag when something is pattern-matching vs. genuinely understood

### ConceptMRI as Reference

When designing a module (especially LLM integration, skills, analysis pipelines), check ConceptMRI's equivalent implementation in `backend/conceptmri/` (or the original repo at `C:\Users\emily\OpenAIHackathon-ConceptMRI\`) for proven patterns. LLMUD designs fresh — don't copy, but learn from what worked. Emily can then compare how each project's approach evolves.

### Research Documents

Four research documents in `docs/research/` provide deep analysis of specific topics:
- `claude_code_patterns.md` — Claude Code integration patterns and LLMUD-specific recommendations
- `tech_stack_review.md` — Library evaluation, risk assessment, version strategy
- `agent_orchestration.md` — Multi-agent framework evaluation and custom IFS architecture
- `ui_research.md` — MUD client, agent dashboard, and hybrid UI patterns

Consult these when working on related areas. They informed the design decisions now captured in the main docs.
