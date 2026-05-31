---
name: agent-architecture
description: >-
  Architectural principles and a build discipline for designing AI agents and
  multi-agent systems WELL — pattern selection, shared state, memory, tools,
  termination, human-in-the-loop, evals, and failure-mode avoidance. Framework
  agnostic, with concrete mappings to LangGraph, the OpenAI Agents SDK, and
  Google ADK. Use this skill whenever the user wants to build, design, scaffold,
  refactor, or debug an AI agent, an "agentic" workflow, a multi-agent system, a
  supervisor/orchestrator, a tool-using assistant, or anything where an LLM takes
  actions in a loop — even if they don't say the word "architecture" or name a
  framework. Reach for it on prompts like "build me an agent that…", "set up a
  multi-agent system", "add memory to my agent", "my agent keeps looping / is
  unreliable / ignores instructions", "should this be one agent or several?", or
  "wire up a supervisor". When in doubt, consult it — getting the architecture
  right up front prevents the most common and expensive agent failures.
---

# Agent Architecture

A discipline for building agents and multi-agent systems that actually work in
production. The goal is not to write the most agents — it's to choose the
**simplest architecture that solves the task**, then make it reliable.

Most agent failures are architectural, not coding problems. They come from
skipping the design decisions below and jumping straight to "spin up some
agents." Follow this build order and you avoid the predictable ones.

## The one mental model that underpins everything

An **agent** is a loop: *reason → act (use a tool) → observe the result → adapt
→ repeat*, until the task is done. A model on its own only generates text; an
agent takes actions and reacts to outcomes.

A **multi-agent system** is just several of these loops coordinated. The
coordination — *who shares what information, and who decides what happens next*
— is called **orchestration**, and it sits on a spectrum:

- **Workflow (you drive):** the path is pre-defined. Predictable, cheap, easy to
  debug. Like a production line.
- **Autonomous (the AI drives):** the path is decided at runtime by a model.
  Flexible, adaptive, but less predictable and more expensive. Like a brainstorm.

Almost every framework primitive you'll touch is one of: an agent, a tool, a
piece of **shared state**, an edge/route/handoff, or a stop condition. Hold that
and any agent codebase becomes readable.

---

## Build order — follow these steps in sequence

### Step 1 — Choose the architecture (don't over-build)

Walk the ladder and stop at the first rung that fits. Each rung adds capability
**and** adds cost, latency, and ways to fail.

1. **Just a model call** — if the task is pure text (summarise, rewrite, answer
   from knowledge) with no live actions. Don't build an agent.
2. **A single agent with tools** — if it needs to *act* (call APIs, search, run
   code) but the approach is one domain and a knowable loop.
3. **A multi-agent workflow** — if the steps are known and repeatable but need
   genuinely diverse expertise. You wire the flow.
4. **An autonomous multi-agent system** — only if the solution path is unknown
   and must be discovered through exploration by several specialists.

Decision signals for "do I need multiple agents?": **decomposable** into
distinct steps, needs **diverse expertise**, each step handles **heavy
context**, and the approach must **adapt** as results arrive. The more that are
true, the more multi-agent earns its complexity. If few are true, a
well-designed single agent almost always beats a sprawling team.

> State your choice and reasoning out loud before building, and offer the
> simpler option if it's close. A 24× token difference between a model call and
> a multi-agent system on a simple task is real money.

### Step 2 — Pick the orchestration pattern

There are six. Name the one you're using and why — this single decision removes
most ambiguity from the build. Full descriptions, trade-offs, and selection
guidance: read **`references/patterns.md`**.

| Pattern | Type | Use when |
| --- | --- | --- |
| Sequential | Workflow | Fixed, ordered steps (A→B→C) |
| Conditional / Supervisor | Workflow | Routing: send each request to the right specialist |
| Parallel | Workflow | Independent subtasks that can run at once |
| Plan-based (orchestrator) | Autonomous | Complex task needing decomposition + oversight + retries |
| Handoff | Autonomous | Peer agents pass control locally in a known domain |
| Group chat | Autonomous | Open-ended work where the path emerges through dialogue |

