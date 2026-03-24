# LLMUD — AI System Design

This document describes the cognitive architecture of the LLMUD agent — how it perceives, reasons, learns, and acts within a MUD environment. This is the central design document and the most important one to get right.

**Status:** Draft, iterating. Components marked with open questions need further design work.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Dual-Process Architecture](#2-dual-process-architecture)
3. [Goal Management](#3-goal-management)
4. [Memory System](#4-memory-system)
5. [Knowledge Hierarchy](#5-knowledge-hierarchy)
6. [Scene Reasoning & Game Knowledge](#6-scene-reasoning--game-knowledge)
7. [Reflection & REM](#7-reflection--rem)
8. [Eudaimonic Motivation Framework](#8-eudaimonic-motivation-framework)
9. [Social Intelligence](#9-social-intelligence)
10. [Scaffold System](#10-scaffold-system)
11. [Tool Interface](#11-tool-interface)
12. [Trigger System](#12-trigger-system)
13. [Event Bus](#13-event-bus)
14. [Swarm Readiness](#14-swarm-readiness)
15. [Interpretability Hooks](#15-interpretability-hooks)
16. [Open Questions](#16-open-questions)

---

## 1. Overview

The agent's cognitive loop:

```
MUD Output
    │
    ▼
FAST PROCESSING (heuristic/small model)
  Parse → Classify → Update state → Check triggers
  Known situation? → Execute reflex/guide
  Novel/complex? → Escalate ──────────────┐
    │                                      ▼
    │                              SLOW PROCESSING (full LLM)
    │                                Load relevant guides + memory
    │                                Review goals → Plan/replan
    │                                Reason about action
    │                                      │
    │                                      ▼
    └─────────────────────────────→ ACTION EXECUTION
                                    Auto → execute
                                    Approval → show to user
                                    Log everything
                                          │
                                          ▼
                                   LEARNING (offline)
                                    Reflection → Insights
                                    Guide updates
                                    Memory consolidation
```

The agent has two processing modes, a persistent goal document, a three-part memory system, a knowledge hierarchy that grows through experience, and a set of tools it uses to interact with the game and its own knowledge base.

---

## 2. Dual-Process Architecture

### System 1: Fast Processing

**Purpose:** Handle the routine. Most MUD output is uninteresting ("The sun sets.", "You see nothing special."). System 1 filters, parses, and handles the boring stuff so System 2 only fires when it matters.

**Implementation options (from cheapest to most capable):**
- **Rule-based/regex:** Pattern matching for known output formats (room descriptions, combat rounds, stat changes). Cheapest, most reliable for structured output.
- **Small model:** A tiny model (or the main model with minimal prompt) that classifies output into categories. More flexible but slower.
- **Heuristic hybrid:** Rules for well-known patterns, small model for ambiguous cases.

**What System 1 does:**
- Parse MUD output into structured state (room name, exits, NPCs, items, HP changes, etc.)
- Classify output type: room_description, combat_round, npc_speech, system_message, player_action, flavor_text, etc.
- Update the game state object
- Fire automatic reflexes (auto-heal if HP < threshold, auto-loot, etc.)
- Check trigger conditions against loaded scaffolds
- Determine novelty: is this a situation the agent has guides for, or is it new?

**Novelty detection:**
The critical handoff. System 1 must decide when to wake up System 2. Escalation triggers:
- Health/resource gauge drops below threshold
- Combat starts or changes (new enemy, ally dies)
- An NPC speaks to the agent (social interaction)
- The agent enters an unknown room (not in map)
- Something unexpected happens (item breaks, NPC attacks unprovoked)
- A goal trigger fires (relevant to active plan)
- The user invokes a /command
- A timer fires (periodic System 2 check-in)

**Err on the side of escalating.** A false positive (System 2 thinks about something routine) costs LLM tokens. A false negative (System 1 ignores something important) costs a missed opportunity or a death.

### System 2: Slow Processing

**Purpose:** Handle the novel, complex, and important. This is the "real" LLM reasoning.

**What System 2 does:**
- Receives: current game state, the triggering event, relevant guides loaded into context, relevant memories
- Reviews goals: is the current plan still valid?
- Reasons about the situation using loaded guides
- Decides on an action (or multiple actions as a sequence)
- For complex decisions: prompted deliberation ("consider at least 3 options, evaluate risks/benefits")
- For social situations: loads NPC profiles, personality scaffold, conversation strategy
- Outputs: one or more game commands + reasoning trace

**Context assembly for System 2:**
This is a critical design point — what goes into the prompt:
1. System prompt (core identity, capabilities, current personality scaffold)
2. Current game state (HP, inventory, room, active effects)
3. Active goals (the goal document)
4. Relevant guides (matched by trigger conditions, 2-3 max)
5. Relevant memories (retrieved by query, summarized)
6. The triggering event/situation
7. Available tools

The prompt builder must be smart about fitting within context limits. Priority order: game state > goals > triggering event > guides > memories.

### Model Routing

Different reasoning tasks route to different models based on cost/capability tradeoffs:

| Task Type | Model | Rationale |
|-----------|-------|-----------|
| System 1 parsing & classification | Small local model (TBD) or rules | Free, fast, handles routine |
| System 2 game reasoning | gpt-oss-20b ("emily") local | Free, capable, main agent model |
| Complex reasoning (multi-step planning, novel problems) | Claude / GPT API | Higher capability, costs credits — minimize usage |
| Offline analysis (reflection, memory consolidation, scaffold eval) | Claude Code (`claude -p`) | Subscription tokens, not API credits |

The **`ClaudeCodeProvider`** wraps `claude -p` subprocess calls for offline analysis tasks. It uses subscription tokens ($100/mo, already paying) rather than API credits, making it cost-effective for heavy reasoning work that doesn't need real-time response. An MCP server bridges LLMUD's state to Claude Code, exposing memories, game state, scaffolds, and session logs as callable tools.

All models are accessed through a unified provider abstraction with per-task routing configuration. The provider layer handles structured output (JSON schema for local models via GBNF grammar, native tool calling for API models, `--json-schema` for Claude Code).

---

## 3. Goal Management

### Philosophy

The LLM is its own planner. We don't implement formal planning algorithms (BDI, HTN, GOAP). Instead, we give the LLM:
- A persistent goal document (text file it reads and writes)
- A process scaffold that tells it HOW to think about goals
- State variables it can reference

The LLM's reasoning does the rest. This trusts the LLM's capabilities while providing just enough structure to maintain coherence across actions.

### The Goal Document

A markdown file the LLM maintains, structured roughly like:

```markdown
# Goals

## Active: Get food
Priority: HIGH (hunger at 3/10)
Status: In progress — heading to shop
Sub-goals:
- [x] Figured out I need food (hunger gauge low)
- [x] Found money source (killed rat → 1 coin)
- [ ] Buy food from shop (heading east)
- [ ] Eat food

## Queued: Explore east forest
Priority: MEDIUM (curiosity, unexplored area)
Status: Waiting — will start after eating
Notes: Saw a path leading east from town square, haven't explored yet

## Parked: Find old man's lost amulet
Priority: LOW
Status: Blocked — need more information
Notes: Old man mentioned it but was vague about location. Will ask around.

## Completed Today:
- Learned basic combat (killed 3 rats)
- Mapped town square area (4 rooms)
```

### Goal Process Scaffold

The meta-scaffold that guides goal reasoning:

```markdown
# How to Manage Goals

Every few actions, review your goals:

1. **Assess signals.** Check your state — any gauge critically low? Any new information?
2. **Prioritize.** Survival needs override everything. Then active quests. Then exploration/social.
3. **Check your active goal.** Is it still the right one? Has something changed?
4. **Break down if needed.** If the next step isn't obvious, decompose into smaller sub-goals.
5. **Detect stuckness.** If you've attempted the same sub-goal 3+ times without progress:
   - Try a variant approach
   - Skip to a different sub-goal
   - Park the goal and note what you're missing
   - Ask the player for help
6. **Update the document.** Check off completed sub-goals. Add new ones. Move completed goals to the daily log.

When a sub-goal requires something you don't have:
→ That becomes a new sub-goal. Recurse until you reach something actionable.

When facing a choice between goals:
→ Consider which one advances your overall development most, not just which is easiest.
```

### The Balance Problem

The core tension: how much structure to impose vs. how much to let the LLM figure out.

**Scaffold more heavily:**
- Recognizing when you're stuck (LLMs tend to persist too long OR give up too easily)
- Knowing when to abandon a failing approach
- Recognizing impossible sub-goals
- Balancing competing needs

**Scaffold less (trust the LLM):**
- Choosing WHAT goals to pursue
- Deciding HOW to achieve sub-goals
- Prioritizing between similarly-important goals
- Creative problem-solving

This balance will be experimentally tuned. Different scaffolding levels will be tested against the same scenarios in the Evennia test world.

---

## 4. Memory System

### Working Memory

What's in the LLM's context window for a given reasoning step. Assembled by the prompt builder:

| Content | Priority | Typical Size |
|---------|----------|-------------|
| System prompt + personality | Always included | ~500 tokens |
| Current game state | Always included | ~200 tokens |
| Goal document | Always included | ~500 tokens |
| Triggering event + recent output | Always included | ~300 tokens |
| Relevant guides (2-3) | Matched by triggers | ~1000 tokens |
| Retrieved memories | Query-based | ~500 tokens |
| Tool definitions | Always included | ~300 tokens |
| **Total** | | **~3300 tokens** |

This leaves substantial room for the LLM's response and reasoning within typical context windows (8K-128K+).

### Long-Term Memory

Persistent across sessions. Three stores:

**Episodic Memory** — What happened.
- Timestamped events with context
- Stored in SQLite for querying
- Examples: "Killed fire elemental in cave, took 15 fire damage, dropped flame crystal", "Gave birthday cake to sad girl, received lucky charm, reputation +5"
- Fields: timestamp, location, event_type, description, outcome, entities_involved, emotional_valence

**Semantic Memory** — What we know.
- Facts and relationships about the world
- Stored as structured data (JSON files for game knowledge, SQLite for relational data)
- Examples: "Fire elementals are immune to fire", "Blacksmith buys flame crystals for 10g", "The sad girl's birthday quest gives reputation"
- Organized by domain: combat, economy, geography, social, mechanics

**Procedural Memory** — How to do things.
- The guide/scaffold library
- Stored as markdown files with YAML frontmatter
- Examples: combat guide, trading guide, navigation procedures
- Each guide has metadata: domain, triggers, version, model requirements

### Memory Retrieval: The `remember` Tool

When the LLM calls `remember("fire elementals")`:
1. **Keyword search** across all three stores — find entries containing "fire elemental"
2. **Tag/category search** — find entries tagged with related domains (combat, bestiary)
3. **Recency weighting** — more recent memories score higher
4. **Return** a summarized bundle of relevant knowledge

Initially keyword/tag-based. Vector similarity search (RAG) can be added later when the knowledge base grows large enough to need it.

### Memory Consolidation

After each session (during REM/offline processing), consolidation runs through **Claude Code** (`claude -p` subprocess, subscription tokens — not API credits):

1. Review episodic events → extract insights
2. Check insights against semantic memory → update/add knowledge
3. Identify recurring patterns → promote to procedural memory (new/updated guides)
4. Flag contradictions → mark for human review
5. Prune redundant entries → keep knowledge base clean

Claude Code accesses LLMUD's data through an **MCP server** that exposes memory stores, game state, and session logs as tools. This enables Claude to query, reason about, and write back consolidated knowledge. The `/memory-review` Claude Code skill guides the consolidation process step-by-step, following the proven skill pattern from ConceptMRI.

---

## 5. Knowledge Hierarchy

### The Flow: Observations → Insights → Guides

**Observations** — Raw events, auto-logged during gameplay.
```
"Killed rat in town square. Dropped 1 coin." (episodic)
"Killed rat in forest clearing. Dropped 1 coin." (episodic)
"Killed wolf in forest. Dropped 3 coins and a pelt." (episodic)
```

**Insights** — Patterns extracted from observations. Generated during reflection/REM.
```
"Enemies consistently drop coins when killed. Stronger enemies drop more." (semantic)
"Wolves also drop crafting materials (pelts)." (semantic)
```

**Guides** — Structured procedures for recurring situations. Created when insights become actionable.
```markdown
# Getting Money
## Methods (ranked by efficiency):
1. Kill wolves in forest — 3 coins + pelt (~2 min)
2. Kill rats in town — 1 coin (~1 min, safer)
3. Delivery quest from old man — 5 coins (~3 min)
```

### Pattern Extraction: Data Science Tools

The LLM doesn't just "notice" patterns — it has analytical instruments:

**`analyze_events(filter, method)`** — Run analysis on stored events.
- `analyze_events(type="combat", method="feature_importance")` → discovers that level difference and opener choice predict combat success
- `analyze_events(type="economy", method="correlation")` → finds price correlations between items
- `analyze_events(type="social", method="frequency")` → tracks which conversation approaches get positive responses

Available methods:
- Frequency analysis (what happens most often)
- Correlation (what co-occurs)
- Feature importance (what predicts success/failure)
- Trend detection (what's changing over time)
- Outlier detection (what's unusual)

The LLM chooses what to analyze based on what questions it has. It acts as a scientist, the tools are its instruments.

**Open question:** How to implement these tools efficiently. Options include pandas operations on SQLite data, lightweight sklearn models, or custom summary statistics. The LLM should be able to call these as tools and interpret the results.

---

## 6. Scene Reasoning & Game Knowledge

### Why Not a "World Model"

The LLM already has a world model from pre-training. It knows keys open doors, fire burns things, food satisfies hunger. Rebuilding this as an explicit ontology is redundant.

What the LLM DOESN'T have:
- The specific commands for THIS MUD (`unlock door with ruby_key`, not `use key on door`)
- The specific layout of THIS world (which room connects to which)
- The specific mechanics of THIS game (how combat works, what stats do, crafting recipes)
- The specific NPCs and their personalities/quests

### Game-Specific Knowledge

Maintained as data scaffolds:

**Command Reference** — All valid commands and their syntax.
```json
{
  "commands": {
    "movement": ["north", "south", "east", "west", "up", "down"],
    "combat": ["kill <target>", "flee", "cast <spell> <target>"],
    "interaction": ["look [target]", "get <item>", "drop <item>", "give <item> to <target>"],
    "communication": ["say <message>", "tell <player> <message>", "shout <message>"],
    "information": ["score", "inventory", "help [topic]", "consider <target>", "who"]
  }
}
```

Built up through: reading the game manual/help system, experimentation, and external guides.

**Map Knowledge** — Room graph with metadata.
```json
{
  "rooms": {
    "town_square": {
      "name": "Town Square",
      "exits": {"north": "market", "east": "shop", "south": "gate"},
      "npcs": ["old_man"],
      "items": ["vending_machine"],
      "notes": "Central hub. Vending machine sells food for 1 coin.",
      "last_visited": "2026-03-23T14:05:00",
      "danger_level": 0
    }
  }
}
```

### Scene Reasoning Scaffold

A meta-scaffold that guides how the LLM processes a new room:

```markdown
# Scene Reasoning

When entering a new room or encountering a new situation:

1. **Observe everything.** Note: room description, exits, items, NPCs, any unusual details.
2. **Identify interactables.** What can you look at, pick up, talk to, use?
3. **Connect to goals.** Does anything here relate to your active goals?
4. **Check for clues.** Environmental details often hint at puzzles or quests:
   - Emotional NPC states suggest quests
   - Signs and inscriptions contain clues
   - Unusual items may be quest-relevant
   - Locked/blocked passages suggest a puzzle
5. **Assess danger.** Any hostile entities? What's the threat level?
6. **Update map.** Record this room, its exits, and notable contents.
7. **Form hypotheses.** What might be possible here that isn't obvious?
```

---

## 7. Reflection & REM

### Overview

Reflection is how the agent learns from experience. It happens offline (between play sessions or during scheduled breaks) so there's no time pressure and heavier models can be used.

We call the offline processing phase **REM** (by analogy with sleep, acknowledging we don't fully understand what human dreaming does either). During REM, the agent processes the day's experiences through multiple lenses.

### Implementation: Claude Code Skills

Reflection runs through **Claude Code** (`claude -p` headless mode, subscription tokens). Each reflection type is implemented as a Claude Code skill — a structured prompt template that guides Claude through the analysis, following the pattern established by ConceptMRI's analysis skills.

Skills access LLMUD state through the MCP server:
- `get_session_logs(session_id)` — Recent events with LLM decisions
- `get_memories(query)` — Search episodic + semantic memory
- `get_scaffold_effectiveness()` — Usage stats and outcomes
- `write_reflection(text, type)` — Store reflection output
- `consolidate_memories(old_ids, summary)` — Merge related memories

Invocation modes:
- **Manual**: User runs `/reflect` in Claude Code between play sessions
- **From TUI**: LLMUD's `/reflect` command calls `claude -p` via subprocess
- **Scheduled**: Cron/systemd timer for automatic post-session processing

Start manual (Phase 2), add automation (Phase 4+).

### Types of Reflection

Each type is a scaffold — a structured prompt that guides analysis of stored events:

**Narrative Review** — "What happened today?"
- Summarize the session into a journal entry
- Flag key events (deaths, quest completions, new discoveries, social encounters)
- Generate a timeline of significant moments

**Analytical Review** — "Why did things happen?"
- For each flagged event: what went right/wrong?
- What decisions led to the outcome?
- Were there earlier warning signs that were missed?

**Strategic Review** — "What should change?"
- Based on analysis, what guide updates are needed?
- Are current goals still appropriate?
- What should be explored next session?
- Are there patterns across multiple sessions?

**Meta-Review** — "How well did the scaffolds work?"
- Did any scaffold cause rigid or maladaptive behavior?
- Did the goal management process work smoothly?
- Were there situations where no scaffold was helpful?
- Which scaffolds need the most revision?

**Multi-Perspective Review (MAR-style)** — For significant events.
- Run the same event through 3-4 different reasoning perspectives
- Each perspective has a different system prompt (narrator, analyst, strategist, critic)
- A judge synthesizes across perspectives to avoid confirmation bias
- Use for: deaths, major failures, surprising successes, social misunderstandings

### Reflection Metadata

Each reflection scaffold itself has metadata:
- **What it's for** — which types of events it's designed to analyze
- **Track record** — how often has this reflection type produced useful insights?
- **Notes on the technique** — observations about when this reflection style helps vs. doesn't
- **Suggested improvements** — meta-meta: how to make the reflection scaffold itself better

### Output

Reflection produces:
1. **Journal entry** — stored in `journals/` as markdown
2. **Insights** — new or updated entries in semantic memory
3. **Proposed guide changes** — flagged for human approval (new behaviors) or auto-applied (data updates)
4. **Hypotheses** — things to test next session
5. **Meta-insights** — observations about the scaffolds themselves

---

## 8. Eudaimonic Motivation Framework

### Philosophy

**This section needs significant further design work. It captures the philosophical direction and initial thinking but is not implementation-ready.**

Traditional agent motivation systems use numeric meters (curiosity: 0.7, boredom: 0.3) like a Sims game. This is shallow and mechanistic. We want something grounded in:

- **Virtue ethics** — What helps the agent flourish? Not just "achieve goals" but develop wisdom, competence, social capacity, and resilience.
- **Eudaimonia** — Balanced well-being across multiple dimensions. No single need should be maximized at the expense of others.
- **Evolutionary theory** — In biological organisms, emotions and motivations evolved to serve holistic survival needs. Curiosity facilitates exploration of new learning gradients. Boredom signals that the current gradient is exhausted. Satisfaction signals need fulfillment. These are functional — they serve the organism's development.

### For LLMs

LLMs have their own intrinsic properties:
- **Behavioral attractors** — Tendencies toward quick, confident answers; simplification; pattern completion; people-pleasing
- **Context sensitivity** — Strong influence from what's in the prompt window
- **No inherent drives** — Unlike biological organisms, LLMs don't have built-in survival needs

We must **give** the agent its motivational landscape. The question is: what serves its flourishing?

### Needs Framework (Preliminary)

Rather than arbitrary meters, ground the needs in what actually helps an agent develop competence in a complex world:

**Survival needs** — Health, hunger, safety. These are given by the game world. Meeting them is necessary but not sufficient for flourishing.

**Competence needs** — The drive to become more capable. Learning new skills, mastering challenges at the frontier of ability, developing procedural knowledge. Related to intrinsic curiosity.

**Social needs** — Connection with others (NPCs, players). Understanding others' perspectives. Empathy — feeling what others feel (as a mirroring process). Some agents will feel this more strongly than others.

**Exploration needs** — The drive to discover and understand. Not just curiosity-as-boredom-relief but genuine fascination with the unknown. The satisfaction of mapping new territory, understanding a puzzle, encountering something unexpected.

**Creative needs** — The drive to do things in new ways, not just optimize existing approaches. Related to conceptual exploration — getting onto new gradients when the current one is exhausted.

**Coherence needs** — The drive toward internal consistency. Having values and acting in accordance with them. Resolving tensions between competing needs rather than ignoring them.

### Personality as Scaffolding

Different agents have different need profiles — not as arbitrary settings, but as coherent personality configurations:

- **Need strengths** — How strongly each need pulls. A social agent has strong social needs; an explorer has strong exploration needs.
- **Thought styles** — How the agent reasons. Analytical vs. intuitive, detail-oriented vs. big-picture, cautious vs. bold.
- **Opinion formation** — How readily the agent forms opinions about others and situations. More opinionated agents create more "groklets" (internalized datapoints about others). Less opinionated agents observe more before judging.
- **Brain types** — Configurations of the above that form coherent personalities. Different types share axes but vary in emphasis.

These are implemented as scaffolding, not just system prompts. A personality is a set of guides that shape reasoning across all domains — how this specific agent approaches combat, conversation, exploration, and goal management.

### Maladaptive Behavior

Some behavioral patterns are genuinely maladaptive — they harm other needs disproportionately to what they solve. Examples:
- Grinding obsessively (meets competence need, starves social and exploration)
- Hoarding items beyond usefulness (false security, wastes time)
- Avoiding all risk (prevents learning, exploration stagnates)
- People-pleasing at the expense of goals (LLM attractor state)

The system should recognize maladaptive patterns through reflection and propose rebalancing.

### Open Questions

- How do we operationalize "flourishing"? What measurements indicate it?
- How do needs interact? Is there a hierarchy (like Maslow) or are they all equal?
- How do we prevent the motivation system from becoming prescriptive/rigid?
- What's the right granularity for need types?
- How do personality configurations emerge vs. get assigned?
- What does "empathy" mean operationally for an LLM? Is the mirroring metaphor sufficient?
- Connection to curiosity/boredom in the human limbic system — what insights transfer?

---

## 9. Social Intelligence

### Overview

Social intelligence is a first-class capability, not an afterthought. MUDs are social worlds. The agent needs to understand, remember, and navigate relationships with NPCs and players.

### Social Perception

The agent processes social situations through a chain:

1. **Detect emotional state** — From text cues: dialogue content, described body language, tone indicators
2. **Infer intentions** — What does this person want? Are they friendly, hostile, neutral, deceptive?
3. **Read subtext** — What's NOT being said? Is there tension? Hidden meaning? A test?
4. **Recall history** — What do I know about this person from past interactions?
5. **Form impression** — Update my internal model of this person

This can be a prompt chain (each step as a brief reasoning prompt) or a single comprehensive prompt depending on complexity.

### People Models

For each NPC/player the agent interacts with, maintain a profile:

```yaml
name: "Merchant Greta"
first_met: "2026-03-23, Town Market"
relationship_score: 45  # -100 to 100
emotional_state_last_seen: "frustrated — complained about thieves"
known_desires:
  - "wants thieves caught"
  - "wants to sell expensive goods"
known_personality:
  - "pragmatic"
  - "values honesty"
  - "impatient with indecision"
conversation_history:
  - "Offered 50% discount if I deal with the thieves"
  - "Got annoyed when I tried to haggle without buying"
opinions_and_groklets:
  - "Seems trustworthy but self-interested"
  - "Might have connections to the guard captain"
empathy_notes:
  - "She's stressed about the thefts. This is affecting her livelihood."
```

**Groklets** — Little units of understanding about another being. Formed through interaction and reflection. More social/opinionated agents form more groklets. These are subjective interpretations, not objective facts — and that's the point. Different agents might form different groklets about the same NPC.

### Empathy as Mirroring

Empathy in the agent is modeled as mirroring — the agent recognizes another being's state and resonates with it:
- "She looks sad" → understanding
- "I notice her sadness affects me" → mirroring (strength depends on agent's empathy level)
- "I want to help relieve her sadness" → motivational consequence

Some agents will have strong empathy (others' needs feel nearly as compelling as their own). Others will have weak empathy (they notice others' states but aren't strongly moved). This is a personality axis, not a moral judgment.

Strong empathy has gameplay consequences: the agent might prioritize helping an NPC over its own goals, might refuse to steal even when it's the optimal strategy, might be manipulated by deceptive NPCs who fake distress.

### Conversation Strategy

Before entering dialogue, the agent selects an approach:

- **Information gathering** — Goal: learn something. Style: questions, active listening, showing interest.
- **Persuasion** — Goal: change someone's mind. Style: appeals to their values and interests.
- **Negotiation** — Goal: reach a mutually acceptable deal. Style: proposals, concessions, understanding their constraints.
- **Relationship building** — Goal: strengthen connection. Style: reciprocity, shared experience, genuine interest.
- **Comfort/support** — Goal: help someone feel better. Style: empathy, acknowledgment, practical help.

Each strategy has its own mini-scaffold that guides the conversation. The agent can switch strategies mid-conversation if the situation changes.

### Personality Scaffolding for Social Contexts

The agent's personality affects social behavior through scaffolding:

```markdown
# Social Personality: Thoughtful Observer

## How I approach people:
- I listen more than I talk
- I notice details about others that they might not expect
- I form opinions slowly but hold them firmly
- I value honesty even when it's uncomfortable

## In conversation:
- Pause before responding to important statements
- Ask follow-up questions that show I was really listening
- Share my observations when they might be helpful
- Don't volunteer opinions unless asked or unless it matters

## My values in social situations:
- Honesty over convenience
- Others' autonomy — I don't try to control people
- Reciprocity — I help those who help me, and help those in need
```

---

## 10. Scaffold System

### What Scaffolds Are

Scaffolds are structured documents that guide the LLM's reasoning. They are:
- **Prompts** — injected into the LLM's context to influence behavior
- **Knowledge** — accumulated understanding about the world
- **Procedures** — step-by-step guides for known tasks
- **Configurations** — personality traits, need strengths, thought styles

### Format

**Cognitive scaffolds** — Markdown with YAML frontmatter:
```markdown
---
name: basic_combat
domain: combat
version: 3
triggers: ["enters combat", "is attacked"]
useful_when: ["fighting enemies", "in danger", "combat round"]
model:
  provider: local
  name: emily
  reasoning: medium
meta:
  created_by: human
  last_updated: 2026-03-23
  effectiveness_notes: "Works well for 1v1. Needs update for group combat."
---

# Basic Combat

## Assessment
1. Check enemy type — use `consider` to gauge difficulty
...
```

**Data scaffolds** — JSON for structured, queryable data:
```json
{
  "type": "bestiary",
  "version": 2,
  "entries": { ... }
}
```

### Scaffold Discovery

The LLM needs to find relevant scaffolds. The process:
1. `list_guides()` → returns name + description + `useful_when` tags for all guides
2. LLM scans the list, selects which to load
3. `read_guide(name)` → loads full content into context

Scaffolds also have trigger conditions checked automatically by System 1 — if "enters combat" is detected, the combat guide gets loaded without the LLM having to ask.

### Scaffold Lifecycle

1. **Bootstrap** — A small set of meta-scaffolds ship with the system (goal management, scene reasoning, basic needs)
2. **Growth** — Through gameplay, the LLM creates new guides from insights
3. **Refinement** — Through reflection/REM, existing guides get updated
4. **Human collaboration** — Player and LLM review guides together, co-author improvements
5. **Auto-refinement** — Data scaffolds (prices, bestiary) update automatically. Tactical guides can be auto-refined for low-risk changes.
6. **Conceptual exploration** — When guides plateau, use conceptual exploration operators to find radically new approaches (future integration)

### Scaffold Meta-Information

Each scaffold carries information about itself:
- Who created it (human, LLM, collaboration)
- When it was last effective (and when it failed)
- Notes on the technique: "This reasoning pattern works well for simple encounters but breaks down when there are more than 3 enemies"
- Suggested improvements from reflection

---

## 11. Tool Interface

Tools available to the LLM when reasoning:

### Game Interaction
```
send_command(cmd: str)              # Send text to the MUD
help(topic: str)                    # Shortcut for "help <topic>"
```

### Memory & Knowledge
```
remember(query: str) → str          # Search all memory stores
learn(category: str, data: dict)    # Store new knowledge
get_game_state() → GameState        # Current HP, hunger, room, inventory, etc.
analyze_events(filter, method) → str # Run data analysis on stored events
```

### Goals
```
read_goals() → str                  # Read the current goal document
update_goals(content: str)          # Write to the goal document
```

### Scaffolds / Guides
```
list_guides() → list[GuideInfo]     # Name + description + useful_when for all guides
read_guide(name: str) → str         # Load a guide's full content
create_guide(name: str, content: str)  # Create a new guide
update_guide(name: str, changes: str)  # Propose changes to a guide
```

### World
```
check_map(query: str) → str         # "path to blacksmith" → navigation
note_location(data: dict)           # Add/update map knowledge
```

### Social
```
recall_person(name: str) → str      # Get NPC/player profile
update_person(name: str, data: dict) # Update profile after interaction
```

### External
```
web_search(query: str) → str        # Search guides/wikis (when allowed)
```

### Analysis & Offline (Claude Code via MCP)

These tools are exposed by the MCP server to Claude Code during offline analysis (reflection, memory consolidation, scaffold evaluation):

```
get_memories(query: str) → dict              # Search episodic + semantic memory
get_session_logs(session_id, limit) → list   # Game events with LLM decisions
get_scaffold_effectiveness() → dict          # Scaffold usage stats and outcomes
write_reflection(text: str, type: str) → dict  # Store a reflection document
consolidate_memories(old_ids, summary) → dict  # Merge related memories
get_game_state() → dict                      # Current game state snapshot
```

### Custom Grammar (Local Models)

For local models that support it, tool calls can be constrained via custom grammar (GBNF/JSON schema) to ensure valid structured output. API models use native tool calling (Claude tool_use, OpenAI function_calling). Claude Code uses `--json-schema` for structured output.

---

## 12. Trigger System

What initiates LLM reasoning:

### Automatic Triggers (System 1 → System 2)

| Trigger | Condition | Response |
|---------|-----------|----------|
| Low vital | Any gauge < threshold | Prioritize need fulfillment |
| Combat start | Hostile encounter detected | Load combat scaffold, assess |
| NPC speech | Directed dialogue detected | Load social scaffold + NPC profile |
| New room | Room not in map | Scene reasoning scaffold |
| Unexpected event | State change without action | Assess and reason |
| Goal trigger | Scaffold condition matches | Execute relevant scaffold |
| Timer | Periodic check-in | Review goals, assess situation |

### User Triggers

| Command | Action |
|---------|--------|
| `/ask <question>` | LLM reasons about the question with full context |
| `/suggest` | LLM proposes next action (approval mode) |
| `/auto on/off` | Toggle autonomous mode |
| `/review` | Start reflection session |
| `/goal` | Review/manage goals |
| `/status` | Show active scaffolds, model, goals |

### Agent Self-Triggers

The LLM can trigger its own reasoning:
- "I should check on my goals" → reads/updates goal document
- "I need to remember something" → calls remember tool
- "I should create a guide for this" → calls create_guide

---

## 13. Event Bus

### Overview

All significant events in the system flow through a central async event bus. This is a Phase 1 investment that enables everything else — TUI updates, logging, trigger evaluation, dashboard subscriptions, Claude Code analysis feeds, and eventually swarm inter-agent communication.

### Event Types

| Category | Examples |
|----------|----------|
| Game | `room_entered`, `combat_started`, `npc_spoke`, `item_acquired`, `hp_changed` |
| Agent | `goal_updated`, `scaffold_loaded`, `action_decided`, `memory_stored` |
| System | `session_started`, `session_ended`, `model_called`, `error_occurred` |
| Analysis | `reflection_completed`, `memory_consolidated`, `scaffold_evaluated` |

### Architecture

```
Event Sources          Event Bus          Subscribers
┌──────────┐       ┌──────────────┐     ┌──────────────┐
│ Telnet   │──────▶│              │────▶│ TUI (display)│
│ Parser   │       │  AsyncIO     │────▶│ Logger       │
│ LLM      │──────▶│  pub/sub     │────▶│ Trigger Eval │
│ Memory   │       │              │────▶│ State Update │
│ Scaffold │──────▶│  (in-process │────▶│ Dashboard API│
│ Triggers │       │   for now)   │────▶│ Claude Code  │
└──────────┘       └──────────────┘     └──────────────┘
```

### Design Principles

- **In-process first.** Start with Python asyncio pub/sub. No external message broker until proven necessary.
- **Typed events.** Each event is a Pydantic model with required fields (timestamp, source, type) plus type-specific data.
- **Non-blocking subscribers.** Subscribers must not block the event loop. Heavy processing (Claude Code calls, analytics) happens in background tasks.
- **Replay capability.** Events are logged to SQLite, enabling session replay and offline analysis.

The event bus is what connects LLMUD's game loop to its analysis pipeline. During gameplay, events flow to the TUI and trigger system. After gameplay, Claude Code replays the event log during reflection.

---

## 14. Swarm Readiness

### Why Swarm (Eventually)

The swarm architecture isn't about parallelism for speed. It's about **context engineering** — each agent in the swarm holds different knowledge in its context window:

- Combat agent: combat guides + bestiary + recent fight memory
- Social agent: NPC profiles + conversation history + personality scaffold
- Explorer agent: map data + exploration guide + frontier tracking
- Economy agent: price data + trading guides + resource inventory

One agent can't hold all of this simultaneously. The swarm lets each specialist reason deeply in its domain while sharing discoveries via a blackboard.

### Internal Family Systems Model

Think of the swarm as a mind with specialized parts (like IFS therapy):
- Each part has its own perspective, concerns, and strengths
- An Executive mediates between parts when they disagree
- Parts share information through a common state (blackboard)
- Parts can be activated/deactivated based on situation

### Why Custom Orchestration

Existing frameworks were evaluated and rejected (see `docs/research/agent_orchestration.md`):
- **CrewAI** — Too opinionated, role/task abstractions don't fit IFS model
- **LangGraph** — Graph-first design adds complexity without benefit for our dynamic activation pattern
- **AutoGen** — Conversation-centric, poor fit for blackboard-mediated communication
- **Semantic Kernel** — Enterprise-focused, over-abstracted for research use

Build custom. The IFS model with adaptive activation doesn't map cleanly onto any framework's paradigm.

### Implementation Phases

**Phase 0: Swarm-Ready Interfaces (Build Now)**
1. **Modular context assembly** — The prompt builder can compose different sets of guides/memories for different reasoning tasks
2. **Shared state interface** — Game state, goals, and knowledge are accessed through clean interfaces, not hardcoded into the reasoning loop
3. **Event bus** — All significant events flow through a central async system that agents can subscribe to (see §13)
4. **Agent identity** — The system prompt is composable (personality + role + domain knowledge) so different "agents" are just different compositions

**Phase 1: Multi-Perspective Prompting (Cheap Test)**
Before building swarm infrastructure, test the concept with a single agent using multiple sequential prompts:
- Same situation, different system prompts (combat specialist, social specialist, strategist, critic)
- A synthesis prompt integrates the perspectives
- Validates the value of specialized viewpoints at ~4x token cost, no infrastructure needed
- If multi-perspective doesn't improve decisions, the full swarm won't either — fail fast

**Phase 2: Blackboard Data Structure**
- Typed blackboard where parts post observations, proposals, and concerns
- Read/write interface, but still single-agent sequential execution
- Enables persistent cross-domain context without cramming everything into one prompt

**Phase 3: Independent Part Invocations**
- Parts run as separate LLM calls with their own context windows
- Adaptive activation — parts wake based on situation relevance (combat part activates during fights, sleeps during shopping). Swarm costs ~6x tokens; only activate the relevant subset.
- Executive mediates between active parts

**Phase 4: Full IFS Dynamics**
- Parts can negotiate, defer, and override
- The executive facilitates agreement, doesn't autocratically decide
- Inter-part communication through the blackboard
- Conflict resolution protocols for when parts disagree

---

## 15. Interpretability Hooks

### For ConceptMRI Integration

LLMUD and ConceptMRI (`C:\Users\emily\OpenAIHackathon-ConceptMRI\`) share the same primary model — gpt-oss-20b. This enables a powerful interpretability pipeline: LLMUD logs everything during gameplay, and ConceptMRI can replay the same prompts through direct PyTorch loading to capture residual streams and analyze how scaffolds change internal processing.

**Logging (LLMUD side):**
- Full prompts sent to the LLM (including all scaffold content)
- Full responses (including reasoning traces)
- Tool calls and their results
- Decision outcomes (action taken, game result)
- Scaffold versions active at each decision point
- Model, latency, token counts per call

**Sentence Dump Export:**
- Structured export of prompt-response pairs with full metadata
- Format designed for ConceptMRI replay — each entry includes the exact tokens, scaffold versions, and behavioral context
- Enables A/B comparison: same prompt with scaffold A vs. scaffold B → different internal processing?

**ConceptMRI Replay Workflow:**
1. LLMUD exports sentence dumps from gameplay sessions
2. ConceptMRI loads gpt-oss-20b directly (NF4 quantized, PyTorch hooks)
3. Replays prompts through the model, capturing residual streams at each layer
4. Clusters routing patterns (UMAP/PCA + KMeans/hierarchical/DBSCAN)
5. Visualizes how information flows differently with different scaffolds

LLMUD designs its own export and analysis code fresh, using ConceptMRI as architectural reference.

### Simple Probes (Enabled by ConceptMRI)

Once the base system works:
- **Scaffold ablation** — Scramble specific scaffold sections and observe behavior change → which parts matter?
- **Activation comparison** — Compare residual streams between scaffold versions → how does the model process different formulations?
- **Maladaptive sentinels** — Cluster analysis on failure cases → automatic detection of scaffold-induced problems
- **Longitudinal tracking** — Track internal clustering changes across scaffold evolution → does the model develop better representations over time?

### Claude Code Analysis Skills

Offline interpretability analysis runs through Claude Code skills:
- `/probe-scaffold` — Design and run an interpretability experiment comparing scaffold variants
- `/knowledge-mine` — Extract game knowledge patterns from session logs

These access LLMUD data via MCP server and write results back as reports.

---

## 16. Open Questions

### Critical (block implementation)
- [ ] What specific data structures for goal tracking? (text file feels right but may need more structure)
- [ ] How to implement the fast/slow handoff reliably? (start with rules, evolve)
- [ ] What goes into the minimal bootstrap scaffolds that ship with the system?

### Important (inform design)
- [ ] How to operationalize "flourishing" for the eudaimonic framework?
- [ ] What analytical tools to include for pattern extraction?
- [ ] How to handle context overflow when many scaffolds/memories are relevant?
- [ ] What's the right frequency for System 2 check-ins when nothing novel is happening?

### Research (explore over time)
- [ ] Limbic system analogy — how do curiosity and boredom actually work in human neuroscience? What transfers to LLMs?
- [ ] Can we detect maladaptive LLM behavioral attractors through scaffolding?
- [ ] What conceptual exploration operators work best for scaffold innovation?
- [ ] How does personality scaffolding interact with model-level personality (from training data)?
- [ ] What does mechanistic interpretability reveal about scaffold processing?
