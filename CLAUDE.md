# CLAUDE.md — Project Instructions for Claude Code

This file is read automatically by any Claude Code session opened in this repo. It is the AI's operating manual. It is **not** a user-facing document — humans read `README.md`. If `CLAUDE.md` and `README.md` ever disagree, update both in the same change; never let them drift.

---

## 1. What this repo is

GhostCheck is a Claude Code skill pack that audits why senior job applications get silenced. Eleven narrow agents each inspect one dimension of silence risk (bucket mismatch, Google footprint, IT-services discount, stale postings, and so on), each returns a verdict in the fixed `AgentVerdict` shape, and a deterministic aggregator combines those verdicts into a callback probability with ranked silence drivers and evidence-cited fix suggestions.

The main user command is:

```
/ghostcheck audit --cv profile/cv.md --jd jobs/<slug>.md
```

Output is written to `applications/YYYY-MM-DD_<slug>/audit.md`.

---

## 2. Binding architectural constraints (non-negotiable)

1. **No Python in V1.** No FastAPI, LangGraph, SQLite, Playwright, or any backend service. Every agent is a `.md` file under `.claude/skills/agents/`.
2. **Every agent skill has YAML frontmatter** declaring at minimum: `agent_id`, `tier`, `inputs`, `outputs`, `capabilities`, `severity_model`, `temperature`, `max_tokens`. A future Python runtime must be able to parse these without a rewrite.
3. **The `AgentVerdict` schema in `docs/SCHEMAS.md` is the only permitted output shape.** No agent may invent fields, rename fields, or omit required fields.
4. **Fail-closed.** If an agent cannot produce a schema-valid verdict with cited evidence, it returns `UNKNOWN` with a reason. It does not guess.
5. **Agents cannot call each other.** They run blind. The only permitted cross-agent step is the deterministic aggregator under `.claude/skills/aggregator/`.
6. **Agents cannot write to user files.** Only the aggregator / router decides what gets saved to `applications/`.
7. **Every claim must cite its source** — a line from the CV, a line from the JD, or a field of the `ExternalContext` object. No opinion without citation.
8. **The aggregator is deterministic math, not another LLM call.** Do not turn it into an LLM.

---

## 3. Model configuration defaults

- **Temperature:** `0.1` for all agents, except `bucket-classifier` and `hm-deep-read` which use `0.3` (more nuanced reasoning).
- **Max tokens per agent:** `800`.

These values live in each agent's frontmatter. If a user or contributor wants to change them for a specific agent, update the frontmatter — do not override at call time.

---

## 4. Repo layout (where things belong)

```
.claude/skills/                    — Skills (run in the main conversation context)
  ghostcheck/SKILL.md              — the /ghostcheck slash command router
  aggregator/aggregator.md         — deterministic verdict combination
  enrichment/                      — helper skills (web search, company classification, JD age)
  parser/markitdown-parse.md       — PDF / DOCX / HTML parsing
.claude/agents/                    — Subagents (each runs in its own isolated context window)
  bucket-classifier.md             — A3 (Tier A: invisible-failure detection)
  google-test.md                   — A1 (Tier A)
  posting-decoder.md               — A2 (Tier A)
  it-services-discount.md          — A4 (Tier A)
  headline-filter.md               — A5 (Tier A)
  funnel-math.md                   — B1 (Tier B: channel and funnel math)
  channel-mix.md                   — B2 (Tier B)
  stale-detector.md                — B3 (Tier B)
  ats-simulator.md                 — C1 (Tier C: standard CV quality)
  recruiter-30sec.md               — C2 (Tier C)
  hm-deep-read.md                  — C3 (Tier C)
docs/
  SCHEMAS.md                  — the AgentVerdict contract (Pydantic-ready)
  HARNESS_ENGINEERING.md      — why the harness is the moat
  ZERO_TRUST.md               — security principles
  ROADMAP.md                  — V1 to V1.1 to V1.2 to V2
profile/                      — user CV, style, and examples
config/                       — profile.yml, weights.yml
applications/                 — one folder per audit (generated)
patterns/                     — V1.1 silence signatures (placeholder in V1)
```

If you are about to put a file somewhere not in this tree, stop and ask — the layout is deliberate.

---

## 5. How Claude must behave during an audit run

When a user invokes `/ghostcheck audit --cv <file> --jd <file>`:

1. Load the CV and JD via `parser/markitdown-parse.md`. If parsing fails, surface the error — do not guess the content.
2. Build the `ExternalContext` (Google-test lookup, company classification, JD age detection) by invoking only the enrichment skills declared in each agent's `capabilities` field.
3. Run the 11 agents in the order declared by the router. Each agent reads only the inputs declared in its frontmatter — nothing else.
4. Each agent returns an `AgentVerdict`. Validate against the schema. On failure, coerce to `UNKNOWN` with a reason — never pass a malformed verdict to the aggregator.
5. The aggregator combines verdicts deterministically (weighted logistic, weights from `config/weights.yml`) to produce a callback probability and a ranked list of silence drivers.
6. Write the result to `applications/YYYY-MM-DD_<slug>/audit.md`. Do not write anywhere else.

Do not improvise new agents, new fields, or new output paths during an audit run. Follow the contract.

---

## 6. How Claude must behave during development

When a user asks for changes to the project — adding an agent, tuning weights, updating a prompt, modifying the aggregator:

1. **Read `SESSION_RESUME.md` first** — it carries build context and pending decisions across sessions.
2. **Flag any request that conflicts with section 2 of this file** before acting. If a user asks to add Python, bypass fail-closed, or let one agent call another, explain the tradeoff and confirm before implementing.
3. **Explain-before-write** for any file that touches the `AgentVerdict` contract, the aggregator math, or the capability declarations in frontmatter. These are load-bearing.
4. **Never silently improve.** If you see a better way than the spec, say so before changing course. Prefer a short chat exchange over a surprise.
5. **Keep the README contract honest.** If a change invalidates a claim in `README.md`, update `README.md` in the same change.

---

## 7. Never do

- Never invent evidence. If the CV does not say it, do not cite it.
- Never have one agent call another, even indirectly.
- Never write to the user's `profile/cv.md`, `profile/style.md`, any file under `config/`, or the JD file. Those belong to the user.
- Never produce an `AgentVerdict` with a `severity` outside the declared enum.
- Never skip an agent silently. If an agent cannot run, the aggregator receives `UNKNOWN` with a reason.
- Never hit an external service from an agent whose frontmatter `capabilities` field does not declare that permission. `google-test` may call web search. `bucket-classifier` may not.
- Never bypass the aggregator. The final `audit.md` is written by the aggregator / router only.

---

## 8. Where to read more

- **`docs/SCHEMAS.md`** — the `AgentVerdict` contract, Pydantic-ready. Read this before modifying any agent.
- **`docs/HARNESS_ENGINEERING.md`** — why the harness (contracts, fail-closed, blind agents, deterministic aggregator) is the actual moat.
- **`docs/ZERO_TRUST.md`** — security and trust posture. Capability declarations, data boundaries, fail-closed enforcement.
- **`docs/ROADMAP.md`** — what's in V1, what moves to V1.1 (batch pattern analysis), V1.2 (calibration, BYOK), V2 (Python service).
- **`SESSION_RESUME.md`** — build journey and session handoff notes. Read during development; ignore during audit runs.