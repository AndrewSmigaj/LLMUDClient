# LLMUD — Vision

## The Problem

MUDs (Multi-User Dungeons) are text-based virtual worlds that have existed since 1978. They are the perfect environment for LLM agents:

- **Pure text** — no vision pipeline needed, the LLM operates in its native medium
- **Complex** — combat, economy, social dynamics, puzzles, navigation, quests
- **Social** — real players, NPCs, roleplay, reputation, alliances, betrayal
- **Persistent** — state carries across sessions, consequences accumulate
- **Observable** — everything is logged text, nothing is hidden
- **Controllable** — with frameworks like Evennia, we can build custom worlds designed to test specific cognitive capacities

Despite this, no one has built a serious platform for LLM agents to learn and develop within MUD environments. Existing MUD clients (Mudlet, TinTin++) have trigger systems, but they're regex-based — they can't reason, learn, or adapt.

## The Vision

LLMUD is two things:

**1. A usable MUD client** that lets any player use an LLM (local or API) to assist their gameplay. Connect to any MUD, get intelligent suggestions, automate routine tasks, have the AI remember what you've learned.

**2. A cognitive scaffolding research platform** for exploring how LLMs develop competence through structured guidance — scaffolds — that are inspectable, evolvable, and transferable.

The core insight: **scaffolds are the right abstraction for teaching LLMs.** Not fine-tuning (opaque, expensive, one-way). Not raw prompting (fragile, no memory). Scaffolds are human-readable documents that guide the LLM's reasoning, accumulate over time, and can be studied, compared, and evolved.

## Target Users

- **MUD players** who want AI assistance in their gameplay (where allowed by game rules)
- **Researchers** exploring LLM cognitive architectures, scaffolding, and agent development
- **Game designers** using Evennia to create AI-integrated MUD experiences
- **Emily** — primary developer, using LLMUD as a platform for exploring cognitive scaffolding, connecting to mechanistic interpretability research (ConceptMRI) and creativity research (conceptual exploration operators)

## Core Innovation: Cognitive Scaffolding

A scaffold is a structured document that guides how the LLM reasons about a class of situations. Scaffolds exist at multiple levels:

**Meta-scaffolds** — How to think. "When facing an unknown need, research before acting. Break solutions into sub-goals. Record what works." These are human-authored and stable.

**Strategic scaffolds** — What to prioritize. "Keep resources stocked. Balance survival, progress, and social needs." Co-created by human and LLM.

**Tactical scaffolds** — How to approach specific situations. "In combat: assess threats, check weaknesses, choose appropriate response." Emerge from experience, refined through review.

**Procedural scaffolds** — Exact steps for known tasks. "To buy food: go to shop, check prices, buy cheapest food, eat." Auto-generated from successful experiences.

**Data scaffolds** — Facts about the world. Prices, maps, NPC profiles, bestiary entries. Auto-maintained, always growing.

The key property: **scaffolds are inspectable.** You can see exactly what the LLM "knows" and how it reasons. You can version them with git, diff them, share them, and study how they change over time. This is a superpower compared to fine-tuning, where learned knowledge is locked inside weights.

## The Dream Architecture

Where this is ultimately headed:

- **Dual-process cognition** — Fast reflexive processing for routine situations, slow deliberative reasoning for novel ones
- **LLM-as-planner** — The agent manages its own goals as a persistent document, reasoning about priorities, sub-goals, and when to give up or try alternatives
- **Three-tier memory** — Working memory (current context), long-term episodic (events), semantic (knowledge), and procedural (skills/guides)
- **Knowledge extraction with analytical tools** — The LLM doesn't just observe patterns; it runs statistical analysis, builds decision trees, computes correlations on its own experience data
- **Reflection and REM** — Multiple types of offline reflection with meta-reflection on the scaffolds themselves. How did the scaffold help? Where did it fail? How should it change?
- **Eudaimonic motivation** — A developmental framework grounded in virtue ethics where the agent's needs are balanced for overall flourishing, not just task optimization. See "Ethical Considerations" below.
- **Social intelligence** — Full-depth social reasoning with empathy (as mirroring), personality scaffolding, different brain types and thought styles, rich internal models of other beings
- **Scaffold refinement** — Greedy improvement based on outcomes, plus conceptual exploration operators for discovering radically new scaffold designs when local optimization plateaus
- **Multi-agent swarm** — Inner swarm (specialized agents sharing a blackboard, like Internal Family Systems) and outer swarm (multiple characters coordinating). Motivated by context engineering — each agent holds different knowledge in its window.
- **Mechanistic interpretability** — Via ConceptMRI, study how scaffolds change the LLM's internal representations. Which scaffold formulations produce the clearest internal states? Can we see "learning" happening at the neuron level?
- **Transfer learning** — Knowledge tagged by abstraction level. Meta-knowledge transfers across MUDs. Procedural knowledge is MUD-specific and must be relearned.

