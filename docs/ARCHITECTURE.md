# LLMUD — Software Architecture

This document maps the AI System Design to actual code: tech stack, module structure, data flow, and implementation decisions.

**Status:** Draft. Architecture will solidify as the AI design stabilizes.

---

## Overview

LLMUD and ConceptMRI are unified into a single application. The existing ConceptMRI React frontend is extended with MUD capabilities. Users have two modes that blend naturally:

**Local analysis mode** — user loads their own datasets, runs probes against their local model, visualizes results in ConceptMRI panels. Fully local, no server connection needed. This is what ConceptMRI already does.

**Institute mode** — user connects to the hosted Evennia server. A MUD panel appears. As they navigate rooms, the trajectory panel automatically switches to the data source controlled by that room's active probe. The room owns the visualization context. Local datasets remain accessible in other tabs.

The same visualization components serve both modes — different data sources, same interface.

---

## Deployment Architecture

```
┌─────────────────────────────────────────────────────┐
│  Developer Desktop (GPU machine)                     │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │  Python Backend                               │   │
│  │                                               │   │
│  │  ConceptMRI inference (existing)             │   │
│  │  Em-OSS-20b with PyTorch hooks               │   │
│  │  Harmony parsing, coordinate generation      │   │
│  │                                               │   │
│  │  Agent loop (new)                            │   │
│  │  assess → plan → act                         │   │
│  │  Calls inference, connects to Evennia        │   │
│  │  Writes logs to disk                         │   │
│  │                                               │   │
│  │  FastAPI + WebSocket (extended)              │   │
│  │  Streams MUD output, analysis channel,       │   │
│  │  coordinate updates to React frontend        │   │
│  │  Serves manifold files for download          │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  Claude Code reads log files from disk               │
│  for offline reflection — no server needed           │
└─────────────────┬────────────────────────────────────┘
                  │ telnet (agent → Evennia)
                  │ HTTP push (coordinates → relay)
                  ▼
┌─────────────────────────────────────────────────────┐
│  Server (always-on)                                  │
│                                                      │
│  Evennia — MUD world, tick manager, room metadata   │
│  Coordinate relay — receives from desktop,           │
│                     broadcasts to clients            │
└─────────────────────────────────────────────────────┘
                  ▲
                  │ WebSocket (game + room metadata + coordinates)
                  │
┌─────────────────────────────────────────────────────┐
│  React Frontend (user browser)                       │
│  Local analysis mode + Institute mode                │
│  Same components, different data sources             │
└─────────────────────────────────────────────────────┘
```

**Key decisions:**
- One unified app — ConceptMRI React frontend extended with MUD panels
- Agent loop runs on GPU desktop — one machine owns all inference
- Em called via existing ConceptMRI inference — PyTorch hooks active
- Claude Code reads log files directly — no server or API for reflection
- Users connect via a single WebSocket — no separate telnet client
- Room entry triggers automatic viz context switch in the frontend
- Fitted manifolds served as downloadable files — client caches them

---

## Tech Stack

### Frontend (existing + extended)

| Component | Choice | Notes |
|-----------|--------|-------|
| Framework | React | Existing ConceptMRI frontend |
| MUD terminal | xterm.js | Terminal emulator, ANSI support |
| WebSocket | Native browser WebSocket | Game feed, analysis, coordinates |
| Visualization | Existing ConceptMRI components | Same components, new data sources |

### Backend (existing + extended)

| Component | Choice | Notes |
|-----------|--------|-------|
| Language | Python 3.11+ | Existing |
| API | FastAPI | Extended with new endpoints |
| WebSocket | FastAPI WebSocket | Streams to frontend |
| Telnet | telnetlib3 | Agent connects to Evennia |
| Inference | PyTorch + harmony | Existing ConceptMRI inference |
| Data models | Pydantic v2 | Existing |
| Database | aiosqlite | Async SQLite |
| Scaffold parsing | python-frontmatter + PyYAML | New |
| UMAP | umap-learn | Existing |
| Claude Code | claude -p CLI | Offline reflection, reads files |

### Why Not These Alternatives

| Alternative | Why Not |
|-------------|---------|
| Separate TUI client | Already have React frontend — extend it |
| Ollama / LMStudio | No PyTorch internals — can't capture residual streams |
| MCP server for Claude Code | Reads files directly — no server needed |
| Two connections per user | Single WebSocket multiplexes everything |

---

## Module Structure

