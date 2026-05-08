# Effective Claude Code Setup

A working guide for individuals, teams, and companies adopting Claude Code. Focused on principles and patterns that hold up as the tooling evolves.

---

## Mental Model

**Brain, hands, session.** Any agentic system splits into three layers: the *brain* (the model plus the harness around it), the *hands* (the sandbox and tools that let it act), and the *session* (the durable log of what happened). Everything you set up — `CLAUDE.md`, skills, subagents, plugins, MCP servers — is harness work that shapes what the brain sees and what the hands can do. Get those right and the model becomes useful. Get them wrong and even the best model underperforms.

**Treat the agent like a brand new employee.** Broadly trained, capable, knows nothing specific about your codebase, conventions, or workflows. Your job is to onboard it.

**Less is more.** Find the smallest set of high-signal tokens that produce the desired outcome. Bloated context degrades performance, increases cost, and slows iteration.

**The model only sees the context window.** Nothing carries between sessions unless you make it carry. Persistence is your responsibility.

**Models predict tokens.** Post-training adds behaviour on top, but the underlying mechanism is sampling from a probability distribution. Models pattern-match on context — garbage in, garbage out. This paradigm is unlikely to shift dramatically; the leverage is in context engineering and the harness, not in waiting for the next model.

---

## Context as a Finite Budget

Performance degrades as the context window fills. This is consistent across every frontier model that has been tested. The practitioner consensus has converged on a hard rule:

**Cap context utilisation well below the auto-compaction trigger.** Compact, clear, or hand off at task boundaries rather than letting the limit drive it. This pattern is sometimes called Frequent Intentional Compaction.

**Why intentional beats automatic:**

- Fresh context beats summarised context. Compaction always loses information; the question is whether you control what's lost or the auto-compactor does.
- Compacting at task boundaries preserves coherent units. Compacting mid-implementation loses ongoing decisions.
- Larger context windows raise the ceiling but the degradation curve still applies. A massive context window does not give you a massive amount of clean reasoning.

**Operating rules:**

- Watch the status bar. Compact proactively at task boundaries, not when you hit the limit.
- Clear context between unrelated tasks. Cheaper than compaction.
- Prefer fresh sessions for new work. Cheaper still.
- When you do compact, give explicit preservation instructions: current goal, files changed, decisions made, errors hit, tests run, next step.
- Don't waste budget on what the model already knows. Spend it on what's unique to your project.

---

## CLAUDE.md and AGENTS.md

Your persistent project context. Most setups support generating a starter from the codebase — use it, then edit aggressively.

**Include:**

- Tech stack with versions and key dependencies. "React 18 with TypeScript, Vite, and Tailwind" — not "React project."
- Non-obvious tooling. "We use `task` not `make`." "Tests run via `bun test`."
- Project structure, the boundaries between packages or modules, and where things live.
- Naming conventions and patterns that aren't obvious from the code.
- Safety-critical rules. "Never edit generated files in `/proto`." "Migrations require review."
- The handful of commands the agent will need most often.

**Don't include:**

- General language or framework knowledge the model already has.
- Information the agent can recover cheaply with `find` and `grep` in a couple of file reads.
- Code-style rules. Use linters and formatters with hooks instead — never send a model to do a linter's job.
- Every command imaginable. Keep it tight.

**Practical limits:**

- Keep it tight. Frontier models reliably follow a finite number of instructions; the harness already consumes a chunk of that budget. Every line you add costs you adherence elsewhere. Targeting a few hundred lines or fewer is a good discipline.
- For sections that should only fire on specific tasks, use conditional wrappers. Make the conditions narrow. Don't wrap project identity, structure, or stack — those are always relevant.
- Don't auto-generate the file and leave it alone. Starter generators produce comprehensive output that almost always needs trimming.
- Don't `@`-mention large reference docs. That embeds the file every turn. Instead, tell the agent when to read it: "For complex Foo usage or if you encounter `FooBarError`, see `path/to/docs.md`."
- Use subdirectory variants in monorepos so package-specific context only loads in the relevant subtree.
- Lazy-load rules that only apply to certain file paths.
- Use deterministic settings for harness-enforced behaviour, not CLAUDE.md. If a rule can be enforced by tooling, enforce it by tooling.

**The test:** any new contributor should be able to launch Claude Code and ask "run the tests" and have it work first try. If it doesn't, your context file is missing essential setup or build commands.

**AGENTS.md** is the cross-tool standard. Many setups read it as a fallback when no Claude-specific file is present. For mixed-tool teams, the practical pattern is an `AGENTS.md` with universal rules and a thin Claude-specific file that imports it and adds Claude-specific behaviour. Don't duplicate code-style rules across both.

