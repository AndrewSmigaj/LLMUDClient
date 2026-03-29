# LLMUD — Requirements

**Status:** Draft. Requirements derived from AI System Design and scenario walkthroughs. Will evolve as design iterates.

---

## Phase 1 Must Requirements (Quick Reference)

FR-NET-01, FR-NET-02, FR-NET-04, FR-NET-05, FR-NET-06, FR-UI-01, FR-UI-02, FR-UI-03, FR-UI-04, FR-PARSE-01, FR-PARSE-02, FR-PARSE-03, FR-PARSE-04, FR-PARSE-05, FR-LLM-01, FR-LLM-02, FR-LLM-03, FR-LLM-04, FR-LLM-05, FR-COG-01, FR-COG-02, FR-COG-03, FR-COG-04, FR-COG-06, FR-GOAL-01, FR-GOAL-02, FR-GOAL-03, FR-MEM-01, FR-MEM-02, FR-MEM-03, FR-MEM-04, FR-MEM-05, FR-KNOW-01, FR-SCENE-01, FR-SCENE-02, FR-SCAFFOLD-01, FR-SCAFFOLD-02, FR-SCAFFOLD-03, FR-SCAFFOLD-04, FR-CMD-01, FR-CMD-02, FR-CMD-03, FR-CMD-04, FR-CMD-05, FR-ACTION-01, FR-ACTION-02, FR-ACTION-04, FR-INT-01, FR-INT-02, FR-INT-03

*Source: AI_SYSTEM_DESIGN.md §The Loop, §Scaffolds, §Tools, §Infrastructure; ARCHITECTURE.md §Module Structure*

---

## Functional Requirements

### FR-NET: Networking & Connection

| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-NET-01 | Agent connects to Evennia via telnet (telnetlib3, async) | Must | 1 |
| FR-NET-02 | Handle telnet protocol negotiation (IAC, NAWS, etc.) | Must | 1 |
| FR-NET-03 | Support GMCP protocol for structured game data | Should | 2 |
| FR-NET-04 | Handle connection drops and reconnection with backoff | Must | 1 |
| FR-NET-05 | Users connect via WebSocket to React frontend (single multiplexed connection) | Must | 1 |
| FR-NET-06 | WebSocket streams: MUD output, analysis channel, coordinates, room context, agent status | Must | 1 |
| FR-NET-07 | Coordinate relay on server broadcasts residual stream coordinates to clients | Should | 2 |

### FR-UI: User Interface (React + xterm.js)

| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-UI-01 | MUD terminal panel (xterm.js) with ANSI color rendering | Must | 1 |
| FR-UI-02 | Analysis channel panel showing agent chain-of-thought | Must | 1 |
| FR-UI-03 | Agent controls: start/stop, autonomy level, reasoning effort | Must | 1 |
| FR-UI-04 | Scaffold browser: list, view, edit scaffolds | Must | 1 |
| FR-UI-05 | Goal viewer showing active goals document | Should | 1 |
| FR-UI-06 | Proposal review panel: approve/reject/edit REM proposals | Should | 2 |
| FR-UI-07 | ConceptMRI visualization panels (existing, extended with live feeds) | Must | 1 |
| FR-UI-08 | Local analysis mode: load datasets, run probes, visualize (existing ConceptMRI) | Must | 1 |
| FR-UI-09 | Institute mode: connect to Evennia, room entry triggers viz context switch | Should | 2 |
| FR-UI-10 | MUD creator panel (host only, disabled by config) | Could | 3 |

### FR-PARSE: MUD Output Parsing

| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-PARSE-01 | Classify output types: room_desc, combat, npc_speech, system_msg, etc. | Must | 1 |
| FR-PARSE-02 | Extract structured data from room descriptions (name, exits, NPCs, items) | Must | 1 |
| FR-PARSE-03 | Extract HP/mana/stats from prompt or GMCP | Must | 1 |
| FR-PARSE-04 | Batch multi-line output into logical units before processing | Must | 1 |
| FR-PARSE-05 | Support Evennia-specific output format | Must | 1 |
| FR-PARSE-06 | Pluggable parser architecture for different MUD types | Should | 2 |
| FR-PARSE-07 | Strip/interpret ANSI codes for semantic meaning | Should | 2 |

