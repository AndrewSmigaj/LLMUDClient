# LLMUD — Document Index

## Suggested Reading Order

1. **[../CLAUDE.md](../CLAUDE.md)** — Start here. Project context, key decisions, routing table for task-specific docs.
2. **[VISION.md](VISION.md)** — Why this exists. The dream, scaffolds as the core idea, ethics.
3. **[AI_SYSTEM_DESIGN.md](AI_SYSTEM_DESIGN.md)** — The cognitive architecture. The Loop, scaffolds, memory, tools, infrastructure.
4. **[ARCHITECTURE.md](ARCHITECTURE.md)** — How the design maps to code. Tech stack, modules, schemas, deployment.
5. Then task-specific: [WORLD_DESIGN.md](WORLD_DESIGN.md), [REQUIREMENTS.md](REQUIREMENTS.md), [DEV_PROCESS.md](DEV_PROCESS.md), [INSTITUTION_DESIGN.md](INSTITUTION_DESIGN.md) as needed.

---

## Documents

| Document | Status | Description | Last Updated |
|----------|--------|-------------|-------------|
| [VISION.md](VISION.md) | Draft | Project vision, goals, ethics, scope | 2026-03-28 |
| [AI_SYSTEM_DESIGN.md](AI_SYSTEM_DESIGN.md) | Draft | Cognitive architecture — the core design | 2026-03-28 |
| [REQUIREMENTS.md](REQUIREMENTS.md) | Draft | Functional & non-functional requirements | 2026-03-28 |
| [ARCHITECTURE.md](ARCHITECTURE.md) | Draft | Software architecture, tech stack, modules | 2026-03-28 |
| [WORLD_DESIGN.md](WORLD_DESIGN.md) | Draft | Evennia test world: micro-worlds, scenarios, object/room specs | 2026-03-28 |
| [INSTITUTION_DESIGN.md](INSTITUTION_DESIGN.md) | Vision | Public research institution: social architecture, AI scientists, visualization | 2026-03-28 |
| [CLAUDE_CODE_GUIDE.md](CLAUDE_CODE_GUIDE.md) | Draft | How to use Claude Code for this project | 2026-03-28 |
| [DEV_PROCESS.md](DEV_PROCESS.md) | Draft | Development workflow and practices | 2026-03-28 |
| [../CLAUDE.md](../CLAUDE.md) | Draft | Project context for Claude Code sessions | 2026-03-28 |
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
- [x] ~~Bootstrap scaffolds~~ → Resolved: meta_assess, meta_plan, meta_goals, meta_scene, meta_needs, meta_learning (AI_SYSTEM_DESIGN.md)
- [x] ~~Evennia test world~~ → Resolved: micro-worlds with tick-based rooms, Scenario 001 specified (WORLD_DESIGN.md)
- [ ] Full content of each bootstrap scaffold — needs drafting
- [ ] How does the assess scaffold decide routine vs. deep reasoning?

### Important (inform upcoming phases)
- [ ] Eudaimonic motivation framework — needs its own design session
- [ ] How to operationalize flourishing?
- [ ] Right frequency for periodic goal review?
- [ ] How does the agent recognize when a scaffold is working against it?

### Research (explore over time)
- [x] ~~Open LLMRI integration points~~ → Resolved: ConceptMRI unified into single app (2026-03-28)
- [ ] Can ConceptMRI detect which attractor is active during reasoning?
- [ ] Does scaffold refinement produce measurable representational change?
- [ ] How does personality scaffolding interact with model-level personality?
- [ ] Can we detect maladaptive attractors before observable behavior?
- [ ] Conceptual exploration operators integration
- [ ] Scaffold-to-internal-representation mapping — ConceptMRI A/B comparison enables this

## Key Decisions Made

*This is the canonical decisions table with rationale and dates. CLAUDE.md has a summary loaded every session.*

