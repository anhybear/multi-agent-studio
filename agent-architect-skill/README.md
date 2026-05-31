# agent-architecture skill — moved

The maintained, canonical version of this skill now lives in the dedicated
skills monorepo (the single source of truth, symlinked into all of Anh's coding
agents):

**→ `anhybear/agent-skills` → `skills/agent-architecture/`**
(local: `~/Code/agent-skills/skills/agent-architecture/`)

It's a portable, framework-agnostic agent/multi-agent **architecture** skill —
pattern selection, shared state, memory, tools, termination, human-in-the-loop,
evals, and failure-mode avoidance — with concrete mappings to **LangGraph, the
OpenAI Agents SDK, Google ADK, and the Claude Agent SDK**, plus an
`install-symlinks.sh` for wiring it into every agent on a machine.

This learning-app repo originally hosted a copy; it was consolidated into
`agent-skills` to keep a single source of truth. Edit it there.
