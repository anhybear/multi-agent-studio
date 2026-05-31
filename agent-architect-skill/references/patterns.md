# The Six Orchestration Patterns

All multi-agent coordination sits on a **spectrum of autonomy**: at one end you
(the developer) define the execution path (workflow patterns); at the other, the
agents decide at runtime (autonomous patterns). Control and flexibility trade off
inversely. There is no "best" pattern — only the right fit for the task.

## Contents
- Workflow patterns: Sequential, Conditional/Supervisor, Parallel
- Autonomous patterns: Plan-based, Handoff, Group chat
- Selection guide
- Hybrid composition & "agents all the way down"
- Cross-cutting: termination & human delegation

---

## Workflow patterns (explicit control — you drive)

Model the system as a graph: **nodes** are units of work (a function or an
agent), **edges** define what runs next. Benefits: you can validate the path
before running, visualize it, and get deterministic behaviour.

### 1. Sequential
Linear A→B→C; each step's output feeds the next. Predictable timing and clear
error isolation (you know exactly which step broke).
- **Use when:** steps are fixed, ordered, and dependent.
- **Example:** research audience → draft copy → fact-check.

### 2. Conditional / Supervisor
A control node reads the request/state and routes to the right next node —
branching logic. The "supervisor" variant is a router that delegates to
specialist agents and collects results.
- **Use when:** "it depends" routing; send each job to the right expert.
- **Example:** triage inbound requests — pricing → agent A, technical → agent B.
- Nesting supervisors yields **hierarchical** systems (multi-tier org charts).

### 3. Parallel
Fan-out into independent branches that run concurrently, then fan-in to combine
results. Requires care at fan-in (wait for all branches) and no side-effects
across branches.
- **Use when:** independent subtasks; you want throughput.
- **Example:** analyse five markets at once, then merge into one report.

> The primary advantage of workflow patterns is **reliability** — you know what
> happens and when. The cost is **flexibility**: they struggle when the optimal
> sequence can't be predetermined. When that assumption breaks, go autonomous.

---

## Autonomous patterns (emergent control — the AI drives)

Control flow is determined at runtime by model reasoning. More adaptive, less
predictable, more expensive (tokens grow with the conversation).

### 4. Plan-based (orchestrator / "project manager")
A single orchestrator agent creates a plan, assigns each step to a specialist
with only the context it needs, **evaluates whether each step succeeded**, and
re-plans or retries on failure. That self-evaluation is **metacognition**.
- **Use when:** complex tasks needing decomposition, oversight, and recovery.
- **Trade-off:** powerful and resilient, but the orchestrator is a single point
  of failure and a reasoning bottleneck.
- **Real systems:** Microsoft Magentic-One (task/progress "ledgers") and
  Anthropic's research system (a lead spawning parallel sub-agents) both use this.

### 5. Handoff
Agents operate with local knowledge and pass control directly to a peer they
know about — like a relay or a "tap on the shoulder." Implemented by representing
agents as tools (a handoff is a tool call: `transfer_to_X`).
- **Use when:** well-defined domains with clear handoff criteria; minimal central
  overhead wanted.
- **Trade-off:** scalable and specialised, but needs care to avoid agents getting
  stuck or cycling.

### 6. Group chat (conversation-driven)
All agents share one conversation; orchestration emerges through turn-taking. A
**selector** decides who speaks next — either **round-robin** (fixed rotation) or
**AI-driven** (an LLM picks the most useful next speaker based on context).
- **Use when:** open-ended, exploratory work where the path emerges through
  dialogue (brainstorms, research + write + critique loops).
- **Trade-off:** transparent and naturally collaborative, but token usage grows
  with conversation length and agents can make conflicting decisions.
- Naturally enables single-agent research patterns like ReAct
  (think→act→observe) and Reflexion (turn failures into lessons in the history).

---

## Selection guide

Ask "what does my task need?", not "which pattern is best?"

| Task characteristic | Pattern |
| --- | --- |
| Fixed, known, ordered steps | Sequential |
| Routing to specialists | Conditional / Supervisor |
| Independent subtasks, want speed | Parallel |
| Unknown decomposition, needs oversight | Plan-based |
| Known domain, peer delegation | Handoff |
| Uncertain path, exploratory | Group chat (AI-driven) |

| System requirement | Lean toward |
| --- | --- |
| High predictability / production reliability | Workflow patterns |
| Maximum autonomy | AI-driven group chat |
| Resource constraints / low coordination overhead | Handoff |
| Scalability | Parallel or Handoff |
| Human oversight | Any pattern + human delegation |

Comparative trade-offs:

| Pattern | Autonomy | Dev control | Complexity |
| --- | --- | --- | --- |
| Sequential | Low | High | Low |
| Conditional/Supervisor | Low | High | Low–Med |
| Parallel | Low | High | Medium |
| Plan-based | Medium | High | Medium |
| Handoff | Medium | Medium | Low |
| Group chat (AI-driven) | High | Low | Medium |

---

## Hybrid composition

Most production systems combine patterns: a **workflow skeleton** for the
predictable parts with **autonomy** injected only where flexibility is essential.
Strategy: prototype with workflows, measure with an eval, find where the fixed
path fails, and convert *only those sections* to autonomous orchestration.

**Agents all the way down:** any node/agent can itself be a full multi-agent
system internally, presenting as a single unit to the layer above. A plan-based
"Coder" agent might internally run a sequential research→code→test workflow. This
keeps each layer simple and lets you choose the right pattern per layer.

---

## Cross-cutting concerns (apply to every pattern)

### Termination — when does it stop?
Without this, autonomous loops run and bill forever. Three mechanisms, usually
composed:
- **Budget-based:** max messages, tokens, time, or iterations.
- **Semantic:** an LLM (or a "task complete" tool the agent calls) judges done.
- **External:** a human or API signal stops it.

Example: "terminate after 10 rounds OR when quality threshold met, whichever
first." Set conservative limits during development, tighten with real data.

### Human delegation — when does it escalate to a person?
- **Rule-based:** explicit triggers (any payment > $10k, 3 failed attempts,
  medical/legal domains). Deterministic and auditable.
- **LLM-based:** the agent reasons about risk/confidence ("this mentions legal
  action — escalate").

Pair this with cost-aware action gating: classify actions by consequence and
require approval for the high-cost ones (see `memory-and-evals.md`).
