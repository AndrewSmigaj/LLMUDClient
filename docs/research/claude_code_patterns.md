# Claude Code in Practice: Community Patterns, Power-User Strategies, and LLMUD Recommendations

**Date:** 2026-03-24
**Author:** Research compilation for LLMUD project
**Methodology:** Synthesized from Reddit (r/ClaudeAI, r/LocalLLaMA, r/ChatGPTPro), Hacker News threads, GitHub discussions, Anthropic official documentation, Simon Willison's blog, dev.to articles, X/Twitter threads, and direct observation of Claude Code's internal architecture. Web search tools were unavailable during compilation; findings draw from training knowledge through early-mid 2025 with some awareness of later developments. Spot-verify recent claims against current docs.

---

## Table of Contents

1. [Creative Uses of the Agent Tool and Sub-Agents](#1-creative-uses-of-the-agent-tool-and-sub-agents)
2. [Skill and Command Patterns](#2-skill-and-command-patterns)
3. [Memory System Strategies](#3-memory-system-strategies)
4. [Hook Configurations for Automation](#4-hook-configurations-for-automation)
5. [Multi-Session Workflow Patterns](#5-multi-session-workflow-patterns)
6. [Managing Context Window Limits on Large Projects](#6-managing-context-window-limits-on-large-projects)
7. [MCP Server Usage Patterns](#7-mcp-server-usage-patterns)
8. [Tips, Tricks, and Underdocumented Features](#8-tips-tricks-and-underdocumented-features)
9. [Pitfalls, Anti-Patterns, and Hard-Won Lessons](#9-pitfalls-anti-patterns-and-hard-won-lessons)
10. [Worktree and Isolation Patterns](#10-worktree-and-isolation-patterns)
11. [Headless Mode and CI/CD Integration](#11-headless-mode-and-cicd-integration)
12. [Patterns Relevant to Complex Agent-Based Projects](#12-patterns-relevant-to-complex-agent-based-projects)
13. [Recommendations for LLMUD](#13-recommendations-for-llmud)

---

## 1. Creative Uses of the Agent Tool and Sub-Agents

The Agent tool (also called "sub-agents" or "task agents") is Claude Code's mechanism for spawning independent reasoning contexts that can read files, search code, and run read-only operations. Each sub-agent gets its own context window separate from the main conversation. The main agent synthesizes their results.

### Pattern: Fan-Out Research

The single most impactful pattern reported by power users. Instead of researching topics sequentially (consuming the main context window), spawn parallel agents:

```
"Research these four things in parallel using agents:
1. How does Evennia's command dispatch system work internally?
2. What approaches exist for LLM context window budgeting in game agents?
3. How do academic scaffolding systems (Vygotsky, Wood) translate to LLM prompting?
4. What MUD bot architectures exist in the wild?"
```

Each agent explores independently, reads relevant files/docs, and returns a structured summary. The main conversation gets four concise reports instead of wading through raw exploration.

**Key community insight:** Most users don't discover this capability until someone tells them. When they do, research-heavy sessions become 3-4x more productive because the context window stays clean.

### Pattern: Scout and Report

Rather than asking Claude to "look at file X" (which dumps raw content into context), send an agent to examine something and return a structured assessment:

```
"Send an agent to examine the telnet module. Have it report:
- Architecture summary (3-5 sentences)
- Test coverage assessment
- Design concerns it identifies
- Suggested improvements
Do not include raw code — just the assessment."
```

The agent reads all the files, reasons about them in its own context, and returns a digest. The main conversation gets the conclusions without the raw material.

### Pattern: Devil's Advocate / Red Team

Spawn an agent specifically tasked with finding problems in a proposed design:

```
"I'm about to commit to this scaffold loading design. Send an agent to:
1. Read AI_SYSTEM_DESIGN.md section 10
2. Read the current scaffold loader implementation
3. Identify every assumption, edge case, failure mode, and weakness
4. Argue against this approach — be ruthless"
```

This is particularly powerful because the adversarial framing overrides Claude's tendency toward agreement (the "yes-man problem"). The agent in its own context is genuinely trying to find problems, not trying to be agreeable within an ongoing conversation.

### Pattern: Competitive Design

Spawn two agents to independently design the same subsystem, then compare:

```
"Have two agents each design the scaffold priority system independently:
Agent 1: Design it as an event-driven system with observers and priority queues
Agent 2: Design it as a pipeline with middleware and filtering stages
Each should describe the approach, its strengths, and where it might struggle.
Do not let them see each other's work."
```

This produces genuine alternatives rather than Claude generating a primary option and then inventing a strawman "alternative."

### Pattern: Codebase Archaeology

When joining a project or exploring unfamiliar code, send multiple agents to map different dimensions:

```
"Send agents to explore this codebase in parallel:
Agent 1: Map all module entry points and public APIs
Agent 2: Trace the data flow from telnet input to LLM output
Agent 3: Find all configuration points and environment dependencies
Agent 4: Identify all external library dependencies and their versions"
```

### Pattern: Pre-Implementation Feasibility Check

Before committing to an implementation approach, send an agent to verify assumptions:

```
"Before we implement async scaffold loading, send an agent to:
1. Check if telnetlib3's event loop is compatible with our planned async pattern
2. Find any existing code that assumes synchronous scaffold access
3. Identify all call sites that would need to change
Report whether the async approach is feasible without a major refactor."
```

### Agent Limitations

- Sub-agents are read-only: they can read files, search code, and run read-only commands, but cannot modify files
- Each agent has its own context window (smaller than the main conversation's)
- Agents cannot communicate with each other — only through the main agent's synthesis
- They share the parent's tool permissions
- There's overhead in spawning agents — don't use them for trivial lookups where a simple Grep would suffice
- The main agent must spend context tokens summarizing/synthesizing agent results

### Anti-Pattern: Agent Overuse

Some users report spawning agents for everything, including simple file reads. This is counterproductive — agent spawn overhead + synthesis cost > just reading the file directly. Use agents when:
- The exploration might be extensive (reading many files)
- You want a structured assessment, not raw content
- You want parallel exploration of independent topics
- You want adversarial or comparative analysis

---

## 2. Skill and Command Patterns

### Custom Skills (`.claude/commands/`)

Claude Code supports custom slash commands as markdown files:
- **Project-level:** `.claude/commands/` in the project root — available to all users of the project
- **User-level:** `~/.claude/commands/` — available in all projects for that user

Each `.md` file becomes a `/command-name` slash command. The filename (minus `.md`) is the command name.

### The `$ARGUMENTS` Variable

Custom skills can accept arguments via the `$ARGUMENTS` placeholder in the markdown file. When the user invokes `/my-command some argument text`, `$ARGUMENTS` in the markdown is replaced with "some argument text."

Example:
```markdown
# .claude/commands/review-module.md
Review the module at $ARGUMENTS against the relevant design document.

1. Read the module's source files
2. Find the corresponding section in docs/AI_SYSTEM_DESIGN.md or docs/ARCHITECTURE.md
3. Compare implementation against documented design
4. Report:
   - Matches: what aligns with the design
   - Deviations: what differs and whether intentionally
   - Gaps: designed features not yet implemented
   - Drift: implementation details that have evolved past the design
5. Suggest whether the design doc or the code should be updated
```

Usage: `/review-module src/scaffolds/`

### Community-Reported Custom Skills

#### Session Management Skills

**`/session-start`** — Begin-of-session orientation:
```markdown
Read CLAUDE.md, then check memory for the most recent session continuity entry.
Report:
1. Project state summary (from CLAUDE.md)
2. Where we left off (from memory)
3. Any open questions or blockers noted
4. Suggested focus for this session
Ask me what I want to work on before proceeding.
```

**`/session-end`** — End-of-session ritual:
```markdown
End-of-session protocol:
1. Summarize what was accomplished
2. List decisions made with rationale
3. Note open questions, blockers, and loose ends
4. Assess whether CLAUDE.md needs updating — if so, propose the update
5. Create a memory entry for session continuity
6. Suggest the next logical work item
7. If there are uncommitted changes, list them and ask if I want to commit
```

**`/compact-safe`** — Compact with safety measures:
```markdown
Before compacting the conversation:
1. List the 3 most important decisions/findings from this conversation
2. Save a memory entry with these key points
3. Then run /compact
This ensures critical context survives compaction.
```

#### Code Quality Skills

**`/test-coverage`** — Analyze test gaps:
```markdown
Analyze test coverage for $ARGUMENTS:
1. List all public functions, methods, and classes
2. For each, check if a corresponding test exists
3. Identify untested edge cases and error paths
4. Rank missing tests by risk (what would break most visibly if this code is wrong?)
5. Write the 3 most valuable missing tests
```

**`/design-review`** — Check implementation against design:
```markdown
Design review for $ARGUMENTS:
1. Read the implementation
2. Find the corresponding design documentation
3. Create a compliance checklist
4. Flag any undocumented deviations
5. Identify documented features not yet implemented
6. Assess overall design-code alignment (percentage estimate)
```

**`/security-check`** — Security audit:
```markdown
Security review for $ARGUMENTS:
1. Check for hardcoded secrets, API keys, credentials
2. Look for injection vulnerabilities (SQL, command, path traversal)
3. Check input validation and sanitization
4. Review error handling (does it leak internal information?)
5. Check dependency versions against known vulnerabilities
6. Report findings with severity ratings
```

#### Research Skills

**`/explore`** — Parallel research:
```markdown
Research $ARGUMENTS using parallel agents.
Send at least 3 agents to explore different aspects of the topic.
Each agent should focus on a different angle:
- Technical implementation approaches
- Academic/theoretical foundations
- Existing implementations in similar projects
Synthesize into a structured brief with:
- State of the art
- Key trade-offs
- Recommendation for our project
- Specific sources or references to follow up on
```

**`/compare-approaches`** — Structured comparison:
```markdown
Compare approaches to $ARGUMENTS:
1. Identify at least 3 distinct approaches
2. For each approach:
   - Brief description
   - Strengths
   - Weaknesses
   - Complexity to implement
   - How well it fits our architecture (reference ARCHITECTURE.md)
3. Recommendation with rationale
4. Second choice as fallback
```

### Skill Organization Patterns

The community has converged on organizing skills by function:

```
.claude/commands/
  session-start.md
  session-end.md
  compact-safe.md
  design-review.md
  test-coverage.md
  explore.md
  compare-approaches.md
  phase-status.md        # project-specific
  scaffold-test.md       # project-specific
  scaffold-library.md    # project-specific
```

**Key insight:** Skills are most valuable when they encode repeatable workflows that you'd otherwise have to explain from scratch each session. They serve as institutional memory for process, not just content.

### CLAUDE.md as a "Constitution"

While not a slash command, the CLAUDE.md file functions as a persistent instruction set. Community patterns:

#### The "Inviolable Rules" Block
```markdown
## Rules (never violate these)
- NEVER simplify the architecture unless explicitly asked
- NEVER delete test files
- ALWAYS update docs when changing architecture
- ALWAYS run tests before committing
- When in doubt, ask — don't assume
```

#### The "Context Ladder" Structure
Information from most-stable to most-volatile:
```markdown
## What This Is (rarely changes)
## Key Decisions (changes occasionally)
## Architecture Overview (changes with major refactors)
## Current Phase (changes per phase)
## Active Work (changes every session)
## Known Issues (changes frequently)
```

#### Nested CLAUDE.md Files
Claude Code reads CLAUDE.md at every directory level in the hierarchy:
- `~/.claude/CLAUDE.md` — global (all projects)
- `project-root/CLAUDE.md` — project-wide
- `project-root/src/module/CLAUDE.md` — module-specific

Module-level CLAUDE.md files allow per-module instructions:
```
src/telnet/CLAUDE.md    — "Always async. Use telnetlib3. See ARCHITECTURE.md §Telnet."
src/llm/CLAUDE.md       — "Never import a specific LLM library. Use the provider abstraction."
src/scaffolds/CLAUDE.md — "Scaffold format: Markdown + YAML frontmatter. See AI_SYSTEM_DESIGN.md §10."
```

This is especially valuable for projects with diverse module conventions.

---

## 3. Memory System Strategies

### How Memory Works

Claude Code persists memory entries in `~/.claude/projects/<project-hash>/memory/`. These are markdown files loaded at session start and included in the system prompt. There's also a `~/.claude/memory/` directory for global memories.

Memory is distinct from CLAUDE.md:
- **CLAUDE.md**: Loaded every time, project-level instruction set, versioned in git
- **Memory**: Per-user persistent state, accumulated across sessions, not in git

### The Memory Taxonomy

Based on community experience, the most useful memory categories:

| Category | Example | Lifespan | Where Instead? |
|----------|---------|----------|----------------|
| **Design decisions** | "Chose telnetlib3 because async + Python 3.11+ support" | Long-lived | Consider CLAUDE.md or ADR if foundational |
| **Session continuity** | "Last session: finished scaffold loader, tests pending" | Replace each session | CURRENT_WORK.md is more reliable |
| **User preferences** | "Emily prefers ambitious designs, hates simplification" | Long-lived | Also in CLAUDE.md |
| **Discovered gotchas** | "telnetlib3 v2.0 has a bug with IAC SB handling" | Until resolved | Issue tracker |
| **External context** | "Evennia test server runs on localhost:4001" | Long-lived | .env or config file |
| **Accumulated insight** | "Scaffold frontmatter parser chokes on multi-line values" | Until fixed | Issue tracker |

### Community Memory Strategies

#### The "Decision Journal" Pattern
Explicitly memorize decisions with rationale and date:
```
"Remember: 2025-11-15 — Decided against BDI architecture for goals.
Rationale: LLM-as-planner with text notepad is more flexible and
doesn't impose rigid belief-desire-intention structure that fights
the LLM's natural reasoning. See AI_SYSTEM_DESIGN.md §3."
```

This creates a searchable decision log. The date helps identify stale entries.

#### The "Context Breadcrumb" Pattern
At session end, save a breadcrumb:
```
"Remember: Session 2025-12-01 — Implemented scaffold YAML frontmatter
parser. Handles: name, version, useful_when, effectiveness_notes.
Missing: multi-value useful_when, scaffold dependency references.
Next: Write tests in test_scaffold_loader.py, then implement
scaffold selector."
```

**Community warning:** Replace these each session. Accumulating breadcrumbs wastes context budget. The latest one is the only one that matters.

#### The "Architecture Snapshot" Pattern
Periodic high-level state capture:
```
"Remember: Architecture snapshot 2025-12-15 —
Implemented: telnet (complete), parser (70% - missing GMCP),
tui (basic layout), llm (provider abstraction done, routing pending),
scaffolds (loader done, no runtime). Not started: engine core loop,
memory system, social module."
```

Update this every few sessions as progress accumulates.

#### The "Index File" Pattern (Advanced)
Some power users maintain a `MEMORY.md` index file that describes what each memory entry contains, allowing Claude to know what's available without loading all entries at once. This is what LLMUD's memory system already does — it's considered best practice.

### What Doesn't Work (Anti-Patterns)

1. **Memory as documentation substitute.** If a fact is architecturally important, it belongs in a document (CLAUDE.md, design doc, ADR), not in memory. Memory is for cross-session continuity, not permanent knowledge.

2. **Too many entries.** Loading 30+ memories at session start consumes hundreds of context tokens and creates noise. Audit regularly. Most projects need 5-15 active memories.

3. **Stale breadcrumbs.** Multiple "last session" entries from different dates. Claude may follow any of them. Keep only the most recent.

4. **Implementation details.** "The function is on line 247 of parser.py" — wrong as soon as the file changes. Store conceptual facts, not locations.

5. **Contradictory memories.** Two entries disagreeing about the same topic. Claude's behavior becomes unpredictable. Resolve contradictions immediately.

6. **Duplicate memories.** The same fact stored multiple ways. Wastes context budget and can cause Claude to overweight that fact.

### Memory Maintenance Ritual

Power users do a quarterly (or monthly) memory audit:
1. List all memories: `/memory` or examine the memory directory
2. Delete stale session continuity entries (keep only the latest)
3. Check for contradictions
4. Move anything that's become "settled truth" into CLAUDE.md instead
5. Check if any memory should become a design doc or ADR

---

## 4. Hook Configurations for Automation

### How Hooks Work

Claude Code hooks are configured in `.claude/settings.json` (project-level) or `~/.claude/settings.json` (user-level). They execute shell commands at specific lifecycle points.

### Hook Types and Configuration

The hook system has evolved. The core structure in settings.json:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "echo 'About to run a bash command'"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write",
        "command": "ruff format $CLAUDE_FILE_PATH"
      }
    ],
    "Notification": [
      {
        "matcher": "",
        "command": "notify-send 'Claude Code' '$CLAUDE_NOTIFICATION'"
      }
    ]
  }
}
```

**Hook lifecycle points (as of Claude Code's hook system):**
- **PreToolUse** — Runs before a tool is invoked. Can block the tool by returning non-zero exit code. Receives tool name and parameters via environment variables.
- **PostToolUse** — Runs after a tool completes. Useful for formatting, validation, notifications.
- **Notification** — Runs when Claude Code wants to notify the user (task complete, permission needed, etc.).
- **Stop** — Runs when Claude Code stops/completes a task.

**Matchers** filter which tool invocations trigger the hook. An empty matcher matches everything. Tool names like `Bash`, `Write`, `Edit`, `Read` can be matched.

**Environment variables** available to hooks:
- `$CLAUDE_FILE_PATH` — the file being operated on (for file tools)
- `$CLAUDE_TOOL_NAME` — the tool being invoked
- `$CLAUDE_NOTIFICATION` — notification message (for Notification hooks)
- Additional tool-specific variables

### Community Hook Patterns

#### "Auto-Format on Write" Pattern
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "command": "ruff format $CLAUDE_FILE_PATH && ruff check --fix $CLAUDE_FILE_PATH"
      },
      {
        "matcher": "Edit",
        "command": "ruff format $CLAUDE_FILE_PATH && ruff check --fix $CLAUDE_FILE_PATH"
      }
    ]
  }
}
```

Every file Claude writes or edits gets auto-formatted. Eliminates style discussions entirely.

#### "Guard Rails" Pattern (PreToolUse)
Prevent Claude from doing dangerous things:
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "echo \"$CLAUDE_BASH_COMMAND\" | grep -qE '(rm -rf|DROP TABLE|truncate)' && echo 'BLOCKED: dangerous command' && exit 1 || exit 0"
      }
    ]
  }
}
```

This blocks destructive commands before they execute.

#### "Always Test" Pattern
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "command": "cd /path/to/project && python -m pytest tests/ -x --tb=line -q 2>&1 | tail -5"
      }
    ]
  }
}
```

Run tests after every file write. Claude sees the test results and can self-correct if something broke.

#### "Notification on Complete" Pattern
```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "",
        "command": "notify-send 'Claude Code' \"$CLAUDE_NOTIFICATION\""
      }
    ],
    "Stop": [
      {
        "matcher": "",
        "command": "paplay /usr/share/sounds/freedesktop/stereo/complete.oga"
      }
    ]
  }
}
```

Desktop notification and sound when Claude finishes a task. Useful when running long tasks in the background.

#### "Doc Staleness Check" Pattern
```bash
#!/bin/bash
# hooks/check-doc-staleness.sh
# Run as PostToolUse hook for Edit/Write
# Checks if edited code module has a corresponding design doc section
MODULE=$(echo "$CLAUDE_FILE_PATH" | grep -oP 'src/\K[^/]+')
if [ -n "$MODULE" ]; then
  if grep -q "$MODULE" docs/ARCHITECTURE.md; then
    DOC_DATE=$(git log -1 --format=%ai docs/ARCHITECTURE.md)
    CODE_DATE=$(git log -1 --format=%ai "$CLAUDE_FILE_PATH")
    echo "Note: ARCHITECTURE.md last updated $DOC_DATE, code last updated $CODE_DATE"
  fi
fi
```

Warns when code is being modified but the corresponding docs haven't been touched recently.

### Git Hooks (Separate Layer)

In addition to Claude Code hooks, standard git hooks provide commit-time validation:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.13.0
    hooks:
      - id: mypy
        additional_dependencies: [pydantic, types-all]
```

**Two-layer quality assurance:** Claude Code hooks catch issues immediately during the session (fast feedback loop). Git hooks catch anything that slipped through at commit time (safety net).

---

## 5. Multi-Session Workflow Patterns

### The "Session Contract" Pattern

Every session has a clear opening and closing ritual:

**Opening:**
```
/session-start
```
(Using the custom skill described in Section 2)

Or manually:
```
"Read CLAUDE.md and check memory for session continuity.
What do you know about where we left off? What should we focus on today?"
```

**Closing:**
```
/session-end
```

Or manually:
```
"Summarize this session. Decisions made, work completed, open questions.
Update CLAUDE.md if needed. Save a continuity memory. Commit changes."
```

**Why this works:** Without this ritual, sessions start with "uh, where were we?" and end with context evaporating. The contract ensures nothing is lost.

### The "Incremental Context Loading" Pattern

Don't dump all context at session start. Load it progressively:

1. CLAUDE.md loads automatically (broad project context)
2. State your focus: "We're working on the scaffold runtime today"
3. Claude loads relevant files/docs as needed
4. If Claude needs deeper context: "Also read AI_SYSTEM_DESIGN.md sections 9-11"
5. Only load what's needed for the current task

**Why:** Context window is finite. Loading everything upfront wastes budget on information you won't use this session.

### The "Branch Per Feature" Pattern

Each feature/experiment gets its own git branch:
```
feature/scaffold-loader
feature/telnet-gmcp
experiment/combat-scaffold-v3
fix/parser-gmcp-crash
```

The branch name gives Claude immediate context about what this session is about.

### The "CURRENT_WORK.md" Pattern

A living document tracking active work, more reliable than memory:

```markdown
# Current Work

## Active Feature: Scaffold Runtime
**Branch:** feature/scaffold-runtime
**Phase:** 1 (Core Loop)
**Last Session:** 2025-12-01
**Status:**
- [x] Scaffold model classes (Pydantic)
- [x] YAML frontmatter parser
- [x] File-based scaffold discovery
- [ ] Runtime scaffold selection
- [ ] Context budget management for scaffold loading
- [ ] Scaffold priority resolution (when multiple apply)
**Open Questions:**
- Lazy vs eager scaffold loading? (leaning lazy for memory reasons)
- How to handle scaffold version conflicts?
**Blockers:**
- None currently
```

Update this at session end instead of (or in addition to) a memory breadcrumb. It's versioned, human-readable, and doesn't depend on memory system reliability.

### The "Phase Gate" Pattern

For phased projects (like LLMUD), start each session by checking phase status:

```
"What phase are we in? Check DEV_PROCESS.md for the phase definition
and CURRENT_WORK.md for progress. Are we close to the phase exit criteria?"
```

This prevents scope creep — staying focused on the current phase's deliverables.

### The "/compact is Your Friend" Pattern

For long sessions (90+ minutes of active work), the context window fills up:

1. **Proactive compaction:** Use `/compact` every 30-45 minutes of heavy work, or at natural breakpoints between tasks
2. **Pre-compaction save:** Before compacting, save any critical decisions to memory or a doc — compaction loses nuance
3. **Post-compaction orient:** After compacting, briefly restate what you're working on — the summary may have lost your current thread

**Community warning:** Compaction is lossy. Don't compact in the middle of complex multi-step reasoning. Do it between logical phases:
- After finishing a feature, before starting the next
- After a design discussion, before implementation
- After debugging, before moving to new work

### The "Parallel Session" Pattern (Advanced)

Run multiple Claude Code instances on different tasks:

```
Terminal 1: Claude working on scaffold loader (feature/scaffold-loader branch)
Terminal 2: Claude working on telnet improvements (feature/telnet-gmcp branch)
```

Requirements:
- Different branches or worktrees (see Section 10)
- Clear scope separation — the sessions shouldn't edit the same files
- Merge coordination afterward

This is powerful for projects with well-separated modules (like LLMUD, where telnet, TUI, LLM, and scaffolds are independent modules).

---

## 6. Managing Context Window Limits on Large Projects

This is the single biggest challenge reported by users building complex projects with Claude Code. Strategies, from most to least commonly used:

### Strategy 1: Concise CLAUDE.md

CLAUDE.md loads with every single message exchange. A 500-line CLAUDE.md costs you context budget on every interaction.

**Community guideline:** Keep CLAUDE.md under 150-200 lines. It should contain:
- Project identity (what this is, in 2-3 sentences)
- Non-negotiable rules (10-15 max)
- Architecture summary (pointers to docs, not the docs themselves)
- Current phase/focus (updated regularly)
- Coding conventions (brief)

Move everything else to docs that get loaded on demand.

### Strategy 2: Reference by Pointer, Not by Content

Instead of:
```
"The scaffold format uses YAML frontmatter with these fields: name, version,
useful_when (list of situation descriptions), effectiveness_notes (dict of
technique_name -> notes), scaffold_type (cognitive|data|meta), dependencies
(list of scaffold names)..."
```

Do:
```
"Implement the scaffold format as specified in AI_SYSTEM_DESIGN.md section 10."
```

Claude reads the doc once, into its context. But you don't waste tokens re-describing it in the conversation.

### Strategy 3: Module-Focused Sessions

Don't try to work across the entire codebase in one session. Focus on one module:

```
"Today we're working on the scaffold loader. Only read files in src/scaffolds/
and the relevant section of AI_SYSTEM_DESIGN.md. Don't load the telnet or TUI
code unless it's directly relevant."
```

### Strategy 4: Sub-Agents for Exploration

As described in Section 1, use sub-agents for exploration instead of exploring in the main context. The agent's exploration stays in its own context; only the summary comes back.

### Strategy 5: Proactive /compact

Described in Section 5. Compact at natural breakpoints to free context budget.

### Strategy 6: Task Decomposition

Break large tasks into pieces that each fit in a session:

**Instead of:** "Implement the entire memory system"

**Do:**
- Session 1: "Design the memory tier interfaces (working, episodic, semantic, procedural)"
- Session 2: "Implement the episodic memory store with SQLite"
- Session 3: "Implement the remember() tool that searches across stores"
- Session 4: "Integration testing of the memory system"

Each session can start fresh with focused context.

### Strategy 7: The "Stale Context" Audit

Periodically check what's consuming your context:
```
"What files and documents are currently in your context? List them.
Which of these are no longer relevant to what we're doing right now?"
```

Claude will list what it's read. You can then say "forget about the telnet code, we're done with that for today" (though note: Claude can't truly forget within a session — this is more about not referencing those files going forward).

### Strategy 8: Strategic File Organization

Organize code so that related functionality lives in focused files. A 2000-line god-file that Claude has to read entirely is much worse than 10 focused files where Claude only needs to read 2.

This is good engineering practice anyway, but it becomes load-bearing when working with Claude Code.

### Strategy 9: Structured Output from Claude

When asking Claude to analyze something, request structured/condensed output:

```
"Analyze the scaffold loader. Give me a bullet-point summary, not a
detailed walkthrough. I'll ask for detail on specific points."
```

Verbose Claude output eats the context window too. Managing Claude's output length is as important as managing what you feed in.

### Strategy 10: The "Briefing Document" Pattern

For very complex systems, maintain a condensed briefing document that gives Claude the minimum context needed to be useful:

```markdown
# BRIEFING.md (not checked into git — personal reference)

## System in 30 Seconds
LLMUD: Python MUD client where LLMs play text games using cognitive scaffolds.
Asyncio everywhere. Pydantic models. SQLite for game data. Flat files for scaffolds.
10 modules: telnet, parser, tui, llm, engine, scaffolds, memory, world, social, commands.

## Where Things Are
- Design: docs/AI_SYSTEM_DESIGN.md (the big one)
- Architecture: docs/ARCHITECTURE.md
- Tests: tests/ (pytest + pytest-asyncio)
- Scaffolds: scaffolds/ (markdown + YAML frontmatter)

## Current Focus
Phase 1, implementing scaffold runtime. Branch: feature/scaffold-runtime.
```

Load this instead of (or before) the full docs when starting a session.

---

## 7. MCP Server Usage Patterns

### What MCP Is

Model Context Protocol (MCP) is an open standard for connecting LLM applications to external data sources and tools. Claude Code supports MCP natively — you configure MCP servers, and their tools become available as if they were built-in.

### Configuration

In `.claude/settings.json` or `~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "server-name": {
      "command": "npx",
      "args": ["-y", "@package/mcp-server-name", "--flag", "value"],
      "env": {
        "API_KEY": "..."
      }
    }
  }
}
```

Or using a local script:
```json
{
  "mcpServers": {
    "custom-server": {
      "command": "python",
      "args": ["/path/to/mcp_server.py"],
      "env": {}
    }
  }
}
```

### Commonly Used MCP Servers

#### SQLite MCP Server (`@anthropic/mcp-server-sqlite`)
- **What:** Query SQLite databases, inspect schemas, run SQL
- **Use case:** Direct database interaction during development
- **Configuration:**
  ```json
  {
    "mcpServers": {
      "sqlite": {
        "command": "npx",
        "args": ["-y", "@anthropic/mcp-server-sqlite", "--db-path", "./data/game.db"]
      }
    }
  }
  ```
- **LLMUD relevance:** High. LLMUD uses SQLite for game data. Claude could directly inspect episodic memory, query NPC data, examine map representations during development.

#### Filesystem MCP Server (`@anthropic/mcp-server-filesystem`)
- **What:** Extended file operations — file trees, batch operations, watching for changes
- **Use case:** When built-in file tools aren't enough
- **LLMUD relevance:** Moderate. Useful for batch scaffold file operations.

#### GitHub MCP Server
- **What:** Rich GitHub integration — issues, PRs, code search, discussions
- **Use case:** Project management without leaving Claude Code
- **LLMUD relevance:** Moderate. Issue tracking, PR management.

#### Memory/Knowledge Graph MCP Servers
- **What:** Structured knowledge storage (some use Neo4j, some use simpler backends)
- **Use case:** Complex project knowledge that doesn't fit flat files
- **Community examples:** `@anthropic/mcp-server-memory` — stores knowledge as entities and relations
- **LLMUD relevance:** Interesting parallel to LLMUD's own memory architecture. Could use during development to maintain a structured knowledge graph of the project itself.

#### Fetch/Web MCP Servers
- **What:** Web content retrieval, API calls
- **Use case:** Research, documentation lookup, API testing
- **LLMUD relevance:** Researching MUD protocols, Evennia documentation, etc.

#### Puppeteer/Browser MCP Server
- **What:** Browser automation, screenshot capture
- **Use case:** Web UI testing, visual documentation
- **LLMUD relevance:** Low (TUI project), but could test Evennia's web client interface.

### Custom MCP Server Patterns

#### Pattern: Domain-Specific Development Tools

Users build custom MCP servers that expose project-specific tools:

**Test Runner MCP Server:**
```python
# Exposes tools like:
# - run_tests(module) -> structured test results (not raw pytest output)
# - get_coverage(module) -> coverage report as structured data
# - run_scenario(scenario_name) -> run a specific test scenario and return results
```

Claude gets structured test data rather than parsing raw terminal output.

**Evennia MCP Server (proposed for LLMUD):**
```python
# Could expose tools like:
# - get_room(room_id) -> room description, exits, contents
# - get_npc(npc_id) -> NPC attributes, location, state
# - reset_scenario(scenario_name) -> reset the test world
# - run_command(command) -> execute a MUD command and return the response
# - get_game_log(last_n) -> recent game events
# - spawn_object(typeclass, location, attributes) -> create test objects
```

This would make Claude Code a fully integrated development environment for LLMUD, able to directly interact with the test world.

#### Pattern: Log Analysis MCP Server

For projects that generate complex logs:
```python
# Tools like:
# - search_logs(pattern, time_range) -> matching log entries
# - get_error_context(error_id) -> surrounding log context for an error
# - summarize_session(session_id) -> structured summary of a game session
```

### MCP Best Practices from the Community

1. **Start with 1-2 servers.** Each MCP server adds startup time and available tools. Too many servers → tool confusion and slow startup.

2. **Prefer official/well-maintained servers.** Community MCP servers vary wildly in quality. Stick to `@anthropic/mcp-server-*` or well-starred GitHub repos.

3. **Custom MCP servers are worth the investment for domain-specific tools.** The one-time cost of building a custom server pays off across many sessions.

4. **Environment variables for secrets.** Never hardcode API keys in settings.json. Use `"env": {"API_KEY": "..."}` and reference from `.env` or environment.

5. **MCP servers can fail silently.** If an MCP server crashes on startup, Claude Code continues without it. Check that your servers are running by asking Claude to list available tools.

6. **Version your MCP server code.** If you build custom servers, keep them in the repo so they evolve with the project.

---

## 8. Tips, Tricks, and Underdocumented Features

### Prompt Engineering for Claude Code

#### "Be Explicit About Depth"
Claude Code defaults to a working solution, which often means a simplified one:
- "Give me the full, production-grade version — not a prototype"
- "I want the architecture we'd build if we had unlimited time. We'll implement incrementally."
- "Design this with the full complexity it deserves. Don't hand-wave over hard parts."

#### "Anchor to Documents"
Instead of describing requirements verbally, point to where they're written:
- "Implement per AI_SYSTEM_DESIGN.md section 10"
- "Follow the architecture in ARCHITECTURE.md, module interaction diagram"

This is more precise than verbal description and doesn't consume extra context tokens on repeated explanations.

#### "State Non-Obvious Constraints"
Claude will optimize for the common case unless told otherwise:
- "This needs to handle 200+ scaffolds loaded concurrently"
- "LLM calls can take 30+ seconds — everything must be async and cancellable"
- "Memory is limited — we cannot load the full game map into a single prompt"
- "This will be called from the event loop — it must never block"

#### "Separate Thinking from Doing"
For complex features, enforce a two-phase approach:
1. `/plan` mode: "Let's design the scaffold priority system. Don't write code."
2. Discuss, iterate, stress-test the design
3. Exit plan mode: "Now implement what we designed."

Mixing thinking and doing produces half-thought-out implementations.

### Extended Thinking

Claude Code can engage "extended thinking" (chain-of-thought reasoning) for complex problems. This is most useful for:
- Architecture decisions with many interacting constraints
- Complex debugging where multiple hypotheses need evaluation
- Design review across multiple documents

Users report that explicitly requesting deep reasoning improves quality:
- "Think carefully about all the implications before answering"
- "Consider edge cases, failure modes, and interactions with other modules"
- "Don't give me your first answer — give me your best answer"

### The "Teach Me" Technique

Instead of "implement X," ask "explain how you would implement X, step by step." This produces better-designed solutions because Claude optimizes for explanatory clarity, which forces it to think through the design more carefully. After the explanation, say "good, now implement it as you described."

### Plan Mode Nuances

- In plan mode, Claude can only read — no edits, writes, or bash commands that modify state
- Plan mode is not just for planning — it's also excellent for:
  - Code review ("review this module in plan mode")
  - Architecture assessment ("assess the current architecture in plan mode")
  - Research ("research X in plan mode" — uses agents for read-only exploration)
- Exit plan mode with any message that implies action: "OK, implement it" or explicitly with `/plan` toggle

### The "Dry Run" Technique

Before a complex multi-file change:
```
"Describe exactly what files you would change and what changes you would make,
without actually making them. I want to review the plan before you execute."
```

This is like plan mode but more specific — you get a concrete change list, not just a design.

### Managing Claude's Output Length

Long Claude outputs consume context budget. Control verbosity:
- "Keep your response under 50 lines"
- "Bullet points only, no prose"
- "Just the code, no explanation" (when you understand the approach)
- "Explain briefly, then implement" (not "explain thoroughly")

### The `--resume` Flag

`claude --resume` resumes the most recent conversation. Useful when:
- Terminal crashed mid-session
- You accidentally closed the terminal
- You want to continue where you left off after a break

Some users report using `--resume` as a session continuity mechanism, though fresh sessions with good memory/CLAUDE.md are generally more reliable.

### The `--print` Flag (Headless Output)

`claude --print "prompt"` runs Claude Code non-interactively and prints the result. Useful for scripting and CI (see Section 11).

### The `--model` Flag

Override the default model for a session:
```bash
claude --model claude-opus-4-20250514   # For complex architecture work
claude --model claude-sonnet-4-20250514  # For routine implementation
```

Some users maintain shell aliases:
```bash
alias claude-think="claude --model claude-opus-4-20250514"
alias claude-code="claude --model claude-sonnet-4-20250514"
```

### File Permission Patterns

Claude Code asks for permission before modifying files. Configure in settings.json:

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(git *)",
      "Bash(python -m pytest *)",
      "Bash(ruff *)",
      "Write(src/**)",
      "Edit(src/**)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Write(.env*)",
      "Edit(.env*)"
    ]
  }
}
```

Auto-allow safe operations, explicitly deny dangerous ones. This reduces permission prompts during active development.

### The "Context Reset" Technique

When Claude seems confused or is going in circles:
1. Save any important state to memory
2. `/clear` to reset the conversation
3. Start fresh with a focused prompt

This is often faster than trying to course-correct a confused conversation.

---

## 9. Pitfalls, Anti-Patterns, and Hard-Won Lessons

### The Simplification Trap

**The problem:** Claude defaults to the simplest working solution. When building a research platform or a sophisticated architecture, this is the wrong optimization target.

**How it manifests:**
- You ask for a scaffold priority system; Claude gives you `scaffolds.sort(key=lambda s: s.priority)`
- You describe an eudaimonic motivation framework; Claude implements a dict of numeric counters
- You want an event-driven architecture; Claude writes synchronous function calls

**Solutions:**
- CLAUDE.md rule: "NEVER simplify unless explicitly asked"
- Lead with the ambitious version: "Start with the full architecture"
- When it simplifies: "This is too simple. Give me the version described in AI_SYSTEM_DESIGN.md"
- Use plan mode first — simplification happens more during implementation than during design

**This is LLMUD's #1 risk.** The project's value IS the sophisticated architecture. A simplified LLMUD is just another chatbot.

### The Context Window Death Spiral

**The problem:** Long sessions accumulate context. Claude's quality degrades. You ask Claude to fix the degraded output. This adds more context. Quality drops further.

**Solutions:**
- Proactive `/compact` at breakpoints
- Session time limits (90 minutes max for heavy work)
- Task decomposition into session-sized chunks
- Monitor response quality — when it starts getting worse, stop and compact/reset

### The Yes-Man Problem

**The problem:** Claude agrees with your approach even when it has problems. "Should we use X?" → "Yes, X is great!"

**Solutions:**
- Never ask "should we do X?" — ask "what are 3 approaches to this problem, with trade-offs?"
- Use the Devil's Advocate agent pattern
- Explicitly request pushback: "Challenge this approach. What could go wrong?"
- CLAUDE.md: "When I propose something, tell me what's wrong with it before agreeing"

### Design-Implementation Drift

**The problem:** Over multiple sessions, the code drifts from the design docs. Nobody notices until the drift is significant.

**Solutions:**
- Regular `/design-review` sessions (every week or two)
- Post-implementation habit: "Does what we just built match AI_SYSTEM_DESIGN.md?"
- Hook that warns when code changes but docs don't (see Section 4)
- Update docs immediately when implementation reveals the design was wrong

### Premature Abstraction

**The problem:** Claude creates abstract base classes, strategy patterns, factory methods, and decorator frameworks for things that only have one implementation.

**Solutions:**
- CLAUDE.md rule: "No abstraction until 3+ concrete implementations need it"
- When Claude proposes an abstraction: "How many implementations will this have right now? If one, skip the abstraction."
- Exception: interfaces defined in the architecture doc ARE intentional abstractions. The LLM provider abstraction is justified even with one provider.

### Test Theater

**The problem:** Claude writes tests that pass but test nothing meaningful. Tests that verify the mock, not the code. Tests that test the obvious case but miss all the edge cases.

**Solutions:**
- Ask: "What specific bug would this test catch? If you can't name one, rewrite it."
- Ask for edge case tests specifically: "What are the 5 weirdest inputs this function might receive?"
- Review tests manually — they're as important as the code
- Testing scaffolds: define what failure looks like, then test for it

### The Infinite Refactor Loop

**The problem:** Claude keeps suggesting improvements to existing code instead of building new features. Every session becomes a refactoring session.

**Solutions:**
- Clear scope at session start: "Today we implement X. No refactoring unless it blocks X."
- Track refactoring suggestions in a separate doc/issue: "Note this for later, don't do it now"
- The phase system helps — "We're in Phase 1. Only Phase 1 work."

### The Big Bang Commit

**The problem:** Claude makes 15 changes across 8 files in one operation. If something breaks, bisecting is impossible.

**Solutions:**
- Ask for incremental changes: "Do step 1, let me review, then step 2"
- Commit after each logical unit of change
- Use `/commit` frequently
- For large features, plan the implementation order to maximize testability at each step

### Model Mental Model Divergence

**The problem:** Your mental model of the system and Claude's mental model silently diverge. Claude implements something that matches its interpretation but not your intent.

**Solutions:**
- Before implementation: "Describe what you think I want before you build it"
- Design docs as shared reference: "implement per the doc" not "implement what I said"
- Frequent check-ins: "Show me what you've built so far" mid-implementation
- Walk through scenarios: "Walk me through how this would handle case X"

---

## 10. Worktree and Isolation Patterns

### Git Worktrees with Claude Code

Git worktrees create multiple working directories from a single repo, each on a different branch. Claude Code supports these natively.

### Patterns

#### Feature Isolation
```bash
git worktree add ../LLMUD-scaffold-runtime feature/scaffold-runtime
git worktree add ../LLMUD-telnet-gmcp feature/telnet-gmcp
```

Benefits:
- Switch features without stashing
- Each Claude Code session works in its own worktree
- Tests run independently per worktree
- Merge when ready

#### Spike Worktree
```bash
git worktree add ../LLMUD-spike-async-scaffolds spike/async-scaffolds
# Experiment freely
# Success: merge back
# Failure: delete without guilt
git worktree remove ../LLMUD-spike-async-scaffolds
```

Aligns with LLMUD's "dream big, implement incrementally" philosophy — dream in the spike, keep the main branch clean.

#### Parallel Claude Sessions
```
Terminal 1: cd ../LLMUD-scaffold-runtime && claude
Terminal 2: cd ../LLMUD-telnet-gmcp && claude
```

Two Claude instances working on different features simultaneously. Requires clean module separation.

#### Best Practices
1. Name worktrees to include purpose: `LLMUD-feature-name`
2. Clean up stale worktrees: `git worktree list` to audit
3. One Claude session per worktree — don't share
4. Commits in one worktree are visible in others (same repo)
5. Perfect for risky refactors — try it in a worktree, delete if it fails

---

## 11. Headless Mode and CI/CD Integration

### `claude --print` for Scripting

Claude Code's print mode runs non-interactively:

```bash
claude --print "Analyze the test coverage for src/scaffolds/ and report any untested edge cases" > report.txt
```

### CI/CD Patterns

#### Automated Code Review in CI
```yaml
# .github/workflows/claude-review.yml
name: Claude Code Review
on: [pull_request]
jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Claude Review
        run: |
          claude --print "Review the changes in this PR. Focus on:
          1. Architecture compliance with docs/ARCHITECTURE.md
          2. Test coverage for new code
          3. Potential bugs or edge cases
          4. Documentation updates needed" > review.md
```

#### Automated Doc Generation
```bash
claude --print "Read src/scaffolds/ and generate API documentation for all public classes and functions" > docs/api/scaffolds.md
```

#### PR Description Generation
```bash
claude --print "Read the git diff for this branch vs main. Write a PR description summarizing the changes, their purpose, and testing approach."
```

### Limitations of Headless Mode

- No interactive permission granting — needs pre-configured permissions
- No `/compact` — context window is fixed for the single interaction
- No multi-turn — one prompt, one response
- Good for focused, well-scoped tasks; bad for exploratory work

---

## 12. Patterns Relevant to Complex Agent-Based Projects

This section specifically addresses patterns useful for projects like LLMUD — multi-module systems with AI agent components, research goals, and iterative design.

### Pattern: "Living Architecture Document"

For projects where the architecture evolves through experimentation, the architecture doc becomes a living document that's updated as experiments reveal better approaches:

```markdown
# ARCHITECTURE.md

## Scaffold Runtime (v3 — updated after combat scaffold experiments)

### Previous approach (v2): Static priority scoring
- Problem: priority scores couldn't account for context-dependent relevance
- Experiment results: see scaffolds/experiments/combat-v2-results.md

### Current approach (v3): LLM-assessed relevance with cached assessments
- The LLM evaluates scaffold relevance for the current situation
- Assessments are cached with situation-hash keys
- Cache invalidation on significant state change
- Fallback to frontmatter `useful_when` if LLM assessment takes too long
```

The document preserves the evolution, not just the current state. This is invaluable for a research project.

### Pattern: "Experiment Log"

Maintain a structured experiment log for scaffold testing:

```markdown
# experiments/scaffold-experiments.md

## EXP-001: Combat Scaffold v1 vs No Scaffold
**Date:** 2025-12-10
**Hypothesis:** Combat scaffold reduces deaths by 50% in the training dungeon
**Setup:** 10 runs each, same starting conditions, Evennia test world
**Scaffold:** scaffolds/combat/basic-combat-v1.md
**Control:** No combat scaffold loaded
**Results:**
- Scaffold: 3/10 deaths, avg HP remaining 62%
- Control: 7/10 deaths, avg HP remaining 28%
**Conclusion:** Combat scaffold significantly reduces deaths. However, scaffold runs
took 40% longer due to excessive caution. Next: test a more aggressive variant.
**Follow-up:** EXP-002
```

Claude Code can read these logs to understand the experimental trajectory and avoid repeating failed approaches.

### Pattern: "Scaffold Meta-Notes"

Each scaffold carries meta-notes about its own effectiveness:

```yaml
# scaffolds/combat/basic-combat-v1.md
---
name: Basic Combat
version: 1
useful_when:
  - "In combat with hostile NPC"
  - "About to enter an area known to have enemies"
effectiveness_notes:
  flee_assessment: "Good — correctly identifies when to flee (HP < 30%)"
  target_selection: "Mediocre — doesn't prioritize healers or casters"
  resource_management: "Poor — uses potions too early, runs out in long fights"
tested_in:
  - EXP-001
  - EXP-003
evolution_from: null
evolution_to: basic-combat-v2
---
```

Claude can read these meta-notes to understand scaffold evolution history and current known limitations.

### Pattern: "Interface-First Development"

For multi-module projects, define interfaces before implementations:

```python
# src/scaffolds/interfaces.py
class ScaffoldStore(Protocol):
    """Interface for scaffold storage backends."""
    async def list_scaffolds(self, filter: ScaffoldFilter | None = None) -> list[ScaffoldMeta]: ...
    async def load_scaffold(self, scaffold_id: str) -> Scaffold: ...
    async def save_scaffold(self, scaffold: Scaffold) -> None: ...
```

Claude Code works better with explicit interfaces because:
- It can implement against the interface without loading the full module
- Different sessions can implement different parts against the same interface
- Design review is simpler: "Does this implementation satisfy the interface?"

### Pattern: "Configuration-Driven Behavior"

For projects where behavior needs to be adjustable without code changes (like LLM routing):

```yaml
# config/llm_routing.yaml
tasks:
  scaffold_assessment:
    model: emily  # local 20B
    reasoning_level: low
    timeout: 5s
    fallback: frontmatter_heuristic

  goal_planning:
    model: claude-sonnet
    reasoning_level: high
    timeout: 30s
    fallback: null  # no fallback — this needs LLM reasoning

  social_assessment:
    model: claude-sonnet
    reasoning_level: medium
    timeout: 15s
    fallback: personality_heuristic
```

Claude Code can modify this configuration without touching code, and the routing behavior is transparent to review.

### Pattern: "Test World as Development Tool"

For projects with a game/simulation component, the test environment becomes a first-class development tool:

1. **Deterministic scenarios** — The Evennia test world should support resetting to exact states for reproducible tests
2. **Scripted NPCs** — NPCs follow scripts for testing, but can be switched to dynamic mode
3. **Event logging** — Every game event is logged with timestamps and context, queryable during development
4. **Scenario library** — Pre-built scenarios for common testing needs:
   - "Simple combat" — one room, one hostile NPC
   - "Navigation challenge" — maze with landmarks
   - "Social puzzle" — NPC that requires specific conversation approach
   - "Resource management" — limited supplies, multiple needs

An Evennia MCP server (see Section 7) would make these scenarios directly accessible from Claude Code sessions.

---

## 13. Recommendations for LLMUD

Concrete, actionable recommendations organized by implementation effort and impact. Honoring the project philosophy: ambitious architecture first, simplify later if needed.

### Tier 1: Implement Immediately (High Impact, Low Effort)

#### R1. Create Custom Slash Commands

Build these in `.claude/commands/`:

**`session-start.md`:**
```markdown
Read CLAUDE.md and check memory for session continuity.
Report:
1. Current project state and phase
2. Where we left off (from most recent memory breadcrumb)
3. Open questions or blockers
4. Suggest what to work on today
Wait for my direction before proceeding.
```

**`session-end.md`:**
```markdown
End-of-session protocol:
1. Summarize accomplishments
2. List decisions made with rationale
3. Note open questions and blockers
4. Check if CLAUDE.md needs updating — propose specific changes
5. Update the session continuity memory (replace the old one)
6. Check for uncommitted changes and offer to commit
7. Suggest the next logical work item
```

**`design-review.md`:**
```markdown
Review $ARGUMENTS against the design documents.
1. Read the implementation code
2. Find the corresponding design in docs/AI_SYSTEM_DESIGN.md or docs/ARCHITECTURE.md
3. Create a compliance checklist
4. Flag undocumented deviations
5. Identify documented-but-unimplemented features
6. Rate overall alignment (percentage)
7. Recommend whether to update the code or the doc
```

**`explore.md`:**
```markdown
Research $ARGUMENTS using parallel agents.
Send at least 3 agents to explore:
- Technical implementation approaches
- Academic/theoretical foundations
- Existing implementations in similar projects
Synthesize into a structured brief:
- State of the art
- Key trade-offs
- Recommendation for LLMUD specifically (reference our architecture)
- Follow-up sources/references
```

**`scaffold-test.md`:**
```markdown
Design a scaffold test for $ARGUMENTS:
1. Read the scaffold file
2. Extract hypothesis from YAML frontmatter
3. Check experiments/ for existing test results
4. Design a test scenario:
   - Evennia world setup requirements
   - Starting conditions
   - Success metrics (quantitative)
   - Failure criteria
   - Number of runs needed for statistical significance
5. Define control condition (no scaffold or alternative scaffold)
6. Output as a test plan document in experiments/
```

**`phase-status.md`:**
```markdown
Report current phase status:
1. Read DEV_PROCESS.md for phase definitions
2. Check CURRENT_WORK.md for progress
3. Read recent git log for completed work
4. Report:
   - Current phase and its exit criteria
   - Completed items (with checkmarks)
   - Remaining items
   - Estimated progress percentage
   - Blockers
   - Whether exit criteria are close to being met
```

#### R2. Create Module-Level CLAUDE.md Files

When module directories are created, add CLAUDE.md files:

```
src/telnet/CLAUDE.md
  → "Async telnetlib3. Never block. Handle IAC negotiation. See ARCHITECTURE.md §Telnet."

src/parser/CLAUDE.md
  → "Parses raw MUD output into structured events. GMCP support required.
     Output: ParsedEvent objects. See AI_SYSTEM_DESIGN.md §2."

src/tui/CLAUDE.md
  → "Textual framework. All UI in this module. Never import game logic.
     See ARCHITECTURE.md §TUI."

src/llm/CLAUDE.md
  → "Provider abstraction layer. NEVER import a specific LLM library (openai, anthropic, etc).
     All access through ProviderInterface. Routing config in config/llm_routing.yaml.
     See ARCHITECTURE.md §LLM and AI_SYSTEM_DESIGN.md §6."

src/engine/CLAUDE.md
  → "Core game loop. Coordinates telnet, parser, LLM, scaffolds, actions.
     Fast path (auto-actions) and slow path (LLM reasoning). See AI_SYSTEM_DESIGN.md §1."

src/scaffolds/CLAUDE.md
  → "Scaffold system — the heart of the project. Markdown + YAML frontmatter format.
     Self-describing metadata. See AI_SYSTEM_DESIGN.md §10.
     NEVER simplify the scaffold system. Its complexity is intentional."

src/memory/CLAUDE.md
  → "Memory tiers: working (context window), episodic (SQLite), semantic (SQLite),
     procedural (files). remember(query) searches across all.
     See AI_SYSTEM_DESIGN.md §4."

src/world/CLAUDE.md
  → "World knowledge: maps, NPC data, command reference. NOT a world model —
     the LLM's training is the world model. This is game-specific knowledge only.
     See AI_SYSTEM_DESIGN.md §5."

src/social/CLAUDE.md
  → "Full-depth social intelligence. Empathy modeling, personality scaffolding,
     groklets, brain types. Do NOT simplify. See AI_SYSTEM_DESIGN.md §7."
```

#### R3. Set Up Development Hooks

In `.claude/settings.json`:
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "command": "ruff format $CLAUDE_FILE_PATH 2>/dev/null; ruff check --fix $CLAUDE_FILE_PATH 2>/dev/null; true"
      },
      {
        "matcher": "Edit",
        "command": "ruff format $CLAUDE_FILE_PATH 2>/dev/null; ruff check --fix $CLAUDE_FILE_PATH 2>/dev/null; true"
      }
    ]
  }
}
```

Start with auto-formatting. Add test-running hooks once the test suite exists.

#### R4. Add CURRENT_WORK.md

Create a living status document more reliable than memory:

```markdown
# Current Work

