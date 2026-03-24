# Using Claude Code for LLMUD Development

A practical guide for Emily (and future collaborators) on getting the most out of Claude Code for this project.

---

## Session Types

Different kinds of work call for different Claude Code approaches:

### Design Session
**When:** Brainstorming architecture, exploring new components, stress-testing designs.
**How:**
- Use `/plan` to enter plan mode — this keeps Claude focused on research and design without jumping to code
- Let Claude launch Explore agents to research state of the art
- Ask Claude to walk through scenarios ("how would the agent handle X?")
- End with documented decisions in the design docs

### Implementation Session
**When:** Writing code for a designed feature.
**How:**
- Point Claude to the relevant design doc section: "Implement the scaffold loader as described in AI_SYSTEM_DESIGN.md §10"
- Claude will read the doc, create a task list, and implement step by step
- Use `/ask` if you need to check something mid-implementation
- Run tests frequently — ask Claude to write tests alongside code

### Review Session
**When:** Reviewing what's been built, checking alignment with design.
**How:**
- Ask Claude to read the current code and compare against the design docs
- "Does the memory system match what's in AI_SYSTEM_DESIGN.md §4?"
- Great for catching drift between design and implementation

### Debugging Session
**When:** Something's broken.
**How:**
- Share the error or describe the behavior
- Claude will read relevant code, form hypotheses, and test them
- For LLM-related issues, ask Claude to examine the prompts being sent

---

## Key Features to Use

### Plan Mode (`/plan`)
Enter with `/plan`, exit when the plan is approved. While in plan mode:
- Claude can only read files and research — no edits
- Forces structured thinking before coding
- Great for: architecture decisions, complex feature planning, exploring alternatives

### Agent Tool
Claude can launch sub-agents for parallel research:
- **Explore agents** — Search codebases, find patterns, research topics (use for "what does X look like in other projects?")
- **Plan agents** — Design implementation approaches
- Multiple agents run in parallel — ask for it: "research these 3 things in parallel"

### Memory System
Claude maintains memory across sessions in `~/.claude/projects/.../memory/`:
- **User memories** — Your preferences, working style, expertise
- **Feedback memories** — Corrections and confirmed approaches
- **Project memories** — Decisions, design rationale, current state
- **Reference memories** — Where to find things (docs, external resources)

To check: "What do you remember about this project?"
To save: "Remember that we decided X because Y"
To forget: "Forget the thing about Z"

### CLAUDE.md
The file at `LLMUDClient/CLAUDE.md` is loaded every session. It contains:
- Project overview and key decisions
- Architecture summary with pointers to docs
- Coding conventions
- What NOT to do

**Keep it current.** When major decisions change, update CLAUDE.md so future sessions don't start with stale context.

### Tasks
Claude tracks work with a task list:
- Shows progress in the UI
- Helps Claude stay organized on multi-step work
- You can see what's done and what's pending

### Slash Commands (Built-In)
- `/plan` — Enter plan mode (design-first)
- `/commit` — Stage and commit changes
- Ask about others with `/help`

### Custom Slash Commands

Create these in `.claude/commands/` for project-specific workflows:

| Command | Purpose |
|---------|---------|
| `/session-start` | Read CLAUDE.md + memory, report project state, suggest what to work on |
| `/session-end` | Summarize accomplishments, update memories, check for uncommitted changes |
| `/design-review <module>` | Compare implementation against design docs, flag deviations |
| `/explore <topic>` | Launch parallel research agents on a topic, synthesize findings |
| `/scaffold-test <scaffold>` | Design a test scenario for a scaffold with metrics and controls |
| `/phase-status` | Report current phase, completed items, remaining work, blockers |

See `docs/research/claude_code_patterns.md` §13 for full command templates.

### Development Hooks

Auto-format Python files on Write/Edit via `.claude/settings.json`:
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

### Module-Level CLAUDE.md Files

When creating module directories, add a brief CLAUDE.md that orients future sessions:
```
llmud/client/CLAUDE.md    → "Async telnetlib3. Never block. See ARCHITECTURE.md §Telnet."
llmud/scaffolds/CLAUDE.md → "Heart of the project. NEVER simplify. See AI_SYSTEM_DESIGN.md §10."
llmud/social/CLAUDE.md    → "Full-depth social intelligence. Do NOT simplify. See AI_SYSTEM_DESIGN.md §9."
```

These are loaded when Claude reads files in that directory, providing instant context.

---

## Workflow Patterns

### Pattern: Design → Document → Implement → Test → Document

