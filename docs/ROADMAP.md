# GhostCheck Roadmap — V1 to V2

This document captures the state of GhostCheck V1 as shipped, the milestones planned for V1.1 and V1.2, and the longer-horizon V2 architecture. It is meant to be honest about what is implemented, what is documented but not yet implemented, and what is intentionally deferred.

The thesis underlying the roadmap: the V1 harness (eleven agents in isolated subagent contexts, deterministic aggregator, schema contracts, fail-closed) is the foundation. Subsequent versions add capabilities on top of that foundation without rewriting it. Because GhostCheck is a markdown-first skill pack rather than a compiled service, additive change is cheaper than redesign.

---

## V1 — Single-application audit (current state)

V1 is functional. Eleven agents and four enrichment skills run end-to-end, producing schema-faithful audit artefacts with cited evidence.

What V1 ships:

- **Eleven agents** producing AgentVerdict-shaped output:
  - Tier A invisible-failure agents: `bucket-classifier`, `google-test`, `posting-decoder`, `it-services-discount`, `headline-filter`.
  - Tier B channel-and-funnel-math agents: `funnel-math`, `channel-mix`, `stale-detector`.
  - Tier C standard-CV-quality agents: `ats-simulator`, `recruiter-30sec`, `hm-deep-read`.
- **Four enrichment skills**: `markitdown-parse`, `google-test-lookup`, `jd-age-detector`, `company-classifier`.
- **Slash command** `/ghostcheck audit --cv <path> --jd <path>` invoking the router skill that orchestrates parsing, enrichment, dispatch to all eleven subagents (Pattern B subagent isolation), schema validation, aggregator math, and audit artefact write.
- **Deterministic aggregator** computing callback probability via weighted-logistic over non-UNKNOWN verdicts: `P = 1 / (1 + exp(k * (z - midpoint)))` with V1 hyperparameters `k = 4.0`, `midpoint = 0.4`, weights from `config/weights.yml`.
- **Schema contracts** in `docs/SCHEMAS.md`: `AgentVerdict` (the cross-agent output shape), `Severity` enum, `Evidence` shape, `ExternalContext` (the enrichment blob), `Company` (with per-field provenance), `GoogleResult`, `UserProfile`, JD frontmatter spec, `applications/<date_slug>/` artefact layout, audit-report contract, and validation rules.
- **Audit artefact** at `applications/YYYY-MM-DD_<slug>/audit.md` plus `verdicts.json` and `context.json`. Privacy-respecting: `applications/*/` is gitignored.
- **Trust posture** documented in `docs/ZERO_TRUST.md`: aligned with Microsoft's *Zero Trust for AI* framework (March 2026), with NIST / CISA / CIS / Microsoft SFI as the standards chain. V1 enforces by design and by schema validation; runtime policy enforcement is V1.2.
- **Harness thesis** documented in `docs/HARNESS_ENGINEERING.md`: why the moat is the system around the agents, not any individual agent prompt.

What V1 does NOT include (intentional deferrals — see V1.1 and beyond below):

- Self-evolution of agent prompts via execution-trace mutation.
- Batch pattern analysis across multiple ghosted applications (the silence-signature feature).
- Calibration of weights and hyperparameters from real outcome data.
- PNG shareable cards or HTML slide decks (visual artefacts beyond `audit.md`).
- Bring-your-own-LLM (BYOK) — V1 runs on Claude Code with the Claude API.
- Multi-user state, web frontend, hosted runtime.

V1's documented technical debt is captured in `SESSION_RESUME.md` section 9 (post-Day-5 cleanup). The architectural debt (structured-extraction layer between parser and agents, hardcoded-content audit, cross-domain validation) is scoped and ready to execute as one focused session before V1.1 work begins.

---

## V1.1 — Self-evolution, batch pattern analysis, shareable artefacts

V1.1 is the first compounding-value milestone. V1 produces per-application audits; V1.1 starts learning across them and producing artefacts that travel beyond the user.

### V1.1-A: Self-evolution via DSPy + GEPA

GhostCheck is markdown-first by design. The V1 agent prompts are plain `.md` files with prompt body plus YAML frontmatter — exactly the artefact shape that an evolutionary text optimiser like DSPy + GEPA can mutate and score against real outcomes.

The path:

- Reference implementation: `NousResearch/hermes-agent-self-evolution` on GitHub (released April 2026, ~65K stars). The architecture: GEPA reads execution traces, mutates skill prompts and tool descriptions, scores variants against outcomes, keeps winners. No GPU training; everything is API-call-and-text-mutation.
- Underlying technique: GEPA (Genetic-Pareto Prompt Evolution), an ICLR 2026 Oral paper by Agrawal et al., titled *"Reflective Prompt Evolution Can Outperform Reinforcement Learning"*. Open-source at `gepa-ai/gepa`. MIT-licensed.
- Why it fits GhostCheck cleanly: V1's `applications/<date_slug>/` folder already contains exactly the execution-trace shape GEPA expects (`audit.md` plus `verdicts.json` is the trace; `outcome.md` written by the user — `callback`, `screened`, `rejected`, `ghosted` — is the ground truth). No schema rework needed to start the evolutionary loop.
- Integration shape: a sibling repository `ghostcheck-evolution` that points at this project's `.claude/agents/*.md` plus `applications/` folder, mutates agent prompts, scores variants against outcome data, and proposes diffs back as pull requests for review.
- What gets optimised: agent prompts (severity rubrics, "what I look for" heuristics, fix-hint language), per-agent weights in `weights.yml`. What stays frozen: capability declarations in frontmatter (security boundaries), the deterministic aggregator math (reproducibility boundary).

