# Effective Claude Code Setup

A working guide for individuals, teams, and companies adopting Claude Code. Focused on principles and patterns that hold up as the tooling evolves.

---

## Mental Model

**Brain, hands, session.** Any agentic system splits into three layers: the *brain* (the model plus the harness around it), the *hands* (the sandbox and tools that let it act), and the *session* (the durable log of what happened). Everything you set up — `CLAUDE.md`, skills, subagents, plugins, MCP servers — is harness work that shapes what the brain sees and what the hands can do. Get those right and the model becomes useful. Get them wrong and even the best model underperforms.

**Treat the agent like a brand new employee.** Broadly trained, capable, knows nothing specific about your codebase, conventions, or workflows. Your job is to onboard it.

**Less is more.** Find the smallest set of high-signal tokens that produce the desired outcome. Bloated context degrades performance, increases cost, and slows iteration.

**The model only sees the context window.** Nothing carries between sessions unless you make it carry. Persistence is your responsibility.

**Models predict tokens.** Post-training adds behaviour on top, but the underlying mechanism is sampling from a probability distribution. Models pattern-match on context — garbage in, garbage out. This paradigm is unlikely to shift dramatically; the leverage is in context engineering and the harness, not in waiting for the next model.

**Don't fix a context problem by switching models.** When an agent fails on a long task, the first lever is context engineering — scope, compaction, scaffolding, tool budget. Model upgrades exist and sometimes matter, but they don't substitute for harness work; they expose where the harness is missing.

**Match the model to the task.** Top-tier models aren't free, and using them for everything wastes both cost and the weekly limits that shape what you can do in a session. Use cheap, fast models for mechanical work — summarising, formatting, classification. Use mid-tier for routine reasoning — drafting code, refactoring, bounded analysis. Reserve top-tier with high effort for the work where getting it wrong is expensive — system design, debugging across many files, reviewing production code, and the orchestrator that routes the others. A common pattern: top-tier plans and verifies, mid-tier executes, cheap-tier handles the boilerplate. The agent's intelligence budget is finite; spend it where it matters.

---

## What Makes Up the Context

Every turn, Claude Code assembles its context window from a fixed set of components:

- **System prompt** — the harness's built-in instructions. You don't control this directly, but it consumes a meaningful share of the budget.
- **CLAUDE.md / AGENTS.md** — your persistent project context. Loaded on every turn.
- **Skills** — names and descriptions of every available skill are loaded into the system prompt at startup. Skill bodies and bundled files load only when invoked.
- **Tool definitions** — every tool exposed (built-in, MCP, custom) has its schema loaded into context before any work begins.
- **Codebase** — files the agent has read this session.
- **Conversation history** — your messages and the agent's previous turns and tool results.

Knowing what's in the context window tells you where the budget goes, and where the leverage is. Bloated tool surfaces, oversized context files, and unfocused conversation history all cost the same thing: tokens that the model has to reason around.

The rest of this guide is about managing each of these components deliberately.

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

### Skills are third-party code

A skill that ships scripts or references arbitrary files has the same blast radius as a plugin hook. Once enabled, it can execute on your machine on every relevant turn. Treat skills the way you'd treat any third-party dependency:

1. Official vendor sources first.
2. Plugins and skills published by the company whose system they integrate with.
3. Curated community sources from known maintainers.
4. Arbitrary repositories on the open internet — review before enabling.

Public skill registries have been found to contain malicious payloads, including credential theft and ransomware staging. Read every `SKILL.md` before enabling. Pay particular attention to bundled scripts and `allowed-tools` declarations. Skills that contain no executable code (instructions only) are roughly as risky as system-prompt content; skills that ship scripts are software you're choosing to run.

---

## Hooks

Skills, CLAUDE.md, and prompts are probabilistic guidance — the agent reads them and *usually* does the right thing. Hooks are different. A hook is a script that fires deterministically on a lifecycle event (before a tool call, after a file write, when a session ends). It runs every time, and it can block.

Use hooks for things that must always happen, not things you'd like to happen:

