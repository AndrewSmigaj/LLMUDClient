# LLMUD — Software Architecture

This document maps the AI System Design to actual code: tech stack, module structure, data flow, and implementation decisions.

**Status:** Draft. Architecture will solidify as the AI design stabilizes.

---

## Tech Stack

| Component | Choice | Rationale |
|-----------|--------|-----------|
| **Language** | Python 3.11+ | Asyncio for networking, rich LLM ecosystem, same language as Evennia, type hints |
| **TUI** | Textual | Modern async TUI on top of Rich. Split panes, widgets, CSS-like styling. Active development. |
| **Telnet** | telnetlib3 | Async telnet client with protocol negotiation support (IAC, NAWS, GMCP) |
| **LLM (local)** | OpenAI-compatible API | Ollama, LMStudio, vLLM all expose this. Universal local model interface. |
| **LLM (API)** | httpx + anthropic SDK | Async HTTP. Anthropic SDK for Claude, httpx for others. |
| **Data models** | Pydantic v2 | Validation, serialization, settings management. Models for game state, scaffolds, config. |
| **Scaffold parsing** | python-frontmatter + PyYAML | Parse YAML frontmatter from markdown scaffold files |
| **Database** | aiosqlite | Async SQLite for episodic memory, session logs, map graph, NPC interactions |
| **Analysis tools** | pandas + scikit-learn (optional) | For the analyze_events tool. Only loaded when needed. |
| **Analytical DB** | DuckDB (Phase 2) | OLAP complement to SQLite for analytical queries on event/LLM log data |
| **Config** | pydantic-settings | Type-safe settings with env var and .env support |
| **TUI charts** | textual-plotext (Phase 2) | Lightweight ASCII-based in-terminal plotting |
| **Testing** | pytest + pytest-asyncio + pytest-cov + pytest-timeout | Standard Python testing. Async, coverage, and timeout support. |
| **Packaging** | pyproject.toml + uv | Modern Python packaging. uv for fast dependency resolution. |
| **Claude Code** | `claude -p` subprocess | Offline analysis runtime using subscription tokens ($100/mo) |
| **MCP** | mcp Python SDK | MCP server exposing LLMUD state to Claude Code |

### Why Not These Alternatives

| Alternative | Why Not |
|-------------|---------|
| LangChain/LangGraph | Adds abstraction we don't need. Our provider abstraction is simpler and specific to our use case. |
| LiteLLM | Good universal adapter but Emily wants custom routing logic per task type. |
| curses/blessed | Textual is more capable and maintainable for complex layouts. |
| PostgreSQL | Overkill for local-first. SQLite keeps everything in one file per character. |
| Redis | No need for a cache server. In-memory dicts + SQLite suffice. |
| pydantic-ai | Evaluate before coding LLM layer — may simplify provider abstraction. Don't commit without evaluation. |

### Risk Mitigations

| Library | Risk | Mitigation |
|---------|------|------------|
| telnetlib3 | Sparse maintenance, small user base | Build abstraction layer so telnet library is swappable. Limit surface area to connection/read/write. |

---

## Module Structure

