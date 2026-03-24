# LLMUD — Requirements

**Status:** Draft. Requirements derived from AI System Design and scenario walkthroughs. Will evolve as design iterates.

---

## Functional Requirements

### FR-NET: Networking & Connection

| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-NET-01 | Connect to any MUD via telnet (host:port) | Must | 1 |
| FR-NET-02 | Handle telnet protocol negotiation (IAC, NAWS, etc.) | Must | 1 |
| FR-NET-03 | Support GMCP protocol for structured game data | Should | 2 |
| FR-NET-04 | Support MSDP protocol for structured game data | Could | 3 |
| FR-NET-05 | Handle connection drops and reconnection | Must | 1 |
| FR-NET-06 | Support SSL/TLS connections | Should | 2 |

### FR-TUI: Terminal User Interface

| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-TUI-01 | Three-pane layout: MUD output, command input, LLM panel | Must | 1 |
| FR-TUI-02 | MUD output pane with scrollback buffer | Must | 1 |
| FR-TUI-03 | Command input with history (up/down arrow) | Must | 1 |
| FR-TUI-04 | LLM panel showing: current state, active goals, suggestions, reasoning | Must | 1 |
| FR-TUI-05 | Approval dialog: accept/reject/retry/edit for LLM actions | Must | 1 |
| FR-TUI-06 | ANSI color rendering in MUD output | Must | 1 |
| FR-TUI-07 | Slash command support in input (/ask, /suggest, etc.) | Must | 1 |
| FR-TUI-08 | Map visualization pane (ASCII or graphical) | Should | 3 |
| FR-TUI-09 | Resizable/configurable pane layout | Could | 3 |
| FR-TUI-10 | Status bar showing connection state, HP, active mode | Should | 1 |

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
| FR-LLM-01 | Support local models via OpenAI-compatible API (Ollama, LMStudio) | Must | 1 |
| FR-LLM-02 | Support Anthropic Claude API | Should | 1 |
| FR-LLM-03 | Support OpenAI API | Should | 2 |
| FR-LLM-04 | Per-task model routing (different model/reasoning for perception vs. tactics vs. RP) | Must | 1 |
| FR-LLM-05 | Custom grammar support for local models (GBNF/JSON schema constraints) | Should | 2 |
| FR-LLM-06 | Native tool calling for API models | Must | 1 |
| FR-LLM-07 | Streaming response support | Should | 2 |
| FR-LLM-08 | Token usage tracking and reporting | Should | 2 |
| FR-LLM-09 | Fallback provider if primary is unavailable | Could | 3 |
| FR-LLM-10 | ClaudeCodeProvider: `claude -p` subprocess wrapper for offline analysis (subscription tokens, not API credits) | Must | 2 |
| FR-LLM-11 | MCP server exposing game state, memories, scaffolds, session logs to Claude Code | Must | 2 |

### FR-COG: Cognitive Architecture

| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-COG-01 | Fast/slow processing split — cheap filtering, expensive reasoning | Must | 1 |
| FR-COG-02 | Novelty detection: trigger slow processing for novel/complex situations | Must | 1 |
| FR-COG-03 | Automatic reflexes: configurable auto-heal, auto-loot, etc. | Must | 1 |
| FR-COG-04 | Prompted deliberation for complex decisions | Must | 1 |
| FR-COG-05 | Context assembly: compose prompts from state + goals + guides + memory | Must | 1 |
| FR-COG-06 | Context overflow management when many things are relevant | Should | 2 |
| FR-COG-07 | Event bus: central async pub/sub for all significant system events | Must | 1 |

### FR-GOAL: Goal Management

| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-GOAL-01 | Persistent goal document (text file) readable/writable by LLM | Must | 1 |
| FR-GOAL-02 | Goal process scaffold guiding prioritization and decomposition | Must | 1 |
| FR-GOAL-03 | Stuckness detection and recovery guidance | Must | 1 |
| FR-GOAL-04 | Goal document displayed in TUI | Should | 1 |
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
| FR-REFL-08 | Reflection via Claude Code skills (manual invocation first, TUI trigger later) | Should | 2 |

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
| FR-SCAFFOLD-03 | list_guides, read_guide, create_guide, update_guide tools | Must | 1 |
| FR-SCAFFOLD-04 | Trigger-based scaffold loading (auto-load relevant guides) | Must | 1 |
| FR-SCAFFOLD-05 | Scaffold versioning and change tracking | Should | 2 |
| FR-SCAFFOLD-06 | Scaffold effectiveness metadata | Should | 3 |
| FR-SCAFFOLD-07 | Conceptual exploration operator integration | Could | 5 |

### FR-CMD: Slash Commands

| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-CMD-01 | /ask — query the LLM about current state | Must | 1 |
| FR-CMD-02 | /suggest — request action suggestion | Must | 1 |
| FR-CMD-03 | /auto — toggle autonomous mode | Must | 1 |
| FR-CMD-04 | /status — show engine state, active scaffolds, model | Must | 1 |
| FR-CMD-05 | /scaffold — list/view/edit scaffolds | Must | 1 |
| FR-CMD-06 | /goal — review/manage goals | Should | 1 |
| FR-CMD-07 | /map — show current map | Should | 3 |
| FR-CMD-08 | /review — start reflection session | Should | 2 |
| FR-CMD-09 | /journal — view session journal | Should | 2 |
| FR-CMD-10 | /model — change model assignment | Should | 2 |
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
| FR-INT-01 | Sentence dump export format for ConceptMRI replay (tokens, scaffold versions, behavioral context) | Should | 2 |
| FR-INT-02 | Full LLM call logging: prompt, response, scaffold versions, decision, outcome, model, latency, tokens | Must | 1 |
| FR-INT-03 | Event log replay capability for offline analysis | Should | 2 |

---

## Non-Functional Requirements

### NFR-PERF: Performance

| ID | Requirement | Priority |
|----|------------|----------|
| NFR-PERF-01 | System 1 processing adds < 50ms latency to MUD output display | Must |
| NFR-PERF-02 | System 2 reasoning should be visibly "thinking" in TUI (not blocking input) | Must |
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
