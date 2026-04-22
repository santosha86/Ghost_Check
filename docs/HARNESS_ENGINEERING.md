# HARNESS_ENGINEERING.md — Why the harness is the moat

*A design note on multi-agent systems, with GhostCheck as the worked example.*

---

## The thesis in one paragraph

The boring part of a multi-agent system is the part you can't copy. LLM prompts are artefacts. Clever ones take an hour to write, extremely clever ones take a weekend, and once they exist a fork takes thirty seconds — copy the markdown, tweak two lines, done. Anyone who reads your repo can own your prompts by Monday morning.

The part nobody can clone by Monday morning is the scaffolding *around* the prompts: the input contract, the output schema, the validation at every boundary, the deterministic combiner, the fail-closed rules, the capability declarations. The boring, rigid, un-LLM part. Call this scaffolding the **harness**. In GhostCheck, the harness is what turns 11 independently-reasoning agents into a reproducible callback probability instead of 11 unrelated opinions. The harness is the moat. This file explains why.

---

## What "harness" means here

The word comes from the literal horse harness — a set of straps that don't make the horses stronger but force independent horses to pull in one direction. In software, the closest analog from the enterprise world is an API gateway, or a service mesh: the microservices do the business logic, and the gateway enforces uniform auth, rate limits, retries, logging, schema validation. The gateway is deliberately boring, and *that is what makes the services safe to expose*.

In a multi-agent AI system, the agents are the smart (and unreliable, occasionally hallucinating) parts. The harness is every non-agent piece around them:

- The shared input and output schema.
- The validator at every boundary.
- The deterministic combiner (the aggregator).
- The capability declarations that say which agent may touch what.
- The fail-closed rule that turns bad outputs into `UNKNOWN` instead of confident lies.

None of that involves an LLM call. All of it is load-bearing. This document walks through the five pillars that hold the GhostCheck harness up — and, by extension, the five pillars any serious multi-agent system needs.