```
LLMUDClient/
├── CLAUDE.md                       # Project context for Claude Code
├── pyproject.toml                  # Package definition and dependencies
├── README.md
│
├── llmud/                          # Main package
│   ├── __init__.py
│   ├── __main__.py                 # Entry point: python -m llmud
│   │
│   ├── client/                     # Telnet layer
│   │   ├── __init__.py
│   │   ├── connection.py           # Async telnet connection management
│   │   ├── protocols.py            # GMCP/MSDP protocol handlers
│   │   └── ansi.py                 # ANSI code parsing and stripping
│   │
│   ├── parser/                     # MUD output parsing
│   │   ├── __init__.py
│   │   ├── base.py                 # Parser interface (ABC)
│   │   ├── classifier.py           # Output type classification
│   │   ├── generic.py              # Generic MUD parser
│   │   └── evennia.py              # Evennia-specific parser
│   │
│   ├── tui/                        # Terminal UI (Textual)
│   │   ├── __init__.py
│   │   ├── app.py                  # Main Textual application
│   │   ├── mud_pane.py             # MUD output display
│   │   ├── input_pane.py           # Command input with /slash support
│   │   ├── llm_pane.py             # LLM status, suggestions, reasoning
│   │   ├── approval.py             # Accept/reject/retry/edit dialog
│   │   └── status_bar.py           # Connection, HP, mode indicators
│   │
│   ├── llm/                        # LLM provider abstraction
│   │   ├── __init__.py
│   │   ├── types.py                # LLMRequest, LLMResponse, Message, Tool models
│   │   ├── provider.py             # Provider interface (ABC)
│   │   ├── local.py                # OpenAI-compatible local provider
│   │   ├── anthropic_provider.py   # Claude API provider
│   │   ├── openai_provider.py      # OpenAI API provider
│   │   ├── claude_code.py          # ClaudeCodeProvider: `claude -p` subprocess (subscription tokens)
│   │   ├── router.py               # Task-type → model routing
│   │   └── grammar.py              # Custom grammar generation for local models
│   │
│   ├── events/                     # Event bus (Phase 1 — enables everything)
│   │   ├── __init__.py
│   │   ├── bus.py                  # Async pub/sub event bus
│   │   ├── types.py                # Typed event models (Pydantic)
│   │   └── log.py                  # Event persistence to SQLite for replay
│   │
│   ├── mcp/                        # MCP server for Claude Code integration
│   │   ├── __init__.py
│   │   └── server.py               # MCP server exposing game state, memories, scaffolds
│   │
│   ├── engine/                     # Cognitive architecture core
│   │   ├── __init__.py
│   │   ├── loop.py                 # Main game loop (orchestrator)
│   │   ├── fast.py                 # System 1: fast processing, classification, triggers
│   │   ├── slow.py                 # System 2: deliberative reasoning
│   │   ├── context.py              # Context assembly (prompt builder)
│   │   └── actions.py              # Action execution + approval flow
│   │
│   ├── goals/                      # Goal management
│   │   ├── __init__.py
│   │   ├── manager.py              # Read/write goal document, track state
│   │   └── process.py              # Goal process scaffold and stuckness detection
│   │
│   ├── memory/                     # Memory system
│   │   ├── __init__.py
│   │   ├── episodic.py             # Event storage (SQLite)
│   │   ├── semantic.py             # Knowledge storage (JSON + SQLite)
│   │   ├── procedural.py           # Scaffold/guide library management
│   │   ├── retrieval.py            # remember() tool implementation
│   │   └── consolidation.py        # Memory consolidation (offline)
│   │
│   ├── knowledge/                  # Knowledge hierarchy + analysis
│   │   ├── __init__.py
│   │   ├── observations.py         # Auto-logging of game events
│   │   ├── insights.py             # Pattern extraction
│   │   └── analysis.py             # analyze_events tool (data science)
│   │
│   ├── scaffolds/                  # Scaffold system
│   │   ├── __init__.py
│   │   ├── loader.py               # Load scaffolds from files
│   │   ├── registry.py             # Index of all scaffolds with metadata
│   │   ├── schema.py               # Pydantic models for scaffold format
│   │   └── defaults/               # Built-in bootstrap scaffolds
│   │       ├── meta_goals.md       # Goal management process
│   │       ├── meta_scene.md       # Scene reasoning
│   │       ├── meta_needs.md       # Basic needs fulfillment
│   │       └── meta_learning.md    # How to learn from experience
│   │
│   ├── social/                     # Social intelligence
│   │   ├── __init__.py
│   │   ├── profiles.py             # NPC/player profile storage
│   │   ├── perception.py           # Emotional state detection, subtext
│   │   └── personality.py          # Personality scaffold management
│   │
│   ├── world/                      # World model (game-specific knowledge)
│   │   ├── __init__.py
│   │   ├── state.py                # Current game state (HP, room, inventory)
│   │   ├── map_graph.py            # Room graph + pathfinding
│   │   ├── commands.py             # Command reference management
│   │   └── tracker.py              # State change detection
│   │
│   ├── reflection/                 # Reflection & REM
│   │   ├── __init__.py
│   │   ├── journal.py              # Session journaling
│   │   ├── review.py               # Reflection scaffolds (narrative, analytical, etc.)
│   │   └── rem.py                  # Offline processing orchestrator
│   │
│   ├── commands/                   # Slash command system
│   │   ├── __init__.py
│   │   ├── registry.py             # Command registry and dispatch
│   │   ├── ask.py                  # /ask
│   │   ├── suggest.py              # /suggest
│   │   ├── auto.py                 # /auto
│   │   ├── status.py               # /status
│   │   ├── scaffold_cmd.py         # /scaffold
│   │   ├── goal_cmd.py             # /goal
│   │   └── review_cmd.py           # /review
│   │
│   ├── tools/                      # Tool definitions for LLM
│   │   ├── __init__.py
│   │   ├── registry.py             # Tool registry
│   │   ├── game_tools.py           # send_command, help
│   │   ├── memory_tools.py         # remember, learn, analyze_events
│   │   ├── goal_tools.py           # read_goals, update_goals
│   │   ├── scaffold_tools.py       # list_guides, read_guide, create_guide
│   │   ├── world_tools.py          # check_map, note_location
│   │   └── social_tools.py         # recall_person, update_person
│   │
│   └── config/                     # Configuration
│       ├── __init__.py
│       ├── settings.py             # Global settings (Pydantic)
│       └── character.py            # Per-character config (model routing, etc.)
│
├── scaffolds/                      # User scaffold library (per-character directories)
│   └── _template/                  # Template character scaffold set
│       ├── cognitive/
│       │   └── .gitkeep
│       └── data/
│           └── .gitkeep
│
├── config/
│   └── default.yaml                # Default configuration
│
├── data/                           # Runtime data (SQLite DBs, logs)
│   └── .gitkeep
│
├── tests/
│   ├── conftest.py                 # Shared fixtures
│   ├── test_connection.py
│   ├── test_parser.py
│   ├── test_scaffolds.py
│   ├── test_memory.py
│   ├── test_engine.py
│   ├── test_goals.py
│   └── test_tools.py
│
├── docs/                           # Design documents
│   ├── VISION.md
│   ├── AI_SYSTEM_DESIGN.md
│   ├── REQUIREMENTS.md
│   ├── ARCHITECTURE.md (this file)
│   ├── INDEX.md
│   ├── CLAUDE_CODE_GUIDE.md
│   ├── DEV_PROCESS.md
│   └── research/                   # Research outputs (reference material)
│       ├── claude_code_patterns.md
│       ├── tech_stack_review.md
│       ├── agent_orchestration.md
│       └── ui_research.md
│
└── evennia_testworld/              # Evennia test server (separate setup)
    └── README.md                   # Setup instructions
```

