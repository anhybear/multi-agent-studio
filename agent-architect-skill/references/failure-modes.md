# Failure Modes: Symptom → Cause → Fix

Multi-agent systems fail in consistent, predictable ways. Use this as a
diagnostic checklist when a system is flaky, expensive, looping, or unreliable.
Almost every fix is an **architecture/specification** change, not a coding heroics
problem — which means you can usually fix it by directing the build better.

| # | Symptom | Root cause | Fix |
| --- | --- | --- | --- |
| 1 | Vague, inconsistent results | **Instructions too thin** | Rewrite instructions as a detailed job description (role, behavioural rules, tool-use guidance, output format, error handling, explicit "done when…"). Grow them from observed eval failures. *Highest-leverage, cheapest fix — do this first.* |
| 2 | Picks the wrong tool / fumbles tools | **Poor or too many tools** | Reliability over breadth: fewer, excellent, well-documented, domain-specific tools. Improve docstrings; return structured errors; cut redundant/overlapping tools that clog context. |
| 3 | Worked, then got worse after a model swap | **Instructions don't match the model** | Prompts aren't portable across models or even versions. Keep model-specific instruction sets and A/B test against your eval whenever you switch. |
| 4 | Runs forever / huge token bill | **No (or weak) termination** | Compose stop conditions: hard cap (max turns/tokens/time) AND a completion signal (a "task complete" tool the agent calls) AND a human stop. Set conservative limits first. |
| 5 | Treats risky actions as casually as safe ones | **No cost-aware delegation** | Classify actions by consequence; gate irreversible/expensive/sensitive ones behind human approval. To an agent all actions look equal — encode the difference. |
| 6 | Wrong pattern for the task (over- or under-powered) | **Pattern mismatch** | Match pattern to task (see `patterns.md`). Don't use AI-driven orchestration for a linear pipeline (3–5× cost, no benefit). Start with the simplest pattern; add autonomy only where a fixed path demonstrably fails. |
| 7 | Repeats the same mistakes every session | **No memory / not learning** | Add long-term memory (RAG/Store) so it accumulates knowledge — but only if an eval shows the overhead pays off. Stateless is fine for independent tasks. |
| 8 | On long tasks, ploughs ahead down a doomed path | **No metacognition** | Use a plan-based orchestrator that evaluates each step's success and re-plans/retries on failure, rather than blindly continuing. |
| 9 | "Is this change better?" — can't tell | **No evals (the meta-failure)** | Build a minimal eval (5–8 tasks + expected outputs + a judge). You cannot optimise what you cannot measure. See `memory-and-evals.md`. |
| 10 | Risky actions slip through without sign-off | **No human delegation** | Define escalation: rule-based (payment > $X, N failures, regulated domains) or LLM-based ("this mentions legal action — escalate") routed to a person. |
| 11 | Quality degrades the longer a single run goes | **Context rot** | Context engineering: summarise old history, trim bulky tool outputs, retrieve memory selectively, scope what each agent sees. Feed it less, but the right less. |

## The bonus failure mode: you probably didn't need multi-agent at all

Multi-agent architectures multiply the surface area for errors. Before building
one, check the four signals — is the task genuinely **decomposable**, does it need
**diverse expertise**, **heavy context** per step, and an **adaptive** approach? If
few are true, a single well-designed agent (or even a direct model call) will be
more reliable, cheaper, and faster. A good single agent routinely beats a badly
designed team.

## Diagnostic flow

1. **Is it producing bad/inconsistent output?** → start at #1 (instructions),
   then #2 (tools), then #3 (model match). Don't reach for a bigger model or more
   agents until instructions and tools are tight.
2. **Is it expensive or non-terminating?** → #4 (termination), #6 (wrong/too-
   autonomous pattern), #11 (context bloat inflating tokens).
3. **Did it do something it shouldn't have?** → #5 and #10 (cost-aware gating +
   human escalation).
4. **Can't tell if your fixes help?** → #9 — build the eval first; it makes every
   other fix measurable.

> The model-optimisation ladder: optimise the **agent system** (instructions →
> tools → orchestration) before touching the **model**. Most teams reach 95%+ on
> agent-system tuning alone. Model-level moves (routing, cascading, distillation,
> finetuning) are warranted only for residual domain-specific failures or for
> efficiency (cost/latency/privacy) at scale. When someone says "let's finetune,"
> first ask: "have we exhausted prompt, tool, and orchestration fixes?"