*(Note: "harness" in AI has a second, unrelated meaning — "eval harness," the scaffolding around models for benchmark scoring. That's not what we mean here. We mean the production-time structural scaffolding around a live multi-agent system.)*

---

## Pillar 1 — One schema for all 11 agents

Every agent in GhostCheck returns the same shape. That shape is `AgentVerdict`, fully specified in `docs/SCHEMAS.md`. Eleven agents, one output contract, zero exceptions.

The reason this matters is not aesthetic. If even one agent invented a new field — a `confidence_adjustment` here, a `company_risk_note` there — the aggregator would need a special case for it. Special cases accumulate. After three of them the aggregator is no longer a combiner; it's an adapter layer. And an adapter layer is where judgment creeps in. "Let me just normalise this" turns into "let me decide what this means," and you now have an aggregator with opinions.

An aggregator with opinions is an aggregator that hallucinates.

The consequence of the single-schema rule is that the GhostCheck aggregator is tiny — roughly forty lines of math. It reads `score`, `severity`, and a weight per agent, applies the weighted-logistic formula, and emits a probability. It has no knowledge of what any individual agent does, and that ignorance is deliberate. You could swap out all 11 agents tomorrow, replace them with 11 different agents, and the aggregator would not need a single line changed.

> **One schema means no one agent gets to be special. And when no one is special, the aggregator doesn't need to be smart.**

---

## Pillar 2 — Agents run blind to each other

No agent in GhostCheck ever sees the output of another agent. Each agent receives the CV, the JD, and the `ExternalContext` enrichment blob — and that is all. Their verdicts converge only at the aggregator.

This is the "jury without deliberation" argument. Imagine if `bucket-classifier` could see what `ats-simulator` had returned before it formed its own judgment. LLMs are exquisitely sensitive to framing; one verdict in the prompt and subsequent verdicts cluster around it. You get apparent consensus, but the consensus is an artefact of the pipeline — not of the evidence.

Fan-out — same input to all agents, independent outputs, combined at the end — keeps each judgment genuinely independent. The aggregator then receives eleven signals that are not secretly the same signal echoed eleven times.

There is a counter-intuitive consequence of this design. Agents will sometimes disagree. `bucket-classifier` says CRITICAL; `ats-simulator` says LOW. In a chained pipeline this would look like a bug, and someone would write "reconciliation logic" to smooth it over. In the GhostCheck harness, disagreement is the signal. A CV that reads as both ATS-compliant *and* seniority-mismatched is telling you something specific about why it's getting ghosted. You'd never see that pattern in a pipeline that silently forced the agents into agreement.

> **Agents that talk to each other agree too much to be useful.**

---

## Pillar 3 — The aggregator is deterministic math, not another LLM

The aggregator lives in `.claude/skills/aggregator/aggregator.md`. It is a weighted-logistic function, nothing more. Weights come from `config/weights.yml`. Verdicts come in, a number comes out. No model is called. No external context is read. No reasoning is done.

This is the harness's anti-hallucination layer.

An LLM aggregator is tempting. It could "explain" its weighting. It could notice that two agents surfaced the same root cause. It could write prettier prose. It could also overrule a CRITICAL verdict because it seemed "too harsh." It could round 34% to "around a third" and lose audit-reproducibility. It could invent a silence driver that no agent surfaced. Any of those behaviours would be impossible to audit — you would never know whether the user's score moved because the evidence changed or because the LLM was in a different mood that afternoon.

A math aggregator cannot do any of these things. If a callback probability moves from 34% to 41% between two audits, exactly one of three things happened: a weight was tuned, a verdict changed severity, or an UNKNOWN became a non-UNKNOWN (see `SCHEMAS.md §13` on weight renormalisation). That is it. The system is deterministic by construction, and determinism here is the difference between a diagnostic tool you can stake a job search on and an oracle you cannot.

> **An orchestrator with opinions is just a twelfth agent in a trench coat.**

---

## Pillar 4 — Markdown-first scales to any runtime

Every agent, the aggregator, the router, and the enrichment skills in GhostCheck are plain markdown files with YAML frontmatter. No Python code runs an agent. The V1 runtime is Claude Code, which reads the markdown and executes the prompt against the Claude API. That is an implementation detail, not a design decision.

Markdown is the lowest-friction container for a prompt-plus-schema-plus-example that is both human-readable and machine-parseable. A contributor reads an agent file like prose. A runtime parses the frontmatter as a data structure. Both representations are exact; neither fights the other.

The pay-off of this choice is felt at the V2 boundary. The day GhostCheck outgrows Claude Code — because of scale, or because a team wants SSO-wrapped access, or because batch pattern analysis (V1.1) needs a real backend — the V2 Python service reads the exact same markdown files. The YAML frontmatter maps onto Pydantic fields. The prompt body becomes an Anthropic API call. The schema section becomes a validator. Content stays, runtime swaps. Nothing to port.

Compare this to the alternative, where the agent logic lives inside Python classes from day one. The "agents" in that world are half prompts and half code. To swap runtimes you have to rewrite, and the rewrite risks breaking the prompts. The markdown-first rule decouples these concerns permanently.

The payoff extends beyond *orchestrating* runtimes to a second kind of runtime — the kind that **rewrites the prompts themselves**. Evolutionary optimizers like DSPy + GEPA (the technique behind Nous Research's Hermes Agent self-evolution, grounded in an ICLR 2026 Oral paper) read agent execution traces, mutate the text of a prompt, score the mutations against real outcomes, and keep the winners. Because every GhostCheck agent is plain markdown — prompt as text, schema as text, examples as text — a GEPA-style evolver can point at `.claude/skills/agents/*.md` and start optimizing without a compilation step. A Python-compiled agent framework makes this expensive: you would need to re-serialize across code-object boundaries every iteration. Markdown is already the serialization format; nothing to convert. This is how GhostCheck reaches V1.1 self-evolution without rewriting V1 — the ROADMAP target is additive, not destructive.

> **The runtime is an implementation detail. The contract is the product.**

---

## Pillar 5 — Skill packs fork better than pip packages

GhostCheck is distributed as a Claude Code skill pack: a git repository of markdown, forked and used directly. It could have been distributed as a pip package (or an npm module, or a SaaS) — and it deliberately isn't. This is a distribution decision that follows from what the project actually is.

A pip package is the right shape for code you want to hide behind a stable API. Users call your functions; they do not read them. A skill pack is the right shape for content you want users to *read*, *edit*, and *re-publish*. And GhostCheck is, fundamentally, opinions-plus-prompts — the whole point of the project is that a forker reads the prompts, disagrees with some, tunes them to their context, and runs them on their own CV.

A pip package punishes that behaviour. To edit a prompt you would monkey-patch, subclass, or fork the whole package. A skill pack rewards the same behaviour natively: edit the markdown, commit, push. No build step, no release cycle, no semver to negotiate.

The second-order consequence is that contributions to GhostCheck look like pull requests to specific agent markdowns, not bug reports against an opaque binary. A hiring coach who specialises in IT-services professionals can fork the repo, tune `it-services-discount.md` to the nuances of their niche, and re-publish their fork as a vertical product. That distribution path does not exist in the pip world.

> **When the product is the prose, distribute the prose.**

---

## What the harness is NOT

A reader encountering this project for the first time will pattern-match it to something they already know, and often the pattern-match is wrong. Four negations to head off the most common wrong mental models.

**It is not a framework.** LangGraph, CrewAI, Autogen — those are frameworks. They ship Python code that you import and extend. The harness ships markdown that you read and fork. You don't install it; you clone it. If you were looking for a library to build multi-agent systems, this is not that — this is a specific multi-agent system, built on a reusable pattern.

**It is not Claude-specific.** Today's runtime is Claude Code. Tomorrow's runtime could be a Python service driving GPT, Llama, a locally-hosted model, or a mix. The agent files carry no Claude-specific assumptions beyond the prompt format, and the prompt format is generic markdown. The runtime is an implementation detail, not a constraint. V1.2's bring-your-own-LLM milestone operationalises this explicitly.

**It is not a state machine.** Agents do not carry state across audits. Each audit run builds its context fresh from the CV, the JD, and the enrichment blob. "Applications" are persisted as files in `applications/` — not as agent memory. If you were expecting a conversational, multi-turn, stateful multi-agent system, this is not that. It is a batch pipeline that produces one artefact per run.

**It is not an eval suite.** The harness does not score agent *correctness* against labelled data. Whether `bucket-classifier` is factually right about Director versus Staff is a separate concern from whether it returned a schema-valid verdict with cited evidence. The former is V1.2's calibration loop, where real callback outcomes feed back into weight tuning. The harness itself only guarantees the latter — that every agent either produces a well-formed verdict or surfaces as UNKNOWN.

---

## Why this matters beyond GhostCheck

The five pillars — one schema for all agents, blind agents, deterministic combiners, markdown-first runtime, skill-pack distribution — are not specific to career auditing. They generalise to any high-stakes agentic decision system where reproducibility, auditability, and composable independent judgments matter more than end-to-end cleverness.

Medical triage. Legal document review. M&A due diligence. Pre-interview candidate screening for your own next hire. Anywhere you want to combine multiple specialist views into one defensible output, the harness pattern applies. Give your agents one output shape. Keep them from influencing each other. Combine them with math, not judgment. Write them as content, not as code. And refuse to let your pipeline guess when it doesn't know.

The prompts are leaves. The agents get better each year as the underlying models improve — for free, on a cadence you don't control. The harness is the trunk. The harness is what you actually build.

> **The prompts get better on their own. The harness only gets better if you engineer it.**
