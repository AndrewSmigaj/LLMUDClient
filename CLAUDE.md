# LLMUD — Project Context

## What This Is

LLMUD is an LLM-powered MUD client and **cognitive scaffolding research platform**. It connects to MUDs via telnet and uses LLMs (local or API) to play, learn, and evolve through structured guidance documents called scaffolds.

This is both a usable tool for MUD players and a research platform for exploring how LLMs develop competence through scaffolded reasoning — with connections to mechanistic interpretability (ConceptMRI) and creativity research (conceptual exploration operators).

## Key Design Decisions

These have been deliberated and decided. Do not re-litigate unless Emily asks. Canonical table with rationale and dates: `INDEX.md` §Key Decisions Made.

- **Language**: Python 3.11+ with asyncio
- **UI**: React frontend extending ConceptMRI, with xterm.js for MUD terminal. Single unified app — local analysis mode + institute mode.
- **Telnet**: telnetlib3 (agent → Evennia connection). Users connect via WebSocket to the React frontend.
- **Model strategy**: gpt-oss-20b local (Em, main agent, free) via ConceptMRI inference server (PyTorch hooks for residual stream capture). Claude Code `claude -p` for offline analysis (subscription tokens).
- **Claude Code as analysis runtime**: `claude -p` reads log files directly from disk. No MCP server needed — Claude Code has filesystem access.
- **ConceptMRI**: Unified into a single application with LLMUD. Same React frontend, same Python backend. ConceptMRI inference server handles Em + PyTorch hooks.
- **Event bus**: Central async pub/sub from Phase 1. All significant events flow through it. Enables UI, logging, triggers, Claude Code analysis.
- **Swarm**: Custom Blackboard-Mediated IFS orchestration (not CrewAI/LangGraph/AutoGen). Build single-agent first, swarm later.
- **Test server**: Self-hosted Evennia MUD instance with tick-based micro-worlds
- **Client is MUD-agnostic**: Agent can connect to any telnet MUD
- **Persistence**: Flat files (git-versioned) for scaffolds/guides/config; SQLite for structured game data (aiosqlite)
- **Scaffold format**: Markdown with YAML frontmatter for cognitive scaffolds; JSON for data scaffolds

## Architecture Philosophy

- **The LLM's training IS the world model.** Don't rebuild what it already knows (keys open doors). Scaffold game-specific knowledge and scene reasoning.
- **LLM-as-planner.** Goals are a text file the LLM reads/writes, not a formal planning engine. Scaffold the PROCESS (how to think about goals) not the CONTENT (what goals to have).
- **Observations → Insights → Guides.** Raw events get logged; patterns get extracted (with data science tools, not just observation); recurring patterns get promoted to structured guides.
- **Eudaimonic motivation, not Sims meters.** The agent's development framework is grounded in virtue ethics and balanced flourishing, not numeric curiosity/boredom counters.
- **Design for swarm, implement single agent.** The architecture supports multi-agent (IFS model, context engineering) but we start with one agent.
- **Scaffold the process, not the content.** Tell the LLM HOW to reason about goals, combat, social situations — not WHAT to do in specific scenarios.

## How to Work on This Project

1. **NO CODING until docs are polished.** Document-first development is a hard rule, not a suggestion.
2. **Don't simplify prematurely.** Emily wants the best possible architecture, not the easiest to build. Dream big in design docs, implement incrementally in code.
3. **Scientific approach.** Scaffolds are hypotheses. Test them, compare them, iterate. The Evennia world supports rewinding/restarting for experimentation.
4. **Check docs/ for context.** The design documents capture extensive brainstorming and should be consulted before making architectural decisions.
5. **ConceptMRI as reference.** When designing modules (especially LLM/analysis/skills), check ConceptMRI for proven patterns. Design fresh — don't copy.

## What to Read When

| Task | Start here | Then |
|------|-----------|------|
| Implementing a backend module | `ARCHITECTURE.md` §Module Structure | `AI_SYSTEM_DESIGN.md` for the design behind it |
| Designing cognitive behavior | `AI_SYSTEM_DESIGN.md` | — |
| Building Evennia content | `WORLD_DESIGN.md` | `ARCHITECTURE.md` §Module Structure (evennia_testworld/) |
| Understanding the vision/motivation | `VISION.md` | `INSTITUTION_DESIGN.md` for the public-facing vision |
| Checking what to build next | `DEV_PROCESS.md` §Phases | `INDEX.md` §Open Design Questions |
| Working with Claude Code on this project | `CLAUDE_CODE_GUIDE.md` | — |
| Looking up a specific requirement | `REQUIREMENTS.md` (search by FR-/NFR- ID) | Source design doc for full spec |
| Checking what was decided and why | `INDEX.md` §Key Decisions Made | — |

## Documentation

- `docs/VISION.md` — Why this exists, the dream, ethics
- `docs/AI_SYSTEM_DESIGN.md` — Cognitive architecture: the loop, scaffolds, memory, tools, research directions
- `docs/ARCHITECTURE.md` — Software architecture: tech stack, module tree, data flow, schemas, WebSocket protocol
- `docs/WORLD_DESIGN.md` — Evennia test world: micro-worlds, scenarios, object/room YAML specs
- `docs/INSTITUTION_DESIGN.md` — Public research institution: social architecture, AI scientists, visualization (vision doc)
- `docs/REQUIREMENTS.md` — Functional and non-functional requirements (FR-/NFR- IDs)
- `docs/INDEX.md` — Document status, open questions, key decisions (canonical), cross-references
- `docs/DEV_PROCESS.md` — Implementation phases, testing, collaboration style
- `docs/CLAUDE_CODE_GUIDE.md` — How to use Claude Code on this project
- `docs/research/` — Pre-decision research artifacts (may reference superseded designs — check main docs for current decisions)

## Conventions

- Python 3.11+, type hints, asyncio for I/O
- Pydantic for data models and config
- pytest + pytest-asyncio for testing
- Module boundaries (under `backend/`): conceptmri, agent, mud, streaming, memory, scaffolds, social, world, reflection, goals, tools, events, api
