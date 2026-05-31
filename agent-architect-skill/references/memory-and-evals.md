# Agent Internals: Tools, Memory, Context, Human-in-the-Loop, Evals

How to design what's *inside* an agent and the guardrails *around* it. These are
framework-agnostic; see `frameworks.md` for the specific primitives.

## Contents
- Tools (the action space)
- Memory (short/long, app- vs agent-managed, RAG)
- Context engineering (avoiding context rot)
- Human-in-the-loop & cost-aware delegation
- Minimal evaluation (the meta-skill)

---

## Tools — the action space

Tools define what an agent can *do*, so they cap its reliability. Principles:

- **Reliability over breadth.** A few excellent, domain-specific tools beat many
  mediocre or overlapping ones. Large toolsets confuse the model and clog
  context. ~10 great tools is a healthier target than 50.
- **The docstring is the interface.** The model decides whether/how to call a
  tool from its name, description, and parameter docs. Write them like you're
  instructing a smart new hire. State what it returns and its constraints (rate
  limits, data freshness, failure modes).
- **Never crash the agent.** Tools should return structured errors, not raise.
  Add retries/backoff for transient failures; validate outputs before returning.
- **Two kinds:** general-purpose (code runner, browser, file ops) and
  domain-specific (your "check brand guidelines" API). Invest in the latter.
- **A "think" tool helps.** A tool that just lets the agent reason/plan before
  acting measurably improves complex tasks. Sometimes the best action is
  structured thought.
- **Agents as tools.** An entire agent (or sub-team) can be exposed to another
  agent as a single tool — the caller sees a tool that returns a result and
  doesn't care about the internal complexity. This is how handoffs and large
  sub-agent systems are built ("agents all the way down").

---

## Memory

Two temporal kinds:

- **Short-term** — working memory for the current task (the conversation/state).
  In a multi-agent system this is the shared context agents coordinate through.
- **Long-term** — knowledge that persists across sessions. Usually implemented
  with **RAG**: store information as searchable vectors, retrieve the relevant
  pieces on demand (backends: vector stores like Chroma, Qdrant, Pinecone).

Two control models:

- **Application-managed** — your code decides what to store and retrieve, on
  fixed rules. Predictable.
- **Agent-managed** — the agent decides, using memory *tools* to curate its own
  knowledge. More autonomous, more flexible.

**Design rule:** not every agent needs long-term memory. Stateless agents are
simpler, faster, and cheaper. Memory adds latency (retrieval) and cost
(embedding/storage). Add it only when an eval shows it improves task success
enough to justify the overhead, and watch for diminishing returns past a handful
of retrieved memories.

---

## Context engineering — avoid "context rot"

A context window is finite, and quality *degrades* when it's overloaded with long
histories, bloated tool outputs, and every retrieved memory. "Context rot" /
"context explosion" shows up as: the agent forgetting earlier instructions and
getting worse the longer a run goes.

Management strategies:
- **Summarise** old conversation history once it grows.
- **Trim** verbose tool outputs to what's needed downstream.
- **Retrieve selectively** — pull only the memories relevant to the current step.
- **Scope sharing** — in plan-based orchestration, give each agent only the
  context it needs, not the whole history.

Think of it like a tight creative brief: feed the agent *less, but the right
less*. If an agent degrades over a long run, suspect context before adding tools
or agents (which usually make it worse).

---

## Human-in-the-loop & cost-aware delegation

Autonomous systems are non-deterministic — the same input can take a different
path each time. You can't make them perfectly predictable, so design controls
around them.

**Classify actions by consequence** and handle each tier differently:

| Cost tier | Characteristics | Examples | Handling |
| --- | --- | --- | --- |
| Low | read-only, reversible | search, read a file, calculate | auto-run |
| Medium | changes things, recoverable | reschedule a meeting | notify after / preview |
| High | irreversible, costly, sensitive | send email, delete data, pay, publish | **require human approval first** |

To an agent, "delete 400 live rows" and "search the web" look equally weighted —
your design must encode the difference. Implement approval as a pause that
surfaces the action and its parameters to a human (LangGraph `interrupt()`,
OpenAI guardrail + approval step, ADK `before_tool` callback).

Tune it with data: a >95% approval rate means you're flagging too much (approval
fatigue); missing real risks means too little.

Also design for the human's experience:
- **Capability discovery** — show what the system is reliably good at (e.g.
  preset example tasks) so people don't throw unsuitable jobs at it.
- **Observability & provenance** — show what each agent did and *why*, with
  sources. "Calendar agent blocked the date" beats "Error: failed."
- **Interruptibility** — let people pause, correct, and resume without losing
  progress (depends on persisted state).

---

## Minimal evaluation — the meta-skill

**You cannot optimise what you cannot measure.** Without an eval, every change to
prompts, tools, or orchestration is a guess. An eval is just: a set of
representative tasks, their expected outcomes, and a judge that scores the
system's answers.

Build the smallest useful one first (5–8 tasks):
1. **Tasks** — representative inputs with clear expected outputs, tagged by
   category (tool use, summarisation, routing, etc.).
2. **Judge** — exact-match where outputs are deterministic; an LLM-as-judge for
   open-ended quality; or reference-free checks (did it call the right tool? cite
   sources?).
3. **Run & compare** — score before and after each change. Now "did that help?"
   is a number.

Use the eval to drive the optimisation order (cheapest, highest-leverage first):
1. Sharpen **instructions** (expand the job description from observed failures).
2. Improve **tools** (docstrings, reliability, cut redundant ones).
3. Fix **orchestration** (right pattern, termination, context scoping).
4. Only then consider **model-level** changes — routing cheap/expensive models,
   cascading, distillation, finetuning — and only for residual domain-specific
   failures or efficiency at scale. Most teams reach 95%+ before this.

Tracking metrics like average rounds-to-completion, token usage distribution, and
failure-mode breakdown (timeout vs. max-rounds vs. completion) also lets you
calibrate termination conditions sensibly.
