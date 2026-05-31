# Framework Mapping: LangGraph · OpenAI Agents SDK · Google ADK

The architecture is the same everywhere; only the vocabulary changes. Use the
master table to translate a design into the user's framework, then the per-
framework sections for idiomatic snippets. **Verify exact API names against
current docs before finalising** — these frameworks move fast — but the concepts
and shapes below are stable.

## Master mapping

| Concept | LangGraph | OpenAI Agents SDK | Google ADK |
| --- | --- | --- | --- |
| Single agent + loop | `create_react_agent(model, tools)` | `Agent(...)` + `Runner.run()` | `LlmAgent(...)` + `Runner` |
| Tool | `@tool` function | `@function_tool` function | Python function (docstring = spec) |
| Shared state | `TypedDict` state + reducers | `Session` / context object | `session.state` + `output_key` |
| How state merges | reducer, e.g. `Annotated[list, add_messages]` | append to session history | write key → next agent reads it |
| Sequential | chained `add_edge(a, b)` | chain agents / handoffs | `SequentialAgent(sub_agents=[…])` |
| Parallel | fan-out edges + reducer fan-in | `asyncio.gather` over runs | `ParallelAgent(sub_agents=[…])` |
| Loop / iterate | cyclic edges + `recursion_limit` | the `Runner` agent loop | `LoopAgent(max_iterations=…)` |
| Conditional / route | `add_conditional_edges(node, router)` | `handoffs=[…]` (LLM routes) | coordinator `LlmAgent` w/ `sub_agents` |
| Plan-based / supervisor | `langgraph-supervisor: create_supervisor(...)` | orchestrator `Agent` using agents-as-tools | coordinator `LlmAgent` delegating to `sub_agents` |
| Handoff | `Command(goto=…)` / `langgraph-swarm` | `handoffs=[agent_b]` → `transfer_to_agent_b` | `sub_agents` (auto-transfer) |
| Agent used as a tool | wrap compiled graph as `@tool` | `agent.as_tool(...)` | `AgentTool(agent=…)` |
| Long-term memory | a `Store` (e.g. vector store) | `Session` persistence | `MemoryService` / session state |
| Human approval / HITL | `interrupt()` + `Command(resume=…)` | guardrail + manual approval step | `before_tool` callback / approval gate |
| Guardrails | middleware / pre-post hooks | input & output guardrails | `before_*` / `after_*` callbacks |
| Tracing / observability | LangSmith | built-in tracing | built-in / Cloud Trace |
| Termination | `recursion_limit` + edge to `END` | `max_turns` | `LoopAgent` max iterations / escalate |
| Build → run boundary | `builder.compile()` then `.invoke()` | define `Agent` then `Runner.run()` | define agent tree then `Runner.run()` |

**The universal idea:** *shared state*. LangGraph's typed state + reducers, ADK's
`session.state` + `output_key`, and OpenAI's Sessions are the same concept —
agents coordinate through a shared store, not by calling each other. Get the
state design right and the rest follows.

---

## LangGraph

Graphs over shared state, with durable execution (checkpointing). Best when you
need explicit control of flow, branching/routing, parallelism, human-in-the-loop,
or resume-after-crash. (LangChain = component toolkit; LangGraph = orchestration;
LangSmith = tracing/evals.)

```python
from langgraph.graph import StateGraph, START, END
from typing import Annotated
from typing_extensions import TypedDict
from operator import add

class State(TypedDict):
    brief: str
    research: str
    concepts: Annotated[list, add]   # reducer: APPEND, don't overwrite

def research(state): return {"research": do_research(state["brief"])}
def ideate(state):   return {"concepts": [draft(state)]}

def route(state) -> str:            # conditional edge = router
    return "legal" if has_claim(state) else END

b = StateGraph(State)
b.add_node("research", research); b.add_node("ideate", ideate)
b.add_edge(START, "research"); b.add_edge("research", "ideate")
b.add_conditional_edges("ideate", route)
graph = b.compile(checkpointer=InMemorySaver())   # state saved per thread_id
graph.invoke({"brief": "..."}, {"configurable": {"thread_id": "acme"}})
```