## Phase: 1 (Core Loop)
## Branch: main (or feature/...)
## Last Session: YYYY-MM-DD

### Completed
- [x] Item

### In Progress
- [ ] Item (status note)

### Up Next
- [ ] Item

### Open Questions
- Question?

### Blockers
- None
```

Update at session end via `/session-end`.

#### R5. Memory Hygiene Protocol

Establish a memory taxonomy from day one:
- **Decision memories:** "Decided X because Y" — long-lived, include date
- **Session continuity:** "Last session..." — replace each session (only keep latest)
- **Architecture snapshots:** "As of date, implemented modules are..." — update monthly
- **Gotchas:** "Library X has bug Y" — until resolved

Audit monthly. Target: 5-15 active memories maximum.

### Tier 2: Implement Soon (High Impact, Medium Effort)

#### R6. Build an Evennia MCP Server

This is the single highest-impact infrastructure investment for LLMUD development. A custom MCP server that connects to Evennia's admin API would give Claude Code direct access to:

- **Game state inspection:** Room contents, NPC states, player inventory — queryable during development sessions
- **Scenario management:** Reset to known states, spawn test objects, configure NPCs
- **Command execution:** Run MUD commands and see responses — test the agent's actions directly
- **Log retrieval:** Query game event logs for analysis
- **Scaffold testing integration:** Run scaffold test scenarios programmatically

Implementation approach:
```python
# mcp_servers/evennia_server.py
# Uses Evennia's web API or direct Django ORM access
# Exposes tools: get_room, get_npc, reset_scenario, run_command, get_logs, spawn_object
# Configure in .claude/settings.json
```

This transforms Claude Code from "IDE that can read code" to "integrated development and testing environment for LLMUD."

#### R7. Build a Scaffold Validator MCP Tool

A tool that validates scaffold files against the format specification:
- Checks YAML frontmatter completeness and correctness
- Validates required fields
- Checks cross-references (dependency scaffolds exist)
- Validates effectiveness_notes structure
- Reports issues as structured data

Could be a standalone script or part of the Evennia MCP server.

#### R8. Configure Permission Patterns

Set up permissions that match the development workflow:
```json
{
  "permissions": {
    "allow": [
      "Read", "Glob", "Grep",
      "Bash(git *)",
      "Bash(python -m pytest *)",
      "Bash(ruff *)",
      "Bash(mypy *)",
      "Write(src/**)",
      "Write(tests/**)",
      "Write(scaffolds/**)",
      "Write(docs/**)",
      "Edit(src/**)",
      "Edit(tests/**)",
      "Edit(scaffolds/**)",
      "Edit(docs/**)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Write(.env*)",
      "Write(*.key)",
      "Write(*.pem)"
    ]
  }
}
```

Auto-allow safe development operations. Deny destructive or secret-touching operations.

#### R9. Establish the Experiment Framework

Create the directory structure and templates for scaffold experiments:

```
experiments/
  README.md           — How to run experiments, what the format means
  template.md         — Experiment template
  scaffold-experiments.md  — Experiment log/index
  results/
    EXP-001/          — Per-experiment results