V1.1-A makes GhostCheck a system that gets better the more you use it. Without it, the agents stay at their V1-hand-tuned defaults forever.

### V1.1-B: Batch pattern analysis (the killer feature)

The V1 audit answers "why was I ghosted on this specific application." The V1.1 batch pattern analysis answers a different and more important question: "what is my **personal silence signature** — the pattern across all my ghostings."

A new agent (`pattern-detector`, placeholder file in `.claude/agents/V1_1-pattern-detector.md`) reads the user's full `applications/` folder — at least five application audits with logged outcomes — and identifies cross-application patterns:

- The dimensions that consistently fire HIGH or CRITICAL across multiple ghostings (e.g., it-services-discount on 6 of 8 ghosted applications).
- The dimensions that fire DIFFERENTLY between callback-bound applications and ghosted applications (the discriminating signals).
- The channel pattern across ghostings vs callbacks (e.g., 90% of callbacks come via referral; 80% of ghostings come via Easy Apply).
- The seniority-bucket pattern (e.g., Director-targeted applications ghost; Principal-targeted convert).

The output is a `patterns/<user>/silence-signature.md` artefact that summarises the user's specific failure mode and prescribes the highest-leverage interventions. This is genuinely novel — no existing CV checker offers cross-application analysis because no existing CV checker stores audits.

### V1.1-C: PNG shareable cards via the Anthropic frontend-design skill

Single-application audits and silence signatures both deserve sharable artefacts beyond markdown. V1.1 adds a render skill that produces PNG cards (Twitter / LinkedIn-shareable) from audit data.

- Driven by Anthropic's `frontend-design` skill (referenced in build prompt section 7).
- One PNG per audit (the headline finding plus probability) and one per silence-signature (the cross-application pattern).
- Optional: HTML report view as well, for interactive viewing.

This is launch-friendly — sharable cards drive distribution.

### V1.1-D: Architecture slide deck (deferred from earlier)

The eight-slide architecture overview (proposed during the build but deferred to launch prep) lands in V1.1. Driven by the `frontend-slides` Claude Code skill (zero-dependency animation-rich HTML presentations).

Locked outline (from `SESSION_RESUME` section 6, 2026-04-22 entry):

1. The problem — silence, not rejection.
2. The eleven agents at a glance, by tier.
3. Pattern A vs Pattern B side-by-side; why blind fan-out matters.
4. The harness in five pillars.
5. Audit flow end-to-end.
6. Zero Trust posture — three Microsoft principles, four GhostCheck rules.
7. V1 to V2 roadmap.
8. Fork and run.

---

## V1.2 — Calibration, BYOK, durable knowledge, runtime trust enforcement

V1.2 closes the gap between "V1.1 self-evolution proposes changes" and "the system rigorously knows whether those changes are working."

### V1.2-A: Calibration loop with real outcomes

V1's hyperparameters (`k = 4.0`, `midpoint = 0.4`) and per-agent weights (`weights.yml`) were chosen by intuition. V1.2 fits them empirically.

The mechanism: as users accumulate `applications/*/outcome.md` data, a calibration job reads (audit verdicts, callback outcomes) pairs and tunes:

- Logistic hyperparameters `k` and `midpoint` to maximise predictive accuracy.
- Per-agent weights in `weights.yml` to up-weight the agents whose verdicts are most predictive of outcomes for THIS user (calibration is per-user; the population-level baseline is also computed and surfaced for forkers).
- Severity rubric thresholds inside agent prompts where applicable.

Aspect to preserve: the formula stays the same. Calibration tunes hyperparameters, not the math.

### V1.2-B: BYOK (bring-your-own-LLM)

V1 runs on Claude Code with the Claude API. V1.2 adds a provider abstraction so the same agent files run on Gemini, Ollama (local models), or any future LLM provider.

- Configuration goes in `config/runtime.yml` — choose the provider, model name, API key path.
- Agent files do not change. They are runtime-agnostic markdown.
- The router resolves the provider at audit time and routes API calls accordingly.

V1.2-B operationalises the harness-engineering thesis: *"The runtime is an implementation detail. The contract is the product."*

### V1.2-C: Karpathy LLM Wiki pattern (compounding knowledge)

Andrej Karpathy's "LLM Wiki" pattern — durable knowledge accumulating across sessions — applied to GhostCheck. As users accumulate applications and the system learns its silence signature, Wiki entries form on:

