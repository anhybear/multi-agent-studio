# agent-architecture — a portable skill

Distilled architectural knowledge for building AI agents and multi-agent systems
well, so your coding agent (Claude Code, Codex, OpenCode, …) references solid
principles whenever it designs or builds an agentic system — on **any** framework
(LangGraph, OpenAI Agents SDK, Google ADK).

It's just markdown, so it's portable. `SKILL.md` is the entry point; the depth
lives in `references/`.

```
agent-architecture/
├── SKILL.md                      ← entry: build discipline + when to use
└── references/
    ├── patterns.md               ← the 6 orchestration patterns
    ├── frameworks.md             ← LangGraph / OpenAI Agents SDK / Google ADK mappings
    ├── memory-and-evals.md       ← tools, memory, context, human-in-the-loop, evals
    └── failure-modes.md          ← symptom → cause → fix diagnostic
```

## Install

### Claude Code (native skill)
Copy the folder into your skills directory and restart:

```bash
# Personal (all projects):
cp -r agent-architect-skill ~/.claude/skills/agent-architecture
# OR project-only:
mkdir -p .claude/skills && cp -r agent-architect-skill .claude/skills/agent-architecture
```

Claude Code reads `SKILL.md`'s frontmatter and auto-invokes it when a task
matches the description. No further wiring needed.

### Codex / OpenCode / other agents
These don't all share Claude's skill auto-loader, but they read an instructions
file (`AGENTS.md`, or the tool's rules/config). Two options:

1. **Drop the folder into your repo** (e.g. `docs/agent-architecture/`) and add
   the pointer below to your `AGENTS.md` (Codex) or rules file (OpenCode).
2. **Paste `SKILL.md` directly** into your global instructions if you want it
   always loaded.

Pointer to add to `AGENTS.md` / your agent's rules:

```markdown
## Building AI agents / multi-agent systems
When designing, building, scaffolding, refactoring, or debugging an AI agent,
an agentic workflow, or a multi-agent system, FIRST read
`docs/agent-architecture/SKILL.md` and follow its build order (choose the
simplest architecture → name the orchestration pattern → design shared state +
each agent → map to the framework in references/frameworks.md → add termination,
human-in-the-loop, and a minimal eval). Consult references/failure-modes.md when
a system is flaky, looping, or expensive.
```

### As a Claude Code plugin (optional)
To distribute via a plugin marketplace, wrap this folder in a plugin: add a
`.claude-plugin/plugin.json` manifest and place this directory under the plugin's
`skills/`. See the Claude Code plugin docs for the current manifest schema.

## Keeping it current
This skill lives alongside the interactive learning app in the same repo, which
is the source of truth. As frameworks evolve, update `references/frameworks.md`
(API names move fast — the concepts are stable) and re-sync to wherever you've
installed it.

## What it will make your agent do
- Stop reflexively reaching for multi-agent designs; choose the simplest thing
  that works.
- Name the orchestration pattern and justify it before building.
- Design shared state and how it merges (the thing that breaks most often).
- Spec few-but-excellent tools, the right memory, and lean context.
- Always add a termination condition, human approval for high-cost actions, and
  a minimal eval.
- Diagnose flaky systems against the ten known failure modes.