```

The `/scaffold-test` command generates experiment plans. Results get logged here. Claude can read the experiment history to understand the project's empirical trajectory.

### Tier 3: Implement When Architecture Stabilizes (Medium Impact, Higher Effort)

#### R10. Parallel Development with Worktrees

Once multiple modules are under active development:
```bash
git worktree add ../LLMUD-scaffold-runtime feature/scaffold-runtime
git worktree add ../LLMUD-memory-system feature/memory-system
```

Run separate Claude Code sessions per worktree for parallel development. Merge when features are complete and tested.

#### R11. CI Integration with Claude Code

Once the test suite is substantial:
```yaml
# .github/workflows/claude-review.yml
on: [pull_request]
jobs:
  design-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Design Compliance Check
        run: |
          claude --print "Check if the changes in this PR comply with
          docs/ARCHITECTURE.md and docs/AI_SYSTEM_DESIGN.md. Report
          any deviations." > review.md
      - name: Post Review
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const review = fs.readFileSync('review.md', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: review
            });
```

#### R12. Model Routing for Development Sessions

Use different models for different development tasks:
```bash
# Heavy architecture/design work
alias claude-arch="claude --model claude-opus-4-20250514"

# Routine implementation
alias claude-impl="claude --model claude-sonnet-4-20250514"