---

## Skills

Skills are reusable, on-demand capabilities. Each skill is a folder with a `SKILL.md` and optional bundled files. They use **progressive disclosure**: only the skill's name and description are loaded into the system prompt at all times; the body and any bundled `references/` or `scripts/` only load when the agent decides to use the skill.

User-invoked commands and model-invoked skills increasingly share the same primitive — defining a skill often gives you a slash command for free.

### When to use skills

Skills are for things *you* know that the model doesn't:

- Team workflows and standard operating procedures.
- Code structure preferences, design patterns, conventions.
- Repeated multi-step processes you find yourself re-explaining.
- Domain knowledge specific to your product or industry.

### When to create one

**Don't create a skill the first time you discover a workflow.** Walk through it manually with the agent at least twice. Watch what works and what doesn't. Then have the agent review what it just did, and codify it afterwards.

The agent needs to see what a "good run" looks like before you can codify it. Premature skill creation locks in patterns that haven't been validated. Independent research has shown that untested context files can actively *hurt* agent performance — getting this wrong has a real cost.

### Authoring discipline

- Keep `SKILL.md` short. Push deeper reference material into linked files.
- Make the description slightly pushy. Models tend to under-trigger skills. Phrase descriptions to invite use ("Use this skill whenever the user mentions X, Y, or Z — even if they don't explicitly ask for it").
- Test before shipping. Use whatever evaluation tooling your harness provides; run skills against held-out test cases and iterate on the description based on failures.
- Version skills like any other software artefact.

### Recursive improvement

When a skill fails:
1. Ask the agent why it failed.
2. Have it diagnose the specific error.
3. Update the `SKILL.md` based on what was learned.
4. Re-run the skill's evals to confirm the fix.

### Build your own

Generic skills are fine starting points. The highest-leverage skills are the ones you build for your own work — they capture taste, conventions, and tribal knowledge the model can't get any other way.

**Security note:** skills can include scripts and reference arbitrary files. Only install skills from sources you trust. Public skill registries have been found to contain malicious payloads. Review every `SKILL.md` before enabling.

---

## Plugins

A plugin is a directory bundling any combination of skills, subagents, hooks, slash commands, MCP server configs, and background monitors. Plugins are the unit of distribution and version control for harness work.

Use them to:

- Share a curated set of skills and subagents across a team.
- Pin specific versions of harness behaviour to a repo.
- Distribute company-wide conventions (review patterns, deployment workflows, security checks) without copy-paste.

For most individuals, you won't need plugins until you're sharing harness work with others. For teams, this is where standard practice lives.

---

## Tools and MCP

Tool definitions get loaded into the context window on every turn. Every tool you expose costs tokens before any work begins.

- A wide tool surface can burn significant context just on definitions.
- Bloated or overlapping tool sets create ambiguous decision points and degrade tool selection. If a human engineer can't tell which of two tools to use in a given situation, the agent can't either.
- Cap concurrent tool servers. A handful per project is a reasonable ceiling.

**Prefer agentic search over RAG-first patterns** for codebase work. Standard tools like `grep`, `find`, and `tail` let the agent pull what it needs just-in-time rather than pre-loading retrievals that may not be relevant.

**For large tool surfaces, use the code-execution pattern.** Instead of exposing tools as flat definitions, expose them as a typed API the agent writes code against, run in a sandbox, with only the relevant final result returned to the model context. Massive token savings, better composition, intermediate data never enters the context window.

**Treat tool servers as microservices.** Per-project credentials, read-only modes until usage patterns are observed, structured audit logs, secrets in a vault — not the JSON config.

---

## Scaling: From One Agent to Many

Going from zero straight to a multi-agent setup is like starting a company and immediately hiring ten people. Scale gradually, and only when the work demands it.

### Stage 1 — Single agent with good context

Start here. Most coding work never needs to leave this stage. Get your context file right, build a handful of skills, learn the harness.

The dominant pattern from people who ship a lot with coding agents is:

1. **Plan first.** Don't let the agent write a line of code until the plan is approved. Iterate on the plan. Ask "what are the flaws in this plan?" Ask the agent to challenge its own assumptions.
2. **Auto-accept once the plan is good.** A solid plan one-shots the implementation almost every time.
3. **Use a managed permission mode for daily work** rather than blanket permission-skipping. Modern agent harnesses include automated permission modes that probe inputs and review outputs for unsafe patterns. Pair them with sandboxing for anything touching real systems.

### Stage 2 — Subagents for context isolation

Add subagents when you find yourself spawning the same kind of investigation repeatedly. Subagents have isolated context windows — only the prompt string crosses into them, only the final message returns. This keeps the main session's context clean.