---

## Data Flow

### Main Game Loop

```
                    ┌──────────────────┐
                    │  Telnet Client   │
                    │  (connection.py) │
                    └────────┬─────────┘
                             │ raw text + protocol data
                             ▼
                    ┌──────────────────┐
                    │  Parser          │
                    │  (classifier +   │
                    │   evennia.py)    │
                    └────────┬─────────┘
                             │ ParsedOutput (structured)
                             ▼
              ┌──────────────────────────────┐
              │  Engine: Fast Processing     │
              │  (fast.py)                   │
              │  • Update game state         │
              │  • Check triggers            │
              │  • Fire reflexes             │
              │  • Novelty check             │
              └──────┬────────────┬──────────┘
                     │            │
              routine│            │novel/complex
                     │            ▼
                     │  ┌──────────────────────┐
                     │  │  Context Assembly    │
                     │  │  (context.py)        │
                     │  │  • State + Goals     │
                     │  │  • Relevant guides   │
                     │  │  • Relevant memories │
                     │  │  • Tools             │
                     │  └─────────┬────────────┘
                     │            │ assembled prompt
                     │            ▼
                     │  ┌──────────────────────┐
                     │  │  LLM Router          │
                     │  │  (router.py)         │
                     │  │  Task → Model        │
                     │  └─────────┬────────────┘
                     │            │ response
                     │            ▼
                     │  ┌──────────────────────┐
                     │  │  Engine: Slow        │
                     │  │  (slow.py)           │
                     │  │  • Parse tool calls  │
                     │  │  • Execute tools     │
                     │  │  • Determine action  │
                     │  └─────────┬────────────┘
                     │            │
                     └────────────┤
                                  │ action(s)
                                  ▼
                    ┌──────────────────────────┐
                    │  Action Execution        │
                    │  (actions.py)            │
                    │  Auto → execute          │
                    │  Approval → TUI dialog   │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │  Logging                 │
                    │  (observations.py)       │
                    │  → Episodic memory       │
                    │  → Session journal       │
                    └──────────────────────────┘
```