Most production systems are **hybrid**: a workflow skeleton with autonomy only
where it earns its keep. And any single agent can itself be a whole system
inside ("agents all the way down") — use the right pattern at each layer.

### Step 3 — Design the shared state and each agent

**Shared state is the backbone of any multi-step system.** Agents should not
call each other directly; they read from and write to a shared state object (a
"whiteboard"). Decide up front: what fields are in the state, and *how each
field updates* — does a new value overwrite, or accumulate/append? (Message
history almost always accumulates.)

For each agent, specify its parts (see **`references/memory-and-evals.md`** for
depth):

- **Tools** — its action space. *Reliability over breadth:* a few excellent,
  well-documented, domain-specific tools beat many mediocre ones. A tool's
  docstring is the instruction the model reads — write it carefully.
- **Memory** — short-term (the current task/conversation) vs. long-term (across
  sessions, usually retrieval/RAG). Not every agent needs long-term memory;
  add it only when it measurably helps.
- **Context** — guard against "context rot": don't stuff the window. Summarise
  old history, trim bulky tool outputs, retrieve memory selectively.
- **Instructions** — a detailed job description (role, tool-use rules, output
  format, error handling, and an explicit "you are done when…"), not a vague
  one-liner. These are the highest-leverage thing to get right.

### Step 4 — Implement it in the user's framework

The principles above are framework-agnostic. Map them to whatever the user is on
— LangGraph, the OpenAI Agents SDK, or Google ADK — using the side-by-side
primitive tables and idiomatic snippets in **`references/frameworks.md`**. If the
user hasn't chosen a framework, that reference also has guidance on picking one.

### Step 5 — Add the guardrails before calling it done

These are the difference between a demo and something trustworthy:

- **Termination** — every autonomous loop needs a stop condition, or it runs (and
  bills) forever. Compose several: a hard cap (max turns/tokens/time) AND a
  completion signal AND a human stop.
- **Human-in-the-loop for high-cost actions** — classify actions by
  consequence; gate irreversible/expensive/sensitive ones (send, delete, pay,
  publish) behind explicit human approval. Read-only actions auto-run.
- **Observability** — stream/trace every step so a human can see what each agent
  did and *why*. You can't debug or trust what you can't see.
- **Evaluation** — you cannot optimise what you cannot measure. Even a tiny eval
  (5–8 representative tasks + expected outcomes + a simple judge) turns "did that
  change help?" into a number instead of a vibe.

---

## The non-negotiables checklist

Before shipping any agentic system, confirm:

- [ ] Chose the **simplest** architecture that fits (didn't reach for multi-agent reflexively).
- [ ] Named the **orchestration pattern** and can justify it.
- [ ] Defined the **shared state** and how each field merges (overwrite vs. accumulate).
- [ ] Each agent has **few, excellent tools** with clear docstrings.
- [ ] **Instructions** read like a detailed job description, including "done when…".
- [ ] A **termination condition** exists (and is composed, for autonomous loops).
- [ ] High-cost actions require **human approval**; read-only actions don't.
- [ ] Steps are **observable/traced**.
- [ ] A minimal **eval** exists so changes can be measured.
- [ ] Context is kept lean (no context rot).

When debugging an existing system that's flaky, looping, expensive, or
unreliable, go straight to **`references/failure-modes.md`** — it maps the ten
common symptoms to their root cause and fix.

## References

- **`references/patterns.md`** — the six orchestration patterns: how each works, trade-offs, when to use, and how to combine them.
- **`references/frameworks.md`** — concept → primitive mappings and idiomatic snippets for **LangGraph**, **OpenAI Agents SDK**, and **Google ADK**, plus how to choose.
- **`references/memory-and-evals.md`** — designing tools, memory (short/long, app- vs agent-managed, RAG), context engineering, human-in-the-loop, and building a minimal eval.
- **`references/failure-modes.md`** — ten predictable failure modes, each with symptom → cause → fix, for diagnosing and hardening systems.