- The user's personal silence signature (the "always check this dimension first" knowledge).
- Cross-user generic patterns (anonymised, opt-in).
- Industry-specific or domain-specific heuristics learned from real audits.

Implementation: `.claude/skills/wiki-compiler/wiki-compile.md` (placeholder in V1) plus a `wiki/` folder. Each wiki entry is a markdown file with metadata; the compiler maintains an index and surfaces relevant entries during future audits.

### V1.2-D: Runtime Zero Trust policy enforcement

V1's Zero Trust posture is documented and enforced by convention plus schema validation. V1.2 adds runtime enforcement: an agent that declares `capabilities: []` cannot call any tool at runtime, period — enforced by the Claude Code runtime, not by our convention-following.

This brings GhostCheck from "Zero Trust at design time" to "Zero Trust at runtime," matching the emerging industry expectation.

---

## V2 — Hosted multi-user runtime, observability, ecosystem

V2 is GhostCheck as a service rather than a fork-and-go skill pack. It does not replace V1; it extends.

### V2-A: Python / FastAPI runtime

A backend service that drives the same eleven agents over the Claude API (or Gemini / Ollama via the V1.2 provider abstraction). The skill pack stays the source of truth for agent definitions; the Python service reads it.

For users: they upload CV and JD, get an audit. No Claude Code installation required.

For coaches and recruiters: they manage multiple candidates' audits with shared infrastructure. Genuinely useful for hiring-coach businesses.

### V2-B: Web frontend

React / TypeScript application that lets users:

- Upload CV and JD via drag-and-drop.
- View the audit interactively (severity bars, evidence cards, fix-hint timelines).
- See their silence signature visualised across multiple applications.
- Export to PNG / PDF / share to LinkedIn.

### V2-C: Microsoft AI Observability framework alignment

Microsoft's *AI Steering Committee's 2026 Checklist: Observability* (April 2026) defines four foundational governance questions for production AI agents: Inventory, Identity, Access, Outcomes. V2 implements them:

- **Inventory** — registry of all agent versions deployed across the platform.
- **Identity** — per-user attribution for every audit run.
- **Access** — capability declarations enforced at the API gateway, not just within Claude Code.
- **Outcomes** — telemetry of audit verdicts vs real callback outcomes, fed back into V1.2 calibration.

This makes GhostCheck deployable into enterprise contexts (career coaches at large firms, internal HR teams at companies running it on candidates with permission, etc.).

### V2-D: Ecosystem and forks

By V2, the harness pattern is well-documented enough that vertical forks become trivial:

- `ghostcheck-electrical-eng` — engineering-services discount, IEEE / ResearchGate surface, hardware-specific keyword extractor.
- `ghostcheck-finance` — finance-firm discount, journal-publication surface, FCA / SEC compliance signals.
- `ghostcheck-medicine` — medical-residency tracks, peer-review surface, board-cert keyword extractor.

Each fork is a few config files plus minor agent prompt edits. The harness stays untouched.

---

## Calibration and accuracy disclaimers

All V1 numbers (severity thresholds, weights, hyperparameters) are intuition-fit defaults. Until V1.2 calibration accumulates enough outcome data to tune empirically, audit probabilities are directional indicators, not statistically precise predictions.

Specifically:

- The 0.14 weight on `bucket-classifier` and 0.06 weight on `hm-deep-read` reflect the build prompt's framing ("bucket is the dominant senior-silence driver"), not regression-fitted importance.
- The four-tier severity-to-score mapping (`CRITICAL=1.0, HIGH=0.75, MEDIUM=0.5, LOW=0.25`) is a uniform-step assumption. Real-world severity-to-impact relationships are likely non-linear.
- The logistic `k = 4.0, midpoint = 0.4` produce sensible-feeling probabilities at the four corners but are not validated against outcome data.

Audits are most useful as **diagnostic hypotheses prioritised by severity**, not as numeric predictions. The cited evidence is more durable than the probability number.

---

## Contributing to the roadmap

V1 ships as an MIT-licensed fork-and-go skill pack. Contributions are welcome via pull requests; the harness pattern (schema contracts, fail-closed semantics, deterministic aggregator) is the load-bearing constraint that all contributions must respect.

Specifically:

- New agents are accepted if they conform to the `AgentVerdict` schema, declare their inputs and capabilities in frontmatter, run blind to other agents, and ship with at least one severity rubric and three reasonable fix-hint examples.
- Schema changes (`SCHEMAS.md`) require explanation of why existing agents would tolerate the change without breaking. Breaking changes need a `schema_version` bump in agent frontmatter.
- Calibration data (anonymised audit outcomes contributed to a shared corpus) accelerates V1.2 — opt-in via `config/share_outcomes.yml` (V1.2 placeholder; not yet implemented).
- Cross-domain forks (electrical engineering, finance, medicine) are encouraged; they validate the harness-portability thesis.

The build prompt at the project root (`GhostCheck_Build_Prompt.md`) plus the session journal (`SESSION_RESUME.md`) carry the architectural decision trail and explain why the system is shaped the way it is. Read both before contributing.