### FR-LLM: LLM Integration

| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-LLM-01 | Em-OSS-20b via ConceptMRI inference server (PyTorch hooks for residual stream capture) | Must | 1 |
| FR-LLM-02 | Harmony response format: analysis, commentary, final channels parsed separately | Must | 1 |
| FR-LLM-03 | Configurable reasoning effort (low/medium/high) via system prompt | Must | 1 |
| FR-LLM-04 | Per-task reasoning routing: assess (low), plan+act routine (medium), plan+act complex (high) | Must | 1 |
| FR-LLM-05 | Native tool calling and structured output | Must | 1 |
| FR-LLM-06 | Fallback to Ollama if ConceptMRI server unreachable (no coordinates, alert user) | Should | 2 |
| FR-LLM-07 | Token usage tracking and reporting | Should | 2 |
| FR-LLM-08 | Claude Code offline reflection: `claude -p` reads log files directly from disk | Must | 2 |
| FR-LLM-09 | Future upgrade path: gpt-oss-120b or API model for reasoning role | Could | 3 |

### FR-COG: Cognitive Architecture

| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-COG-01 | Four-phase loop: Assess → Plan → Act → Learn | Must | 1 |
| FR-COG-02 | Assess scaffold determines reasoning depth: routine vs. complex | Must | 1 |
| FR-COG-03 | Context assembly: system prompt + game state + goals + scaffolds + memories + tools | Must | 1 |
| FR-COG-04 | Scaffold-driven behavior: each phase guided by loaded scaffolds | Must | 1 |
| FR-COG-05 | Context priority ordering with defined drop policy (least-relevant scaffolds first, oldest memories first) | Should | 2 |
| FR-COG-06 | Event bus: central async pub/sub for all significant system events (Pydantic models, logged to SQLite) | Must | 1 |

### FR-GOAL: Goal Management

| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-GOAL-01 | Persistent goal document (text file) readable/writable by LLM | Must | 1 |
| FR-GOAL-02 | Goal process scaffold guiding prioritization and decomposition | Must | 1 |
| FR-GOAL-03 | Stuckness detection and recovery guidance | Must | 1 |
| FR-GOAL-04 | Goal document displayed in UI (GoalViewer component) | Should | 1 |
| FR-GOAL-05 | Goal history tracking (completed goals log) | Should | 2 |

### FR-MEM: Memory System

| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-MEM-01 | Episodic memory: timestamped events in SQLite | Must | 1 |
| FR-MEM-02 | Semantic memory: facts and relationships (JSON + SQLite) | Must | 1 |
| FR-MEM-03 | Procedural memory: guide/scaffold library (markdown files) | Must | 1 |
| FR-MEM-04 | remember(query) tool: search across all memory stores | Must | 1 |
| FR-MEM-05 | learn(category, data) tool: store new knowledge | Must | 1 |
| FR-MEM-06 | Memory consolidation during REM/offline | Should | 2 |
| FR-MEM-07 | Vector similarity search (RAG) for fuzzy retrieval | Could | 4 |

### FR-KNOW: Knowledge & Analysis

| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-KNOW-01 | Observation logging (auto-record events during gameplay) | Must | 1 |
| FR-KNOW-02 | Insight extraction during reflection | Should | 2 |
| FR-KNOW-03 | Guide creation from insights | Should | 2 |
| FR-KNOW-04 | analyze_events tool with statistical methods | Should | 2 |
| FR-KNOW-05 | Data science tools accessible to LLM (correlations, feature importance, trends) | Could | 3 |

### FR-SCENE: Scene Reasoning & Game Knowledge

| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-SCENE-01 | Command reference scaffold (valid commands and syntax) | Must | 1 |
| FR-SCENE-02 | Scene reasoning scaffold (how to process new rooms/situations) | Must | 1 |
| FR-SCENE-03 | Auto-map: build room graph from exploration | Should | 2 |
| FR-SCENE-04 | Pathfinding on room graph | Should | 3 |
| FR-SCENE-05 | Game manual integration (load help system knowledge) | Should | 2 |

### FR-REFL: Reflection & REM

| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-REFL-01 | Session journaling (auto-generate session summary) | Should | 2 |
| FR-REFL-02 | Narrative review scaffold | Should | 2 |
| FR-REFL-03 | Analytical review scaffold | Should | 2 |
| FR-REFL-04 | Strategic review scaffold | Should | 2 |
| FR-REFL-05 | Meta-review scaffold (review the scaffolds themselves) | Should | 2 |
| FR-REFL-06 | Multi-perspective review (MAR-style) | Could | 3 |
| FR-REFL-07 | Automated REM scheduling (daily/weekly offline analysis) | Could | 4 |
| FR-REFL-08 | Reflection via Claude Code skills (manual invocation first, UI trigger later) | Should | 2 |

### FR-SOCIAL: Social Intelligence

| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-SOCIAL-01 | NPC/player profile storage and retrieval | Should | 2 |
| FR-SOCIAL-02 | Emotional state detection from dialogue | Should | 2 |
| FR-SOCIAL-03 | Conversation strategy selection | Should | 3 |
| FR-SOCIAL-04 | Personality scaffolding for social behavior | Should | 3 |
| FR-SOCIAL-05 | Empathy modeling (mirroring, variable strength) | Could | 4 |
| FR-SOCIAL-06 | Groklet formation (subjective impressions of others) | Could | 4 |

### FR-SCAFFOLD: Scaffold System

| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-SCAFFOLD-01 | Load scaffolds from markdown+YAML files | Must | 1 |
| FR-SCAFFOLD-02 | Load data scaffolds from JSON files | Must | 1 |
| FR-SCAFFOLD-03 | search_scaffolds, list_scaffolds, read_scaffold, create_scaffold, update_scaffold tools | Must | 1 |
| FR-SCAFFOLD-04 | Trigger-based scaffold loading via frontmatter triggers (auto-load relevant scaffolds) | Must | 1 |
| FR-SCAFFOLD-05 | Scaffold versioning and change tracking | Should | 2 |
| FR-SCAFFOLD-06 | Scaffold effectiveness metadata | Should | 3 |
| FR-SCAFFOLD-07 | Conceptual exploration operator integration | Could | 5 |

### FR-CMD: Agent Interaction Commands

These are interface-agnostic agent commands — they may be invoked via UI buttons, API endpoints, or text input depending on the interface context.

| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-CMD-01 | ask — query the LLM about current state | Must | 1 |
| FR-CMD-02 | suggest — request action suggestion | Must | 1 |
| FR-CMD-03 | auto — toggle autonomous mode | Must | 1 |
| FR-CMD-04 | status — show engine state, active scaffolds, model | Must | 1 |
| FR-CMD-05 | scaffold — list/view/edit scaffolds | Must | 1 |
| FR-CMD-06 | goal — review/manage goals | Should | 1 |
| FR-CMD-07 | map — show current map | Should | 3 |
| FR-CMD-08 | review — start reflection session | Should | 2 |
| FR-CMD-09 | journal — view session journal | Should | 2 |
| FR-CMD-10 | model — change model assignment | Should | 2 |
| FR-CMD-11 | Extensible command registry for plugins | Could | 3 |

### FR-ACTION: Action Control

| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-ACTION-01 | Automatic actions (execute without approval) | Must | 1 |
| FR-ACTION-02 | Approval-required actions (accept/reject/retry/edit) | Must | 1 |
| FR-ACTION-03 | Configurable action approval levels per situation | Should | 2 |
| FR-ACTION-04 | Action logging (every action + outcome) | Must | 1 |
| FR-ACTION-05 | Autonomy spectrum (spectator/advisor/assistant/autonomous) | Should | 3 |

