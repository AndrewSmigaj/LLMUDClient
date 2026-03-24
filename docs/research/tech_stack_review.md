# LLMUD Tech Stack Review — Critical Deep-Dive

**Date:** 2026-03-24
**Status:** Research document. Recommendations at end.
**Note:** Web access was unavailable during research. All assessments are based on knowledge current through May 2025. Items marked with [VERIFY] should be checked against live PyPI/GitHub data before acting on recommendations.

---

## Table of Contents

1. [Textual (TUI Framework)](#1-textual-tui-framework)
2. [telnetlib3 (Async Telnet)](#2-telnetlib3-async-telnet)
3. [Pydantic v2 (Data Models)](#3-pydantic-v2-data-models)
4. [aiosqlite (Async SQLite)](#4-aiosqlite-async-sqlite)
5. [python-frontmatter (YAML Frontmatter)](#5-python-frontmatter-yaml-frontmatter)
6. [LLM Provider Libraries](#6-llm-provider-libraries)
7. [pytest + pytest-asyncio (Testing)](#7-pytest--pytest-asyncio-testing)
8. [Terminal Graphical Capabilities](#8-terminal-graphical-capabilities)
9. [Hybrid TUI+GUI Approaches](#9-hybrid-tuigui-approaches)
10. [Analysis Tools (pandas + sklearn)](#10-analysis-tools-pandas--sklearn)
11. [Recommendations Summary](#11-recommendations-summary)

---

## 1. Textual (TUI Framework)

### Maintenance State

**Actively maintained.** Textual is developed by Textualize (the company behind Rich), led by Will McGuinness. As of early 2025, it was on roughly a biweekly release cadence with major feature additions throughout 2024. The project had 25k+ GitHub stars and a healthy contributor community. Textualize operates as a commercial entity (Textual Web is their monetization path), which provides financial sustainability for the OSS library.

[VERIFY] Check latest release on PyPI. Textualize has been through some organizational changes; confirm the project is still actively receiving commits.

**Key milestones reached by mid-2025:**
- CSS-based styling system (mature)
- Widget tree with composable layouts
- Async-native (built on asyncio)
- Rich text rendering (inherits all of Rich's formatting)
- Reactive attributes and data binding
- Command palette
- Built-in development tools (devtools, snapshot testing)
- Textual Web (serve TUI apps in a browser)

### Graphical Capabilities — The Critical Question

**What Textual CAN do:**

1. **Rich text rendering** — Full ANSI/Rich markup, tables, trees, markdown rendering. This covers MUD output display comprehensively.

2. **Sparklines** — Built-in `Sparkline` widget for inline data visualization. Suitable for HP/mana trend lines, token usage graphs, simple metrics.

3. **Canvas/plotting via textual-plotext** — The `textual-plotext` library wraps plotext (terminal plotting) as a Textual widget. This gives you:
   - Line plots, bar charts, scatter plots, histograms
   - All rendered in Unicode braille characters in the terminal
   - Suitable for agent analytics dashboards (token usage over time, scaffold effectiveness plots, combat outcome distributions)

4. **Rich Pixels** — The `rich-pixels` library can render images as colored Unicode blocks within Rich/Textual. Resolution is extremely low (each "pixel" is a terminal cell), but it can render small icons or very rough maps.

5. **ASCII/Unicode art** — Box drawing, colored blocks, Unicode symbols. You can build surprisingly detailed maps using Unicode box-drawing characters and colored backgrounds.

6. **DataTable widget** — Built-in sortable, scrollable data table. Good for bestiary entries, command references, inventory displays.

7. **Tree widget** — Built-in collapsible tree. Good for goal hierarchies, scaffold browsing.

8. **Markdown widget** — Renders markdown inline. Good for scaffold preview.

**What Textual CANNOT do (or does poorly):**

1. **Real graphical rendering** — No pixel-level drawing. No SVG. No actual images (beyond the crude rich-pixels approximation). You cannot render a real graph visualization, a proper map with icons, or any kind of interactive visual diagram.

2. **Interactive graph layouts** — No force-directed graphs, no node-and-edge diagrams with mouse interaction. The map_graph.py room graph visualization would be limited to ASCII art or crude Unicode representations.

3. **Mouse-interactive visual elements** — While Textual supports mouse events on widgets, you cannot create a draggable map, a zoomable chart, or a click-to-navigate graph layout. Everything is grid-of-characters.

4. **Smooth animation** — Terminal refresh rates and character-cell rendering mean animations are limited to ~30fps character changes. No smooth scrolling of graphical elements.

5. **Rich data visualization** — No equivalent of D3, matplotlib, or Plotly interactivity. textual-plotext gives you static terminal plots, but they re-render from scratch on resize/update.

### Gotchas for LLMUD

1. **Thread safety** — Textual is asyncio-native but all widget updates MUST happen on the main thread via `call_from_thread()` or `post_message()`. Since the telnet connection and LLM calls are async, this should be fine, but mixing threads (e.g., for blocking LLM SDKs) requires care.

2. **ANSI passthrough** — MUD output contains raw ANSI escape codes. Textual's `RichLog` widget can handle Rich markup, but raw ANSI from a MUD needs to be parsed and converted to Rich markup first. The `ansi.py` module in the architecture handles this, but it's a nontrivial translation layer. Watch for edge cases with MUD-specific ANSI (blinking text, 256-color, etc.).

3. **High-throughput output** — MUDs can produce large volumes of text (combat spam, channel chatter). Textual's `RichLog` can handle this, but you need to manage scrollback buffer size. Textual's virtual scrolling helps, but very long sessions can accumulate memory.

4. **Input focus model** — Textual's focus system is widget-based. The MUD input pane needs to maintain focus while other panes update. This is supported but requires explicit focus management. The approval dialog (FR-TUI-05) needs to steal focus and return it cleanly.

5. **Terminal compatibility** — Textual works well in most modern terminals, but performance varies. Windows Terminal is good; legacy cmd.exe is poor. macOS Terminal.app has quirks. WSL2 (your environment) generally works well via Windows Terminal.

6. **Textual Web** — The "serve TUI in browser" feature is interesting for the monitoring dashboard use case. You could potentially have the TUI running in a terminal AND a web view of the same app. However, Textual Web was still maturing as of mid-2025 and adds deployment complexity.

### Alternatives

| Alternative | Pros | Cons | Verdict |
|-------------|------|------|---------|
| **urwid** | Mature, flexible, good for custom widgets | Old API, less pretty, no Rich integration, poor CSS-like styling | Worse for LLMUD |
| **prompt_toolkit** | Good for input/completion, used by IPython | Not designed for full-app TUI layouts, limited widgets | Too limited |
| **blessed/blessings** | Simple terminal abstraction | Low-level, you'd build everything yourself | Too low-level |
| **curses** | Standard library, universal | Extremely low-level, painful to build complex UIs | Far too low-level |
| **Textual** | Best widget system, Rich rendering, active development, async-native | No real graphics, terminal-only | **Best choice for TUI** |
| **PyTermGUI** | Lightweight, similar concept to Textual | Much smaller community, fewer widgets, less mature | Not worth switching |

**Verdict: Keep Textual.** It is the clear best-in-class TUI framework for Python. The graphical limitations are inherent to terminal UIs, not Textual-specific. See Section 9 for hybrid approaches that address the graphics gap.

---

## 2. telnetlib3 (Async Telnet)

### Maintenance State

**This is the biggest concern in the stack.**

telnetlib3 was created by Jeff Quast (jquast). As of mid-2025, the library had not seen a release since 2023 (version 2.0.4 was the latest). The GitHub repository showed sporadic maintenance — issues were being filed but responses were slow. The library was functional but not actively developed.

[VERIFY] Check PyPI for any releases after 2.0.4. Check GitHub issues/PRs for recent activity.

**Key facts:**
- Python's standard `telnetlib` was deprecated in 3.11 and removed in 3.13. telnetlib3 is the primary async replacement.
- telnetlib3 uses asyncio properly (it was designed for it).
- It supports IAC negotiation, NAWS (window size), and basic protocol features.
- GMCP support: telnetlib3 does NOT natively support GMCP (Generic MUD Communication Protocol). GMCP is transmitted as telnet sub-negotiation (IAC SB GMCP ... IAC SE), and while telnetlib3 can handle raw sub-negotiation, you'd need to write the GMCP parsing layer yourself.
- MSDP support: Same story — possible via sub-negotiation handling, but no built-in support.

### Gotchas for LLMUD

1. **GMCP implementation** — This is significant. GMCP is your FR-NET-03 (Should, Phase 2) requirement. Many modern MUDs use GMCP for structured data (room info, character stats, map data). Without it, you're relying entirely on text parsing. telnetlib3 gives you the raw sub-negotiation hooks, but you need to:
   - Register for GMCP option negotiation
   - Parse GMCP package names and JSON payloads
   - Handle GMCP handshake (Core.Hello, Core.Supports.Set)
   - This is ~200-400 lines of custom code on top of telnetlib3

2. **Encoding issues** — MUDs vary wildly in encoding (Latin-1, UTF-8, CP437 for box drawing). telnetlib3 handles encoding but defaults may not match every MUD. You'll need configurable encoding per connection.

3. **Connection resilience** — telnetlib3's reconnection handling is basic. You'll need to build your own reconnection logic with backoff (FR-NET-05).

4. **SSL/TLS** — telnetlib3 supports TLS (FR-NET-06) via asyncio SSL contexts, but the documentation is sparse.

5. **Python 3.13+ compatibility** — Since stdlib telnetlib was removed in 3.13, some projects that depended on it broke. telnetlib3 is independent of stdlib telnetlib, so it should be fine, but [VERIFY] this.

### Alternatives

| Alternative | Pros | Cons | Verdict |
|-------------|------|------|---------|
| **telnetlib3** | Async-native, most feature-complete Python async telnet lib | Maintenance concerns, no GMCP built-in | Current choice |
| **asynctelnet** | Fork/alternative async telnet | Even less maintained than telnetlib3 | Worse |
| **miniboa** | Designed for MUD servers | Server-side, not client | Wrong direction |
| **Raw asyncio sockets** | Full control, no dependency risk | You'd reimplement all telnet protocol negotiation (IAC, option negotiation, sub-negotiation) — hundreds of lines of fiddly RFC 854 compliance | Only if telnetlib3 dies |
| **mudtelnet** | MUD-specific telnet library | Very niche, likely unmaintained [VERIFY] | Probably not viable |
| **Twisted conch** | Mature networking framework | Twisted is a massive dependency, different async model (not asyncio-native), overkill | Too heavy |

**Verdict: Keep telnetlib3, but with risk mitigation.** The library works and does what we need. The maintenance concern is real but the API surface we use is stable (it's implementing RFC 854 which doesn't change). Risk mitigation:
- Pin the version in dependencies.
- Build the GMCP layer as a clean abstraction over telnetlib3's sub-negotiation hooks, so if we ever need to swap the transport, the GMCP code is portable.
- If telnetlib3 truly dies, the fallback is raw asyncio sockets with a thin telnet protocol layer — painful but doable, and our abstraction layer in `client/connection.py` already isolates this.

---

## 3. Pydantic v2 (Data Models)

### Maintenance State

**Excellent.** Pydantic is one of the most actively maintained Python libraries. As of mid-2025, Pydantic v2 was mature and stable (v2.7+ range). Samuel Colvin's company (Pydantic Services) backs it commercially. Monthly releases, active issue triage, large contributor base.

Pydantic v2 was a major rewrite (Rust core via pydantic-core) that brought significant performance improvements over v1. The migration from v1 to v2 caused ecosystem churn in 2023-2024, but by 2025 the ecosystem had fully adapted.

### Gotchas for LLMUD

1. **Serialization for LLM tools** — When defining tool schemas for LLM function calling, Pydantic's `.model_json_schema()` generates JSON Schema. However, the output may include Pydantic-specific extensions (like `$defs` for nested models) that some LLM APIs handle differently. Test tool schemas against both Anthropic and OpenAI APIs early.

2. **Validation performance** — Pydantic v2 is fast (Rust core), but if you're validating every MUD output line through Pydantic models in the hot path (System 1), measure the overhead. For the ParsedOutput model, consider whether dataclasses or NamedTuples might be faster for the most frequent, simplest cases.

3. **Settings management** — `pydantic-settings` (separate package in v2) is excellent for the config system. It supports YAML (with a custom source), env vars, and dotenv. Make sure to use `pydantic-settings` rather than building custom config loading.

4. **Strict mode** — Pydantic v2's strict mode prevents coercion (e.g., string "1" won't become int 1). For MUD data parsing where types are inherently ambiguous, you probably want coercion (lax mode, the default). But for config and scaffold schemas, strict mode catches errors early.

### Alternatives

No serious alternatives worth considering. Pydantic v2 is the standard for Python data validation. The alternatives:
- **attrs** — Good but less ecosystem integration, no JSON Schema generation, no settings management.
- **dataclasses** — Standard library but no validation. Used for simple internal types where validation overhead is unwanted.
- **msgspec** — Faster than Pydantic for pure serialization, but less ecosystem integration and fewer features.

**Verdict: Keep Pydantic v2. No concerns.**

---

## 4. aiosqlite (Async SQLite)

### Maintenance State

**Stable but low-activity.** aiosqlite was created by Amethyst Reese (now at Meta). As of mid-2025, the library was in maintenance mode — it works, it's stable, but new features are rare. Last major release was 0.19.0 (2023) or 0.20.0 (2024) [VERIFY].

The library is a thin wrapper that runs sqlite3 operations in a thread pool (since Python's sqlite3 module is synchronous). It's architecturally simple and unlikely to have hidden bugs.

[VERIFY] Check if there's been any release activity in 2025-2026. Check if the project has been transferred to a new maintainer.

### Gotchas for LLMUD

1. **Not truly async** — aiosqlite uses `asyncio.get_event_loop().run_in_executor()` under the hood. Each database call runs in a thread. For LLMUD's use case (relatively low write frequency, reads mostly during context assembly), this is fine. But if you're doing high-frequency writes (logging every MUD output line), batch your inserts.

2. **WAL mode** — Enable WAL (Write-Ahead Logging) mode on your SQLite database for concurrent read/write access. Without WAL, a write blocks all reads. One line: `await db.execute("PRAGMA journal_mode=WAL")`.

3. **Connection management** — aiosqlite connections should be long-lived (one per character session). Don't open/close per query. Use `async with aiosqlite.connect(path) as db:` at session scope.

4. **JSON in SQLite** — Your schema uses JSON columns (entities, exits, etc.). SQLite has built-in JSON functions (json_extract, json_each) since 3.38 (2022). Use them for querying rather than loading all rows and filtering in Python.

5. **Memory with large session logs** — session_logs will grow fast (every MUD output/input line). Consider periodic archival or a retention policy. SQLite handles databases up to 281 TB in theory, but query performance degrades without proper indexing on timestamp/session_id.

### Alternatives

| Alternative | Pros | Cons | Verdict |
|-------------|------|------|---------|
| **aiosqlite** | Simple, works, lightweight | Thread-based async, slow maintenance | Current choice |
| **sqlite3 + run_in_executor** | No extra dependency, same behavior | You'd rewrite what aiosqlite already does | Pointless |
| **databases** (encode) | Async database abstraction, supports SQLite, PostgreSQL, MySQL | Heavier dependency, SQLite backend uses aiosqlite anyway | Overkill |
| **sqlalchemy async** | Full ORM with async support, SQLite backend | Massive dependency for what we need, ORM overhead | Too heavy |
| **DuckDB** | Excellent for analytics queries, columnar storage | Not great for OLTP (frequent small writes), different paradigm | Wrong tool for primary storage (but interesting for analytics — see Section 10) |
| **libsql / Turso** | SQLite fork with async native driver, better replication | Adds infrastructure complexity, early-stage | Watch for future |

**Verdict: Keep aiosqlite.** It's simple, it works, the maintenance concern is minimal because the library is essentially "done" — it's a thin wrapper with a stable API surface. Just make sure to enable WAL mode and add proper indexes.

---

## 5. python-frontmatter (YAML Frontmatter)

### Maintenance State

**Stable, low-activity maintenance.** python-frontmatter is a small utility library by Chris Amico. It does one thing: parse YAML/TOML/JSON frontmatter from text files (the `---\n...\n---` format used in Jekyll, Hugo, etc.). Last release was likely in 2023-2024 [VERIFY].

The library is simple enough that maintenance concerns are minimal. It depends on PyYAML for YAML parsing.

### Gotchas for LLMUD

1. **PyYAML security** — python-frontmatter uses PyYAML. PyYAML's `yaml.load()` with the default Loader is a known security risk (arbitrary code execution). python-frontmatter uses `yaml.safe_load()` by default, which is safe. Verify this is still the case in whatever version you install.

2. **TOML frontmatter** — python-frontmatter also supports TOML frontmatter (using `+++` delimiters). If you ever want to use TOML for scaffold metadata, it's already supported.

3. **Custom handlers** — python-frontmatter supports custom post/handler classes if you need to extend the metadata format.

4. **Limited validation** — python-frontmatter parses frontmatter into a dict. It doesn't validate the structure. You'll need Pydantic models (scaffold schema.py) to validate that parsed frontmatter matches expected scaffold schemas. This is already planned in the architecture.

### Alternatives

| Alternative | Pros | Cons | Verdict |
|-------------|------|------|---------|
| **python-frontmatter** | Simple, works, supports YAML/TOML/JSON | Minimal maintenance | Current choice |
| **Manual parsing** | No dependency, ~20 lines of code | Why bother? Edge cases in frontmatter parsing are annoying | Not worth it |
| **gray-matter (JS port)** | N/A — JavaScript only | Wrong language | N/A |
| **frontmatter** (PyPI) | Another frontmatter parser | Less popular, no advantage | No reason to switch |

**Verdict: Keep python-frontmatter.** It's a tiny, stable utility. The only risk is PyYAML as a transitive dependency, but PyYAML is ubiquitous and well-maintained.

---

## 6. LLM Provider Libraries

### Current Plan: anthropic SDK + openai SDK

#### anthropic SDK

**Maintenance state:** Excellent. Anthropic maintains their official Python SDK actively, with releases tracking every API update. As of mid-2025, the SDK supported Messages API, tool use, streaming, batch API, prompt caching, and extended thinking. The SDK is auto-generated from their API spec but with hand-tuned ergonomics.

**Gotchas:**
1. **Streaming + tool use** — Streaming responses with tool calls requires careful handling. The SDK provides helper classes (`MessageStream`) but the event model is complex. You need to accumulate `content_block_delta` events and detect `tool_use` blocks.
2. **Async client** — `anthropic.AsyncAnthropic()` is fully async. Good. Uses httpx internally.
3. **Prompt caching** — Claude supports prompt caching for system prompts and tool definitions. For LLMUD, where the system prompt + scaffold context is often similar across calls, this can significantly reduce costs and latency. But the cache key is exact-match on prefix — any change to the system prompt invalidates the cache.
4. **Extended thinking** — Claude's extended thinking feature (chain-of-thought before responding) is powerful for System 2 reasoning. The SDK supports it but the response format differs (thinking blocks + response blocks).
5. **Beta headers** — Some features require beta headers (e.g., `anthropic-beta: prompt-caching-2024-07-31`). These change. Keep up with API changelog.
6. **Rate limiting** — The SDK raises `RateLimitError`. Your router needs to handle this gracefully, potentially falling back to local models.

#### openai SDK

**Maintenance state:** Excellent. OpenAI's Python SDK is actively maintained and is the de facto standard for OpenAI-compatible APIs. Crucially for LLMUD, **Ollama, LMStudio, vLLM, and most local model servers expose an OpenAI-compatible API**, making the openai SDK the universal local model client.

**Gotchas:**
1. **Compatibility variations** — "OpenAI-compatible" is a loose standard. Different local servers implement different subsets:
   - Ollama: Supports chat completions, streaming, basic tool use. Does NOT support all OpenAI parameters (logprobs, logit_bias vary by version).
   - LMStudio: Similar to Ollama. Tool use support varies.
   - vLLM: More complete OpenAI compatibility, including guided generation (grammar constraints).
   - **You MUST test your provider code against each server you intend to support.** Don't assume feature parity.
2. **base_url override** — For local models, you set `base_url="http://localhost:11434/v1"` (Ollama) or similar. The SDK handles this but some edge cases (e.g., model listing endpoint) may differ between servers.
3. **Tool use with local models** — Tool calling with local models is unreliable compared to API models. Small models (7B-20B) frequently produce malformed tool calls, call nonexistent tools, or ignore tool results. Your System 1 (fast processing) should probably not use tool calling; reserve it for System 2 with capable models.
4. **Grammar/guided generation** — The openai SDK doesn't expose grammar constraints (GBNF, JSON schema enforcement) because OpenAI's API doesn't support it. But vLLM, llama.cpp, and Ollama do support it via extra parameters. You'll need to pass these as `extra_body` parameters or use the server's native API directly.
5. **Async client** — `openai.AsyncOpenAI()` is fully async. Uses httpx internally.

#### Alternatives to Consider

| Alternative | Pros | Cons | Verdict |
|-------------|------|------|---------|
| **anthropic + openai SDKs** | Official, well-maintained, full feature access | Two SDKs to maintain, different response formats | Current choice |
| **LiteLLM** | Unified interface for 100+ providers, handles format translation | Adds abstraction layer, less control over provider-specific features (caching, thinking), dependency risk | **Not for primary use, but worth watching** |
| **httpx direct** | Full control, no SDK dependency, minimal surface area | You'd rewrite auth, streaming, error handling, retries | Too much work for API providers |
| **aisuite** | Lightweight multi-provider abstraction by Andrew Ng | Very new, limited features, no tool use support (as of mid-2025) | Too immature |
| **pydantic-ai** | Pydantic-native LLM abstraction with tool calling | Newer library, opinionated, may not fit custom routing needs | **Interesting — worth evaluating** |
| **instructor** | Structured output extraction from LLMs | Focused on structured extraction, not general agent use | Useful as a complement, not a replacement |
| **magentic** | Decorator-based LLM function calling | Too opinionated for our use case | Not suitable |

**Assessment of pydantic-ai:** This is worth a closer look. pydantic-ai (by the Pydantic team) provides a typed, Pydantic-native interface for LLM interactions with built-in tool calling. Since LLMUD already uses Pydantic v2 for everything, the integration would be natural. It supports multiple providers (OpenAI, Anthropic, Groq, Ollama) through a unified interface. The concern is whether it's flexible enough for LLMUD's custom routing (per-task model selection, grammar constraints for local models, etc.).

[VERIFY] Check pydantic-ai's current state. If it supports custom routing and provider-specific features, it could replace both SDKs with a single abstraction.

**Verdict: Keep anthropic + openai SDKs for now.** The custom provider abstraction in `llm/provider.py` already isolates these dependencies. But evaluate pydantic-ai as a potential unification layer — if it's mature enough and flexible enough, it could simplify the provider layer significantly while staying in the Pydantic ecosystem.

---

## 7. pytest + pytest-asyncio (Testing)

### Maintenance State

**Both excellent.** pytest is the gold standard for Python testing. pytest-asyncio is actively maintained and supports modern asyncio patterns.

As of mid-2025:
- pytest: v8.x, active development, monthly releases.
- pytest-asyncio: v0.23+ (possibly v1.0 by now [VERIFY]), with the `auto` mode that automatically handles async test functions.

### Gotchas for LLMUD

1. **pytest-asyncio mode** — Use `asyncio_mode = "auto"` in pyproject.toml so you don't need `@pytest.mark.asyncio` on every test. Cleaner.

2. **Event loop scope** — pytest-asyncio creates a new event loop per test by default. If you need shared async fixtures (e.g., a database connection), use `scope="session"` or `scope="module"` on the fixture. Be careful: shared async fixtures with `scope="session"` require `loop_scope="session"` in newer pytest-asyncio.

3. **Mocking LLM responses** — For engine/cognitive tests, you'll need mock LLM providers that return canned responses. Build a `MockProvider(LLMProvider)` early. Also consider recording/replaying real LLM responses for regression tests.

4. **Textual testing** — Textual has its own testing framework (`textual.testing`) with `App.run_test()` that simulates user input and checks widget state. Use it for TUI tests. It's well-documented.

5. **Telnet testing** — For telnet tests, you'll need a mock telnet server. Consider writing a simple asyncio server that sends canned MUD output, or use `unittest.mock.AsyncMock` to mock the connection.

### Additional Testing Libraries to Consider

- **pytest-cov** — Coverage reporting. Essential.
- **pytest-timeout** — Prevent hanging async tests. Very useful for networking tests.
- **hypothesis** — Property-based testing. Could be valuable for parser testing (generate random MUD output, verify parser doesn't crash).
- **syrupy** — Snapshot testing. Useful for verifying LLM prompt assembly doesn't change unexpectedly.

**Verdict: Keep pytest + pytest-asyncio. Add pytest-cov and pytest-timeout.**

---

## 8. Terminal Graphical Capabilities

This section addresses the core question: what graphical elements can we render in the terminal, and how good are they?

### plotext

**What it is:** A Python library for plotting directly in the terminal using Unicode characters (braille dots, block elements). No GUI dependencies.

**Maintenance:** Actively maintained as of mid-2025. The author (Savino) has been consistent with releases.

**Capabilities:**
- Line plots, scatter plots, bar charts, histograms, candlestick charts
- Multiple subplots
- Date/time axes
- Color support (ANSI 256 + RGB where terminal supports it)
- Image rendering (using braille characters — very low resolution)
- Animation (re-rendering plots)

**For LLMUD:**
- **Token usage over time** — line plot, works well
- **Scaffold effectiveness** — bar chart comparing scaffold versions, works well
- **Combat outcome distribution** — histogram, works well
- **Agent state sparklines** — inline mini-plots, works well
- **Map rendering** — poor. A room graph as a plotext scatter plot with labels would be barely usable
- **Relationship graphs** — not supported (no node-edge layout)

### textual-plotext

**What it is:** A Textual widget that wraps plotext. Renders plotext charts inside a Textual pane.

**Maintenance:** Community-maintained, follows plotext and Textual releases.

**For LLMUD:** This is the right way to add charts to the TUI. A dedicated analytics pane could show real-time plots of agent metrics.

### rich-pixels

**What it is:** Renders images as colored Unicode half-block characters within Rich/Textual.

**For LLMUD:** Limited utility. Could render a tiny map icon or ASCII art portrait, but the resolution is too low for meaningful visualization. A 100-character-wide terminal gives you roughly 100x50 "pixels" in the best case.

### Unicode Box Drawing for Maps

The most practical approach for terminal map rendering:

```
    ╔═══════╗     ╔═══════╗
    ║ Town  ║─────║ Shop  ║
    ║ Square║     ╚═══╤═══╝
    ╚═══╤═══╝         │
        │         ╔═══╧═══╗
    ╔═══╧═══╗     ║ Back  ║
    ║ Road  ║─────║ Alley ║
    ╚═══════╝     ╚═══════╝
```

This approach:
- Works in any terminal
- Can be colored (current room highlighted, dangerous areas in red)
- Can be generated programmatically from the room graph
- Is the standard approach for MUD map display
- Can include mini-info (room name, danger indicators)

**Limitation:** Large maps require scrolling/panning, and the ASCII representation gets cluttered beyond ~20-30 rooms visible at once.

### Sixel Graphics / Kitty Graphics Protocol

Some modern terminals (Kitty, WezTerm, iTerm2, foot) support inline image protocols:
- **Sixel** — An old DEC protocol for inline raster images, revived by modern terminals
- **Kitty Graphics Protocol** — Kitty terminal's proprietary but powerful inline image protocol

Libraries like `term-image` can render actual images in supporting terminals. This would allow:
- Real graphical maps (generated with matplotlib/graphviz, rendered inline)
- Agent state diagrams
- Scaffold relationship visualizations

**The problem:** Terminal support is inconsistent. Windows Terminal gained Sixel support only recently [VERIFY]. Not all users will have compatible terminals. This cannot be the primary rendering path — only an enhancement.

### Assessment

**For Phase 1-2:** Unicode box-drawing maps + textual-plotext charts are sufficient and universally compatible.

**For Phase 3+:** Consider Sixel/Kitty protocol for enhanced graphics in supporting terminals, with Unicode fallback.

**For rich interactive visualization (graph layouts, interactive maps, scaffold builders):** The terminal is not the right medium. See Section 9.

---

## 9. Hybrid TUI+GUI Approaches

This is the most architecturally significant question. LLMUD needs:
- **Gameplay** — Text display, command input, action approval. TUI is PERFECT for this.
- **Monitoring** — Agent state, goal progress, scaffold status, token usage. TUI is adequate.
- **Analysis** — Charts, graphs, statistical visualizations. TUI is marginal (textual-plotext helps).
- **Visualization** — Room graph maps, NPC relationship webs, scaffold dependency graphs. TUI is poor.
- **Authoring** — Scaffold editing, goal editing, configuration. TUI is workable but clunky.

### Option A: Pure TUI (Current Plan)

**Pros:** Single process, no web server, no browser dependency, everything in one terminal. Simplest to develop and deploy. True to the MUD spirit.

**Cons:** Limited visualization. Map display is crude. No interactive graph layouts. Analysis charts are low-resolution.

**When it's enough:** If visualization needs stay modest — ASCII maps, sparklines, simple bar charts — pure TUI works. For a research platform, though, you'll eventually want better visualization.

### Option B: TUI + Textual Web Dashboard

Textual Web can serve a Textual app in a browser. You could have:
- Primary TUI in terminal (gameplay)
- Textual Web app in browser (monitoring dashboard)

**Pros:** Same codebase, same widgets, no new framework to learn.

**Cons:** Textual Web is still Textual — same character-cell rendering, just in a browser. You gain accessibility (anyone with a browser) but not graphical capability. You don't get D3 charts or interactive graphs.

### Option C: TUI + Separate Web Dashboard (FastAPI/Flask + HTMX/React)

Run a lightweight web server alongside the TUI that serves a monitoring/analysis dashboard:

```
Terminal                    Browser
┌─────────────┐            ┌──────────────────────┐
│  LLMUD TUI  │            │  Dashboard           │
│  (Textual)  │◄──────────►│  (FastAPI + HTMX)    │
│             │  shared    │  - D3 map graph       │
│  gameplay   │  state     │  - Plotly charts      │
│  input/out  │  via       │  - Scaffold browser   │
│  LLM panel  │  asyncio   │  - Session analytics  │
│  approval   │  events    │  - Agent state        │
└─────────────┘            └──────────────────────┘
```

**Pros:**
- Gameplay stays in terminal where it belongs
- Dashboard gets full web capabilities (D3, Plotly, interactive graphs, image rendering)
- Dashboard is optional — TUI works standalone
- FastAPI is async-native, shares the event loop
- HTMX keeps it lightweight (no full SPA framework needed)
- Multiple people could view the dashboard (spectator mode)

**Cons:**
- Two UI systems to maintain
- Need a shared state mechanism (event bus, shared memory objects)
- More complexity
- Browser dependency for advanced features

**Architecture sketch:**
- `llmud/dashboard/` module with FastAPI app
- WebSocket connection for real-time updates
- Shared state via the existing engine state + event system
- Optional: `--dashboard` flag to enable, `--dashboard-port 8080`

### Option D: TUI + Electron/Tauri Desktop App

A native desktop app wrapping both the terminal and a web view:

**Pros:** Single window, native feel, full graphical capabilities.

**Cons:** MASSIVE increase in complexity. Electron is heavy. Tauri is lighter (Rust) but you'd need to bridge Python<->Rust. This is a different project.

**Verdict: Not appropriate for LLMUD's scale.**

### Option E: Full Web Client (Abandon TUI)

Replace the TUI entirely with a web-based client:

**Pros:** Unlimited graphical capability, no terminal constraints, wider accessibility.

**Cons:** Loses the terminal aesthetic and MUD culture alignment. More complex. Requires a browser. Moves away from the "MUD client" identity.

**Verdict: Against the project vision.** LLMUD is a MUD client. MUD clients are terminal programs.

### Recommendation

**Phase 1-2: Pure TUI (Option A).** Focus on gameplay, LLM integration, and core cognitive architecture. Use textual-plotext for basic charts, Unicode for maps.

**Phase 3+: TUI + Web Dashboard (Option C).** When you need rich visualization for research (scaffold effectiveness analysis, map graph visualization, agent analytics), add a lightweight FastAPI dashboard. Key design decisions:

1. **Make it optional.** The TUI must work standalone. The dashboard is a research tool.
2. **Use FastAPI + HTMX + a few JS libraries** (D3 for graphs, Plotly for charts). NOT a full SPA.
3. **Share state via an event bus** — the engine already emits events for logging; the dashboard subscribes to the same events.
4. **Consider NiceGUI** [VERIFY maintenance state] — a Python library that creates web UIs with minimal JS. It supports Plotly, AG Grid, and various visualization widgets, all from Python. Could be faster to develop than raw FastAPI+HTMX.

---

## 10. Analysis Tools (pandas + sklearn)

### Current Plan

The architecture lists `pandas + scikit-learn (optional)` for the `analyze_events` tool. This tool lets the LLM run statistical analysis on its own experience data (combat outcomes, price trends, NPC behavior patterns).

### pandas

**Maintenance:** Excellent. pandas 2.x is actively maintained with regular releases.

**For LLMUD:**
- Great for loading SQLite data (`pd.read_sql_query`), computing statistics, aggregations, time series analysis
- The LLM's `analyze_events` tool would build pandas queries and interpret results
- Heavy dependency (~30MB) for what might be simple aggregations

**Gotcha:** pandas is a large import. If it's loaded on startup, it adds 1-2 seconds. Load lazily (`importlib.import_module` in the analysis tool, not at top level).

### scikit-learn

**Maintenance:** Excellent. Actively maintained by a large team.

**For LLMUD:**
- Decision trees for combat strategy analysis (feature importance: what factors predict combat success?)
- Clustering for NPC behavior categorization
- Linear regression for price trend prediction
- Dimensionality reduction for visualizing scaffold embedding spaces

**Gotcha:** Even larger dependency than pandas. Lazy loading is essential.

### Alternatives and Complements

| Tool | Use Case | Pros | Cons |
|------|----------|------|------|
| **pandas** | Data manipulation, aggregation | Standard, powerful, LLM knows it well | Heavy import |
| **polars** | Same as pandas, but faster | Much faster than pandas, lazy evaluation, Rust core | LLMs may know it less well for code generation, API is different |
| **DuckDB** | SQL analytics on structured data | Blazing fast for analytical queries, can query SQLite directly, SQL interface (LLMs are great at SQL) | Another dependency, overlaps with SQLite |
| **scikit-learn** | ML models, feature importance | Standard, comprehensive | Heavy, might be overkill |
| **scipy.stats** | Statistical tests | Lighter than sklearn for pure statistics | Less ML capability |
| **numpy** | Numerical computation | Already a dependency of pandas/sklearn | Too low-level alone |

### Interesting Idea: DuckDB for Analytics

DuckDB can directly query SQLite databases with zero-copy. Instead of loading data into pandas, the LLM could write SQL analytics queries that DuckDB executes against the SQLite memory.db:

```python
import duckdb
conn = duckdb.connect()
conn.execute("ATTACH 'memory.db' AS mem (TYPE sqlite)")
result = conn.execute("""
    SELECT event_type, AVG(emotional_valence), COUNT(*)
    FROM mem.episodes
    WHERE location LIKE '%dungeon%'
    GROUP BY event_type
    ORDER BY COUNT(*) DESC
""").fetchdf()  # Returns pandas DataFrame
```

**Why this is compelling:**
- LLMs are better at generating SQL than pandas code
- SQL is safer to execute (no arbitrary Python)
- DuckDB is extremely fast for analytical queries
- It can produce pandas DataFrames when needed
- It adds analytical SQL functions (window functions, statistics) that SQLite lacks

### Recommendation

**Keep pandas as the primary analysis framework** — it's what the LLM will be most fluent with.

**Add DuckDB as an optional analytical query engine** — for SQL-based analytics that run directly against memory.db. This is especially powerful because:
1. The LLM generates SQL (its strongest structured output format)
2. No need to load all data into Python memory
3. DuckDB's analytical functions exceed SQLite's

**Defer scikit-learn** — Start with pandas aggregations and DuckDB SQL. Add sklearn only when you have a specific ML need (feature importance for combat analysis, clustering for NPC behavior). Most early "analysis" will be aggregations, correlations, and trends — pandas and DuckDB handle those fine.

**Lazy load everything** — None of these should be imported at startup. Load them in the `analyze_events` tool implementation only.

---

## 11. Recommendations Summary

### Keep (No Changes Needed)

| Library | Confidence | Notes |
|---------|------------|-------|
| **Python 3.11+ / asyncio** | High | Correct foundation |
| **Textual** | High | Best TUI framework, no real competitor |
| **Pydantic v2** | High | Industry standard, excellent maintenance |
| **python-frontmatter** | High | Simple, stable, does its job |
| **pytest + pytest-asyncio** | High | Standard, well-maintained |
| **anthropic SDK** | High | Official, well-maintained, need provider-specific features |
| **openai SDK** | High | Universal local model interface, well-maintained |

### Keep with Risk Mitigation

| Library | Risk | Mitigation |
|---------|------|------------|
| **telnetlib3** | Low maintenance activity; no GMCP support | Pin version; build clean abstraction layer in `client/connection.py` so transport is swappable; implement GMCP as a separate layer on top of sub-negotiation hooks |
| **aiosqlite** | Low maintenance activity | Library is "done" (stable, thin wrapper); pin version; enable WAL mode; add proper indexes on timestamp/session_id columns |

### Add

| Library | Purpose | When |
|---------|---------|------|
| **textual-plotext** | Charts and plots in the TUI | Phase 2 — when analytics pane is built |
| **plotext** | Terminal plotting (dependency of textual-plotext) | Phase 2 |
| **pytest-cov** | Test coverage reporting | Phase 1 |
| **pytest-timeout** | Prevent hanging async tests | Phase 1 |
| **DuckDB** | Analytical SQL queries against memory.db | Phase 2 — when `analyze_events` tool is built |
| **pydantic-settings** | Settings management (separate from pydantic in v2) | Phase 1 — if not already planned |

### Evaluate Further

| Library | What to Evaluate | When |
|---------|-----------------|------|
| **pydantic-ai** | Could unify anthropic+openai SDKs into single Pydantic-native provider abstraction. Check if it supports custom routing, grammar constraints, provider-specific features (prompt caching, extended thinking) | Before Phase 1 coding of `llm/` module |
| **NiceGUI** | For the web dashboard (Option C). Check if it's mature enough for a research monitoring dashboard | Phase 3 planning |
| **polars** | If pandas import time or performance becomes a bottleneck | Phase 2-3 |
| **hypothesis** | Property-based testing for parser robustness | Phase 2 |

### Future Architecture Decision: Hybrid TUI+GUI

| Phase | UI Strategy |
|-------|-------------|
| **Phase 1-2** | Pure TUI. Textual handles everything. ASCII/Unicode maps, textual-plotext for basic charts. |
| **Phase 3** | Evaluate whether research visualization needs justify a web dashboard. If yes: FastAPI + HTMX + Plotly/D3, running as optional companion to the TUI. Consider NiceGUI as a rapid development option. |
| **Phase 4+** | Web dashboard with interactive map graph (D3 force layout), scaffold relationship visualization, agent analytics suite. TUI remains primary gameplay interface. |

### Do NOT Add

| Library | Why Not |
|---------|---------|
| **LangChain / LangGraph** | Adds abstraction LLMUD doesn't need. Custom provider layer is simpler and more transparent. |
| **LiteLLM** | Good idea but conflicts with LLMUD's need for provider-specific features (prompt caching, grammar constraints, extended thinking). The custom router in `llm/router.py` is the right approach. |
| **Electron / Tauri** | Massive complexity increase for marginal benefit. The TUI+web dashboard approach is better. |
| **Full web UI framework (React, Vue)** | Against project identity. LLMUD is a terminal MUD client. |
| **Redis** | No need for a cache server. In-memory state + SQLite suffices. |
| **PostgreSQL** | Overkill for local-first architecture. |

---

## Appendix A: Dependency Risk Matrix

| Library | Maintenance Risk | Replacement Difficulty | Mitigation Priority |
|---------|-----------------|----------------------|-------------------|
| textual | Low (commercial backing) | High (deeply integrated in TUI) | Low |
| telnetlib3 | **Medium** (sparse releases) | Medium (abstracted in connection.py) | **High** — build clean abstraction |
| pydantic | Very Low (commercial backing) | Very High (used everywhere) | None needed |
| aiosqlite | Low-Medium (stable but quiet) | Low (thin wrapper, easy to replace) | Low |
| python-frontmatter | Low (simple, stable) | Very Low (trivial to replace) | None needed |
| anthropic SDK | Very Low (official) | Low (abstracted in provider.py) | None needed |
| openai SDK | Very Low (official) | Low (abstracted in provider.py) | None needed |
| pytest | Very Low (ecosystem standard) | Very High (all tests depend on it) | None needed |

## Appendix B: Import Time Budget

For a TUI application, startup time matters. Estimated cold import times:

| Library | Approx Import Time | Strategy |
|---------|-------------------|----------|
| textual | ~300ms | Unavoidable (TUI framework) |
| pydantic | ~200ms | Unavoidable (core models) |
| rich | ~150ms | Imported by textual |
| anthropic | ~100ms | Load when first API call is made |
| openai | ~100ms | Load when first API call is made |
| aiosqlite | ~10ms | Negligible |
| python-frontmatter | ~10ms | Negligible |
| pandas | ~1000-2000ms | **Lazy load only** — import in analyze_events tool |
| scikit-learn | ~2000-3000ms | **Lazy load only** — import in analysis tool |
| duckdb | ~200ms | **Lazy load only** — import in analysis tool |
| plotext | ~100ms | Lazy load when analytics pane is opened |

**Target:** Main TUI should be responsive within 1-2 seconds of launch. Lazy-load anything not needed for initial gameplay display.

## Appendix C: Version Pinning Strategy

Given the maintenance concerns with some libraries, use conservative pinning in `pyproject.toml`:

```toml
[project]
dependencies = [
    "textual>=1.0,<2.0",          # Pin major version, allow minor updates
    "telnetlib3>=2.0,<3.0",       # Pin major version
    "pydantic>=2.0,<3.0",         # Pin major version
    "pydantic-settings>=2.0,<3.0",
    "aiosqlite>=0.19,<1.0",       # Pin minor floor
    "python-frontmatter>=1.0,<2.0",
    "anthropic>=0.30,<1.0",       # Anthropic SDK, allow updates
    "openai>=1.0,<2.0",           # OpenAI SDK v1.x
]

[project.optional-dependencies]
analysis = [
    "pandas>=2.0,<3.0",
    "duckdb>=0.9,<2.0",
]
ml = [
    "scikit-learn>=1.3,<2.0",
]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "pytest-cov>=4.0",
    "pytest-timeout>=2.0",
]
dashboard = [
    # Phase 3: web dashboard dependencies
    # "fastapi>=0.100",
    # "uvicorn>=0.20",
]
```

[VERIFY] All version numbers against current PyPI releases before using.