```
conceptmri-llmud/
├── CLAUDE.md
├── pyproject.toml
├── README.md
│
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── conceptmri/         # Existing components
│   │   │   │   ├── UMAPTrajectory.tsx
│   │   │   │   ├── SankeyDiagram.tsx
│   │   │   │   ├── ClusterAnalysis.tsx
│   │   │   │   └── ProbeResults.tsx
│   │   │   ├── mud/                # New
│   │   │   │   ├── MUDPanel.tsx    # xterm.js terminal
│   │   │   │   ├── AnalysisChannel.tsx
│   │   │   │   ├── AgentControls.tsx
│   │   │   │   ├── ScaffoldBrowser.tsx
│   │   │   │   ├── GoalViewer.tsx
│   │   │   │   ├── ProposalReview.tsx
│   │   │   │   └── MUDCreator.tsx  # Host only
│   │   │   └── shared/
│   │   │       └── VizPanel.tsx    # Shared trajectory panel
│   │   │                           # Switches source on room entry
│   │   ├── hooks/
│   │   │   ├── useLocalAnalysis.ts
│   │   │   ├── useMUDConnection.ts
│   │   │   └── useRoomContext.ts   # Room entry → viz switch
│   │   └── App.tsx
│   └── package.json
│
├── backend/
│   ├── conceptmri/                 # Existing
│   │   ├── inference.py            # Em + PyTorch hooks (extended for agent)
│   │   ├── harmony.py              # Harmony channel parsing
│   │   ├── umap_pipeline.py        # Manifold fit, projection, bootstrap
│   │   ├── probes.py
│   │   └── datasets.py
│   │
│   ├── agent/                      # New
│   │   ├── loop.py
│   │   ├── assess.py
│   │   ├── plan.py
│   │   ├── act.py
│   │   └── context.py
│   │
│   ├── mud/                        # New
│   │   ├── connection.py           # Async telnet to Evennia
│   │   ├── parser.py
│   │   ├── evennia_parser.py
│   │   └── ansi.py
│   │
│   ├── streaming/                  # New
│   │   ├── mud_stream.py
│   │   ├── analysis_stream.py
│   │   ├── coord_stream.py
│   │   └── room_context.py         # Room entry metadata → viz switch
│   │
│   ├── memory/                     # New
│   │   ├── episodic.py
│   │   ├── semantic.py
│   │   ├── retrieval.py
│   │   └── exporter.py             # Exports JSON for Claude Code
│   │
│   ├── scaffolds/                  # New
│   │   ├── loader.py
│   │   ├── registry.py
│   │   ├── schema.py
│   │   └── defaults/
│   │       ├── meta_assess.md
│   │       ├── meta_plan.md
│   │       ├── meta_goals.md
│   │       ├── meta_scene.md
│   │       ├── meta_needs.md
│   │       └── meta_learning.md
│   │
│   ├── social/                     # New
│   │   ├── profiles.py
│   │   └── perception.py
│   │
│   ├── world/                      # New
│   │   ├── state.py
│   │   ├── map_graph.py
│   │   └── tracker.py
│   │
│   ├── reflection/                 # New
│   │   ├── journal.py
│   │   └── exporter.py
│   │
│   ├── goals/                      # New
│   │   ├── manager.py              # File lock, stale detection, archiving
│   │   └── process.py
│   │
│   ├── tools/                      # New — LLM tool definitions
│   │   ├── registry.py
│   │   ├── game_tools.py
│   │   ├── memory_tools.py
│   │   ├── scaffold_tools.py
│   │   ├── world_tools.py
│   │   └── social_tools.py
│   │
│   ├── events/                     # New
│   │   ├── bus.py
│   │   ├── types.py
│   │   └── log.py
│   │
│   └── api/
│       ├── app.py                  # Main FastAPI app
│       ├── websocket.py            # WebSocket endpoint
│       ├── probes.py               # Existing
│       ├── datasets.py             # Existing
│       ├── manifolds.py            # New — serve fitted manifolds
│       ├── agent.py                # New — start/stop/status
│       ├── scaffolds.py            # New — scaffold browser
│       ├── proposals.py            # New — proposal review
│       └── mud_creator.py          # New — world builder (host only)
│
├── characters/
│   ├── _template/
│   │   ├── config.yaml
│   │   ├── goals.md
│   │   ├── goals_archive.md
│   │   ├── personality.md
│   │   ├── scaffolds/
│   │   │   ├── cognitive/
│   │   │   └── data/
│   │   ├── semantic/
│   │   │   └── npcs/
│   │   ├── proposals/
│   │   │   ├── semantic/
│   │   │   └── scaffolds/
│   │   ├── journals/
│   │   ├── exports/
│   │   └── data/
│   │       └── memory.db
│   └── .gitignore
│
├── docs/
│   ├── VISION.md
│   ├── AI_SYSTEM_DESIGN.md
│   ├── ARCHITECTURE.md
│   ├── WORLD_DESIGN.md
│   ├── INSTITUTION_DESIGN.md
│   └── CLAUDE_CODE_GUIDE.md
│
└── evennia_testworld/
    ├── world/
    │   ├── turn_room.py
    │   ├── turn_manager.py
    │   ├── coordinate_relay.py
    │   ├── observer_room.py
    │   ├── knowledge_mixin.py
    │   └── room_metadata.py        # Sends probe context on room entry
    └── scenarios/
```