### Offline (REM) Flow — Powered by Claude Code

```
Session Logs + Episodic Memory
          │
          ▼
┌──────────────────────────────────────────────┐
│  MCP Server (mcp/server.py)                   │
│  Exposes: memories, game state, scaffolds,    │
│  session logs, scaffold effectiveness         │
└──────────────────┬───────────────────────────┘
                   │ MCP protocol
                   ▼
┌──────────────────────────────────────────────┐
│  Claude Code (`claude -p`, subscription)      │
│  Invoked via: manual /reflect, TUI command,   │
│  or scheduled cron/systemd timer              │
│                                               │
│  Runs reflection skills:                      │
│  ├─→ /reflect → Narrative + Analytical Review │
│  ├─→ /memory-review → Consolidation           │
│  ├─→ /scaffold-eval → Effectiveness analysis  │
│  └─→ /knowledge-mine → Pattern extraction     │
└──────────────────┬───────────────────────────┘
                   │ writes back via MCP
                   ▼
┌──────────────────────────────────────────────┐
│  Results                                      │
│  • Journal entries → journals/                │
│  • Insights → semantic memory                 │
│  • Guide change proposals → scaffold library  │
│  • Meta-insights → scaffold effectiveness DB  │
└──────────────────────────────────────────────┘
```

---

## LLM Provider Architecture

```python
# Interface (provider.py)
class LLMProvider(ABC):
    async def complete(self, request: LLMRequest) -> LLMResponse: ...
    async def complete_stream(self, request: LLMRequest) -> AsyncIterator[str]: ...
    def supports_tools(self) -> bool: ...
    def supports_grammar(self) -> bool: ...

# ClaudeCodeProvider (claude_code.py)
class ClaudeCodeProvider(LLMProvider):
    """Calls `claude -p` via subprocess. Uses subscription tokens, not API credits.
    For offline analysis only — not real-time gameplay."""

    async def complete(self, request: LLMRequest) -> LLMResponse:
        # subprocess.run(["claude", "-p", prompt, "--bare",
        #     "--output-format", "json", "--max-turns", "5"])
        ...

# Router (router.py)
class ModelRouter:
    """Routes task types to providers based on character config."""

    def route(self, task_type: str) -> LLMProvider:
        """Look up task_type in config, return configured provider."""
        # config:
        #   perception: local/small (fast, free)
        #   tactics: local/emily (capable, free)
        #   roleplay: local/emily (capable, free)
        #   complex_reasoning: anthropic/claude-sonnet (high capability, API credits)
        #   reflection: claude-code (subscription tokens, offline only)
        #   memory_consolidation: claude-code (subscription tokens, offline only)
        #   scaffold_eval: claude-code (subscription tokens, offline only)
        #   default: local/emily
```

### Model Strategy

