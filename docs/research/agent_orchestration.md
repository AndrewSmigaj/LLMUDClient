# Multi-Agent Orchestration Research for LLMUD

**Date:** 2026-03-24
**Status:** Deep research document. Findings inform LLMUD's eventual swarm architecture.
**Scope:** Framework internals, coordination patterns, blackboard architectures, inter-agent protocols, IFS-style models, memory sharing, cost/latency analysis, context engineering, build-vs-adopt recommendation.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Framework Internals](#2-framework-internals)
3. [Coordination Patterns That Ship in Production](#3-coordination-patterns-that-ship-in-production)
4. [Blackboard Architectures](#4-blackboard-architectures)
5. [Agent-to-Agent Communication Protocols](#5-agent-to-agent-communication-protocols)
6. [IFS-Style Agent Models](#6-ifs-style-agent-models)
7. [Memory Sharing Between Agents](#7-memory-sharing-between-agents)
8. [Cost and Latency Analysis](#8-cost-and-latency-analysis)
9. [Context Engineering Strategies](#9-context-engineering-strategies)
10. [Build vs Adopt](#10-build-vs-adopt)
11. [Recommendation for LLMUD](#11-recommendation-for-llmud)

---

## 1. Executive Summary

The multi-agent orchestration landscape as of early 2026 has consolidated around a few key patterns (sequential, concurrent, handoff, group chat, magentic/dynamic planning) with implementations across CrewAI, LangGraph, AutoGen/Microsoft Agent Framework, and Semantic Kernel. None of these frameworks were designed for IFS-style "inner swarm" architectures where specialized agents share a blackboard with executive mediation. They are designed for enterprise workflows, customer support routing, and code generation pipelines.

**Key finding:** LLMUD should build its own lightweight orchestration layer rather than adopting an existing framework. The IFS model, real-time game-playing requirements, eudaimonic motivation, and tight integration with the scaffold system create unique constraints that no existing framework accommodates. However, the *patterns* from these frameworks (particularly the magentic/task-ledger pattern and the blackboard pattern) are directly applicable and should be studied carefully.

The right approach: build a thin orchestration layer using LLMUD's existing architecture (which already has the right interfaces: modular context assembly, shared state via clean interfaces, message bus for tool calls), informed by the patterns documented here, and implement swarm features incrementally after the single-agent system proves out the scaffold architecture.

---

## 2. Framework Internals

### 2.1 CrewAI

**Architecture:** CrewAI organizes agents into "Crews" with defined "Processes" that determine execution order. The two core process types are sequential (agents execute in a fixed order, each receiving the output of the previous) and hierarchical (a manager agent delegates tasks to workers and synthesizes results).

**Under the hood:**
- Each agent is a wrapper around an LLM call with a role description, goal, backstory, and set of tools.
- A Crew is a container that holds agents and tasks. Tasks have descriptions, expected outputs, and assigned agents.
- State passes between agents as string context -- the output of one agent gets injected into the prompt of the next.
- In hierarchical mode, a "manager" agent is auto-created that uses LLM reasoning to decide which agent handles which task. This is essentially the handoff pattern with the manager as router.
- Memory is handled through a shared "crew memory" -- a combination of short-term (conversation within the current execution), long-term (persisted across runs via embeddings in a local vector store), and entity memory (extracted entities and relationships).
- Delegation works through tool calling: when delegation is enabled, agents have a "delegate_work" tool that routes sub-tasks to other crew members.

**Strengths:** Easy to get started, good for linear workflows, built-in memory abstraction.

**Weaknesses for LLMUD:**
- Process types are coarse (sequential or hierarchical) -- no native blackboard pattern.
- State passing is string-based, not structured. You can't have fine-grained shared state.
- The hierarchical manager is a generic LLM-based router, not a domain-specific executive.
- No native support for agents with persistent identity or evolving scaffolds.
- Designed for batch task completion, not continuous real-time operation.
- The memory system uses embeddings/vector search, which adds infrastructure overhead and doesn't align with LLMUD's SQLite + keyword approach.

### 2.2 LangGraph

**Architecture:** LangGraph models agent workflows as state machines (directed graphs) where nodes are computation steps and edges are transitions. State is a typed object (usually a TypedDict or Pydantic model) that flows through the graph, being read and modified by each node.

**Under the hood:**
- The core primitive is a `StateGraph` with typed state. Each node is a function `(state) -> state_update`.
- Edges can be conditional: a routing function examines state and returns the name of the next node.
- State updates are merged using "reducers" -- for example, a message list uses an append reducer, while a string field uses replacement.
- Checkpointing is built in: after each node executes, state is persisted to a checkpoint store (SQLite, PostgreSQL, or custom). This enables time-travel debugging and human-in-the-loop interruption.
- For multi-agent, LangGraph uses the "supervisor" pattern: one node (the supervisor) decides which agent-node to invoke next by routing based on the current state. Agents are subgraphs or tool-calling nodes.
- There is also a "hierarchical teams" pattern where supervisors manage sub-teams, each of which is its own graph.
- Message passing between agents happens through the shared state object -- agents read from and write to state fields.

**Strengths:** Extremely flexible graph-based execution, excellent checkpointing/state persistence, typed state with reducers, good for complex conditional workflows.

**Weaknesses for LLMUD:**
- The graph must be defined ahead of time (nodes and edges are static, even if edge routing is dynamic). This doesn't naturally accommodate an IFS model where "parts" activate and deactivate fluidly.
- The abstraction is designed for request-response workflows, not continuous game loops.
- Heavy dependency chain (LangChain ecosystem).
- State management is powerful but assumes a single state object flowing through nodes, not a blackboard that multiple independent agents read/write concurrently.
- The supervisor pattern maps roughly to our Executive, but LangGraph's supervisor is a routing function, not a rich mediator with its own reasoning about part dynamics.

### 2.3 AutoGen / Microsoft Agent Framework

**Architecture:** AutoGen went through a major redesign from v0.2 to v0.4. The v0.4+ architecture (which evolved into Microsoft Agent Framework) is built on an **actor model** with event-driven message passing.

**Under the hood:**
- Agents are actors that communicate by publishing and subscribing to messages on topics.
- The runtime manages message delivery, agent lifecycle, and distributed execution.
- Messages are typed (not just strings) and can carry structured payloads.
- Conversation patterns are built on top of this: `RoundRobinGroupChat`, `SelectorGroupChat` (LLM picks next speaker), `Swarm` (agents hand off to each other), `MagenticOneGroupChat` (the MagenticOne pattern with a task ledger).
- The MagenticOne pattern is particularly interesting: an Orchestrator agent maintains a "task ledger" -- a running plan with goals, sub-goals, progress, and a "fact sheet" of accumulated knowledge. It reasons about which agent to invoke next, can backtrack, and continuously evaluates whether the goal is met.
- State management varies by pattern: group chats accumulate a message history, while the MagenticOne ledger is an explicit structured document.
- AutoGen/Agent Framework supports multiple orchestration patterns natively: Sequential, Concurrent, GroupChat, Handoff, and Magentic.

**Strengths:** Actor model is well-suited for concurrent agent systems. MagenticOne's task ledger is conceptually close to LLMUD's goal document. Rich orchestration patterns. Good separation of agent logic from coordination logic.

**Weaknesses for LLMUD:**
- Heavy framework with significant abstraction overhead.
- Designed for enterprise scenarios (customer support, SRE automation), not game-playing.
- The actor model adds distributed systems complexity we don't need for a local application.
- MagenticOne assumes agents are tool-using executors; our "parts" are more like perspectives that contribute different kinds of reasoning to a shared situation.
- No native concept of scaffolds, personality, or evolving agent identity.

### 2.4 Semantic Kernel Agent Framework

**Architecture:** Semantic Kernel's Agent Framework is built on top of the broader Semantic Kernel ecosystem (plugins, function calling, prompt templates). Agents are abstractions over LLM capabilities with typed message passing.

**Under the hood:**
- The base `Agent` class provides identity, instructions, and tool access.
- `AgentThread` manages conversation state -- either locally (chat history passed each invocation) or remotely (via service-managed threads like OpenAI Assistants).
- Orchestration patterns are implemented as experimental features: Concurrent, Sequential, Handoff, GroupChat, and Magentic.
- All patterns share a unified invocation interface: create orchestration with agents, start a runtime, invoke with a task, await results.
- The `InProcessRuntime` handles local message delivery between agents.
- Input/output transforms allow data adaptation between agents and external systems.
- Human-in-the-loop is supported across patterns.
- Agents can be defined declaratively via YAML specs and registered with a type registry.

**Strengths:** Clean unified API across patterns. Good .NET and Python support. Declarative agent specs.

**Weaknesses for LLMUD:**
- Still experimental (orchestration features are pre-release).
- Designed around request-response patterns, not continuous operation.
- Same fundamental limitation as others: no native blackboard, no IFS-style part dynamics, no scaffold system.
- Enterprise-focused with heavy Azure integration assumptions.

### 2.5 Framework Comparison Matrix

| Feature | CrewAI | LangGraph | AutoGen/MAF | Semantic Kernel |
|---------|--------|-----------|-------------|-----------------|
| Core abstraction | Crew + Tasks | State Graph | Actor + Messages | Agent + Thread |
| State passing | String context | Typed state with reducers | Typed messages | Chat history / Thread |
| Multi-agent pattern | Sequential, Hierarchical | Supervisor, Hierarchical Teams | Round Robin, Selector, Swarm, MagenticOne | Sequential, Concurrent, Handoff, GroupChat, Magentic |
| Persistence | Vector store memory | Checkpointing (SQLite/PG) | Message history | Thread-based |
| Concurrency | Limited | Graph-level parallelism | Actor-based | Runtime-managed |
| Dynamic routing | Via delegation tool | Conditional edges | Agent handoff / selector | Pattern-dependent |
| Blackboard support | No | Shared state object (limited) | No native | No |
| Continuous operation | No | No (request-response) | No | No |
| Python support | Yes | Yes | Yes | Yes |
| Complexity | Low | Medium | High | Medium |

---

## 3. Coordination Patterns That Ship in Production

Based on the Azure Architecture Center's comprehensive documentation (updated Feb 2026) and framework implementations, here are the patterns that people actually ship:

### 3.1 Sequential (Pipeline)

**How it works:** Agents execute in a fixed linear order. Each agent's output becomes the next agent's input. Common state accumulates through the pipeline.

**Production use cases:** Document processing (draft -> review -> compliance -> risk assessment), data transformation pipelines, progressive refinement workflows.

**Relevance to LLMUD:** Limited direct relevance. The dual-process architecture (System 1 -> System 2) is already a two-stage pipeline. Could be useful for REM reflection (narrative review -> analytical review -> strategic review).

### 3.2 Concurrent (Fan-out/Fan-in)

**How it works:** Multiple agents receive the same input and process it independently in parallel. An aggregator collects results and synthesizes.

**Production use cases:** Multi-perspective analysis (financial: fundamental + technical + sentiment), ensemble decision making, parallel independent subtasks.

**Relevance to LLMUD:** High relevance for the IFS model. Multiple "parts" could analyze the same game situation from different perspectives (combat assessment, social reading, exploration opportunity, resource management) and an executive synthesizes. This is the closest standard pattern to what we want.

### 3.3 Handoff (Dynamic Delegation)

**How it works:** One agent handles a task until it recognizes it needs specialist help, then transfers control to another agent. Only one agent is active at a time. The receiving agent can hand off further.

**Production use cases:** Customer support triage, escalation systems, multi-domain problem solving where the relevant domain emerges during processing.

**Relevance to LLMUD:** Moderate. Could model how the single agent switches between "modes" -- combat mode hands off to social mode when an NPC starts talking during a fight. But in the IFS model, parts don't hand off control; they all contribute to a shared deliberation.

### 3.4 Group Chat (Multi-Agent Debate)

**How it works:** Multiple agents participate in a shared conversation thread. A chat manager determines who speaks next. The conversation accumulates in a shared history. Supports maker-checker loops (one agent proposes, another critiques).

**Production use cases:** Brainstorming, consensus building, collaborative problem solving, quality gate workflows.

**Relevance to LLMUD:** High conceptual relevance. The IFS "parts" model is essentially an internal group chat where different perspectives debate. The chat manager maps to the Executive/Self. The maker-checker loop maps to a part proposing an action and the Self evaluating it against other parts' concerns.

### 3.5 Magentic (Dynamic Task Ledger)

**How it works:** A manager agent maintains a "task ledger" -- a living document of goals, sub-goals, progress tracking, and a fact sheet. The manager consults specialized agents, iterates and backtracks, dynamically builds and refines the plan, and continuously checks whether the goal is met or stalled.

**Production use cases:** SRE incident response, open-ended problem solving, complex planning where the solution path isn't known upfront.

**Relevance to LLMUD:** Very high. This is the closest existing pattern to what LLMUD already does with its goal document. The task ledger IS the goal document. The manager IS the executive/Self. The specialized agents ARE the IFS parts. The difference: MagenticOne was designed for one-shot task completion, while LLMUD needs continuous operation with an evolving task ledger.

### 3.6 The Missing Pattern: Blackboard

None of the five standard patterns explicitly implement a blackboard architecture, though elements appear in several. The blackboard pattern is the most relevant for LLMUD and is discussed in depth in Section 4.

### 3.7 Pattern Recommendations for LLMUD

The LLMUD swarm should be a **hybrid of concurrent analysis + group chat deliberation + magentic task ledger**, unified by a blackboard. Specifically:

1. **Game event arrives** -> Multiple parts analyze concurrently from their specialized perspectives (concurrent pattern).
2. **Parts write assessments to the blackboard** -> The Executive reads all assessments.
3. **If assessments conflict** -> The Executive convenes a deliberation (group chat pattern) between the relevant parts.
4. **Executive updates the goal document** (magentic task ledger pattern) and decides on action.
5. **Action executes** -> results feed back into the loop.

This hybrid doesn't map cleanly to any single framework's implementation, which is a strong signal for building custom.

---

## 4. Blackboard Architectures

### 4.1 Classical Blackboard Pattern

The blackboard architecture originated in AI research in the 1970s-80s (Hearsay-II speech understanding system). The core components:

- **Blackboard:** A shared data structure that holds the current problem state. Organized into regions or levels of abstraction. All knowledge sources read from and write to the blackboard.
- **Knowledge Sources (KS):** Independent, specialized modules that can contribute to the solution. Each monitors the blackboard for patterns it can act on. They don't communicate with each other directly -- only through the blackboard.
- **Control Component:** Monitors the blackboard and decides which knowledge source to activate next. Often based on which KS's preconditions are met by the current blackboard state.

### 4.2 Why Blackboard Fits the IFS Model

The IFS therapeutic model maps remarkably well to the blackboard architecture:

| IFS Concept | Blackboard Concept |
|-------------|-------------------|
| Self (core, wise mediator) | Control Component |
| Parts (protectors, exiles, firefighters) | Knowledge Sources |
| Internal experience being processed | Blackboard content |
| Parts "blending" (taking over) | A KS dominating the blackboard |
| Self "unblending" (regaining perspective) | Control component restoring balance |
| Parts sharing their perspective | KS writing assessments to blackboard |
| Self listening to all parts | Control component reading all KS outputs |

### 4.3 Modern Blackboard Implementations

**The problem:** There are very few modern, production-quality blackboard framework implementations for LLM agents. The pattern is well-understood theoretically but under-implemented in the current agent framework ecosystem. Here's what exists:

**Custom implementations dominate.** Most teams building blackboard-style systems implement them custom. The pattern is simple enough that a framework isn't strictly necessary:

```python
# Minimal blackboard pattern
class Blackboard:
    def __init__(self):
        self._state = {}
        self._subscribers = []

    def write(self, key: str, value: Any, source: str):
        self._state[key] = {"value": value, "source": source, "timestamp": now()}
        self._notify(key)

    def read(self, key: str) -> Any:
        return self._state.get(key)

    def query(self, pattern: str) -> dict:
        return {k: v for k, v in self._state.items() if matches(k, pattern)}
```

**LangGraph's shared state** is the closest framework approximation. A `StateGraph` with typed state and reducers can function as a simple blackboard. But it lacks the control component (which KS to activate) and the pattern-matching trigger mechanism (KSs monitoring for relevant changes).

**Redis/shared memory approaches.** Some production systems use Redis or shared dictionaries as a blackboard, with agents polling for changes. This works but loses the reactive trigger mechanism.

### 4.4 Blackboard Pitfalls

From studying implementations and post-mortems:

**Race conditions with concurrent writers.** When multiple KSs write to the blackboard simultaneously, you need conflict resolution. Options: locking (kills concurrency), CRDTs (complex), event sourcing (append-only log of changes), or single-writer-per-region (partition the blackboard).

**Blackboard bloat.** Without pruning, the blackboard grows unboundedly. Every part that writes an assessment adds to state. Need a garbage collection strategy -- stale assessments expire, superseded entries get replaced.

**Control component complexity.** Deciding which KS to activate next is itself a hard problem. In classical systems, this was often the biggest bottleneck. For LLM-based systems, the Executive (control component) can use LLM reasoning to make this decision, which is elegant but adds latency and cost.

**Context window limits.** Even with a blackboard, the Executive needs to read enough of the blackboard to make good decisions. If the blackboard has 8 parts' worth of assessments plus game state plus goals, that may exceed context limits -- which was the problem the swarm was supposed to solve.

**Recommendation for LLMUD:** Use a **structured, partitioned blackboard** where:
- Each part writes to its own region (combat_assessment, social_assessment, etc.)
- The Executive reads summaries from each region, not full content
- Full part outputs are available on demand if the Executive needs to drill down
- An event log tracks all blackboard changes for interpretability
- Use Python's asyncio primitives for concurrency control, not distributed systems tools

### 4.5 Blackboard Design for LLMUD

```
Blackboard Structure:
├── game_state/           # Written by System 1 parser
│   ├── room
│   ├── vitals
│   ├── inventory
│   └── recent_events[]
├── assessments/          # Written by parts (knowledge sources)
│   ├── combat/           # Combat part's current assessment
│   │   ├── threat_level
│   │   ├── recommended_action
│   │   └── confidence
│   ├── social/           # Social part's current assessment
│   │   ├── npc_states
│   │   ├── conversation_opportunity
│   │   └── relationship_implications
│   ├── exploration/      # Explorer part's assessment
│   │   ├── unknown_exits
│   │   ├── points_of_interest
│   │   └── discovery_potential
│   └── resource/         # Resource manager's assessment
│       ├── needs_status
│       ├── supply_concerns
│       └── economic_opportunities
├── goals/                # Shared goal document (read by all, written by Executive)
├── deliberation/         # Active deliberation thread (when parts disagree)
│   ├── topic
│   ├── contributions[]
│   └── resolution
└── action_queue/         # Executive's decided actions
    ├── pending[]
    └── history[]
```

---

## 5. Agent-to-Agent Communication Protocols

### 5.1 Model Context Protocol (MCP)

**What it is:** An open-source protocol (created by Anthropic) for connecting AI applications to external data sources and tools. Uses JSON-RPC 2.0 over stdio or Streamable HTTP transport.

**Architecture:**
- **MCP Host:** The AI application (e.g., Claude Desktop, VS Code) that manages one or more MCP Clients.
- **MCP Client:** Maintains a dedicated connection to an MCP Server.
- **MCP Server:** Provides context to clients via three primitives: Tools (executable functions), Resources (data), and Prompts (templates).
- The protocol is stateful with lifecycle management (capability negotiation handshake).
- Servers can send notifications when available tools/resources change.
- Client primitives include Sampling (server asks client's LLM for completions) and Elicitation (server asks user for input).

**What MCP is NOT:** MCP is not an agent-to-agent communication protocol. It's a tool/resource exposure protocol. It standardizes how an LLM application discovers and uses external capabilities. Think of it as "how an agent finds and uses tools" not "how agents talk to each other."

**Relevance to LLMUD:** MCP could be useful for exposing LLMUD's tools (game commands, memory, scaffolds) to the LLM in a standardized way. It would also enable external tools (web search, wikis) to plug in via standard MCP servers. However, it does NOT solve inter-agent communication for the swarm. MCP is a tool protocol, not a coordination protocol.

**Practical status:** Mature and widely supported. Claude, ChatGPT, VS Code, Cursor, and many other clients support MCP. The SDK is available in Python and TypeScript. Worth adopting for the tool interface layer regardless of swarm architecture.

### 5.2 Google's Agent-to-Agent Protocol (A2A)

**What it is:** Google's protocol for agent-to-agent communication, announced in 2025. Designed to let agents from different vendors/frameworks discover each other and collaborate.

**Architecture:**
- Agents publish "Agent Cards" -- JSON descriptions of their capabilities, endpoints, and supported interaction modes.
- Communication happens over HTTP with JSON-RPC or similar structured messaging.
- Supports task delegation (one agent sends a task to another), status updates, and artifact exchange.
- Designed for cross-organization agent interop, not intra-application agent coordination.

**Relevance to LLMUD:** Very limited for the inner swarm. A2A is designed for agents that are separate services communicating across network boundaries. LLMUD's IFS parts are intra-process agents sharing memory. A2A might become relevant for the "outer swarm" (multiple LLMUD characters coordinating), but that's far future.

**Practical status:** Early. Less mature than MCP. The spec is published but ecosystem adoption is still growing. Not recommended for adoption now.

### 5.3 Custom Protocols

**What actually ships:** Most production multi-agent systems use custom message passing built on their framework's primitives:

- **AutoGen/Agent Framework:** Typed messages published to topics via the actor runtime.
- **LangGraph:** Shared state mutations with typed reducers.
- **CrewAI:** String-based context passing between tasks, delegation via tool calls.
- **Semantic Kernel:** Chat message content types through orchestration runtimes.

**For LLMUD's inner swarm,** the simplest effective protocol is:

```python
@dataclass
class PartMessage:
    source: str            # Which part sent this
    target: str | None     # None = broadcast to blackboard
    msg_type: str          # "assessment", "concern", "request", "response"
    content: dict          # Structured payload
    priority: float        # How urgent this part considers its message
    timestamp: datetime
```

Messages flow through the blackboard, not directly between parts. The Executive reads all messages and decides what to route, amplify, or suppress. This is simpler than actor-based message passing and maps directly to the IFS model where Self mediates all communication.

### 5.4 Protocol Recommendation

- **Adopt MCP** for the tool interface layer (game tools, memory tools, scaffold tools). This gives standardized tool discovery and invocation, and lets external MCP servers plug in.
- **Build custom** for inter-part communication. A simple blackboard-mediated message protocol is all that's needed. Don't over-engineer this with actor frameworks or distributed protocols.
- **Ignore A2A** for now. Revisit when/if the outer swarm (multi-character coordination) becomes relevant.

---

## 6. IFS-Style Agent Models

### 6.1 Existing Implementations

Research into IFS-inspired multi-agent architectures reveals surprisingly little direct implementation. The concept is discussed in several contexts:

**Therapeutic/coaching chatbots:** Some IFS-informed chatbot projects use multiple prompts or roles to simulate different IFS "parts" in a therapeutic conversation. These are prompt-engineering exercises, not true multi-agent systems. They typically use a single LLM with role-switching, not separate agents.

**Society of Mind / Minsky-inspired systems:** Marvin Minsky's "Society of Mind" (1986) proposed that intelligence arises from the interaction of many simple "agents" (not LLM agents but computational modules). Some multi-agent LLM projects reference this, but implementations tend to be standard multi-agent patterns (debate, voting) without the nuanced part-dynamics of IFS.

**Multi-perspective reasoning:** The "Multi-Agent Reasoning" (MAR) pattern, where the same situation is analyzed from multiple perspectives with a judge/synthesizer, is widely implemented. This is conceptually close to IFS but lacks the persistent identity, emotional valence, and protective function of IFS parts.

**Generative Agents (Stanford, 2023):** Park et al.'s "Generative Agents: Interactive Simulacra of Human Behavior" gave individual LLM agents persistent memory, reflection, and planning. While not multi-agent-within-one-mind, the memory architecture (observation -> reflection -> planning) is directly relevant to how LLMUD's parts could maintain their own episodic context.

### 6.2 What Makes IFS Different from Standard Multi-Agent

The IFS model has properties that distinguish it from typical multi-agent orchestration:

**Parts have emotional valence and protective functions.** A "protector" part doesn't just have a different knowledge domain -- it has a *motivation* (keep the system safe) and an *emotional quality* (anxiety, vigilance). This is deeper than role assignment.

**Parts can "blend" with Self.** In IFS, when a part becomes dominant, it "blends" with Self and the person loses perspective. In a multi-agent system, this would mean a specialist agent's assessment becomes the system's action without executive mediation. This is a failure mode we want to detect and prevent.

**Self is not just a router.** In standard multi-agent, the orchestrator/supervisor routes tasks to specialists. In IFS, Self has its own qualities (curiosity, calm, compassion, clarity, confidence, courage, creativity, connectedness -- the "8 Cs"). Self doesn't just pick the best specialist; it holds space for all parts and makes decisions that honor everyone's concerns.

**Parts have relationships with each other.** Some parts polarize against each other (a risk-taking explorer vs. a cautious protector). Some parts protect other parts (a manager part suppressing an exile's grief). These dynamics affect which parts speak up and how their input is weighted.

**Parts can be activated or deactivated.** Not all parts are relevant all the time. In a combat situation, the exploration part might be dormant. But it should still be *available* if something exploration-relevant happens mid-combat.

### 6.3 Mapping IFS to LLMUD's Agent Architecture

| IFS Concept | LLMUD Implementation |
|-------------|---------------------|
| **Self** | The Executive agent. Has its own system prompt emphasizing the 8 Cs. Reads the blackboard, mediates between parts, makes final decisions. Uses the fullest context (goal document + part summaries + current situation). |
| **Manager parts** (proactive protectors) | Agents like ResourceManager ("make sure we don't run out of food"), SafetyMonitor ("assess danger before acting"). These run on triggers and write cautionary assessments to the blackboard. |
| **Firefighter parts** (reactive protectors) | Emergency response agents that activate in crisis: CombatResponder (HP drops below threshold), FleeAdvisor (overwhelming danger detected). High-priority, interrupt other processing. |
| **Exile parts** (vulnerable, carry burdens) | In LLMUD's context, these might be the parts that carry past failure experiences: "last time we tried this approach, we died." They surface relevant negative memories. This maps to episodic memory retrieval with emotional valence filtering. |
| **Blending** | Detected when a single part's assessment dominates the Executive's decision-making over multiple turns without other parts being consulted. The meta-review (REM) can detect this pattern. |
| **Unblending** | The Executive deliberately soliciting input from dormant parts: "What does the Explorer think about this situation?" Even if it seems combat-focused, there might be an escape route worth noting. |
| **Polarization** | When two parts give contradictory assessments (Explorer: "go deeper into the dungeon" vs. SafetyMonitor: "retreat to town"). The Executive must hold both perspectives without dismissing either. |

### 6.4 Implementation Approach

Rather than starting with a full IFS multi-agent system, the path from single agent to IFS swarm should be:

**Phase 1 (current): Single agent with role-switching scaffolds.** The single agent loads different scaffolds for different situations (combat scaffold, social scaffold, exploration scaffold). This is implicitly "switching which part is active." The scaffold system already supports this.

**Phase 2: Named perspectives in deliberation.** During System 2 reasoning, the prompt includes multiple named perspectives: "Consider this from the Combat Advisor's perspective... Now from the Explorer's perspective... Now from the Social Observer's perspective..." This is MAR-style multi-perspective reasoning within a single LLM call. Cheap, no multi-agent overhead, and captures most of the benefit.

**Phase 3: Blackboard with persistent parts.** Each part becomes a separate LLM invocation with its own system prompt, loaded scaffolds, and relevant memory context. Parts write to the blackboard. The Executive reads summaries and mediates. This is the full IFS swarm.

**Phase 4: Dynamic part activation.** Parts aren't always running -- they activate based on blackboard triggers (similar to how scaffolds have `useful_when` tags). The Executive can also explicitly summon dormant parts. Parts develop over time through reflection, accumulating their own observations and evolving their scaffolds.

---

## 7. Memory Sharing Between Agents

### 7.1 The Core Problem

In a multi-agent system, each agent has a limited context window. The whole point of the swarm (for LLMUD) is that each agent holds different context. But agents need to share discoveries, coordinate on goals, and avoid duplicating work.

### 7.2 Approaches in Practice

**Shared message history (AutoGen, Semantic Kernel).** All agents in a group chat see the accumulated conversation. Simple but doesn't scale -- the history grows linearly and eventually exceeds context limits.

**Shared state object (LangGraph).** A typed state object is passed between agents, with each agent reading/writing specific fields. Better than raw message history because state is structured and can be selectively loaded.

**Vector store memory (CrewAI, some LangChain setups).** Agents write observations to a shared vector store and retrieve via semantic similarity. Good for large knowledge bases but adds latency and infrastructure.

**Blackboard with summaries.** Agents write full assessments to the blackboard but the Executive (and other agents) read summaries. Each agent can request the full content of a specific region if needed. This is the most efficient for IFS-style systems.

### 7.3 Memory Architecture for LLMUD's Swarm

The existing LLMUD memory architecture (episodic + semantic + procedural) maps naturally to multi-agent:

**Shared across all parts (read-only for parts, write by logging system):**
- Episodic memory (SQLite) -- what happened. Any part can `remember(query)`.
- Semantic memory (JSON/SQLite) -- what we know. Shared world knowledge.
- Game state (blackboard) -- current HP, room, inventory.
- Goal document -- shared goals (written only by Executive).

**Per-part context (loaded into each part's prompt):**
- The part's system prompt (role, personality, concerns).
- The part's domain scaffolds (combat scaffolds for combat part, social scaffolds for social part).
- Recent relevant episodic memories filtered by the part's domain.
- The part's own previous assessments (continuity).

**Written by parts to the blackboard:**
- Current situation assessment from this part's perspective.
- Concerns, opportunities, warnings.
- Confidence level and reasoning.

**Written by Executive only:**
- Goal document updates.
- Action decisions.
- Deliberation resolutions.
- Requests for specific parts to weigh in.

### 7.4 Context Budget Per Agent

For a typical game situation with the swarm active:

| Agent | Context Budget | Contents |
|-------|---------------|----------|
| Combat Part | ~4K tokens | System prompt (300) + combat scaffolds (1500) + recent combat memories (500) + current game state (200) + goal summary (200) + current situation (300) + tools (500) |
| Social Part | ~4K tokens | System prompt (300) + social scaffolds (1500) + NPC profiles (500) + current game state (200) + goal summary (200) + current situation (300) + tools (500) |
| Explorer Part | ~4K tokens | System prompt (300) + exploration scaffolds (1000) + map data (800) + current game state (200) + goal summary (200) + current situation (300) + tools (500) |
| Resource Part | ~3K tokens | System prompt (300) + resource scaffolds (800) + inventory/economy data (500) + current game state (200) + goal summary (200) + current situation (300) + tools (500) |
| Executive | ~6K tokens | System prompt (500) + part summaries (1500) + goal document (500) + current game state (200) + current situation (300) + deliberation history (1000) + meta-scaffolds (1000) + tools (500) |

Total per reasoning cycle: ~21K tokens input across 5 agents. Compare to single-agent: ~3.3K tokens (from AI_SYSTEM_DESIGN.md). The swarm costs roughly 6x the single agent in input tokens, plus 5x the LLM calls.

---

## 8. Cost and Latency Analysis

### 8.1 Single Agent vs Multi-Agent

**Single agent with role-switching (current LLMUD design):**
- 1 LLM call per reasoning cycle
- ~3.3K input tokens + ~500 output tokens
- Latency: 1-3 seconds (local model) or 2-5 seconds (API)
- Cost per cycle: ~$0.001-0.005 (API) or ~$0 (local)

**Multi-agent swarm (5 parts + executive):**
- 5 part calls (can be parallel) + 1 executive call (sequential after parts)
- ~21K input + ~2.5K output tokens total
- Latency if parallel: max(part_latencies) + executive_latency = 2-4 seconds (local) or 3-7 seconds (API)
- Latency if sequential: sum of all calls = 6-18 seconds (local) or 12-30 seconds (API)
- Cost per cycle: ~$0.006-0.030 (API) or ~$0 (local)

**Key insight:** With local models, the swarm is feasible on cost but latency is the constraint. Running 5 parts sequentially on a single GPU would be 5x slower. Parallel execution requires either multiple GPUs or batched inference (which many local inference servers support but with diminishing throughput per request).

### 8.2 When Is Multi-Agent Actually Worth It?

Based on production reports and benchmarks:

**Multi-agent is worth it when:**
- Context window limits are a genuine bottleneck (single agent can't hold all relevant info).
- Different subtasks genuinely benefit from different system prompts and scaffolds.
- The task requires integrating perspectives that would conflict in a single prompt (e.g., "be cautious" and "be bold" in the same system prompt degrades both).
- Verification/criticism loops improve output quality enough to justify the cost.

**Multi-agent is NOT worth it when:**
- A single well-scaffolded agent can handle the task with role-switching.
- The overhead of coordination (executive reasoning, context summarization) exceeds the benefit of specialization.
- Latency constraints make sequential multi-agent calls impractical.
- The "agents" would just be doing what a single agent with different scaffolds loaded could do.

**For LLMUD specifically:**
- In routine gameplay (walking around, buying items, simple combat), single agent with appropriate scaffolds is sufficient. Multi-agent adds latency without proportional benefit.
- In complex situations (social+combat+exploration convergence, multi-step quest planning, situations where the agent is stuck), the swarm's multi-perspective analysis genuinely adds value.
- Recommendation: **adaptive activation** -- the single agent operates normally, and the swarm activates only when the Executive detects a situation that would benefit from multi-perspective reasoning. This mirrors how IFS parts activate in response to triggers, not constantly.

### 8.3 Real-World Cost Numbers

From production multi-agent deployments (as of early 2026):

- **Customer support (handoff pattern, 3-5 agents):** $0.02-0.08 per interaction. Most cost is in the first triage agent.
- **Code generation (maker-checker, 2 agents):** $0.03-0.15 per task. The checker catches ~15-25% of issues, arguably worth 2x cost.
- **Research/analysis (concurrent, 4 agents):** $0.10-0.50 per analysis. High value tasks justify cost.
- **Game-playing agents (single agent):** No good production benchmarks for multi-agent game-playing. Single-agent game bots (with tool use) typically run $0.01-0.05 per decision cycle on API models.

For LLMUD with local models, direct cost is electricity + hardware amortization. The real cost is latency and GPU memory.

---

## 9. Context Engineering Strategies

### 9.1 The Fundamental Problem

The LLMUD AI_SYSTEM_DESIGN.md already articulates this: context engineering is the primary rationale for the swarm. A single agent can't hold combat guides + social scaffolds + map data + NPC profiles + full goal document + relevant memories + analytical tools all at once. The swarm distributes this across specialist windows.

### 9.2 Strategies for Context Distribution

**Strategy 1: Specialist Windows (the IFS approach)**
Each part holds domain-specific context. The combat part has combat guides and bestiary; the social part has NPC profiles and conversation history. Parts only need their domain context plus a summary of the current situation.

Pros: Clean separation, each part reasons deeply in its domain. Parts' context windows can be small (4K-8K sufficient).
Cons: Parts can miss cross-domain connections (the NPC you're fighting is the blacksmith's son -- combat part doesn't know the social implications). Requires the Executive to synthesize across domains.

**Strategy 2: Compressed Shared Context**
Instead of distributing context, compress it. Use summaries, abstractions, and progressive detail. All agents get a compressed overview; they can request expansion of specific sections.

Techniques:
- **Hierarchical summarization:** Full game state -> 1-paragraph summary. Full goal document -> active goal + 1 line per queued goal. Full NPC profile -> name + relationship score + current emotional state.
- **Lazy loading:** Start with summaries in context. Agent requests full detail via tool call (`read_guide("combat")`, `recall_person("merchant_greta")`). Only loads what's actually needed.
- **Sliding window with compression:** Recent events in full detail, older events as summaries, oldest as statistical aggregates.

Pros: Avoids information silos. Single agent can hold compressed version of everything.
Cons: Compression loses nuance. LLMs are sensitive to what's in context -- a summary of a combat guide is less effective than the full guide.

**Strategy 3: Context-on-Demand (the LLMUD scaffold approach)**
This is what LLMUD already does: the prompt builder assembles context dynamically based on the current situation. Scaffolds have `useful_when` tags. The `remember()` tool retrieves relevant memories. The system loads only what's relevant.

Pros: Efficient, adaptive, no multi-agent overhead. Already designed and partially implemented.
Cons: Still limited by a single context window. If a situation requires combat + social + exploration knowledge simultaneously, you're back to the overflow problem.

**Strategy 4: Hybrid (Recommended)**
Use Strategy 3 (context-on-demand) for the single agent. When the context builder detects that the relevant context exceeds the window (too many scaffolds triggered, too many relevant memories, multi-domain situation), it automatically escalates to the swarm, where specialists can each hold their domain context in full.

This is the adaptive activation approach from Section 8.2. The single agent handles 80% of situations. The swarm handles the 20% where context overflow would force lossy compression.

### 9.3 Context Assembly for the Executive

The Executive has the hardest context engineering problem: it needs to understand all parts' perspectives while also holding the goal document, current situation, and meta-scaffolds.

Approach: **Structured summary protocol.** Each part writes its assessment in a constrained format:

```markdown
## [Part Name] Assessment
**Situation:** [1 sentence]
**Recommendation:** [1 sentence]
**Confidence:** [high/medium/low]
**Key concern:** [1 sentence]
**Needs from Executive:** [none / more info / deliberation requested]
```

At ~50 tokens per part summary, 5 parts = 250 tokens. The Executive can hold all summaries plus full game state, goals, and meta-scaffolds within ~6K tokens. If a part flags "deliberation requested," the Executive loads that part's full assessment for deeper engagement.

---

## 10. Build vs Adopt

### 10.1 Framework Adoption Analysis

| Framework | IFS Fit | Game-Playing Fit | Scaffold Integration | Python | Verdict |
|-----------|---------|-------------------|---------------------|--------|---------|
| CrewAI | Poor (sequential/hierarchical only) | Poor (batch tasks) | None | Yes | **No** |
| LangGraph | Fair (flexible graph, but static topology) | Fair (could model game loop as graph) | None | Yes | **Maybe for state management ideas** |
| AutoGen/MAF | Fair (MagenticOne is closest to our model) | Poor (enterprise focus) | None | Yes | **Study MagenticOne, don't adopt framework** |
| Semantic Kernel | Fair (multiple patterns) | Poor (request-response) | None | Yes | **No** |

### 10.2 What Frameworks Would Give Us

- Pre-built orchestration patterns (but we need a hybrid pattern none of them implement).
- State persistence/checkpointing (but we already have SQLite + file persistence designed).
- Agent lifecycle management (but our agents are simpler -- same LLM with different prompts).
- Logging/observability instrumentation (but we need custom interpretability hooks for Open LLMRI).

### 10.3 What Frameworks Would Cost Us

- **Dependency weight.** Every framework adds substantial dependencies. LangChain ecosystem alone pulls in dozens of packages.
- **Abstraction mismatch.** Our agents aren't independent services -- they're perspectives within a single mind. Frameworks model agents as independent entities.
- **Loss of control.** Framework message passing, state management, and execution patterns would need to be understood deeply to debug and modify. We'd be fighting the framework's assumptions.
- **Impedance mismatch with scaffolds.** No framework has any concept of scaffolds, guide loading, or evolving agent identity. We'd need to bolt this on.
- **Continuous operation.** All frameworks assume request-response or batch task patterns. We need continuous real-time operation in a game loop.

### 10.4 What To Borrow From Frameworks

Even though we should build custom, there's significant design wisdom to extract:

**From LangGraph:**
- Typed state with reducers. The blackboard should use Pydantic models with defined merge strategies.
- Checkpointing. Persist blackboard state to SQLite after each reasoning cycle for recovery and interpretability.
- Conditional routing based on state inspection. The Executive's decision about which parts to activate should be modeled as state-dependent routing.

**From AutoGen/MagenticOne:**
- Task ledger pattern. The goal document IS a task ledger. Study MagenticOne's implementation of progress tracking, stall detection, and dynamic replanning -- these are exactly the problems LLMUD's goal management needs to solve.
- Typed messages. Inter-part communication should use typed messages (Pydantic models), not raw strings.

**From Semantic Kernel:**
- Unified orchestration interface. All orchestration patterns should share a common invocation pattern so we can swap between single-agent, concurrent parts, and full swarm without changing calling code.
- Declarative agent specs. Parts could be defined declaratively (YAML/markdown) with their role, scaffolds, tools, and activation triggers. This aligns with LLMUD's scaffold-as-configuration philosophy.

**From the Azure Architecture Center patterns:**
- Adaptive complexity. Start with single agent, escalate to multi-agent only when needed (their "start with the right level of complexity" guidance).
- Context compaction between agents. Summarize, don't pass raw.
- Iteration caps on deliberation to prevent infinite loops.
- Per-agent model selection (not every part needs the most capable model).

**From MCP:**
- Standardized tool interface. Use MCP's tool discovery and invocation patterns for the tool layer, even if we don't use MCP as a protocol.

### 10.5 Verdict: Build Custom

**Build a lightweight, custom orchestration layer** using:
- Python asyncio for concurrency
- Pydantic models for typed blackboard state and messages
- LLMUD's existing provider abstraction for LLM calls
- LLMUD's existing scaffold system for part configuration
- SQLite for blackboard persistence and event logging
- A simple pub/sub mechanism for blackboard change notifications

This will be simpler and more maintainable than any framework adoption, while being perfectly tailored to the IFS model and game-playing requirements.

---

## 11. Recommendation for LLMUD

### 11.1 Architecture

Build a **Blackboard-Mediated IFS Swarm** with **Adaptive Activation**.

```
                    ┌─────────────────────┐
                    │  MUD Output (text)  │
                    └─────────┬───────────┘
                              │
                    ┌─────────▼───────────┐
                    │  System 1 (Fast)    │
                    │  Parse, classify,   │
                    │  update blackboard  │
                    │  game_state region  │
                    └─────────┬───────────┘
                              │
                    ┌─────────▼───────────┐
                    │  Complexity Check   │
                    │  Can single agent   │◄── Simple? → Single Agent (System 2)
                    │  handle this?       │              with role-switching scaffolds
                    └─────────┬───────────┘
                              │ Complex / multi-domain
                    ┌─────────▼───────────┐
                    │  Part Activation    │
                    │  Which parts are    │
                    │  relevant?          │
                    └─────────┬───────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
    ┌─────────▼──────┐ ┌─────▼──────┐ ┌──────▼─────────┐
    │  Combat Part   │ │ Social Part│ │ Explorer Part  │ ...
    │  (own context  │ │ (own ctx   │ │ (own ctx       │
    │   window)      │ │  window)   │ │  window)       │
    └────────┬───────┘ └─────┬──────┘ └──────┬─────────┘
             │               │               │
             └───────────────┼───────────────┘
                             │ write assessments
                    ┌────────▼────────────┐
                    │    BLACKBOARD       │
                    │  (structured,       │
                    │   partitioned,      │
                    │   Pydantic models)  │
                    └────────┬────────────┘
                             │ read summaries
                    ┌────────▼────────────┐
                    │    EXECUTIVE        │
                    │  (Self / Mediator)  │
                    │  Reads all parts,   │
                    │  holds goals,       │
                    │  decides action     │
                    │                     │
                    │  If conflict →      │
                    │  convene            │
                    │  deliberation       │
                    └────────┬────────────┘
                             │
                    ┌────────▼────────────┐
                    │   Action Decision   │
                    │   → game command    │
                    │   → goal update     │
                    │   → scaffold update │
                    └─────────────────────┘
```

### 11.2 Implementation Phases

**Phase 0 (Now): Single agent with swarm-ready interfaces.**
Already in the design. The key interfaces that must be clean:
- `engine/context.py` — composable context assembly (already planned)
- `world/state.py` — game state as a data object, not embedded in logic (already planned)
- `goals/manager.py` — goal document as shared state (already planned)
- `scaffolds/registry.py` — scaffolds discoverable and loadable by name (already planned)
- `tools/registry.py` — tools scoped and registerable (already planned)

**Phase 1: Multi-Perspective Prompting (cheap swarm simulation).**
Before building actual multi-agent, add multi-perspective reasoning to System 2. When a situation is complex, the prompt includes:

```markdown
## Analyze from Multiple Perspectives

### Combat Assessment
Consider threats, defensive options, and tactical advantage.

### Social Assessment
Consider NPC relationships, conversation opportunities, and reputation.

### Exploration Assessment
Consider unknown areas, discoverable items, and environmental clues.

### Resource Assessment
Consider current supplies, economic opportunities, and survival needs.

## Synthesis
Weigh all perspectives. What action best serves your active goals while honoring all concerns?
```

This costs 1 LLM call instead of 5, captures most of the multi-perspective benefit, and requires zero new infrastructure. Measure how well this works before building the full swarm.

**Phase 2: Blackboard Data Structure.**
Implement the blackboard as a Pydantic model with regions. Even with a single agent, the blackboard provides:
- Structured game state (replacing ad-hoc state tracking)
- Assessment caching (reuse previous assessments if the situation hasn't changed)
- Event logging for interpretability
- The interface for Phase 3

```python
class Blackboard(BaseModel):
    game_state: GameState
    assessments: dict[str, PartAssessment]
    goals: GoalDocument
    deliberation: Deliberation | None
    action_queue: list[Action]
    event_log: list[BlackboardEvent]
```

**Phase 3: Independent Part Invocations.**
Add the ability to invoke specialist LLM calls with per-part context:
- Each part gets its own system prompt, scaffolds, and memory context
- Parts write structured assessments to the blackboard
- The Executive reads summaries and makes decisions
- Start with 2-3 parts (combat + social + explorer), add more as needed
- Use asyncio to run parts concurrently (parallel LLM calls)

**Phase 4: Full IFS Dynamics.**
Add the IFS-specific behaviors:
- Part activation triggers (which parts wake up for which situations)
- Blending detection (is one part dominating without executive mediation?)
- Deliberation protocol (when parts disagree, structured debate)
- Part evolution (parts accumulate experience and update their own scaffolds)
- Part relationships (polarization detection, protective dynamics)

### 11.3 Key Design Decisions

**1. Adaptive activation, not always-on swarm.** The swarm is expensive (5-6x tokens, additional latency). Activate it only when the complexity detector determines single-agent + role-switching is insufficient. Triggers: multi-domain situations, stuck on a goal, conflicting signals, novel+complex scenarios.

**2. Blackboard, not direct message passing.** Parts communicate only through the blackboard. This gives the Executive full visibility and control, mirrors IFS (Self mediates all part communication), and makes the system fully observable for interpretability.

**3. Executive is an LLM call, not a rule engine.** The Executive should use LLM reasoning to mediate between parts, not hardcoded priority rules. This lets the Executive develop wisdom over time through reflection and scaffold evolution.

**4. Parts are defined declaratively.** Each part is a scaffold file:

```yaml
---
name: combat_advisor
role: "Combat assessment specialist"
personality: "Vigilant, tactical, protective. Prioritizes survival."
domain: combat
activation_triggers: ["combat_start", "hostile_detected", "hp_low"]
scaffolds: ["basic_combat", "advanced_tactics", "bestiary_reference"]
memory_filter: "type:combat"
priority_weight: 0.8  # High weight in danger situations
---
```

This means new parts can be added by creating new scaffold files. No code changes needed.

**5. Per-part model selection.** Different parts can use different models. The combat part might use a fast local model (it mostly pattern-matches threat levels). The social part might use a more capable model (social reasoning is harder). The Executive should always use the best available model.

**6. Instrument everything.** Every blackboard write, every part invocation, every executive decision gets logged to the `llm_log` table with full prompts and responses. This feeds Open LLMRI analysis and REM reflection.

### 11.4 Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Swarm latency makes gameplay feel slow | High | High | Adaptive activation -- only use swarm when needed. Parallel part execution. Fast local models for simple parts. |
| Parts produce redundant/low-value assessments | Medium | Medium | Track per-part usefulness in meta-review. Deactivate consistently unhelpful parts. Measure whether multi-perspective actually improves decisions vs. single agent. |
| Executive becomes the bottleneck (needs to read everything) | Medium | High | Structured summary protocol limits Executive context. Lazy loading of full assessments only when needed. |
| Blackboard state grows unboundedly | Medium | Medium | Assessments expire after N cycles. Only most recent assessment per part per situation retained. Event log rotates to SQLite archive. |
| Building swarm infrastructure delays core agent development | High | High | **Phase 1 (multi-perspective prompting) first.** Don't build Phase 3 until single agent is working well and Phase 1 shows clear value. |
| The IFS metaphor is cute but doesn't actually improve performance | Medium | Medium | Measure. Compare single-agent, Phase 1 multi-perspective, and Phase 3 full swarm on the same Evennia scenarios. If the swarm doesn't measurably improve decisions, it's not worth the complexity. |

### 11.5 What To Build Now

For the current single-agent implementation, ensure these swarm-ready properties:

1. **`engine/context.py` must be truly composable.** The context builder should accept a "context spec" (which scaffolds, which memories, which state regions) and assemble a prompt from it. Different specs = different "parts" later.

2. **Game state must be a clean data object** (`world/state.py`), not embedded in the engine loop. This becomes the `game_state` region of the blackboard.

3. **Goal document must be accessible through an interface** (`goals/manager.py`), not directly file-read by the engine. This becomes shared blackboard state.

4. **Tool registry must support scoping** (`tools/registry.py`). Today all tools are available. Later, different parts get different tool subsets.

5. **LLM provider abstraction must support concurrent calls** (`llm/router.py`). Today it routes one call at a time. Later, it dispatches parallel part calls.

6. **Logging must capture full prompts and responses** (`llm_log` table). This is needed for interpretability regardless of swarm, but becomes essential for understanding multi-agent dynamics.

All of these are already in the ARCHITECTURE.md design. The swarm doesn't require architectural changes -- it requires the existing design to be implemented cleanly.

---

## Sources and References

### Framework Documentation
- Semantic Kernel Agent Architecture: https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-architecture
- Semantic Kernel Agent Orchestration: https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-orchestration/
- Azure Architecture Center -- AI Agent Orchestration Patterns: https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns
- Model Context Protocol Architecture: https://modelcontextprotocol.io/docs/learn/architecture
- MCP Introduction: https://modelcontextprotocol.io/introduction
- CrewAI Documentation: https://docs.crewai.com/concepts/crews
- LangGraph Multi-Agent Concepts: https://langchain-ai.github.io/langgraph/concepts/multi_agent/
- AutoGen User Guide: https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/index.html

### Research Papers and Projects
- Microsoft MagenticOne: https://www.microsoft.com/en-us/research/articles/magentic-one-a-generalist-multi-agent-system-for-solving-complex-tasks/
- Park et al. (2023), "Generative Agents: Interactive Simulacra of Human Behavior" (Stanford)
- Wu et al. (2023), "AutoGen: Enabling Next-Gen LLM Applications via Multi-Agent Conversation" (Microsoft Research)
- Minsky (1986), "The Society of Mind" (MIT Press)
- Schwartz (1995), "Internal Family Systems Therapy" (Guilford Press)

### Foundational Architecture Patterns
- Erman et al. (1980), "The Hearsay-II Speech-Understanding System" (original blackboard architecture)
- Corkill (1991), "Blackboard Systems" (AI Magazine survey)
- Hewitt (1973), "A Universal Modular ACTOR Formalism" (actor model)

### Google A2A Protocol
- https://google.github.io/A2A/
