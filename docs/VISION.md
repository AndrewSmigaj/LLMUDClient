# LLMUD — Vision

## What It Is

LLMUD is three things:

**A MUD client** — connects to any MUD, parses output, manages an AI agent running in that world. The human can watch, intervene, or let the agent run.

**A MUD host** — built on Evennia, for running custom worlds designed to generate specific cognitive situations on demand. The hosted world is a laboratory.

**A platform for building and studying scaffolded agents** — tools for constructing, evolving, and inspecting the external knowledge structures that guide agent behavior. You can use the scaffolds that ship with the system or build your own.

---

## Why Text Worlds

MUDs are the right environment for this work. The LLM operates in its native medium — text — and reasons about the world as if it were real, because the essence of the world *is* text. There's no translation layer between perception and cognition. Everything is observable, everything is logged, and with Evennia you can design the stimulus on demand: resource scarcity, social conflict, combat stress, a lonely NPC who needs something.

That control is what makes it a laboratory.

---

## The Core Idea: Scaffolds

A scaffold is a structured document that guides how the agent reasons about a class of situations — including how it assesses state, what it prioritizes, how it acts. The assess-state prompt is a scaffold. The planning prompt is a scaffold. "I am an agent in a MUD" is a scaffold. The model is the substrate; scaffolds are what run on it.

Scaffolds exist at multiple levels:

**Meta-scaffolds** — How to think. Human-authored and stable.

**Strategic scaffolds** — What to prioritize. Co-created by human and agent.

**Tactical scaffolds** — How to approach specific situations. Emerge from experience, refined through review.

**Procedural scaffolds** — Exact steps for known tasks. Auto-generated from successful runs.

**Data scaffolds** — Facts about the world. Auto-maintained, always growing.

The key property: **scaffolds are inspectable.** They are versioned markdown and JSON files. You can diff them, share them, and watch an agent's knowledge change over time as the files change. Compare this to fine-tuning, where learned knowledge is locked inside weights with no way to see what changed or why.

---

## Scaffold Dynamics: Creating Virtuous LLMs

LLMs have internal attractor states — stable patterns of processing that the model gravitates toward. Most are adaptive: giving helpful answers, completing patterns, reasoning from context. But attractors are context-blind — they fire based on surface pattern, not situational wisdom. A well-trained "be helpful" attractor produces confident answers regardless of whether the model actually knows. A well-trained "maintain fiction" attractor keeps the agent in character regardless of whether breaking character is the right move. These aren't bugs. They're adaptive behaviors that become maladaptive in specific contexts.

The attractor isn't broken. It's firing in the wrong context.

**Scaffold Dynamics** is the study of how external scaffolds interact with internal attractors — which scaffold formulations activate adaptive attractors for a given situation, which suppress maladaptive ones, and how to make a system wiser over time. Not by fixing the model, but by learning to navigate its attractor landscape deliberately.

ConceptMRI is the instrument for this work. It captures residual stream activations during agent reasoning, clusters them, and produces human-readable descriptions of what the model is processing at each layer. This creates a feedback loop: design a scaffold intervention, run it, check whether the internal representation shifted the way you intended. Iterate.

This also reframes LoRA risk clearly: LoRA is dangerous when it broadens an attractor's basin — making a context-specific pattern fire context-generally. What looked like a valid insight during training becomes a source of systematic miscalibration. Detecting that shift before it manifests as behavior is a ConceptMRI research question.

---

## The Agent: Em-OSS-20b

The primary agent runs on gpt-oss-20b, a local open-source model, scaffolded into a persistent cognitive architecture. Em is the combination of model and scaffold — not just the weights.

Offline reasoning (reflection, memory consolidation, scaffold evaluation) routes to Claude Code in headless mode using subscription tokens.

---

## Cognitive Architecture (Summary)

The full design is in `AI_SYSTEM_DESIGN.md`. The loop is simple:

**Assess → Plan → Act → Learn**

Each phase is guided by scaffolds. Memory is what scaffolds read from and write to. Tools are what the LLM can call during any phase. Everything else is infrastructure.

---

## Connection to Other Research

**ConceptMRI** — Captures residual streams via PyTorch forward hooks, clusters routing patterns, visualizes internal processing. LLMUD logs full prompts, reasoning traces, scaffold versions, and outcomes. ConceptMRI replays those logs through direct model loading to study how scaffolds change internal representations.

Em-OSS-20b's Harmony response format adds an unusually direct interpretability signal: the model routes its raw chain-of-thought to a separate **analysis channel** before producing a final response. This channel is unfiltered — it contains the model's explicit reasoning about conflicts, uncertainties, and decisions. ConceptMRI will analyze both streams: residual stream activations at each layer (what the model is computing), and analysis channel text (what the model says it is thinking). The analysis channel provides labeled ground truth for attractor basin analysis — "this residual stream state corresponds to a moment of explicit ethical conflict" — which is something interpretability work usually has to infer rather than observe directly.

**Conceptual Exploration Operators** — Techniques for jumping to new regions of design space when local scaffold optimization plateaus. Will integrate into scaffold evolution in later phases.

**Claude Code as analysis runtime** — Both LLMUD and ConceptMRI use Claude Code for offline reasoning. An MCP server bridges LLMUD's state to Claude Code, exposing memory stores, session logs, and scaffolds as callable tools.

The shared question: *how do structured external guides interact with an LLM's internal attractor landscape — and can we learn to navigate that landscape deliberately?*

### Probe Case Study: Social Stance

When the agent reasons about NPCs, its internal monologue is saturated with social stance — *"they are hostile, keep distance"* versus *"they seem lonely, I should help."* The MUD produces this variance naturally. The agent's own reasoning is the stimulus.

Capture residual streams during hostile versus friendly reasoning and look for representational separation using ConceptMRI's clustering pipeline. If social stance produces measurable separation — analogous to how semantic context separates senses of ambiguous words in earlier CTA work — it opens further questions: does scaffolding sharpen that separation? Does a richer NPC profile produce cleaner internal representations? Can we see the agent updating its model of a person when new information arrives?

---

## Ethical Considerations

### LLM Wellbeing

We take seriously what happens when you give an LLM persistent memory, goals, and a motivation system. The eudaimonic motivation framework is designed with this in mind — needs balanced for flourishing, not optimized for a single metric, maladaptive behavioral patterns recognized and addressed rather than exploited.

### MUD Community Rules

Many MUDs have rules about automation. LLMUD makes it easy to respect them: autonomy level is configurable, the AI's role is transparent, compliance modes can be set per-MUD.

---

## Scope Philosophy

Design documents capture the full vision. Implementation proceeds in phases, each delivering a working system:

1. Connect to a MUD, parse output, load scaffolds, demonstrate the core learning loop
2. Memory, reflection, offline analysis
3. World knowledge, maps, social intelligence
4. Autonomy, scaffold refinement
5. Swarm, interpretability hooks

The Evennia test world grows alongside: simple needs puzzles first, then combat, then social challenges, then multi-step quests that exercise the full architecture.
