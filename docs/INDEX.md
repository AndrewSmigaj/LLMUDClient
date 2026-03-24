# LLMUD — Document Index

## Documents

| Document | Status | Description | Last Updated |
|----------|--------|-------------|-------------|
| [VISION.md](VISION.md) | Draft | Project vision, goals, ethics, scope | 2026-03-23 |
| [AI_SYSTEM_DESIGN.md](AI_SYSTEM_DESIGN.md) | Draft | Cognitive architecture — the core design | 2026-03-23 |
| [REQUIREMENTS.md](REQUIREMENTS.md) | Draft | Functional & non-functional requirements | 2026-03-23 |
| [ARCHITECTURE.md](ARCHITECTURE.md) | Draft | Software architecture, tech stack, modules | 2026-03-23 |
| [CLAUDE_CODE_GUIDE.md](CLAUDE_CODE_GUIDE.md) | Draft | How to use Claude Code for this project | 2026-03-23 |
| [DEV_PROCESS.md](DEV_PROCESS.md) | Draft | Development workflow and practices | 2026-03-23 |
| [../CLAUDE.md](../CLAUDE.md) | Draft | Project context for Claude Code sessions | 2026-03-23 |
| [research/claude_code_patterns.md](research/claude_code_patterns.md) | Complete | Claude Code integration patterns and recommendations | 2026-03-24 |
| [research/tech_stack_review.md](research/tech_stack_review.md) | Complete | Library evaluation, risk assessment, version strategy | 2026-03-24 |
| [research/agent_orchestration.md](research/agent_orchestration.md) | Complete | Multi-agent framework evaluation and custom IFS design | 2026-03-24 |
| [research/ui_research.md](research/ui_research.md) | Complete | MUD client, agent dashboard, hybrid UI patterns | 2026-03-24 |

## Status Definitions

- **Draft** — Initial content, actively being iterated
- **Review** — Content is substantial, needs review pass for consistency
- **Stable** — Reviewed, consistent, safe to implement against (still can change)

## Open Design Questions

### Critical (block Phase 1 implementation)
- [ ] Goal document format — is plain markdown enough or do we need lightweight structure?
- [ ] Fast/slow handoff — start with regex rules? What patterns to match?
- [ ] Bootstrap scaffolds — what ships with the system?
- [ ] Evennia test world — room layout, NPCs, basic quests for testing

### Important (inform upcoming phases)
- [ ] Eudaimonic motivation framework — needs its own design session
- [ ] Data science tools for analyze_events — which methods, what interface?
- [ ] Context overflow strategy — what gets dropped when everything is relevant?
- [ ] Social intelligence depth — how much to implement vs. defer?

### Research (explore over time)
- [ ] Limbic system analogy — curiosity/boredom neuroscience
- [ ] Conceptual exploration operators integration
- [x] ~~Open LLMRI integration points~~ → Resolved: ConceptMRI replay via sentence dump export (2026-03-24)
- [ ] Scaffold-to-internal-representation mapping — ConceptMRI A/B comparison enables this

## Key Decisions Made

| Decision | Rationale | Date |
|----------|-----------|------|
| Python + asyncio | Rich LLM ecosystem, same language as Evennia, mature async | 2026-03-23 |
| Textual for TUI | Modern, async, Rich-based, split panes | 2026-03-23 |
| LLM-as-planner (not formal BDI) | Trust LLM reasoning, scaffold the process not the content | 2026-03-23 |
| No separate world model | LLM's training is the world model; scaffold game-specific knowledge only | 2026-03-23 |
| Hybrid scaffold format | Markdown+YAML for cognitive, JSON for data | 2026-03-23 |
| Flat files + SQLite persistence | Local-first, portable, human-readable scaffolds, queryable events | 2026-03-23 |
| Eudaimonic motivation (not numeric meters) | Grounded in virtue ethics, balanced flourishing | 2026-03-23 |
| Design for swarm, implement single agent | Context engineering rationale, IFS model, future extensibility | 2026-03-23 |
| Custom provider abstraction (not LiteLLM) | Need per-task routing with reasoning levels | 2026-03-23 |
| Separate repos, ConceptMRI as reference | LLMUD designs fresh, ConceptMRI for reference. Compare how each evolves. | 2026-03-24 |
| Claude Code as analysis runtime (`claude -p`) | Subscription tokens for offline analysis (reflection, memory, scaffolds). Saves API credits. | 2026-03-24 |
| Model strategy: local 20B + smaller + API + CC | gpt-oss-20b main, smaller TBD fast, API for complex, Claude Code for analysis | 2026-03-24 |
| Custom swarm orchestration (not framework) | CrewAI/LangGraph/AutoGen/SK evaluated and rejected. Blackboard-mediated IFS. | 2026-03-24 |
| Hybrid UI: Textual TUI + web dashboard | TUI for gameplay (Phase 1), FastAPI+SvelteKit for research mgmt (Phase 3+) | 2026-03-24 |
| Event bus from Phase 1 | Central async pub/sub. Enables TUI, logging, triggers, dashboard, Claude Code. | 2026-03-24 |
| DuckDB for analytical queries (Phase 2) | OLAP complement to SQLite for event/LLM log analysis | 2026-03-24 |

## Cross-References

- VISION.md §Ethics → AI_SYSTEM_DESIGN.md §8 (Eudaimonic Motivation)
- VISION.md §Connection to Other Research → AI_SYSTEM_DESIGN.md §15 (Interpretability Hooks)
- AI_SYSTEM_DESIGN.md §11 (Tool Interface) → ARCHITECTURE.md §Module Structure (tools/)
- AI_SYSTEM_DESIGN.md §10 (Scaffolds) → ARCHITECTURE.md §Persistence (per-character directory)
- AI_SYSTEM_DESIGN.md §13 (Event Bus) → ARCHITECTURE.md §Module Structure (events/)
- AI_SYSTEM_DESIGN.md §14 (Swarm) → research/agent_orchestration.md
- AI_SYSTEM_DESIGN.md §15 (Interpretability) → ConceptMRI (`C:\Users\emily\OpenAIHackathon-ConceptMRI\`)
- REQUIREMENTS.md FR-COG → AI_SYSTEM_DESIGN.md §2 (Dual-Process)
- REQUIREMENTS.md FR-LLM → ARCHITECTURE.md §LLM Provider Architecture
- REQUIREMENTS.md FR-INT → AI_SYSTEM_DESIGN.md §15 (Interpretability Hooks)
- ARCHITECTURE.md §UI Architecture → research/ui_research.md
- ARCHITECTURE.md §Dependency Version Strategy → research/tech_stack_review.md Appendix C
- DEV_PROCESS.md §Phase 1-2 → ARCHITECTURE.md §Module Structure (events/, mcp/)
- CLAUDE_CODE_GUIDE.md → research/claude_code_patterns.md