- Format and lint code after every write.
- Block destructive commands before they run.
- Run tests before allowing a commit.
- Inject project-specific context at session start.
- Notify external systems on session end.

The mental model: skills capture knowledge the agent should reach for; hooks enforce invariants the agent cannot break. They're complementary, not competing — most serious setups use both.

Hooks belong in your context file's territory of *deterministic enforcement*. Anything you've been tempted to write as "never do X" or "always do Y" in CLAUDE.md is probably better expressed as a hook. The instruction can be ignored under context pressure; the hook cannot.

Hooks aren't a security boundary — a sufficiently determined prompt injection can find ways around them, and the script itself runs with whatever permissions you give it. But for the everyday case of "I want this thing to happen reliably," hooks are the only layer of the stack that gives you *reliably*.

---

## When to Reach Beyond the Basics

The setup described above — single agent, good context, a handful of skills, built-in tools, plan-first workflow — is enough for most work. Most teams never need more than this. The agent already has filesystem, shell, and search; through those primitives it can read your code, run your tests, hit APIs via curl, query databases via the CLI, and interact with anything you have a command-line tool for.

The serious practitioners shipping agentic-coding workflows in production take a measured view of MCPs and plugins — they're tools of last resort behind CLIs and skills, not defaults. Reach for them when you hit specific limits.

### MCPs — when the agent needs to develop against a system, not just call it

The distinction matters. *Calling* a system is firing a request and parsing the response — `curl` handles that fine. *Developing against* a system means reading state, planning changes, making coherent updates, verifying, and iterating with the system's model in the agent's head while it works.

MCPs earn their place in the second case. Strong examples:

- **Stripe** — building billing logic, reasoning about products and prices, configuring webhooks, verifying test charges. The agent benefits from the system's vocabulary, not just its endpoints.
- **Terraform** — reasoning about state, planning changes, understanding resource dependencies. Far better than parsing `terraform plan` output.
- **AWS and other cloud providers** — reasoning about resource graphs, IAM, networking, infrastructure relationships rather than calling individual endpoints.
- **Databases with complex schemas** — structured introspection, type-aware queries, schema-aware migrations. Use read-only roles and never point at production.
- **Browser automation** — there's no curl-equivalent for clicking, filling forms, or asserting on rendered output.
- **Observability** — feeding live production telemetry (logs, traces, metrics, incidents) into the agent during a debugging session.

When MCPs are usually not worth it:

- Your own codebase — the agent has the filesystem already.
- Your own services — the agent can read the handler code and call the API directly.
- Stateless API surfaces where a well-written CLI works just as well. Most public APIs fall into this bucket.
- Anything you'd add "just in case." Tool definitions cost context every turn, regardless of whether they're used.

**Treat MCPs as a context-budget item, not a feature checklist.** Tool definitions get loaded into context on every turn, before the agent does any work. Three thoughtfully chosen servers will leave the agent room to think; ten will starve it. Real configurations have been measured consuming nearly half the context window on tool definitions alone before any task begins. If your harness lets you defer tool definitions until needed (sometimes called tool search or progressive loading), turn it on.

The ecosystem is also moving from "load every tool definition upfront" to "let the agent write code against a typed SDK and execute it in a sandbox" — a pattern variously called Code Mode or code-execution-with-MCP. Expect MCPs you adopt today to evolve toward this model. Servers that already work this way (a single `search` and `execute` interface backed by a typed API) cost a fraction of the context of traditional tool-registry MCPs.

A handful of MCPs per project is a reasonable ceiling. Pick the ones that match systems you actively develop against; skip the rest.

For everything else — agentic search over RAG-first patterns, treating tool servers as microservices with per-project credentials, read-only modes, audit logs, secrets in a vault — the same hygiene applies as it does to any external dependency.

### Plugins — when you need to share your setup, not extend it

Plugins are a distribution mechanism. They don't give the agent any capability that skills, subagents, hooks, and MCP configs don't already provide individually — what they give you is packaging.

Reach for them when:

- You're sharing harness work across a team and want everyone to have the same skills, subagents, and hooks without copy-paste.
- You want to version-control your harness setup alongside the code it operates on.
- You're distributing patterns publicly.