| Role | Model | Access | Cost |
|------|-------|--------|------|
| Main game agent | gpt-oss-20b ("emily") | Local, OpenAI-compatible API | Free |
| Fast/routine tasks | Smaller local model (TBD) | Local, OpenAI-compatible API | Free |
| Complex reasoning | Claude / GPT via API | API calls | Credits (minimize) |
| Offline analysis | Claude Code (`claude -p`) | Subprocess, subscription tokens | Already paying $100/mo |
| Interpretability | gpt-oss-20b direct PyTorch | Direct loading with hooks (ConceptMRI) | Free (GPU time) |

### Request Model

```python
class LLMRequest(BaseModel):
    task_type: str                  # "perception", "tactics", "roleplay", "reflection", etc.
    system_prompt: str              # Assembled from personality + role + context
    messages: list[Message]         # Conversation history
    tools: list[ToolDef] | None     # Available tool definitions
    grammar: str | None             # Custom grammar constraint (local models)
    max_tokens: int = 2048
    temperature: float = 0.7
```

---

## Persistence Architecture

### Per-Character Directory

```
characters/
└── adventurer_one/
    ├── config.yaml              # Model routing, preferences
    ├── goals.md                 # Current goal document
    ├── personality.md           # Personality scaffold
    ├── scaffolds/
    │   ├── cognitive/           # Reasoning guides (markdown)
    │   │   ├── combat.md
    │   │   ├── exploration.md
    │   │   └── social.md
    │   └── data/                # World knowledge (JSON)
    │       ├── bestiary.json
    │       ├── commands.json
    │       ├── map.json
    │       ├── npcs.json
    │       └── prices.json
    ├── journals/                # Session journals (markdown)
    │   └── 2026-03-23.md
    └── data/
        └── memory.db            # SQLite: episodic, session logs, analytics
```

### SQLite Schema (memory.db)

```sql
-- Episodic memory
CREATE TABLE episodes (
    id INTEGER PRIMARY KEY,
    timestamp TEXT NOT NULL,
    location TEXT,
    event_type TEXT NOT NULL,  -- combat, social, discovery, death, quest, etc.
    description TEXT NOT NULL,
    outcome TEXT,
    entities TEXT,             -- JSON array of involved entities
    emotional_valence REAL,    -- -1.0 to 1.0
    session_id TEXT
);

-- Session logs (raw)
CREATE TABLE session_logs (
    id INTEGER PRIMARY KEY,
    timestamp TEXT NOT NULL,
    session_id TEXT NOT NULL,
    direction TEXT NOT NULL,   -- 'in' (from MUD) or 'out' (to MUD)
    raw_text TEXT NOT NULL,
    parsed_type TEXT,          -- room_desc, combat, etc.
    parsed_data TEXT           -- JSON structured parse result
);

-- LLM interactions (for interpretability / ConceptMRI replay)
CREATE TABLE llm_log (
    id INTEGER PRIMARY KEY,
    timestamp TEXT NOT NULL,
    session_id TEXT NOT NULL,
    task_type TEXT NOT NULL,
    provider TEXT NOT NULL,    -- "local", "anthropic", "openai", "claude_code"
    model TEXT NOT NULL,
    prompt_text TEXT,          -- Full prompt sent (for ConceptMRI sentence dump export)
    response_text TEXT,        -- Full response received
    prompt_tokens INTEGER,
    completion_tokens INTEGER,
    latency_ms INTEGER,        -- Response time
    prompt_hash TEXT,          -- For deduplication analysis
    scaffold_versions TEXT,    -- JSON: {"combat": 3, "goals": 5} — scaffold versions active
    scaffolds_loaded TEXT,     -- JSON array of scaffold names
    tools_called TEXT,         -- JSON array of tool calls
    action_taken TEXT,
    outcome TEXT
);

-- Map graph
CREATE TABLE rooms (
    id TEXT PRIMARY KEY,       -- Generated from room name + description hash
    name TEXT NOT NULL,
    description TEXT,
    exits TEXT NOT NULL,       -- JSON: {"north": "room_id", ...}
    npcs TEXT,                 -- JSON array
    items TEXT,                -- JSON array
    notes TEXT,
    danger_level INTEGER DEFAULT 0,
    first_visited TEXT,
    last_visited TEXT,
    visit_count INTEGER DEFAULT 0
);
```