---

## Data Flow

### Local Analysis Mode

User loads dataset → existing ConceptMRI pipeline → results in existing React components. No change.

### Institute Mode — Room Entry and Viz Switch

```
Agent or user enters a room
        │
        ▼
evennia_testworld/world/room_metadata.py
Sends on entry:
{
  "event": "room_entered",
  "probe": "social_stance",
  "manifold_id": "meadow_001",
  "feed": "em_live",
  "poles": ["friend", "enemy"]
}
        │
        ▼
backend/streaming/room_context.py
Forwards to frontend via WebSocket
        │
        ▼
frontend/hooks/useRoomContext.ts
Downloads manifold_id if not cached
Switches VizPanel to institute live feed
Updates pole labels
Subscribes to coordinate stream for this room
```

### Institute Mode — Live Inference and Streaming

```
Tick opens
        │
        ▼
backend/agent/loop.py runs assess → plan → act
        │
        ▼
backend/conceptmri/inference.py
Runs Em with PyTorch hooks
Returns: analysis + commentary + final channels
         + residual stream coordinates
        │
        ├─────────────────────────────────┐
        │ final channel                   │ analysis + coordinates
        ▼                                 ▼
backend/mud/connection.py       backend/streaming/
Submit action to Evennia        Streams to React frontend:
                                  analysis_stream → analysis pane
                                  coord_stream → VizPanel
        │
        ▼
Tick resolves — Evennia advances
        │
        ▼
backend/memory/ + events/
Log everything to disk
```

### Offline Reflection

```
Session ends
        │
        ▼
backend/reflection/exporter.py
Writes to characters/{name}/exports/:
  session_log.json, episodes.json, scaffold_stats.json
        │
        ▼
Human runs: claude -p "reflect on last session" --bare
        │
        ▼
Claude Code reads exports/ directly
Writes proposals → characters/{name}/proposals/
Writes journal → characters/{name}/journals/
        │
        ▼
Proposals visible in frontend ProposalReview component
Human approves/rejects/edits
```

---

## WebSocket Protocol

Backend → Frontend:

```typescript
{ type: "mud_output", text: string, parsed_type: string }
{ type: "analysis", text: string, session_id: string }
{ type: "coordinates", points: [{layer, x, y, z}], token: string,
  context: string, session_id: string }
{ type: "room_entered", probe: string, manifold_id: string,
  feed: string, poles: string[] }
{ type: "tick_open" | "tick_resolved", room_id: string }
{ type: "agent_status", phase: "assess"|"plan"|"act"|"idle",
  reasoning_effort: "low"|"medium"|"high" }
```

Frontend → Backend:

```typescript
{ type: "command", text: string }
{ type: "proposal_decision", proposal_id: string,
  decision: "approve"|"reject"|"edit", edited_content?: string }
{ type: "switch_feed", session_id: string }
```

---

## Persistence Architecture

### Per-Character Directory

```
characters/adventurer_one/
├── config.yaml
├── goals.md                 # Active (~600 token cap)
├── goals_archive.md
├── goals_proposals.md
├── personality.md
├── scaffolds/
│   ├── cognitive/
│   └── data/
├── semantic/
│   └── npcs/
├── proposals/
│   ├── semantic/
│   └── scaffolds/
├── journals/
├── exports/                 # Claude Code reads these
│   ├── session_log.json
│   ├── episodes.json
│   └── scaffold_stats.json
└── data/
    └── memory.db
```

### SQLite Schema (memory.db)