Skip them when you're working solo or with a small team that can sync via the codebase. Plugins solve a distribution problem; if you don't have one, they add ceremony without benefit.

**Trust hierarchy matters.** A plugin can execute arbitrary code on your machine. The reasonable order of trust, roughly:

1. Official vendor marketplace published by the harness maintainer.
2. Plugins published by the company whose system they integrate with (the people who run the service publishing their own plugin).
3. Curated community marketplaces from known maintainers.
4. Arbitrary repositories on the open internet.

Treat the last tier the way you'd treat any unreviewed dependency. Read the manifest, look at the hooks and scripts, decide if you trust them.

### The pattern

MCPs and plugins aren't starting points. They're responses to friction you've actually felt. If you haven't hit the friction, you don't need them yet — and adding them speculatively makes your setup worse, not better.

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

Set the model tier per subagent. Subagents doing bounded, routine work — explorers, test runners, file searchers — can run on a mid-tier model. Subagents making judgment calls that matter — reviewers, security auditors, architects — should run on the top tier. The orchestrator (the main session) runs top-tier with high effort because routing decisions cascade; getting them wrong wastes every downstream call.

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

## Cross-Session State

The agent has no memory between sessions. Anything you want to carry forward — plans, research, decisions, postmortems — has to live somewhere it can read again later.

The pattern that's emerged across serious setups is a project-scoped, version-controlled directory of dated Markdown documents. Different teams use different names — `thoughts/`, `ai-docs/`, `decisions/`, a `CHANGELOG.md` at the root — but the structure is consistent:

- A single root location, checked into the repo.
- Subdirectories by artefact type: research, plans, decisions, postmortems.
- Dated, descriptive filenames.
- The agent reads these at the start of relevant sessions and writes new ones at phase boundaries.

This is what survives compaction. It's also what lets a fresh session pick up where a previous one left off — including a session run by a different team member, a different agent, or a future you who's forgotten the context.

A few principles:

- Write at phase boundaries (after research, after a plan is approved, after a debugging session resolves), not continuously. Small, decisive artefacts beat sprawling logs.
- Keep them human-readable. Other people on the team will read these too.
- Reference them deliberately. Tell the agent in CLAUDE.md when to look there ("Before starting a new feature, check `decisions/` for prior conventions"), rather than @-mentioning everything.

Build small read-side helpers — slash commands or subagents that find and re-attach relevant prior artefacts — once you have enough of them to need indexing.

---

## Workflow Patterns

A handful of patterns that keep showing up in successful adoptions:

**Plan-first.** Approval gates beat speed. The cost of iterating on a plan is tiny compared to the cost of iterating on a half-implemented feature.

**Verification beats execution.** A large share of useful work is planning and review, not generation. Allocate accordingly.

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
| Hooks for invariants | Use hooks for anything that must always happen |
| Skills are third-party code | Apply the same trust hierarchy as plugins |
| Persist what matters | Cross-session state lives in version-controlled Markdown |
| Start simple | One agent → subagents → multi-agent, in that order |
| Plan first | Approve the plan before any code is written |
| Verification > execution | Spend more time on plans and reviews than on generation |
| Build your own | Generic skills are starters, custom skills are leverage |
| Watch tool bloat | Tool definitions cost tokens before any work begins |
| Cap tool servers | A handful per project is a reasonable ceiling |
| Sandbox automation | Managed permissions plus sandbox for anything touching real systems |
| Trust the harness | Models are capable; context and harness are the differentiator |
| Context > model upgrade | Fix the harness before reaching for a bigger model |
| Match model to task | Top tier for orchestration and judgment; mid for routine; cheap for mechanical |
| Keep questioning the harness | Re-evaluate scaffolding as models improve |

---

## Further Reading

The fastest way to track how this evolves is the official engineering blog and documentation for whatever agent harness you're using. The principles in this guide are drawn from that primary material plus practitioner reports; specific feature names, version numbers, and thresholds shift constantly and have been kept generic on purpose.

If you want named sources, see `../CHANGELOG.md` for the dated references that informed each section.