---

## UI Architecture

### Hybrid Approach

| Interface | Purpose | Tech | Phase |
|-----------|---------|------|-------|
| **Textual TUI** | Gameplay — MUD output, commands, LLM panel | Textual + Rich | 1 |
| **Web Dashboard** | Research management — analytics, scaffold browser, memory explorer | FastAPI + SvelteKit | 3+ |

Both connect through the **event bus + state API**:
- TUI subscribes to game/agent events for real-time display
- Dashboard subscribes to the same events for visualization and analysis
- State API provides read access to game state, memories, scaffolds, and LLM logs
- Claude Code analysis results flow back through the event bus for both interfaces

The TUI is the primary interface. The web dashboard is an add-on for research workflows — browsing scaffold effectiveness, visualizing memory networks, analyzing LLM decision patterns across sessions.

---

## Extension Points

### For Future Swarm (Custom Orchestration)

Swarm uses custom Blackboard-Mediated IFS architecture, not third-party frameworks (see AI_SYSTEM_DESIGN.md §14). These interfaces support adding multi-agent without refactoring:

1. **`engine/context.py`** — Context assembly is already a separate concern. Different "agents" = different context compositions.
2. **`tools/registry.py`** — Tool registry can scope tools per agent. Combat agent gets combat tools only.
3. **`world/state.py`** — Game state is accessed through an interface that could become a shared blackboard.
4. **`goals/manager.py`** — Goal document could become per-agent or shared.
5. **`events/bus.py`** — Event bus supports multi-subscriber pattern. Parts subscribe to relevant event types.

### For Plugins

1. **Parser plugins** — Implement `BaseParser` ABC for new MUD types
2. **Provider plugins** — Implement `LLMProvider` ABC for new LLM services
3. **Command plugins** — Register new slash commands via `CommandRegistry`
4. **Scaffold types** — Frontmatter schema is extensible; new scaffold types can define custom fields
5. **Analysis methods** — New statistical methods can be added to the analysis tool

---

## Testing Strategy

| Level | What | How |
|-------|------|-----|
| **Unit** | Individual modules (parser, memory, scaffolds, router) | pytest, mock LLM responses |
| **Integration** | Module interactions (engine + parser + memory) | pytest-asyncio, fixture-based |
| **LLM** | Actual LLM reasoning (does it follow scaffolds?) | Integration tests with real/mock model |
| **E2E** | Full loop against Evennia | Automated telnet session with scripted scenarios |
| **Scaffold** | Does a specific scaffold produce desired behavior? | Scenario replay with A/B comparison |

---

## Deployment

Local-first. No cloud services required for core functionality.

```bash
# Install
pip install -e .
# or
uv pip install -e .

# Run
python -m llmud --host localhost --port 4000 --character adventurer_one

# Run offline reflection (via Claude Code)
claude -p "Run /reflect for adventurer_one last session" --bare

# Run with specific config
python -m llmud --config config/custom.yaml
```

---

## Dependency Version Strategy

Pin major+minor, allow patch updates:

```toml
# pyproject.toml
[project]
dependencies = [
    "textual>=0.50,<1.0",
    "rich>=13.0,<14.0",
    "pydantic>=2.5,<3.0",
    "pydantic-settings>=2.0,<3.0",
    "aiosqlite>=0.19,<1.0",
    "python-frontmatter>=1.0,<2.0",
    "httpx>=0.27,<1.0",
    "anthropic>=0.40,<1.0",
    "telnetlib3>=2.0,<3.0",
]
```

Lock exact versions with `uv lock` for reproducibility. Update quarterly unless security patch needed.
