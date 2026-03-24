# LLMUD UI Architecture Research

**Date:** 2026-03-24
**Purpose:** Comprehensive research into the best UI architecture for managing AI agents in virtual worlds. LLMUD needs both text gameplay AND rich graphical management interfaces — maps, charts, scaffold builders, agent state visualization.

**Constraint:** This research was compiled from deep knowledge of these technologies. Web access was unavailable during writing, so some very recent (late 2025/early 2026) developments may not be captured. The architectural patterns and technology assessments are sound regardless.

---

## Table of Contents

1. [MUD Client UI Patterns](#1-mud-client-ui-patterns)
2. [Agent Monitoring Dashboards](#2-agent-monitoring-dashboards)
3. [Terminal Graphics Capabilities](#3-terminal-graphics-capabilities)
4. [Hybrid TUI+GUI Approaches](#4-hybrid-tuigui-approaches)
5. [Scaffold Management UI](#5-scaffold-management-ui)
6. [World/Map Visualization](#6-worldmap-visualization)
7. [Web Dashboard for Monitoring](#7-web-dashboard-for-monitoring)
8. [Reference Implementations](#8-reference-implementations)
9. [Concrete Recommendation for LLMUD](#9-concrete-recommendation-for-llmud)

---

## 1. MUD Client UI Patterns

### Mudlet — The Gold Standard for MUD Client UI

Mudlet is the most feature-rich open-source MUD client and the closest analog to what LLMUD needs. It's written in C++/Qt with Lua scripting.

**Core UI architecture:**
- **Main text output pane** — ANSI-rendered scrollback buffer with clickable links, inline images (via MXP protocol), and rich text formatting. This is the heart — everything else orbits it.
- **Dockable panels** — Mudlet uses Qt's QDockWidget system. Panels can be docked to any edge, floated as separate windows, tabbed together, or hidden. This is critical: users arrange their workspace to match their playstyle.
- **Mapper widget** — Mudlet's crown jewel. A real-time 2D map that:
  - Auto-maps as you explore (via GMCP/MSDP room data or regex parsing)
  - Renders rooms as colored circles/squares connected by lines (exits)
  - Supports multiple z-levels (floors) with layer switching
  - Color-codes rooms by type (shop=yellow, danger=red, quest=blue)
  - Shows current position with a highlighted marker
  - Supports custom room symbols/icons
  - Allows manual editing (drag rooms, add labels, create areas)
  - Uses a force-directed layout algorithm by default but allows manual positioning
  - Pathfinding with speedwalking (click a room, auto-walk there)
  - Renders in a Qt OpenGL or QPainter widget — it's a true graphical canvas, not ASCII
- **Gauge bars** — HP/mana/stamina displayed as colored progress bars, typically docked at the top or bottom. Driven by GMCP data or regex captures.
- **Button bars** — Clickable command buttons that users define. Common for combat macros, movement, frequently used commands.
- **Split views** — Users can create additional text output panes (called "miniconsoles") that capture specific types of output. Example: a "combat log" miniconsole that only shows combat text, a "chat" miniconsole for communication channels.
- **Trigger/alias/timer management** — A tree-view editor where you define patterns (regex or substring), associated actions (Lua scripts), and organize them into folders. Visual indicators show which triggers are active.
- **Script editor** — Embedded code editor with Lua syntax highlighting for writing complex automation scripts.

**Key insight from Mudlet:** The power is in the composability. The mapper, gauges, miniconsoles, and buttons are all independent widgets that the user composes into their ideal layout. The text pane is always primary, but the graphical elements aren't afterthoughts — they're first-class citizens with dedicated rendering.

**What Mudlet does NOT have that LLMUD needs:**
- AI reasoning visualization
- Goal/scaffold management UI
- Agent state monitoring
- Decision trace visualization
- Anything resembling an agent dashboard

### TinTin++ — Minimalist Terminal Power

TinTin++ is the terminal purist's MUD client. Pure C, runs in any terminal.

**UI approach:**
- **Split screen** — Divides the terminal into regions using line-based splitting. Top region for output, bottom for input, optional side panels.
- **Map** — Generates ASCII maps in a terminal split pane. The maps use Unicode box-drawing characters and color. Rooms are represented as `[ ]` blocks connected by lines. The mapper is fully automatic and quite capable, but it's text — no graphical canvas.
- **No graphical widgets** — Everything is text. Gauges are drawn as ASCII progress bars. Status is shown as text lines.
- **Scripting** — Its own scripting language (not Lua/Python). Powerful but idiosyncratic.

**Key insight from TinTin++:** ASCII maps are genuinely useful and readable. You don't NEED a graphical canvas for a functional map. TinTin++ proves that a purely text-based map can support exploration, pathfinding, and orientation. The question is: can you go further?

### Blightmud — Modern Terminal MUD Client in Rust

Blightmud is the newer entrant, written in Rust, designed for terminal use.

**UI approach:**
- **Terminal-native** — Runs in any terminal emulator. Uses crossterm/termion for terminal control.
- **Split pane** — Output pane + input line, similar to TinTin++ but cleaner.
- **Lua scripting** — Modern Lua 5.4 scripting with async support.
- **Plugin system** — Packages of Lua scripts that can extend functionality.
- **GMCP support** — Can receive structured data from MUDs.
- **No built-in mapper** — This is a significant gap. Blightmud relies on plugins for mapping, and those plugins produce ASCII output.
- **TLS support** — Modern security features TinTin++ lacks.
- **Text-only UI** — No graphical elements at all.

**Key insight from Blightmud:** A modern, clean codebase proves that a terminal MUD client can be maintainable and extensible. But the lack of a mapper shows the limitation of pure terminal clients — mapping is the killer feature that drives users to Mudlet.

### MUSHclient — Windows-Native with Plugins

MUSHclient is a Windows MUD client written in C++ with COM/ActiveX plugin architecture.

**UI approach:**
- **MDI (Multiple Document Interface)** — Each MUD connection is a child window. Panels are separate windows within the main frame.
- **Miniwindows** — Custom-drawn overlay panels rendered on the main output. These are bitmapped canvases that can display anything: maps, gauges, images, custom graphics. This is MUSHclient's superpower.
- **Plugin ecosystem** — Miniwindow-based plugins provide:
  - Graphical health/mana bars with gradients and borders
  - Interactive clickable maps (bitmap-rendered, not ASCII)
  - Chat capture windows
  - Spell/cooldown trackers
  - Compass roses
- **Scripting** — Supports Lua, Python, JScript, VBScript, PerlScript via COM.

**Key insight from MUSHclient:** The miniwindow concept — a scriptable bitmap canvas overlaid on the text output — is powerful. It lets plugin authors create rich graphical displays without the client itself needing to understand every possible visualization. The client provides the canvas; the scripter provides the content.

### Synthesis: What the Best MUD Clients Teach Us

| Feature | Mudlet | TinTin++ | Blightmud | MUSHclient |
|---------|--------|----------|-----------|------------|
| Graphical map | Yes (Qt canvas) | ASCII only | No built-in | Via miniwindow plugins |
| Gauge bars | Yes (Qt widgets) | ASCII | No built-in | Via miniwindows |
| Custom panels | Yes (dockable) | Text split | No | Miniwindows |
| Script editor | Yes (embedded) | No | No | Limited |
| Text rendering | Excellent | Good | Good | Good |
| Platform | Cross-platform | Linux/Mac | Cross-platform | Windows |
| Architecture | Monolithic Qt | C terminal | Rust terminal | COM/ActiveX |

**The pattern:** Every successful MUD client starts with excellent text handling and then layers graphical elements on top. The text stream is always the primary interface. Graphical elements (maps, gauges) are secondary but critically important for serious players.

**The gap LLMUD fills:** None of these clients have any concept of AI agent management. They have triggers (regex pattern matching), aliases (text substitution), and scripts (procedural automation). LLMUD needs everything they have for gameplay PLUS an entirely new category of UI: agent monitoring, scaffold management, goal visualization, and decision trace inspection.

---

## 2. Agent Monitoring Dashboards

### LangSmith

LangSmith (by LangChain) is the most widely-deployed LLM agent tracing platform.

**Key visualizations:**
- **Trace waterfall** — A timeline view showing each step of an agent's execution as nested spans. Each span shows: the LLM call, tool calls, retrieval steps, and their durations. This is the most valuable visualization for debugging agent behavior.
- **Run tree** — Hierarchical view of parent-child relationships between steps. An agent run contains LLM calls, which contain tool calls, which contain sub-steps.
- **Token usage charts** — Bar/line charts showing token consumption over time, per model, per task type.
- **Latency distribution** — Histograms of response times by step type.
- **Feedback/evaluation views** — Tables showing human ratings, auto-eval scores, and regression comparisons.
- **Dataset management** — UI for curating prompt/response pairs for evaluation.
- **Prompt playground** — Interactive prompt editing and testing (less relevant for LLMUD).

**What's useful for LLMUD:** The trace waterfall is the killer feature. Seeing "MUD output received -> Fast processing (2ms) -> Novel detected -> Context assembled (scaffolds: combat_basic, map_data) -> LLM call (340ms, 1.2k tokens) -> Tool calls: [remember('fire elemental'), check_map('path to safety')] -> Action: flee north" as a visual timeline would be extremely valuable for debugging and research.

### AgentOps

AgentOps focuses on agent session monitoring and cost tracking.

**Key visualizations:**
- **Session timeline** — Horizontal timeline of an agent's session with events plotted.
- **Cost tracking** — Real-time cost accumulation charts (per model, per session, over time).
- **Event stream** — Live feed of agent actions, decisions, and outcomes.
- **Error tracking** — Highlighted failures with stack traces and context.
- **Agent comparison** — Side-by-side sessions for A/B testing different configurations.
- **Replay** — Step through a past session, seeing what the agent "saw" at each point.

**What's useful for LLMUD:** Session replay is directly relevant — being able to replay a MUD session and see what the agent decided at each point, what scaffolds were loaded, what the goal state was. Agent comparison maps directly to scaffold A/B testing.

### Braintrust

Braintrust focuses on evaluation and experimentation for LLM applications.

**Key visualizations:**
- **Experiment table** — Side-by-side comparison of different model/prompt configurations across the same test cases.
- **Score distributions** — Box plots and histograms of evaluation scores.
- **Diff view** — Show exactly what changed between two prompt versions and how outputs differ.
- **Regression detection** — Automated flagging when a change makes things worse.

**What's useful for LLMUD:** The experiment table and diff view map directly to scaffold comparison. "Scaffold v3 vs v4 on the same 10 combat scenarios — which produces better outcomes?"

### Weights & Biases

W&B is broader (ML experiment tracking) but has relevant patterns.

**Key visualizations:**
- **Run dashboards** — Customizable grid of charts (line, bar, scatter, histogram) showing any logged metric over time.
- **Parallel coordinates** — Visualize hyperparameter sweeps (scaffold parameters could map here).
- **Tables** — Rich data tables with inline images, audio, and nested objects.
- **Custom panels** — User-defined Vega-Lite chart specifications.
- **System metrics** — GPU/CPU/memory monitoring alongside model metrics.

**What's useful for LLMUD:** The customizable dashboard pattern. Different research questions need different charts. A fixed dashboard won't serve Emily's research needs — she needs to compose the visualizations she cares about for a given experiment.

### Phoenix (Arize) and LangFuse

These are open-source alternatives to LangSmith.

**Phoenix** provides:
- Trace visualization with span-level detail
- Embedding drift detection
- Retrieval evaluation (precision, recall of retrieved context)
- Open-source, self-hosted

**LangFuse** provides:
- Similar trace views to LangSmith
- Cost analytics
- Prompt management and versioning
- Open-source with a generous free tier

**What's useful for LLMUD:** Both prove that the core agent monitoring UX (trace waterfall, cost tracking, evaluation comparison) can be built as self-hosted web applications. LLMUD doesn't need to depend on external services.

### Synthesis: Standard Agent Monitoring Visualizations

The consensus across all these tools:

1. **Trace/timeline view** — THE essential visualization. Shows the sequence of decisions, tool calls, and outcomes over time.
2. **Cost/token tracking** — Bar charts, cumulative line charts, per-model breakdowns.
3. **Evaluation comparison** — Side-by-side tables/charts comparing configurations.
4. **Live event stream** — Real-time feed of what the agent is doing.
5. **Error/failure highlighting** — Visual callouts for problems.
6. **Session replay** — Step through past runs.

For LLMUD specifically, we also need:
7. **Scaffold effectiveness over time** — Which scaffolds are loaded, how often, and how outcomes correlate.
8. **Goal progress visualization** — Goal tree with completion status and time-on-goal.
9. **Memory growth** — How the knowledge base expands over sessions.
10. **Eudaimonic balance** — Radar/spider chart showing need fulfillment across dimensions.

---

## 3. Terminal Graphics Capabilities

### What Textual Can Do (Current State)

Textual is LLMUD's chosen TUI framework. Here is what it actually supports:

**Layout and structure:**
- CSS-like styling with selectors, pseudo-classes, animations, transitions
- Flexbox-inspired layout with docking
- Tabbed content, collapsible panels, tree views
- Modal dialogs, notification toasts
- Split views with draggable dividers (planned/in development)
- Scrollable containers with virtual scrolling for large datasets

**Built-in widgets:**
- DataTable — High-performance sortable table with millions of rows (virtual scrolling)
- Tree — Expandable tree view
- Markdown — Rendered markdown display
- RichLog — Append-only log display (perfect for MUD output)
- Input/TextArea — Text input with history
- Button, Checkbox, RadioSet, Select, Switch — Standard form widgets
- ProgressBar — Animated progress bars
- Sparkline — Inline sparkline charts
- ContentSwitcher, TabbedContent — View switching
- Header, Footer — App chrome
- Static — Rich text display
- ListView, OptionList — Scrollable item lists
- DirectoryTree — File browser
- LoadingIndicator — Spinner

**What Textual does NOT have natively:**
- No general-purpose charting (no line charts, bar charts, scatter plots)
- No canvas widget for arbitrary 2D drawing
- No image rendering (images in terminal require protocol support)
- No graph/network visualization
- No node-based editor
- No map widget

**Key limitation:** Textual is excellent for structured text UIs — forms, tables, trees, logs — but it is not a graphical canvas. For charts and maps, you need either terminal graphics protocols or a separate graphical surface.

### plotext — Terminal Charts

plotext is a Python library that renders charts as colored Unicode text.

**Capabilities:**
- Line plots, scatter plots, bar charts (horizontal and vertical)
- Histograms, box plots
- Heatmaps (using colored Unicode blocks)
- Date/time axes
- Multiple series, legends, titles
- Color themes
- Outputs to stdout as text — can be captured and displayed in a Textual widget

**Quality assessment:** plotext charts are surprisingly readable for simple data. Line charts and bar charts work well. Scatter plots are usable. Heatmaps are good for low-resolution data. The resolution is limited by terminal cell size (each character cell = 1 pixel), but Unicode half-blocks and Braille characters can achieve ~2x or 8x resolution respectively.

**For LLMUD:** plotext is the pragmatic choice for charts in the TUI. Token usage over time, scaffold effectiveness scores, goal completion timelines — all work well as plotext renders.

### Sparklines and Inline Visualizations

Textual has a built-in Sparkline widget. Additionally:
- Unicode block characters (▁▂▃▄▅▆▇█) create mini bar charts inline
- Braille characters (⠀⠁⠂...⣿) provide 2x4 dot resolution per character cell — enough for small line charts
- Colored text + box-drawing characters create basic gauge bars
- Progress bars with percentage and ETA

**For LLMUD:** Sparklines in the status bar for real-time metrics (tokens/min, actions/min, HP trend). Inline gauges for need fulfillment levels.

### Sixel Graphics

Sixel is a terminal graphics protocol from the VT340 era (1980s) that has been revived.

**How it works:** Encodes bitmap images as escape sequences in the terminal output stream. The terminal renders them as actual pixel graphics inline with text.

**Terminal support:**
- **Supported:** mlterm, mintty, WezTerm, foot, iTerm2 (macOS), Windows Terminal (since ~2024), xterm (with -ti 340 flag), Contour, ctx
- **NOT supported:** gnome-terminal, Konsole, Alacritty (by design philosophy — it delegates to Sixel alternatives)
- **Partial/via patches:** various others

**Quality:** True pixel-resolution images. You can display actual charts, maps, photographs — anything a bitmap can represent.

**Python libraries:**
- `libsixel` — C library with Python bindings
- `PIL/Pillow` — Generate images, then encode to Sixel
- `plotly` + Sixel renderer — Generate plotly charts, render as Sixel images in terminal

**For LLMUD:** Sixel could enable high-quality maps and charts directly in the terminal. But terminal support is fragmented, and integration with Textual is non-trivial (Textual manages its own rendering pipeline and doesn't natively output Sixel).

### Kitty Image Protocol

Kitty's terminal image protocol is more modern than Sixel.

**How it works:** Transmits image data (PNG/RGB) to the terminal via escape sequences. The terminal composites the image into the cell grid. Supports partial updates, animations, z-layering.

**Terminal support:**
- **Supported:** kitty, WezTerm, Konsole (recent versions), Ghostty
- **NOT supported:** most others

**For LLMUD:** Even more fragmented than Sixel. Not a reliable universal approach.

### Rich Pixels

The `rich-pixels` library converts images to Rich-renderable content using colored Unicode half-blocks (▀▄). Each character cell represents 2 pixels vertically.

**Quality:** Recognizable images but low resolution. Good for thumbnails, icons, simple diagrams. Not good for detailed maps or charts.

**For LLMUD:** Could be used for small map thumbnails or icons in the TUI, but not for primary map display.

### Assessment: Terminal Graphics Viability

| Approach | Resolution | Terminal Support | Textual Integration | Verdict |
|----------|-----------|-----------------|---------------------|---------|
| Unicode text (plotext) | ~1 char/pixel | Universal | Easy (render as text) | **Best for charts** |
| Braille characters | 2x4 dots/char | Universal | Easy | Good for sparklines/small charts |
| Unicode half-blocks | 1x2 pixels/char | Universal | Easy | Good for simple images |
| Sixel | Full pixel | ~60% of terminals | Hard (bypass Textual rendering) | Fragile |
| Kitty protocol | Full pixel | ~30% of terminals | Hard | Too fragmented |

**Bottom line:** Terminal graphics are good enough for status displays, sparklines, simple charts, and basic ASCII maps. They are NOT good enough for rich interactive maps, node-based editors, or complex data visualizations. For those, you need a graphical surface — which means a hybrid approach.

---

## 4. Hybrid TUI+GUI Approaches

This is the critical architectural decision. There are five main patterns:

### Pattern A: Terminal + Embedded Web View

**How it works:** The TUI application launches an embedded browser (via CEF, webview2, or similar) within or alongside the terminal. The web view renders rich content.

**Examples:**
- Some Electron apps embed terminal emulators alongside web content
- VS Code's terminal panel is a web-based terminal emulator running inside Electron

**Pros:**
- Single window experience
- Web technologies for rich UI (React, D3, Canvas)
- No separate process management

**Cons:**
- Electron-level resource overhead for the web view
- Complex embedding — putting a web view inside a terminal app is architecturally awkward
- The terminal and web view are different rendering systems fighting for the same window

**Verdict for LLMUD:** Architecturally backwards. You'd end up building an Electron app with a terminal widget, not a terminal app with a web widget. If you're going to embed web views, just build a web app.

### Pattern B: Terminal + Separate Web Dashboard

**How it works:** The TUI runs in the terminal. A separate web server (FastAPI/Flask) runs alongside it, serving a web dashboard. The user opens the dashboard in their browser. Both connect to the same backend state.

**Examples:**
- Jupyter: terminal process + web UI
- Tensorboard: training process + web dashboard
- Many MLOps tools: CLI + web monitoring
- Phoenix (Arize): Python instrumentation + web trace viewer
- LangFuse: SDK instrumentation + web dashboard

**Pros:**
- Best of both worlds: native terminal for gameplay, full web capabilities for management
- Web UI gets the entire modern web stack: React/Svelte, D3.js, Canvas/WebGL, any charting library
- Clean separation of concerns: TUI for real-time gameplay interaction, web for analysis/management
- Can be developed incrementally: TUI first, web dashboard added later
- Multiple users/researchers can view the dashboard simultaneously
- Dashboard can be accessed remotely
- No terminal capability constraints for the management UI

**Cons:**
- Two interfaces to maintain
- Context switching between terminal and browser
- Need to synchronize state between TUI and web
- Additional dependency (web framework, frontend build toolchain)

**Verdict for LLMUD:** **This is the most promising pattern.** The gameplay stays in the terminal where it belongs (text MUD = text interface). The management/research UI goes in the browser where it has unlimited graphical capability.

### Pattern C: Terminal + Separate Desktop GUI (tkinter/Qt/GTK)

**How it works:** The TUI spawns a separate graphical window using a desktop GUI toolkit.

**Examples:**
- Mudlet is essentially this pattern (Qt window with text pane + graphical panels)
- Some debuggers: CLI debugger + graphical variable inspector

**Pros:**
- Rich graphical capabilities
- Native performance for rendering
- Can create map editors, chart views, node editors

**Cons:**
- Desktop GUI dependency (Qt/GTK are heavy)
- Platform-specific issues (especially on WSL — GUI from WSL requires WSLg/X11 forwarding)
- You're now building two applications
- Less flexible than web (no remote access, harder to iterate on layout)
- Emily is on WSL — Qt/tkinter windows from WSL are possible but fiddly

**Verdict for LLMUD:** Qt/tkinter would be powerful but introduces significant platform complexity, especially on WSL. The web approach gives similar capabilities with better portability.

### Pattern D: Full Web UI with Terminal Emulator Embedded

**How it works:** The entire application is a web app. The MUD text output is rendered in an embedded terminal emulator (xterm.js) within the browser.

**Examples:**
- MUDs with web clients (Evennia's built-in web client uses this pattern)
- Cloud IDEs (Gitpod, GitHub Codespaces): web app with xterm.js terminal
- Wetty, ttyd: web-based terminal access

**Pros:**
- Unified interface — everything in one browser window
- Full web capabilities everywhere
- Cross-platform by default
- Can embed the terminal in any layout alongside dashboards
- Evennia already has a web client — could build on that precedent

**Cons:**
- Loses the "real terminal" feel that MUD players value
- xterm.js doesn't perfectly replicate terminal behavior (ANSI edge cases, performance with high-throughput output)
- Heavier than a native terminal
- Requires a web server even for local use
- Browser tab management, memory overhead

**Verdict for LLMUD:** This is viable and has the best UI integration story, but it sacrifices the terminal-native experience. Many MUD players specifically prefer terminal clients. Could be offered as an alternative to the TUI, not a replacement.

### Pattern E: Progressive Enhancement (TUI-first, Web Dashboard Later)

**How it works:** Start with a pure TUI (Textual). Build the complete gameplay experience there. When management/research features are needed, add a web dashboard as an optional companion. The TUI works standalone; the web dashboard adds capabilities.

**Implementation:**
1. Phase 1: Textual TUI with gameplay, basic status displays, ASCII map
2. Phase 2: FastAPI backend that exposes agent state, traces, scaffold data
3. Phase 3: Web dashboard (Svelte/React) consuming the API
4. Phase 4: Rich visualizations in the dashboard (interactive maps, scaffold editor, trace viewer)

**Pros:**
- Delivers a working product at every phase
- TUI works without web server for simple use
- Web dashboard is purely additive
- Clean architectural separation forces good API design
- Each interface does what it's best at

**Cons:**
- Some duplication of state display (both TUI and web show agent status)
- Must design the data API upfront even if web comes later
- Takes longer to reach the "full vision" UI

**Verdict for LLMUD:** **This is the recommended approach.** It matches the project's "dream big, implement incrementally" philosophy. The TUI is the game client; the web dashboard is the research platform.

### Textual-Web: A Special Case

Textual has a feature called `textual-web` (and earlier `textual-serve`) that serves a Textual app as a web application. The Textual app renders in the browser via a WebSocket connection.

**How it works:** Your Textual TUI app runs on a server. A JavaScript client in the browser receives render updates and sends input events. The app looks and behaves exactly like it does in the terminal, but in a browser tab.

**Relevance for LLMUD:**
- This does NOT give you web capabilities (React, D3, Canvas). It gives you the exact same TUI in a browser.
- Useful for remote access to the TUI, not for rich graphical interfaces.
- Could be a nice-to-have for accessing the MUD client from a phone/tablet.
- NOT a replacement for a proper web dashboard.

---

## 5. Scaffold Management UI

Scaffolds are LLMUD's core abstraction. They need a serious management interface.

### What Scaffold Management Needs

1. **Browse & search** — See all scaffolds, filter by domain/type/status
2. **Read & edit** — View scaffold content, edit markdown+YAML
3. **Version history** — See how a scaffold changed over time (git integration)
4. **Diff view** — Compare two scaffold versions side-by-side
5. **Effectiveness tracking** — Charts showing scaffold performance over time
6. **Dependency graph** — Which scaffolds reference or build on other scaffolds
7. **Visual builder** — Compose new scaffolds from templates/components
8. **A/B comparison** — Run two scaffold versions against the same scenarios, compare outcomes
9. **Metadata editing** — Edit triggers, domains, model requirements visually

### Node-Based Editors: The Ambitious Vision

Node-based visual editors (like Unreal Blueprints, ComfyUI, Node-RED, Rete.js) represent computational flows as connected nodes on a canvas.

**Relevance to scaffolds:**
- A scaffold's reasoning flow ("assess threat -> check weaknesses -> choose response -> execute") could be represented as connected nodes
- The meta-scaffold system (which scaffolds load which other scaffolds) is a graph
- The knowledge hierarchy (observations -> insights -> guides) is a flow
- The dual-process architecture (fast processing -> novelty check -> slow processing) is a flow

**Available node editor libraries (web):**
- **Rete.js** — TypeScript node editor framework. Full-featured: node creation, connection dragging, zoom/pan, minimap, node grouping. Well-documented.
- **React Flow** — React-based node/edge graph editor. Used by many AI tools. Highly customizable, good performance, great documentation. This is what most modern AI workflow tools use.
- **LiteGraph.js** — Used by ComfyUI. Lightweight, canvas-based. Less polished than React Flow but battle-tested.
- **Baklava.js** — Vue-based node editor. Clean API.
- **elkjs** — Layout algorithm library (Eclipse Layout Kernel). Can auto-layout node graphs. Used with React Flow for automatic positioning.

**Assessment for LLMUD scaffolds:**
A full node-based scaffold editor would be extremely cool but is a significant engineering effort. The simpler and more immediately useful version is:
- A **graph visualization** of scaffold relationships (read-only, using D3 or React Flow in view mode)
- A **form-based scaffold editor** (edit YAML frontmatter fields via form widgets, edit markdown body in a code editor)
- A **diff view** for scaffold versions (using a standard code diff library)
- **Save the node editor for later** — when the scaffold system is mature enough that visual composition adds real value

### Version Diff Views

For scaffold versioning, the key pattern is split-pane diff display:

**Libraries:**
- **Monaco Editor** (VS Code's editor component) — Has built-in diff view. Shows two versions side by side with insertions/deletions highlighted. Overkill for just diffs but excellent if you also want editing.
- **diff2html** — Renders git diffs as HTML. Supports side-by-side and inline modes. Lightweight.
- **react-diff-viewer** — React component for side-by-side text diffs. Clean, customizable.

**For LLMUD:** Since scaffolds are markdown files tracked in git, the natural approach is `git diff` + a web-based diff renderer. Show scaffold changes alongside their effectiveness metrics: "Version 3 added the 'assess retreat options' step, and combat survival rate went from 60% to 78%."

### Effectiveness Dashboards

Scaffold effectiveness needs:
- **Time series** — Scaffold performance metric over time (sessions, days)
- **Before/after comparison** — How did a scaffold change affect outcomes?
- **Correlation view** — When this scaffold is loaded, what happens to [metric]?
- **Usage frequency** — How often is each scaffold triggered?
- **Co-occurrence** — Which scaffolds are loaded together?

These are standard chart types (line, bar, scatter, heatmap) that any web charting library handles. The challenge is data collection and metric definition, not visualization.

### Scaffold Relationship Graphs

The scaffold ecosystem is a graph:
- Meta-scaffolds reference strategy scaffolds
- Strategy scaffolds reference tactical scaffolds
- Tactical scaffolds reference data scaffolds
- Some scaffolds trigger the loading of other scaffolds
- The goal document references active scaffolds

Visualizing this as a force-directed graph or hierarchical layout would help understand the scaffold ecosystem. Libraries:
- **D3.js force simulation** — The classic. Flexible, powerful, learning curve.
- **vis.js Network** — Simpler API for network graphs. Good for moderate-size graphs.
- **Cytoscape.js** — Full graph theory library with visualization. Supports complex layouts.
- **React Flow** in view mode — Can display graphs with automatic layout via elkjs/dagre.

**For LLMUD:** A force-directed graph of scaffold relationships, color-coded by type (meta=purple, strategic=blue, tactical=green, procedural=yellow, data=gray), with node size indicating usage frequency and edge thickness indicating co-loading frequency. Interactive: click a node to see the scaffold, hover for summary.

---

## 6. World/Map Visualization

### How Existing MUD Mapping Solutions Work

**The fundamental data structure:** A room graph. Nodes are rooms (with metadata: name, description, exits, NPCs, items, danger level). Edges are exits (with metadata: direction, door status, one-way flag).

**Auto-mapping process:**
1. Player moves (e.g., "north")
2. Parser extracts room name + exits from the new room description (or from GMCP data)
3. Room is identified (by name + description hash, or GMCP room ID)
4. If new: create node, connect to previous room via the direction traveled
5. If known: just update the "current room" marker
6. Handle edge cases: one-way exits, teleportation, rooms with identical names

**Mudlet's mapper (the gold standard):**
- Stores rooms in a SQLite database
- Each room has x, y, z coordinates (for layout)
- Renders rooms as colored shapes on a 2D canvas
- Lines connect rooms along exit directions
- Current room is highlighted
- Areas (groups of rooms) can be defined and have their own maps
- Manual room dragging to fix layout
- Custom room colors and symbols
- Room search by name/label
- Speed-walking (click destination, auto-generate movement commands)
- Multiple z-levels shown as tabs or overlays

**TinTin++ ASCII mapper:**
- Renders rooms as `[#]` blocks in a terminal grid
- Uses box-drawing characters for connections
- Color-codes rooms by type
- Centers on current position
- Shows a configurable radius around current position
- Simpler than Mudlet's mapper but functional in pure text

### Map Rendering Approaches for LLMUD

#### Option 1: ASCII Map in TUI (Textual widget)

Render a text-based map as a Textual widget. Each room is a Unicode character block, connections are lines.

```
     ┌─────┐   ┌─────┐
     │Shop │───│Market│
     └──┬──┘   └──┬──┘
        │         │
     ┌──┴──┐   ┌──┴──┐
     │Town │───│Gate │
     │Squar│   │     │
     └──┬──┘   └─────┘
        │
     ┌──┴──┐
     │Taver│
     │ n   │
     └─────┘
```

**Pros:** Works in any terminal, integrates natively with Textual, no external dependencies.
**Cons:** Low information density, can't show much detail per room, doesn't scale well to large maps, no interactive features (can't click rooms, no zoom/pan).

**Implementation:** Custom Textual widget that renders the map graph as a text grid. Use Unicode box-drawing characters. Color rooms by type. Highlight current room. Show a viewport centered on current position with configurable radius.

#### Option 2: Canvas-Based Map in Web Dashboard

Use a web canvas (HTML5 Canvas, SVG, or WebGL) for a full graphical map.

**Libraries:**
- **Pixi.js** — 2D WebGL renderer. Excellent performance for large maps. Used by many game UIs. Supports sprites, tilemaps, interaction.
- **Konva.js** — 2D Canvas framework with object model. Simpler than Pixi for non-game uses. Drag, zoom, events.
- **D3.js** — SVG-based. Good for graph layouts. Force-directed or hierarchical. Less performant for very large maps.
- **Leaflet.js** — Web map framework (for geographic maps) that can be repurposed for game maps using custom tile layers. Some MUD mapping projects use Leaflet.
- **React Flow / xyflow** — Could represent rooms as nodes and exits as edges. Already has zoom, pan, minimap. Would need custom node rendering.
- **cytoscape.js** — Graph visualization library. Force-directed layouts, custom styling, interaction.

**Best choice for LLMUD:** Pixi.js for performance, or Konva.js for simplicity. The map will grow large (hundreds or thousands of rooms), so performance matters. A Pixi.js renderer with:
- Rooms as colored rectangles with text labels
- Exits as lines between rooms
- Pan and zoom (mouse/touch)
- Current room highlighted with a pulsing marker
- Click room for detail popup (description, NPCs, items, danger level, visit history)
- Minimap in corner
- Filter/highlight rooms by criteria (danger level, visited recently, contains quest NPC)
- NPC location tracking (show known NPC positions as icons)
- Pathfinding visualization (highlight the path when speed-walking)

#### Option 3: 2D Tile Map

Represent the MUD world as a tile grid, like a roguelike.

**Pros:** Very familiar game UI pattern, high information density per room.
**Cons:** MUD room graphs don't naturally map to grids (rooms can have irregular connectivity). Requires a graph-to-grid layout algorithm.

**Assessment:** Tile maps work well for MUDs that have grid-based worlds (many do). For non-grid MUDs, a graph-based map is more natural. LLMUD should support both.

#### Option 4: 3D Isometric

A 2.5D isometric view of the world.

**Assessment:** Cool-looking but adds enormous complexity for minimal information gain. Not recommended for LLMUD.

### NPC Location Tracking

The agent tracks NPCs and their locations. Visualization:
- **On the map:** NPC icons at their known locations, with a "last seen" timestamp
- **NPC table:** Sortable list of known NPCs with location, relationship score, quests
- **Movement patterns:** If the agent has enough data, show NPC patrol routes or probability heatmaps

### Pathfinding Visualization

When the agent plans a route:
- **Highlight the path** on the map (rooms and exits in a distinct color)
- **Show the command sequence** (n, n, e, e, s)
- **Show estimated travel time** and danger level along the route
- **Alternative paths** if available (safer but longer, etc.)

### Recommendation for LLMUD Map

**Phase 1 (TUI):** ASCII map widget in Textual. Simple but functional. Centers on current room, shows nearby rooms, color-codes by type.

**Phase 2 (Web):** Full interactive map in the web dashboard using Pixi.js or Konva.js. Graph-based layout. Pan, zoom, click-for-details. NPC tracking. Path visualization.

---

## 7. Web Dashboard for Monitoring

### Should the Monitoring Be a Separate Web UI?

**Yes.** The arguments are overwhelming:

1. **Capability ceiling** — The terminal can't render interactive maps, node editors, complex charts, or rich data tables with the fidelity needed for research. The web can.
2. **Separation of concerns** — Gameplay is real-time, sequential, text-based. Research/management is exploratory, non-linear, visual. Different interfaces serve different cognitive modes.
3. **Precedent** — Every successful ML/AI monitoring tool (TensorBoard, W&B, LangSmith, MLflow, Phoenix) is a web dashboard. This is a solved UX pattern.
4. **Incremental development** — The TUI ships first as a working client. The dashboard is additive.
5. **Multi-user** — Emily might want a collaborator to view the dashboard while she plays. Or view it on a different screen.

### Architecture: FastAPI + Svelte/React

**Backend: FastAPI**

FastAPI is the natural choice given LLMUD is Python:
- Async-native (matches LLMUD's asyncio architecture)
- Automatic OpenAPI documentation
- Pydantic model serialization (LLMUD already uses Pydantic)
- WebSocket support for real-time updates
- Lightweight, fast startup

**The FastAPI server would expose:**
```
GET  /api/agent/state          — Current agent state (goals, HP, room, mode)
GET  /api/agent/traces         — Recent decision traces
GET  /api/agent/traces/{id}    — Single trace detail
WS   /api/agent/stream         — Real-time event stream (WebSocket)

GET  /api/scaffolds            — List all scaffolds with metadata
GET  /api/scaffolds/{name}     — Scaffold content + history
PUT  /api/scaffolds/{name}     — Update scaffold
GET  /api/scaffolds/{name}/diff — Diff between versions

GET  /api/map/rooms            — All rooms in the map graph
GET  /api/map/current          — Current room + nearby rooms
GET  /api/map/path?from=X&to=Y — Pathfinding result

GET  /api/memory/episodes      — Query episodic memory
GET  /api/memory/knowledge     — Query semantic memory
GET  /api/memory/stats         — Memory size, growth rate

GET  /api/metrics/tokens       — Token usage over time
GET  /api/metrics/costs        — Cost tracking
GET  /api/metrics/scaffolds    — Scaffold effectiveness metrics
GET  /api/metrics/goals        — Goal completion stats

GET  /api/sessions             — List past sessions
GET  /api/sessions/{id}/replay — Session replay data
```

**Frontend: Svelte or React?**

Both are excellent. Key considerations:

**Svelte:**
- Less boilerplate, faster development for small-medium dashboards
- Excellent reactivity model
- Smaller bundle size
- SvelteKit provides SSR, routing, API integration
- Growing ecosystem but smaller than React
- Emily may or may not have React experience; Svelte is easier to learn

**React:**
- Larger ecosystem of components (React Flow, Recharts, Monaco, etc.)
- More existing examples/documentation for every visualization type
- Next.js provides full framework features
- More likely Emily has encountered it
- More verbose but more established patterns

**Recommendation:** Svelte with SvelteKit for the dashboard. The smaller bundle, cleaner code, and faster development cycle are advantages for a research tool. The component ecosystem is sufficient: Svelte wrappers exist for D3, Pixi.js, Monaco, and most charting libraries. If a critical library only has React bindings, Svelte can embed React components.

However, if Emily has strong existing React experience, React is fine too. The architectural pattern matters more than the framework choice.

### Real-Time Updates: WebSocket Architecture

The dashboard needs real-time data. The pattern:

1. **LLMUD engine** generates events (MUD output parsed, LLM called, action taken, scaffold loaded, goal updated, etc.)
2. **Event bus** in the Python backend (asyncio queue) distributes events to subscribers
3. **FastAPI WebSocket endpoint** subscribes to the event bus and pushes events to connected dashboards
4. **Frontend** receives events and updates the UI reactively

This is the same pattern used by LangSmith, Phoenix, and LangFuse for live trace viewing.

### Dashboard Layout

A recommended dashboard layout for LLMUD:

**Top bar:** Connection status, current character, session duration, token budget remaining

**Left sidebar:** Navigation
- Overview (live)
- Agent State
- Traces
- Map
- Scaffolds
- Memory
- Metrics
- Sessions (replay)

**Main content area pages:**

**Overview (live):**
- Real-time event stream (scrolling log of decisions and actions)
- Current game state (room, HP, inventory — compact display)
- Active goals (collapsible tree)
- Active scaffolds (list with trigger indicators)
- Quick metrics (tokens used, actions taken, time in session)

**Agent State:**
- Goal document (rendered markdown, live-updating)
- Eudaimonic need radar chart (spider plot of need fulfillment)
- Current processing mode (fast/slow indicator)
- Active scaffold stack (what's loaded in context)
- Working memory visualization (what's in the current prompt)

**Traces:**
- Trace waterfall (timeline of decisions with expandable detail)
- Filter by type (perception, tactics, social, goal management)
- Click to expand: full prompt, response, tool calls, outcome
- Cost per trace

**Map:**
- Interactive Pixi.js/Konva.js map (see Section 6)
- Room detail panel on click
- NPC locations overlay
- Path planning tool
- Heatmap overlay options (visit frequency, danger, recency)

**Scaffolds:**
- Grid/list of all scaffolds with metadata cards
- Click to view/edit (Monaco editor for markdown, form for frontmatter)
- Version history with diff viewer
- Scaffold relationship graph
- Effectiveness chart per scaffold

**Memory:**
- Episodic memory table (filterable, sortable DataTable)
- Semantic knowledge browser (tree by domain)
- Memory growth chart over time
- Knowledge graph visualization (entities and relationships)

**Metrics:**
- Token usage (by model, by task type, over time)
- Action distribution (what types of actions, how often)
- Scaffold usage (load frequency, effectiveness correlation)
- Goal completion rates
- Session comparison charts

**Sessions (replay):**
- List of past sessions
- Click to replay: step through the session, seeing MUD output, agent decisions, scaffold state, and goal state at each point
- Compare two sessions side by side

---

## 8. Reference Implementations

### Open-Source Projects Combining Gameplay + Management

**Evennia Web Client** (`evennia.game_template.web`)
- Evennia itself ships with a web client that serves the MUD through a browser
- Uses Django + WebSockets
- Basic text display + some graphical elements
- Demonstrates the "web MUD client" pattern
- Relevant because LLMUD's test server IS Evennia

**Voyager (MineDojo)** — LLM agent for Minecraft
- Uses GPT-4 to play Minecraft autonomously
- Has a skill library (equivalent to scaffolds) that grows over time
- Open-source: https://github.com/MineDojo/Voyager
- Monitoring is primarily log-based — no rich dashboard
- But the architecture (skill library, curriculum, self-verification) is a close cousin of LLMUD's scaffold system

**BabyAGI / AutoGPT / AgentGPT**
- Autonomous LLM agent frameworks
- AgentGPT has a web UI showing task decomposition, agent state, and execution logs
- AutoGPT has a web dashboard (in newer versions) with agent monitoring
- These demonstrate the "agent + web dashboard" pattern for LLM agents

**Open Interpreter**
- LLM that executes code on your computer
- Has both a terminal interface and a web interface
- The web interface shows code execution, output, and agent reasoning
- Demonstrates progressive enhancement (terminal-first, web-added)

**Langroid**
- Multi-agent LLM framework with a built-in monitoring dashboard
- Shows agent conversation flows, tool usage, and decision traces
- Uses a web dashboard alongside the Python API

**CrewAI**
- Multi-agent framework
- CrewAI Studio provides a web-based visual agent builder (node-based)
- Demonstrates the node-based editor pattern for agent workflow design

**Phaser.js MUD clients** (various)
- Some projects use Phaser.js (game engine) to create graphical MUD clients in the browser
- These render tile maps, character sprites, and chat alongside traditional MUD text
- Show that the "graphical layer on top of text MUD" pattern works

**mmomern (MUD client in React)**
- React-based MUD client with graphical map
- Uses a canvas overlay for the map
- Integrates health bars and status displays
- Open source, demonstrates the web-based MUD client pattern

### Research Tools with Relevant UI Patterns

**MLflow**
- ML experiment tracking with a web dashboard
- Metric charts, artifact browser, model registry
- The experiment comparison UI is relevant for scaffold A/B testing

**Streamlit**
- Python framework for data apps
- Relevant pattern: write Python, get web dashboard
- Could be used for quick prototype dashboards before building a full Svelte/React frontend
- Limitation: not designed for real-time streaming or complex interaction

**Gradio**
- Similar to Streamlit but focused on ML model demos
- Less relevant for monitoring but demonstrates quick Python-to-web-UI

**Panel (HoloViz)**
- Python dashboarding library built on Bokeh
- Supports real-time streaming, complex layouts, interactive widgets
- Could be an intermediate step between "no web UI" and "full Svelte dashboard"
- Runs from Python, no frontend build step needed

---

## 9. Concrete Recommendation for LLMUD

### The Architecture: Layered Progressive Enhancement

```
┌──────────────────────────────────────────────────────────────┐
│                    LLMUD Architecture                        │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐     │
│  │              Layer 1: Core Engine                    │     │
│  │  Telnet, Parser, LLM, Engine, Memory, Scaffolds     │     │
│  │  (Python asyncio — the brain)                       │     │
│  └────────────────────┬────────────────────────────────┘     │
│                       │                                      │
│           ┌───────────┴───────────┐                          │
│           │    Event Bus (async)  │                          │
│           │    State API layer    │                          │
│           └───┬───────────────┬───┘                          │
│               │               │                              │
│  ┌────────────┴─────┐  ┌─────┴──────────────────────┐       │
│  │  Layer 2: TUI    │  │  Layer 3: Web Dashboard     │       │
│  │  (Textual)       │  │  (FastAPI + Svelte)         │       │
│  │                  │  │                              │       │
│  │  • MUD output    │  │  • Interactive map           │       │
│  │  • Command input │  │  • Trace waterfall           │       │
│  │  • LLM panel     │  │  • Scaffold editor           │       │
│  │  • Status bar    │  │  • Scaffold graph             │      │
│  │  • ASCII map     │  │  • Metrics charts            │       │
│  │  • Goal display  │  │  • Memory browser            │       │
│  │  • Sparklines    │  │  • Session replay            │       │
│  │  • Approval UI   │  │  • Goal visualization        │       │
│  └──────────────────┘  │  • Eudaimonic balance chart  │       │
│                        │  • Node editor (future)      │       │
│                        └──────────────────────────────┘       │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐     │
│  │              Layer 4: textual-web (optional)         │     │
│  │  Serve TUI in browser for remote access             │     │
│  └─────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────┘
```

### Technology Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| **Core engine** | Python 3.11+ asyncio | Already decided |
| **TUI** | Textual | Already decided |
| **TUI charts** | plotext + Sparkline widget | Simple terminal charts for at-a-glance metrics |
| **TUI map** | Custom Textual widget (Unicode) | ASCII map for basic orientation |
| **Internal event bus** | asyncio queues + observer pattern | Connect engine to both TUI and web |
| **Web API** | FastAPI | Async-native, Pydantic integration, WebSocket support |
| **Web frontend** | SvelteKit | Clean, fast, sufficient ecosystem. (React is acceptable alternative.) |
| **Web charts** | Apache ECharts or Recharts (React) / LayerChart (Svelte) | Feature-rich charting with good defaults |
| **Web map** | Pixi.js or Konva.js | Interactive 2D map with good performance for large graphs |
| **Web code editor** | Monaco Editor (VS Code editor) | Scaffold editing with syntax highlighting and diff view |
| **Web graph viz** | React Flow (with Svelte wrapper) or D3 force | Scaffold relationship graph, decision trees |
| **Real-time updates** | WebSocket (FastAPI -> Svelte) | Live dashboard updates during gameplay |
| **Session replay** | Custom (backed by SQLite session logs) | Step through past sessions |

### Implementation Phases

#### Phase 1: TUI Game Client (Implement Now)

This is already planned in the architecture docs. Build:

- **MUD output pane** — Textual RichLog widget with ANSI rendering, scrollback
- **Command input** — TextInput with history, slash command support
- **LLM panel** — Collapsible side panel showing:
  - Current processing mode (fast/slow indicator)
  - Active scaffolds (list)
  - Last LLM reasoning (summarized)
  - Suggested action with approval buttons
- **Status bar** — Connection state, HP/mana gauges (Unicode bars), current room name, autonomy mode
- **Goal display** — Rendered markdown of current goal document, collapsible

**Key architectural decision for Phase 1:** Build the **event bus** now, even though only the TUI consumes it. Every significant event (MUD output parsed, LLM called, action taken, scaffold loaded, goal updated, memory stored) should go through a central event system. This is what the web dashboard will subscribe to later.

```python
# events.py — build this in Phase 1
class EventBus:
    """Central event distribution. TUI subscribes now. Web subscribes later."""

    async def emit(self, event: Event) -> None:
        """Emit to all subscribers."""
        for subscriber in self._subscribers:
            await subscriber(event)

    def subscribe(self, callback: Callable[[Event], Awaitable[None]]) -> None:
        self._subscribers.append(callback)

@dataclass
class Event:
    type: str          # "mud.output", "llm.call", "action.taken", etc.
    timestamp: datetime
    data: dict         # Event-specific payload
    session_id: str
```

Also build the **state API layer** — a clean interface for reading agent state that both TUI and (future) web dashboard can use:

```python
# state_api.py — build this in Phase 1
class AgentStateAPI:
    """Clean read interface for agent state. Consumed by TUI and web."""

    def get_game_state(self) -> GameState: ...
    def get_goals(self) -> GoalDocument: ...
    def get_active_scaffolds(self) -> list[ScaffoldInfo]: ...
    def get_recent_traces(self, limit: int) -> list[Trace]: ...
    def get_map_data(self) -> MapGraph: ...
    def get_memory_stats(self) -> MemoryStats: ...
```

#### Phase 2: Enhanced TUI + API Foundation

Enhance the TUI with graphical elements and lay the API groundwork:

- **ASCII map widget** — Custom Textual widget showing the explored map centered on current room
  - Unicode box-drawing characters for rooms and connections
  - Color-coded rooms (safe=green, danger=red, shop=yellow, quest=blue, unexplored exits=dim)
  - Current room highlighted
  - Configurable viewport radius
  - Show room names for nearby rooms

- **Sparkline metrics** — Small sparkline charts in the status area:
  - Token usage rate (tokens/minute over last 30 minutes)
  - HP trend
  - Actions per minute

- **plotext charts** — Available via a `/chart` command or in a dedicated panel:
  - Token usage bar chart by model
  - Scaffold usage frequency
  - Goal completion timeline

- **FastAPI server** — Starts alongside the TUI (or on demand via `/dashboard` command):
  - Exposes the AgentStateAPI as REST endpoints
  - WebSocket endpoint for live event streaming
  - Serves static files for the web frontend
  - Auto-generates OpenAPI docs

- **Data export** — JSON export of traces, scaffolds, metrics for external analysis

#### Phase 3: Web Dashboard — Core Views

Build the Svelte frontend:

- **Overview page** — Live event stream, current state, active goals, quick metrics
- **Trace viewer** — Waterfall timeline of agent decisions
  - Each trace expands to show: trigger, context assembly, LLM call (prompt/response), tool calls, action, outcome
  - Filter by type, time range, scaffold involvement
  - Color-code by decision quality (success/failure/unknown)
- **Map page** — Interactive Pixi.js/Konva.js map
  - Room graph with zoom, pan, click-for-details
  - Current room marker (live-updating via WebSocket)
  - NPC location overlay
  - Danger heatmap overlay
  - Visit frequency heatmap overlay
  - Pathfinding visualization
- **Scaffold browser** — Grid of scaffold cards
  - Click to view full content
  - Basic metadata editing (form for YAML frontmatter)
  - Usage statistics per scaffold

#### Phase 4: Web Dashboard — Advanced Features

- **Scaffold editor** — Monaco Editor integration for editing scaffolds in the browser
  - Markdown preview pane
  - YAML frontmatter form editor
  - Version history with diff view (side-by-side, inline)
  - "Test this scaffold" button (queue it for next relevant situation)

- **Scaffold relationship graph** — D3/React Flow visualization
  - Nodes = scaffolds, color-coded by type
  - Edges = references/triggers between scaffolds
  - Size = usage frequency
  - Click to navigate to scaffold detail
  - Filter by domain

- **Metrics dashboard** — Rich charts
  - Token usage over time (stacked area chart by model)
  - Cost tracking (cumulative line chart)
  - Scaffold effectiveness (line chart per scaffold version)
  - Goal completion rates (bar chart)
  - Action distribution (pie/donut chart)
  - Session comparison (parallel coordinates or grouped bar chart)

- **Memory browser** — Searchable, filterable views of all memory stores
  - Episodic memory table with time range filter
  - Semantic knowledge tree by domain
  - Knowledge graph (entities as nodes, relationships as edges)
  - Memory growth over time chart

- **Session replay** — Step through past sessions
  - Playback controls (play, pause, step forward/back, speed control)
  - Synchronized views: MUD output, agent state, goals, scaffolds, map position
  - Annotations (add notes to specific moments)
  - Compare two sessions side by side

#### Phase 5: Web Dashboard — Research Tools

- **Eudaimonic balance visualization** — Spider/radar chart showing need fulfillment
  - Axes: survival, competence, social, exploration, creative, coherence
  - Track balance over time (animated or time-slider)
  - Highlight imbalance warnings

- **Scaffold A/B testing UI**
  - Define experiment: scaffold A vs scaffold B, test scenarios
  - Run experiments (automated replay against Evennia test world)
  - Results comparison table and charts
  - Statistical significance indicators

- **Node-based scaffold composer** (aspirational)
  - React Flow / Rete.js based visual scaffold builder
  - Drag and drop reasoning steps
  - Connect steps with flow arrows
  - Export to markdown scaffold format
  - Import existing scaffolds into visual representation

- **Interpretability hooks**
  - Export interface for Open LLMRI
  - Prompt/response pairs with full metadata
  - Scaffold version linkage to behavioral changes
  - Activation visualization integration (when Open LLMRI supports it)

### Why This Order?

The phasing follows several principles:

1. **Value at every phase.** Phase 1 delivers a working MUD client. Phase 2 adds visual polish and the API foundation. Phase 3 delivers the research dashboard. Phases 4-5 add depth.

2. **Architectural decisions front-loaded.** The event bus and state API (Phase 1) are architectural investments that enable everything else. Building them early avoids expensive refactoring.

3. **TUI first for gameplay, web for research.** The TUI is where you PLAY the MUD — it needs to be responsive, text-focused, and low-latency. The web dashboard is where you STUDY the agent — it needs charts, graphs, and rich interaction. Different tools for different cognitive modes.

4. **Web dashboard complexity escalates.** Overview and traces (Phase 3) are the most immediately useful. Scaffold editing and metrics (Phase 4) support iterative development. A/B testing and node editors (Phase 5) support mature research workflows.

5. **No premature optimization.** ASCII maps in Phase 2 serve the need while the interactive map is built in Phase 3. plotext charts in Phase 2 serve while the rich web charts are built in Phase 4. Each simpler version validates the need before investing in the richer version.

### Key Architectural Patterns to Adopt

**1. Event sourcing for agent state.**
Every agent decision, action, and state change is an event. The event log IS the source of truth. Both TUI and web dashboard are projections of the event stream. This enables replay, debugging, and analysis.

**2. WebSocket for live updates.**
The web dashboard connects via WebSocket and receives events in real-time. No polling. The dashboard feels alive while you play.

**3. Shared Pydantic models.**
The FastAPI backend and the core engine share Pydantic models. No serialization mismatch. The API contract IS the data model.

**4. Composable dashboard.**
The web dashboard should use a widget/panel system (like Grafana or W&B dashboards) where the researcher can compose their own views from available visualization components. Don't lock into a fixed layout.

**5. Git-native scaffold management.**
Scaffolds are files on disk, tracked by git. The web scaffold editor writes files and makes commits. Version history uses `git log`. Diffs use `git diff`. Don't reinvent version control.

### What NOT to Build

- **Electron wrapper.** Don't package the web dashboard as a desktop app. A browser tab is fine.
- **Custom terminal protocol.** Don't implement Sixel/Kitty graphics. Terminal support is too fragmented. Use text for the TUI, web for graphics.
- **3D anything.** No isometric maps, no 3D world views. 2D graph maps are the right abstraction for MUD worlds.
- **Mobile app.** textual-web provides mobile access to the TUI if needed. The web dashboard works on tablets.
- **Real-time multiplayer dashboard.** The dashboard is for one researcher at a time. Don't build collaborative editing.

### Cost/Complexity Assessment

| Component | Effort | Value | Priority |
|-----------|--------|-------|----------|
| Event bus + state API | Low | Critical (enables everything) | Phase 1 |
| Textual TUI (basic) | Medium | High (core gameplay) | Phase 1 |
| ASCII map widget | Medium | Medium (nice orientation tool) | Phase 2 |
| plotext charts in TUI | Low | Low-Medium (at-a-glance metrics) | Phase 2 |
| FastAPI backend | Medium | Critical (enables web) | Phase 2 |
| Svelte overview + traces | Medium | High (core research tool) | Phase 3 |
| Pixi.js interactive map | High | High (the killer feature) | Phase 3 |
| Scaffold browser | Medium | High (core workflow) | Phase 3 |
| Monaco scaffold editor | Medium | High (core workflow) | Phase 4 |
| Version diff viewer | Low | Medium | Phase 4 |
| Scaffold graph viz | Medium | Medium | Phase 4 |
| Rich metrics dashboard | Medium | High (research insight) | Phase 4 |
| Memory browser | Medium | Medium | Phase 4 |
| Session replay | High | High (research insight) | Phase 4 |
| Eudaimonic balance viz | Low | Medium (unique to LLMUD) | Phase 5 |
| Scaffold A/B testing | High | Very High (research) | Phase 5 |
| Node-based editor | Very High | Medium (aspirational) | Phase 5+ |

### Summary

LLMUD's UI architecture should be a **terminal client for gameplay** (Textual TUI with ASCII map, sparklines, and status gauges) paired with a **web dashboard for management and research** (FastAPI + SvelteKit with interactive maps, trace viewers, scaffold editors, and metrics charts). The two are connected by an event bus and shared state API, both built into the core engine from Phase 1.

This gives Emily the best of both worlds: a responsive text interface for playing MUDs (where text IS the medium), and an unlimited graphical canvas for the research platform (where charts, maps, and visual editors are essential). No compromises on either side.

The critical early investment is the **event bus** and **state API layer** — these are cheap to build, they enforce good architectural separation, and they make the web dashboard a natural extension rather than a bolt-on. Everything else follows incrementally.