### FR-INT: Interpretability & Analysis

| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-INT-01 | Full LLM call logging: prompt, all three harmony channels, scaffold versions, coordinates, outcome, latency, tokens | Must | 1 |
| FR-INT-02 | Residual stream coordinates captured via ConceptMRI inference server on every call | Must | 1 |
| FR-INT-03 | Analysis channel stored as `response_analysis` — primary signal for attractor conflict research | Must | 1 |
| FR-INT-04 | Live coordinate streaming to React frontend for real-time UMAP visualization | Should | 2 |
| FR-INT-05 | Session export for Claude Code reflection (session_log.json, episodes.json, scaffold_stats.json) | Should | 2 |
| FR-INT-06 | Event log replay capability for offline analysis | Should | 2 |

---

## Non-Functional Requirements

### NFR-PERF: Performance

| ID | Requirement | Priority |
|----|------------|----------|
| NFR-PERF-01 | Assess phase (low reasoning effort) completes within 2 seconds | Must |
| NFR-PERF-02 | Agent reasoning state visible in UI via WebSocket (phase + reasoning effort) without blocking user input | Must |
| NFR-PERF-03 | Telnet connection handles high-throughput MUD output without dropping data | Must |
| NFR-PERF-04 | Memory retrieval responds within 1 second for keyword search | Should |

### NFR-EXT: Extensibility

| ID | Requirement | Priority |
|----|------------|----------|
| NFR-EXT-01 | Pluggable parser architecture (add parsers for new MUD types) | Should |
| NFR-EXT-02 | Pluggable LLM provider architecture | Must |
| NFR-EXT-03 | Scaffold system supports custom scaffold types | Should |
| NFR-EXT-04 | Command system supports custom commands | Should |
| NFR-EXT-05 | Architecture supports future multi-agent without major refactor | Must |

### NFR-PORT: Portability

| ID | Requirement | Priority |
|----|------------|----------|
| NFR-PORT-01 | Runs on Linux, macOS, Windows (WSL) | Must |
| NFR-PORT-02 | No external service dependencies for core functionality (local-first) | Must |
| NFR-PORT-03 | Character data (scaffolds, memory, config) is portable (flat files + SQLite) | Must |
| NFR-PORT-04 | Scaffold format is human-readable and editable with any text editor | Must |

### NFR-SEC: Security

| ID | Requirement | Priority |
|----|------------|----------|
| NFR-SEC-01 | LLM-authored scripts run in sandbox with restricted permissions | Must |
| NFR-SEC-02 | API keys stored securely (not in scaffold files) | Must |
| NFR-SEC-03 | No arbitrary code execution from MUD output | Must |

### NFR-OBS: Observability

| ID | Requirement | Priority |
|----|------------|----------|
| NFR-OBS-01 | Full prompt/response logging for interpretability analysis | Must |
| NFR-OBS-02 | Action audit trail (what was decided, why, what happened) | Must |
| NFR-OBS-03 | Token usage tracking per session | Should |
| NFR-OBS-04 | Export logs in structured format (JSON) for ConceptMRI replay and analysis | Should |

### NFR-UX: User Experience

| ID | Requirement | Priority |
|----|------------|----------|
| NFR-UX-01 | User can always type MUD commands directly, regardless of AI state | Must |
| NFR-UX-02 | AI suggestions never block user input | Must |
| NFR-UX-03 | Clear indication of what the AI is doing/thinking | Should |
| NFR-UX-04 | Easy to disable AI entirely (pure telnet mode) | Should |

### NFR-COST: Cost Management

| ID | Requirement | Priority |
|----|------------|----------|
| NFR-COST-01 | Offline analysis uses Claude Code subscription tokens, not API credits | Must |
| NFR-COST-02 | Routine gameplay uses local models (free), API only for complex reasoning | Must |
| NFR-COST-03 | Token usage tracking per model provider to monitor spend | Should |