Common subagent patterns:

- A read-only **reviewer** that examines code with fresh context, avoiding the bias of reviewing what it just wrote.
- A read-only **explorer** for codebase search and investigation.
- Domain specialists: security review, infrastructure audit, test runner.
- Spec → architect → implementer → tester pipelines.

### Stage 3 — Agent teams for collaboration

Multi-agent orchestration becomes useful when:

- Teammates need to challenge each other (multiple hypotheses on a bug, made to disprove each other).
- Cross-layer changes where each layer has a teammate (frontend, backend, tests).
- Long-horizon multi-feature builds where coordination overhead is worth paying.

Pre-conditions before adopting:

- You have evals to know whether the multi-agent setup is actually doing better than a single agent.
- The task value justifies it. Multi-agent setups burn token budget linearly with team size, plus coordination overhead.
- You're consistently running three or more agents per task.

### When parallel multi-agent works, and when it doesn't

The reference case for unstructured parallel multi-agent (running many agents at once with no orchestrator) is work that **decomposes into many independent verifiable subtasks**. Give each agent a distinct failing test and a known-good oracle, and they can converge on the answer.

Parallel multi-agent **stalls** when work converges — when every agent ends up fixing the same bug and overwriting each other. For continuous builds, the better pattern is structured: a planner agrees on a contract, a generator builds against it, an evaluator verifies the output before accepting it.

For most teams, single-agent + plan mode + auto-accept beats both.

---

## Permission Modes and Sandboxing

Unrestricted automation flags are being phased out for daily use in favour of:

- **Managed permission modes.** Multi-layer defence that probes tool calls before they fire and reviews outputs after, pausing on suspicious patterns.
- **Sandboxing.** Run the agent's actions inside a container or VM where the blast radius is limited.

Practical rule: never run unrestricted agent automation on a workstation that has access to credentials you'd care about losing. Either sandbox the actions, or confirm them.

---

## Workflow Patterns

A handful of patterns that keep showing up in successful adoptions:

**Plan-first.** Approval gates beat speed. The cost of iterating on a plan is tiny compared to the cost of iterating on a half-implemented feature.

**Verification beats execution.** A large share of useful work is planning and review, not generation. Allocate accordingly.

**Use a `thoughts/` directory** (or equivalent) for cross-session state. Plans, research, decisions go there explicitly. This is what survives compaction.

**Tracer-bullet PRs over horizontal phasing.** Break work into vertical slices that go end-to-end (DB + service + UI) rather than horizontal phases (all DB first, then all API, then all UI). Agents default to horizontal; correct them. Vertical slices give end-to-end signal early.

**Ship demos, not docs.** The codebase is the source of truth. Documentation drifts; running code doesn't.

**Use deterministic tools wherever you can.** Linters, formatters, type checkers, hooks — never put their job in your context file or a skill if a tool can enforce it.

**Run a clean review pass.** A reviewer subagent with fresh context catches things the writing agent missed because it can't see its own assumptions.

**Re-evaluate the harness as models improve.** Every component in your harness encodes an assumption about what the model can't do on its own — and those assumptions go stale as models improve. What needed scaffolding a year ago may not need it now.

---

## Quick Reference

| Principle | Action |
|-----------|--------|
| Context is finite | Compact at task boundaries before the limit, prefer fresh sessions |
| Less is more | Cut anything the model already knows |
| Treat as new hire | Document what's specific to *your* project |
| Progressive disclosure | Use skills for on-demand expertise |
| Walk before codifying | Don't write a skill on first encounter — do it manually first |
| Test your context | Use eval frameworks; untested skills can hurt performance |
| Recursive improvement | Failed skill → diagnose → update → re-eval |
| Start simple | One agent → subagents → multi-agent, in that order |
| Plan first | Approve the plan before any code is written |
| Verification > execution | Spend more time on plans and reviews than on generation |
| Build your own | Generic skills are starters, custom skills are leverage |
| Watch tool bloat | Tool definitions cost tokens before any work begins |
| Cap tool servers | A handful per project is a reasonable ceiling |
| Sandbox automation | Managed permissions plus sandbox for anything touching real systems |
| Trust the harness | Models are capable; context and harness are the differentiator |
| Keep questioning the harness | Re-evaluate scaffolding as models improve |

---

## Further Reading

The fastest way to track how this evolves is the official engineering blog and documentation for whatever agent harness you're using. The principles in this guide are drawn from that primary material plus practitioner reports; specific feature names, version numbers, and thresholds shift constantly and have been kept generic on purpose.

If you want named sources, see `../CHANGELOG.md` for the dated references that informed each section.