| Decision | Rationale | Date |
|----------|-----------|------|
| Python + asyncio | Rich LLM ecosystem, same language as Evennia, mature async | 2026-03-23 |
| Unified React frontend (ConceptMRI + LLMUD) | Extend existing ConceptMRI React app with MUD panels; xterm.js for terminal | 2026-03-28 |
| ConceptMRI unified into single app | Same React frontend, same Python backend; inference server handles both | 2026-03-28 |
| ConceptMRI inference server (not Ollama/LMStudio) | Need PyTorch hooks for residual stream capture; Ollama has no internals access | 2026-03-28 |
| Claude Code reads files directly (no MCP) | Filesystem access is sufficient; no server needed for offline reflection | 2026-03-28 |
| LLM-as-planner (not formal BDI) | Trust LLM reasoning, scaffold the process not the content | 2026-03-23 |
| No separate world model | LLM's training is the world model; scaffold game-specific knowledge only | 2026-03-23 |
| Hybrid scaffold format | Markdown+YAML for cognitive, JSON for data | 2026-03-23 |
| Flat files + SQLite persistence | Local-first, portable, human-readable scaffolds, queryable events | 2026-03-23 |
| Eudaimonic motivation (not numeric meters) | Grounded in virtue ethics, balanced flourishing | 2026-03-23 |
| Design for swarm, implement single agent | Context engineering rationale, IFS model, future extensibility | 2026-03-23 |
| Em-OSS-20b via ConceptMRI inference | Primary agent on local GPU, harmony format, residual stream capture | 2026-03-28 |
| Custom swarm orchestration (not framework) | CrewAI/LangGraph/AutoGen/SK evaluated and rejected. Blackboard-mediated IFS. | 2026-03-24 |
| Event bus from Phase 1 | Central async pub/sub. Enables UI, logging, triggers, Claude Code. | 2026-03-24 |
| Tick-based micro-worlds | Evennia test world uses turn-based rooms for reproducible scenario research | 2026-03-28 |

## Scratchpad Documents

Working architecture proposals in `docs/scratchpad/`. See `docs/scratchpad/README.md` for template and rules.

| Document | Status | Target Doc | Created |
|----------|--------|-----------|---------|
| *(none yet)* | | | |

---

## Cross-References

- VISION.md §Ethics → AI_SYSTEM_DESIGN.md §Research Directions (Motivation and Flourishing)
- VISION.md §Connection to Other Research → AI_SYSTEM_DESIGN.md §Interpretability Hooks
- AI_SYSTEM_DESIGN.md §Tools → ARCHITECTURE.md §Module Structure (backend/tools/)
- AI_SYSTEM_DESIGN.md §Scaffolds → ARCHITECTURE.md §Persistence (per-character directory)
- AI_SYSTEM_DESIGN.md §Infrastructure (Event Bus) → ARCHITECTURE.md §Module Structure (backend/events/)
- AI_SYSTEM_DESIGN.md §Research Directions (IFS/Swarm) → research/agent_orchestration.md
- AI_SYSTEM_DESIGN.md §Interpretability Hooks → ARCHITECTURE.md §Data Flow (Live Inference and Streaming)
- REQUIREMENTS.md FR-COG → AI_SYSTEM_DESIGN.md §The Loop
- REQUIREMENTS.md FR-LLM → AI_SYSTEM_DESIGN.md §The Agent, ARCHITECTURE.md §Tech Stack
- REQUIREMENTS.md FR-INT → AI_SYSTEM_DESIGN.md §Interpretability Hooks
- ARCHITECTURE.md §Deployment Architecture → research/ui_research.md
- ARCHITECTURE.md §Dependencies → research/tech_stack_review.md
- WORLD_DESIGN.md §Micro-Worlds → ARCHITECTURE.md §Module Structure (evennia_testworld/)
- WORLD_DESIGN.md §Scenarios → AI_SYSTEM_DESIGN.md §Research Directions (Experimental Design)
- INSTITUTION_DESIGN.md §Visualization Layer → ARCHITECTURE.md §Data Flow (Live Inference)
- DEV_PROCESS.md §Phases → ARCHITECTURE.md §Module Structure
- CLAUDE_CODE_GUIDE.md → research/claude_code_patterns.md