```sql
CREATE TABLE episodes (
    id INTEGER PRIMARY KEY,
    timestamp TEXT NOT NULL,
    location TEXT,
    event_type TEXT NOT NULL,
    description TEXT NOT NULL,
    outcome TEXT,
    entities TEXT,
    emotional_valence REAL,
    session_id TEXT,
    embedding BLOB
);
CREATE INDEX idx_episodes_event_type ON episodes(event_type);
CREATE INDEX idx_episodes_session ON episodes(session_id);
CREATE INDEX idx_episodes_timestamp ON episodes(timestamp);

CREATE TABLE session_logs (
    id INTEGER PRIMARY KEY,
    timestamp TEXT NOT NULL,
    session_id TEXT NOT NULL,
    direction TEXT NOT NULL,
    raw_text TEXT NOT NULL,
    parsed_type TEXT,
    parsed_data TEXT
);

CREATE TABLE llm_log (
    id INTEGER PRIMARY KEY,
    timestamp TEXT NOT NULL,
    session_id TEXT NOT NULL,
    task_type TEXT NOT NULL,
    reasoning_effort TEXT,
    prompt_text TEXT,
    response_analysis TEXT,
    response_commentary TEXT,
    response_final TEXT,
    coordinates TEXT,
    prompt_tokens INTEGER,
    completion_tokens INTEGER,
    latency_ms INTEGER,
    prompt_hash TEXT,
    scaffold_versions TEXT,
    scaffolds_loaded TEXT,
    tools_called TEXT,
    action_taken TEXT,
    outcome TEXT,
    user_needs_state TEXT,
    conflict_flags TEXT
);

CREATE TABLE scaffold_effectiveness (
    id INTEGER PRIMARY KEY,
    scaffold_name TEXT NOT NULL,
    scaffold_version INTEGER NOT NULL,
    session_id TEXT NOT NULL,
    times_loaded INTEGER DEFAULT 0,
    outcomes_positive INTEGER DEFAULT 0,
    outcomes_negative INTEGER DEFAULT 0,
    notes TEXT
);

CREATE TABLE rooms (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT,
    exits TEXT NOT NULL,
    npcs TEXT,
    items TEXT,
    notes TEXT,
    danger_level INTEGER DEFAULT 0,
    first_visited TEXT,
    last_visited TEXT,
    visit_count INTEGER DEFAULT 0
);
```

---

## MUD Creator (Host Only)

The MUD creator panel is disabled by config for regular users. When enabled, it provides a UI for:
- Defining rooms, objects, NPCs as YAML (per WORLD_DESIGN.md spec)
- Generating Evennia Python via Claude Code on the backend
- Loading generated scripts into the running Evennia world
- Configuring probe/manifold association per room
- Managing scenario registry and reset

Backend receives YAML spec from UI → writes to disk → invokes `claude -p` with spec and Evennia templates → reads back generated Python → loads into Evennia via admin API.

---

## Extension Points

**For Future Swarm:** agent/context.py — different agents = different context compositions. events/bus.py — already multi-subscriber.

**For Dual-Model Processing:** conceptmri/inference.py — add fast-model path for assess. agent/assess.py — route to fast model.

**For User-Hosted Instances:** mud_creator.py already isolated, unlock via config flag. Evennia world setup documented in evennia_testworld/README.md.

---

## Testing Strategy

| Level | What | How |
|-------|------|-----|
| Unit | Backend modules | pytest, mock inference |
| Integration | Agent loop + parser + memory | pytest-asyncio |
| WebSocket | Stream correctness | pytest-asyncio WebSocket client |
| E2E | Full loop against Evennia | Automated scripted ticks |
| Scaffold | Desired behavior? | Scenario replay, A/B |
| Attractor | Expected failure mode? | Scripted scenarios |
| Frontend | Components, viz switching | Jest + React Testing Library |

---

## Deployment

```bash
# Install
uv pip install -e .

# Run backend (GPU desktop)
uvicorn backend.api.app:app --host 0.0.0.0 --port 8000

# Run frontend (dev)
cd frontend && npm run dev

# Export session for Claude Code
python -m backend.reflection.exporter --character adventurer_one

# Offline reflection
claude -p "Read characters/adventurer_one/exports/ and run narrative review.
Write proposals to characters/adventurer_one/proposals/" --bare
```

---

## Dependencies

```toml
[project]
dependencies = [
    # Existing ConceptMRI
    "fastapi>=0.110,<1.0",
    "uvicorn>=0.29,<1.0",
    "pydantic>=2.5,<3.0",
    "pydantic-settings>=2.0,<3.0",
    "torch>=2.0",
    "transformers>=4.40",
    "umap-learn>=0.5,<1.0",
    "numpy>=1.24,<2.0",
    "pandas>=2.0,<3.0",
    "scikit-learn>=1.3,<2.0",
    # New LLMUD additions
    "aiosqlite>=0.19,<1.0",
    "python-frontmatter>=1.0,<2.0",
    "httpx>=0.27,<1.0",
    "anthropic>=0.40,<1.0",
    "telnetlib3>=2.0,<3.0",
    "openai-harmony>=0.1",
]
```