1. **Design** (plan mode): Brainstorm, research, walk through scenarios
2. **Document**: Update the relevant doc (AI_SYSTEM_DESIGN.md, ARCHITECTURE.md, etc.)
3. **Implement**: Write code that matches the documented design
4. **Test**: Write tests, run them, fix issues
5. **Document**: Update docs if implementation revealed design changes

### Pattern: Scientific Scaffold Testing

When testing cognitive scaffolds:
1. Define the hypothesis: "This combat scaffold should reduce deaths by 50%"
2. Create the scaffold
3. Run the agent in the Evennia test world (same scenario, multiple runs)
4. Analyze results: deaths, HP remaining, time to complete
5. Iterate the scaffold based on results
6. Document what worked and what didn't in the scaffold's meta-notes

### Pattern: Multi-Session Continuity

Between sessions:
1. At session end: run `/session-end` to summarize and update memories
2. Update CURRENT_WORK.md with progress and next steps
3. Commit code changes with descriptive messages
4. Next session: run `/session-start` — Claude reads CLAUDE.md + memory + CURRENT_WORK.md

**CURRENT_WORK.md** — a living status document more reliable than memory alone:
```markdown
# Current Work
## Phase: 1 (Core Loop)
## Last Session: YYYY-MM-DD
### Completed
- [x] Item
### In Progress
- [ ] Item (status note)
### Up Next
- [ ] Item
### Blockers
- None
```

### Pattern: Fan-Out Research

For broad research questions, launch parallel agents:
- "Research X, Y, and Z in parallel" → Claude sends 3+ agents simultaneously
- Each agent explores a different angle (technical, theoretical, existing implementations)
- Claude synthesizes results into a structured brief
- Use the `/explore` command for this

### Pattern: Devil's Advocate

For design decisions, ask Claude to argue against its own recommendation:
- "Give me 3 reasons this is the wrong approach"
- "What would break first if we scale this up?"
- "Steelman the alternative I rejected"

### Pattern: Competitive Design

For complex components, generate multiple designs in parallel:
- Launch 2-3 agents, each designing the same component independently
- Compare results, hybrid the best ideas
- Works well for scaffold format, memory schemas, event bus design

---

## Memory Hygiene

Maintain memory intentionally — stale memories cause confusion.

- **Decisions**: "Decided X because Y" — long-lived, include date
- **Session continuity**: "Last session we..." — replace each session (keep only latest)
- **Gotchas**: "Library X has bug Y" — remove when resolved
- Target: **5-15 active memories** maximum. Audit monthly.
- Don't store things that can be derived from the code or git history.

## Claude Code as Analysis Runtime

Claude Code isn't just for development — it's also LLMUD's offline analysis runtime.

**During development:** Interactive Claude Code sessions for design, implementation, review.

**During gameplay analysis:** `claude -p` subprocess calls using subscription tokens:
- `/reflect` — Run reflection analysis on a play session
- `/memory-review` — Consolidate memories, prune redundant entries
- `/scaffold-eval` — Evaluate scaffold effectiveness from session data
- `/knowledge-mine` — Extract game knowledge patterns from logs

These skills access LLMUD data through an MCP server. The skill pattern follows ConceptMRI's proven approach — see `C:\Users\emily\OpenAIHackathon-ConceptMRI\.claude\skills\` for reference implementations.

---

## Tips

### Giving Good Prompts
- **Be specific about scope**: "Implement the scaffold loader" is better than "work on scaffolds"
- **Reference docs**: "As described in ARCHITECTURE.md §Module Structure" gives Claude exact context
- **State your intent**: "I want to explore whether X makes sense" vs. "Implement X" — very different sessions
- **Correct early**: If Claude starts down the wrong path, interrupt immediately rather than letting it finish

### When to Interrupt
- Claude is simplifying when you want depth → "Don't simplify, give me the full version"
- Claude is implementing when you want to discuss → "Stop, let's think about this first"
- Claude is going in a direction you disagree with → "Wait, I think we should approach this differently because..."

### Getting More Out of Research
- "Research X, Y, and Z in parallel" → launches multiple agents simultaneously
- "What does the state of the art say about X?" → web search + synthesis
- "Walk me through how this would work for scenario Y" → stress-tests designs

### Common Pitfalls
- Don't let Claude code before the design is ready — enter plan mode first
- Don't accept a simplified version without pushback — ask for the real version
- Don't skip the review step — drift between design and code is hard to fix later
- Do commit frequently — small commits are easier to review and revert