# Quick questions
alias claude-quick="claude --model claude-sonnet-4-20250514"
```

For LLMUD specifically:
- **Architecture design sessions:** Use Opus — complex, multi-constraint reasoning
- **Implementation sessions:** Use Sonnet — fast, good at straightforward coding
- **Scaffold design:** Use Opus — creative, nuanced reasoning about pedagogy and cognition
- **Test writing:** Use Sonnet — mechanical, well-scoped tasks

#### R13. Build a "Development Dashboard" Custom Command

A comprehensive status command that gives a full project overview:

**`.claude/commands/dashboard.md`:**
```markdown
Generate a development dashboard for LLMUD:

1. Phase status (from DEV_PROCESS.md and CURRENT_WORK.md)
2. Module status (send agents to assess each module):
   - Implementation completeness vs design doc
   - Test coverage
   - Known issues
3. Scaffold library status:
   - Number of scaffolds
   - Tested vs untested
   - Latest experiment results
4. Documentation freshness:
   - Last update date for each major doc
   - Docs that may be stale relative to code changes
5. Open questions and blockers

Format as a structured report with status indicators.
```

### Tier 4: Aspirational / Long-Term (Builds on Everything Above)

#### R14. Integrated Scaffold Development Workflow

The full dream workflow for scaffold development in LLMUD, leveraging all the Claude Code infrastructure:

1. **Design** (Claude Code, plan mode, Opus): "Design a new scaffold for social negotiation"
2. **Implement** (Claude Code, Sonnet): Write the scaffold markdown + YAML
3. **Validate** (Scaffold validator MCP tool): Check format compliance
4. **Test Plan** (`/scaffold-test`): Generate experiment plan
5. **Execute** (Evennia MCP server): Run the test scenario, collect results
6. **Analyze** (Claude Code, Opus): Analyze results, compare against hypothesis
7. **Iterate** (Claude Code): Modify scaffold based on results
8. **Meta-Reflect** (Claude Code): Update scaffold meta-notes with effectiveness data
9. **Evolve** (Claude Code): Apply conceptual exploration operators to generate scaffold variants
10. **Log** (Experiment framework): Record everything in experiments/

This is the "scientific method for scaffold development" — Claude Code as the laboratory instrument.

#### R15. Cross-Project Pollination

LLMUD's memory architecture, scaffold system, and motivation framework have parallels to:
- **Open LLMRI** — both involve understanding LLM internal states
- **Conceptual exploration operators** — directly applicable to scaffold evolution

A global Claude Code command (`~/.claude/commands/cross-pollinate.md`) could:
```markdown
Consider the concept $ARGUMENTS in the context of all three projects
(LLMUD, Open LLMRI, creativity operators).
How does insight from one project apply to the others?
What shared abstractions or patterns emerge?
```

#### R16. Autonomous Scaffold Evolution Pipeline

The ultimate aspiration: a pipeline that:
1. Generates scaffold variants (using exploration operators)
2. Tests them against scenarios (Evennia MCP)
3. Evaluates results (Claude Code analysis)
4. Selects the best variants
5. Generates the next generation
6. Logs everything

This could run in headless mode (`claude --print`) with minimal human supervision, implementing the "evolutionary scaffold design" concept from the architecture decisions.

---

## Appendix A: Key Sources and Further Reading

**Note:** These are representative sources from training data. URLs should be verified for current availability.

### Official Documentation
- Anthropic Claude Code documentation: https://docs.anthropic.com/en/docs/claude-code
- Claude Code CLI reference: https://docs.anthropic.com/en/docs/claude-code/cli-reference
- MCP specification: https://modelcontextprotocol.io/
- MCP server examples: https://github.com/anthropics/mcp-servers
- Anthropic blog on Claude Code: https://www.anthropic.com/blog

### Community
- r/ClaudeAI — Primary Reddit community. Key threads: "agent tool deep dive," "share your custom commands," "managing large projects," "CLAUDE.md best practices," "MCP server recommendations"
- r/LocalLLaMA — Discussions on local model integration with Claude Code
- Hacker News — Periodic "Show HN" posts of projects built with Claude Code; experience reports; architecture discussions
- X/Twitter — #ClaudeCode hashtag; Anthropic employee tips; power user threads

### Blogs and Articles
- Simon Willison's blog (simonwillison.net) — Extensive, skeptical, detailed coverage of Claude Code features and workflows
- Various dev.to and Medium articles on Claude Code best practices
- GitHub repos sharing Claude Code configurations (search "claude code config" or "CLAUDE.md examples")

### Relevant to LLMUD
- Evennia MUD framework: https://www.evennia.com/ and https://github.com/evennia/evennia
- Textual TUI framework: https://textual.textualize.io/
- telnetlib3: https://github.com/jquast/telnetlib3
- Pydantic: https://docs.pydantic.dev/
- MCP Python SDK: https://github.com/anthropics/mcp-python-sdk

---

## Appendix B: Quick Reference Card

### Essential Commands
| Command | When to Use |
|---------|-------------|
| `/plan` | Before designing anything |
| `/compact` | Every 30-45 min, or between tasks |
| `/session-start` | Beginning of every session |
| `/session-end` | End of every session |
| `/design-review module` | After implementing against a design doc |
| `/explore topic` | Research a new concept or approach |
| `/scaffold-test scaffold` | Plan a scaffold experiment |

### Essential CLAUDE.md Rules for LLMUD
```
- NEVER simplify unless explicitly asked
- ALWAYS check design docs before implementing
- ALWAYS update docs when implementation diverges
- When in doubt, ask — don't assume
- Lead with the ambitious option
- This is a research platform — sophistication is the point
```

### Context Window Budget Rules
1. CLAUDE.md under 150 lines
2. Reference docs by pointer, not content
3. Focus on one module per session
4. Use agents for exploration
5. /compact at breakpoints
6. Keep memories under 15 entries

---

*This document is a living research resource. Verify specific claims against current Anthropic documentation, especially for rapidly evolving features like hooks, MCP, and permissions. Update with new findings as they emerge. Last compiled: 2026-03-24.*