- **Single agent:** `from langgraph.prebuilt import create_react_agent`.
- **Supervisor (plan-based):** `from langgraph_supervisor import create_supervisor`.
- **Handoff (swarm):** `from langgraph_swarm import create_swarm`, or `Command(goto=…)`.
- **HITL:** `interrupt(...)` pauses; resume with `Command(resume=value)` — works because the checkpointer persists state.

## OpenAI Agents SDK

Lightweight, agent-loop-first. `Runner` runs the loop: call model → handle tool
calls → follow handoffs → stop on final output or `max_turns`. Handoffs and
agents-as-tools are first-class; tracing is built in.

```python
from agents import Agent, Runner, function_tool

@function_tool
def search_web(query: str) -> str:
    """Search the web. Returns top results as text."""
    return do_search(query)

researcher = Agent(name="Researcher", instructions="Gather cited facts.",
                   tools=[search_web])
writer     = Agent(name="Writer", instructions="Draft copy from the research.")

# Plan-based / supervisor via agents-as-tools:
producer = Agent(
    name="Producer",
    instructions="Assign work, check it, decide what's next.",
    tools=[researcher.as_tool(tool_name="research",
                              tool_description="Run the researcher")],
)
# Handoff (router): the LLM calls transfer_to_writer when appropriate
triage = Agent(name="Triage", instructions="Route the request.",
               handoffs=[researcher, writer])

result = await Runner.run(triage, "Write a post on solar trends", max_turns=12)
```

- **Memory:** `Sessions` for persistent working context across turns.
- **Guardrails:** input guardrails (validate before running) and output guardrails (validate the result); run in parallel to the agent.
- **Termination:** `max_turns` on `Runner.run`.
- **Workflow patterns:** sequential = chain `Runner.run` calls or handoffs; parallel = `asyncio.gather` over independent runs.

## Google ADK

Has dedicated **workflow agents** for the deterministic patterns and LLM-driven
`sub_agents` for autonomy. Coordination is via shared `session.state` (write with
`output_key`, the next agent reads it).

```python
from google.adk.agents import LlmAgent, SequentialAgent, ParallelAgent, LoopAgent

researcher = LlmAgent(name="researcher", model="gemini-2.0-flash",
                      instruction="Gather facts.", tools=[search_tool],
                      output_key="research")          # writes to session.state
writer     = LlmAgent(name="writer", model="gemini-2.0-flash",
                      instruction="Draft from {research}.", output_key="draft")

pipeline = SequentialAgent(name="campaign",           # fixed order
                           sub_agents=[researcher, writer])

gather = ParallelAgent(name="markets",                # concurrent
                       sub_agents=[market_a, market_b, market_c])

refine = LoopAgent(name="refine", max_iterations=3,   # iterate until good
                   sub_agents=[writer, critic])
```

- **Plan-based / supervisor:** a coordinator `LlmAgent` with `sub_agents`; the LLM transfers control to the right one. Wrap an agent as a tool with `AgentTool`.
- **Conditional/routing:** a coordinator `LlmAgent` chooses among `sub_agents`.
- **Guardrails / HITL:** `before_tool` / `after_tool` (and model/agent) callbacks; gate high-cost tools there.
- **Memory:** `SessionService` for working state; `MemoryService` for long-term recall.

---

## Choosing a framework (if the user hasn't)

- **LangGraph** — maximum control over flow and state; first-class
  human-in-the-loop, checkpointing/resume, branching, parallelism. Best when
  reliability and durable, inspectable execution matter, or the graph is complex.
- **OpenAI Agents SDK** — lightest touch; excellent if you're already in the
  OpenAI ecosystem and want handoffs + agents-as-tools with minimal ceremony and
  built-in tracing. Great for getting a capable agent loop running fast.
- **Google ADK** — strong when you want explicit workflow agents
  (Sequential/Parallel/Loop) alongside LLM-driven delegation, and especially in
  the Google Cloud / Gemini ecosystem.

For a genuinely simple "model + a couple of tools, one shot," you may not need a
framework at all. Reach for one when you need: state across steps, branching/
routing, human approval, parallelism, durability, or multiple coordinated agents.
Pick the framework, then map the design using the table above — the architecture
doesn't change.
