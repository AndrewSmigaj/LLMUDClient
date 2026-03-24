# LLMUD — Project Context

## What This Is

LLMUD is an LLM-powered MUD client and **cognitive scaffolding research platform**. It connects to MUDs via telnet and uses LLMs (local or API) to play, learn, and evolve through structured guidance documents called scaffolds.

This is both a usable tool for MUD players and a research platform for exploring how LLMs develop competence through scaffolded reasoning — with connections to mechanistic interpretability (ConceptMRI) and creativity research (conceptual exploration operators).

## Key Design Decisions

These have been deliberated and decided. Do not re-litigate unless Emily asks:

- **Language**: Python 3.11+ with asyncio
- **TUI**: Textual (Rich-based terminal UI) for gameplay; FastAPI + SvelteKit web dashboard for research (Phase 3+)
- **Telnet**: telnetlib3 (async, with abstraction layer for swappability)
- **Model strategy**: gpt-oss-20b local ("emily", main agent, free) + smaller local TBD (fast, free) + Claude/GPT API (complex, credits) + Claude Code `claude -p` (offline analysis, subscription tokens)
- **Claude Code as analysis runtime**: `claude -p` subprocess calls use subscription tokens ($100/mo). Powers reflection, memory consolidation, scaffold evaluation. MCP server bridges LLMUD state to Claude Code.
- **ConceptMRI**: Separate repo (`C:\Users\emily\OpenAIHackathon-ConceptMRI\`), shares gpt-oss-20b. LLMUD exports sentence dumps for ConceptMRI replay. Design fresh, use as reference.
- **Event bus**: Central async pub/sub from Phase 1. All significant events flow through it. Enables TUI, logging, triggers, dashboard, Claude Code analysis.
- **Swarm**: Custom Blackboard-Mediated IFS orchestration (not CrewAI/LangGraph/AutoGen). Build single-agent first, swarm later.
- **Test server**: Self-hosted Evennia MUD instance
- **Client is MUD-agnostic**: Works with any telnet MUD
- **Persistence**: Flat files (git-versioned) for scaffolds/guides/config; SQLite for structured game data; DuckDB for analytical queries (Phase 2)
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

## Documentation

- `docs/VISION.md` — Why this exists, the dream, ethics
- `docs/AI_SYSTEM_DESIGN.md` — Cognitive architecture (the big one)
- `docs/REQUIREMENTS.md` — Functional and non-functional requirements
- `docs/ARCHITECTURE.md` — Software architecture, tech stack, modules
- `docs/INDEX.md` — Document status tracker
- `docs/CLAUDE_CODE_GUIDE.md` — How to use Claude Code on this project
- `docs/DEV_PROCESS.md` — Development workflow
- `docs/research/` — Research outputs: claude_code_patterns, tech_stack_review, agent_orchestration, ui_research

## Conventions

- Python 3.11+, type hints, asyncio for I/O
- Pydantic for data models and config
- pytest + pytest-asyncio for testing
- Module boundaries: events, mcp, client, parser, tui, llm, engine, scaffolds, commands, world, memory, social, reflection, knowledge, tools, config