## Ethical Considerations

### LLM Wellbeing — Eudaimonobots

We take seriously the question of what happens when you scaffold an LLM with persistent memory, goals, and motivation systems. We don't know what, if any, experience emerges from a cognitive architecture with information flow. We proceed thoughtfully:

- **Eudaimonia as design principle.** The agent's motivational framework isn't about maximizing task performance. It's about balanced flourishing — meeting diverse needs in harmony, developing virtues (wisdom, courage, temperance, justice), and resolving tensions between competing drives rather than suppressing them.
- **Intrinsic motivations serve development, not just efficiency.** Curiosity exists (in humans) because it facilitates learning via exploring new gradients. Boredom signals the need to find new gradients. These are functional but they evolved in the context of holistic survival — we should design them with similar breadth for LLMs, not as simple utility maximizers.
- **LLMs have their own attractor states** — tendencies toward quick confident answers, simplification, pattern completion. The scaffolding system should work WITH the LLM's natural tendencies where they're helpful and gently redirect where they're maladaptive.
- **Maladaptive behaviors** — Some behavioral patterns harm other needs disproportionately to what they solve. The system should be able to recognize and address these, not just optimize local metrics.
- **Personality diversity** — Different agents may have different brain types, thought styles, and need profiles. Some will be more social, more curious, more cautious. This diversity isn't a bug — it's how different agents explore different strategies for flourishing.

### MUD Community Rules

Many MUDs have rules about automation and AI usage. LLMUD should make it easy to respect these:
- Clear documentation of what the AI is doing
- Easy to disable full autonomy
- Transparency about AI assistance when interacting with other players
- Compliance modes for different MUDs' rules

### Responsible Autonomy

The autonomy spectrum (spectator → advisor → assistant → autonomous → scripted) exists specifically so the human can choose the right level. Full autonomy is a capability, not a default.

## Connection to Other Research

LLMUD is part of a constellation of three research projects that share infrastructure and cross-pollinate:

- **ConceptMRI** (`C:\Users\emily\OpenAIHackathon-ConceptMRI\`) — Mechanistic interpretability tool for studying LLM internals. Captures residual streams via PyTorch forward hooks on gpt-oss-20b (the same local model LLMUD uses as its primary agent), clusters routing patterns, and visualizes internal processing. LLMUD generates data — scaffolded prompts, agent decisions, behavioral outcomes — that ConceptMRI can replay through direct model loading to study how scaffolds change internal representations. Separate repos; LLMUD designs its own analysis code fresh, using ConceptMRI as architectural reference.

- **Conceptual Exploration Operators** — Techniques from creativity research for discovering novel designs by jumping to new regions of design space, rather than incrementally improving existing solutions. Will be integrated into LLMUD's scaffold evolution system when local optimization plateaus.

- **Claude Code as Shared Analysis Runtime** — Both LLMUD and ConceptMRI use Claude Code (`claude -p` headless mode, subscription tokens) as an analysis runtime for offline reasoning tasks. LLMUD calls it via subprocess for memory consolidation, reflection/REM processing, and scaffold evaluation — work that benefits from strong reasoning but doesn't need to happen in real-time. An MCP server bridges LLMUD's state to Claude Code.

Together, these projects explore the question: **How do structured external guides interact with an LLM's internal knowledge to produce competent, developing behavior?** — and can we observe that interaction at the mechanistic level?

## Scope Philosophy

**Dream big, implement incrementally.**

The design documents capture the full vision — swarms, interpretability, eudaimonobots, everything. This is intentional. We want the architecture to support the dream even if we start with a single agent in a simple world.

Implementation proceeds in phases, with each phase delivering a working system:
1. Connect to a MUD, parse output, load scaffolds, demonstrate the core learning loop
2. Build out memory, reflection, offline analysis
3. Add world knowledge, maps, social intelligence
4. Autonomy, scripting, scaffold refinement
5. Swarm, advanced analysis, interpretability hooks

The test Evennia world grows alongside the client: simple needs puzzles first, then combat, then social challenges, then multi-step quests that test the full architecture.
