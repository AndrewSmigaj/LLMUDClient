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

1. Before writing a new module, check that its design is documented in AI_SYSTEM_DESIGN.md or ARCHITECTURE.md
2. If the design doesn't exist or is vague, enter plan mode and flesh it out
3. If implementation reveals the design was wrong, update the doc BEFORE moving on
4. Don't defer doc updates — stale docs are worse than no docs

---

## Implementation Phases

### Phase 1: Core Loop
**Goal:** Connect to Evennia, parse output, load scaffolds, demonstrate auto + approval actions.

Subsystems:
- Telnet connection (client/) — with abstraction layer for telnetlib3 swappability
- Basic TUI (tui/)
- MUD parser — Evennia format (parser/)
- LLM provider — local model support (llm/)
- Scaffold loader (scaffolds/)
- Engine — fast/slow processing (engine/)
- Goal document — basic read/write (goals/)
- Action execution — auto + approval (engine/actions.py)
- Slash commands — /ask, /suggest, /auto, /status, /scaffold (commands/)
- Bootstrap scaffolds — meta-goals, meta-scene, meta-needs (scaffolds/defaults/)
- **Event bus** — central async pub/sub (events/) — build from Phase 1, enables everything
- **MCP server skeleton** — expose basic game state (mcp/)
- **LLM call logging** — full prompt/response/scaffold versions for interpretability (FR-INT-02)
- Evennia test world — basic rooms, NPC, combat encounter

**Exit criteria:** Agent connects to Evennia, parses room descriptions, loads a combat scaffold, suggests an action when entering a room with an NPC, and the user can approve/reject it. All significant events flow through the event bus.

### Phase 2: Memory & Reflection
**Goal:** The agent remembers things between sessions and learns from experience.

Subsystems:
- Episodic memory — SQLite storage (memory/)
- Semantic memory — knowledge base (memory/)
- remember() tool (tools/)
- Session journaling (reflection/)
- Reflection scaffolds — narrative, analytical, strategic (reflection/)
- Memory consolidation (memory/)
- Observation auto-logging (knowledge/)
- **ClaudeCodeProvider** — `claude -p` subprocess wrapper (llm/claude_code.py)
- **Claude Code reflection skills** — /reflect, /memory-review, /scaffold-eval
- **MCP server expansion** — expose memories, scaffolds, session logs
- **Sentence dump export** — for ConceptMRI replay (FR-INT-01)
- **DuckDB** — analytical queries on event/LLM log data

**Exit criteria:** After a play session, run offline reflection via Claude Code. The agent extracts insights, proposes guide updates, and the next session starts with accumulated knowledge.

### Phase 3: Knowledge & World
**Goal:** The agent builds a rich understanding of the game world.

Subsystems:
- Auto-mapping (world/)
- NPC profiles (social/)
- Command reference (world/)
- Pathfinding (world/)
- analyze_events tool (knowledge/)
- Insight extraction pipeline (knowledge/)
- Guide auto-creation from insights (scaffolds/)

**Exit criteria:** After 5 sessions, the agent has a map, knows NPC personalities, has a price database, and has created its own tactical guides from experience.

### Phase 4: Social & Personality
**Goal:** The agent handles social situations with depth.

Subsystems:
- Personality scaffolding (social/)
- Conversation strategy engine (social/)
- Empathy modeling (social/)
- Groklet formation (social/)
- Social perception chain (social/)

**Exit criteria:** The agent can complete the birthday gift quest (social puzzle) by reading emotional cues, connecting environmental clues, and choosing appropriate social actions.

### Phase 5: Autonomy & Advancement
**Goal:** The agent operates autonomously with the user supervising.

Subsystems:
- Full autonomy mode
- LLM-authored scripts (sandboxed)
- Advanced scaffolding — scaffold meta-notes, self-critique
- REM scheduling (automated offline analysis)
- Eudaimonic motivation (to the extent designed)
- Conceptual exploration operator integration

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

When designing a module (especially LLM integration, skills, analysis pipelines), check ConceptMRI's equivalent implementation at `C:\Users\emily\OpenAIHackathon-ConceptMRI\` for proven patterns. LLMUD designs fresh — don't copy, but learn from what worked. Emily can then compare how each project's approach evolves.

### Research Documents

Four research documents in `docs/research/` provide deep analysis of specific topics:
- `claude_code_patterns.md` — Claude Code integration patterns and LLMUD-specific recommendations
- `tech_stack_review.md` — Library evaluation, risk assessment, version strategy
- `agent_orchestration.md` — Multi-agent framework evaluation and custom IFS architecture
- `ui_research.md` — MUD client, agent dashboard, and hybrid UI patterns

Consult these when working on related areas. They informed the design decisions now captured in the main docs.
