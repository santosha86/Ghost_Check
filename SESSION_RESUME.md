# GhostCheck — Session Resume

**Last updated:** 2026-04-21
**Next Claude:** read this file FIRST, then `GhostCheck_Build_Prompt.md`, then greet the user.

---

## 0. How to use this file (instructions for the next Claude)

Santosh is continuing the GhostCheck build from a different machine. Your local memory files from the previous session do not travel across machines — **this file IS the memory**. Do not skip any section.

**Before writing any code or explanation:**
1. Read this whole file.
2. Read `GhostCheck_Build_Prompt.md` (the full build spec — binding).
3. Briefly greet Santosh: confirm you've read both, state where we are in the plan, and ask if he wants to adjust anything before the next file.
4. **Do not re-run the full Multi-Agent Solutions 101 preamble** unless he asks — he already went through it on 2026-04-21 and the vocabulary list in section 5 below shows what he knows.
5. Follow the Teach Mode rules in section 2 strictly. They are binding and override default terseness.

---

## 1. Who Santosh is

- 16 years in AI / enterprise architecture (SABIC, Aramco, STC, SEC).
- Currently applying to senior AI roles and experiencing "silent ghosting" — this is the pain driving GhostCheck.
- **First multi-agent project** of his own. Technically strong, but new to multi-agent orchestration specifically.
- Builds in public: ship GhostCheck, post results, attract inbound opportunities.
- Wants to **learn deeply** — he must understand every file before it is written.

**Framing guidance:**
- Safe to use terms like "schema contract", "fail-closed", "orchestration" after a one-line definition.
- He is NOT new to YAML, markdown, JSON — don't over-explain those.
- He IS new to: Claude Code skill packs specifically, AgentVerdict-style contracts, the framework-vs-skill-pack distinction, logistic aggregation math.

---

## 2. Teach Mode rules (BINDING — override default terseness)

From `GhostCheck_Build_Prompt.md` section 9. These apply to every file, every explanation, for the whole build:

1. **Explain-before-write.** Before creating ANY file, write a short explanation in chat (what / why / how it fits). Wait for "continue" or follow-up questions before writing.
2. **Define vocabulary first.** Before any tricky section (frontmatter, prompt template, aggregator math), define the term in plain English. Never assume he knows a term — saying it back is fine reinforcement.
3. **TEACH_NOTES after each file.** After creating each file, produce a small `docs/TEACH_NOTES/XX-<topic>.md` capturing the key learning. These accumulate into a re-readable course.
4. **Checkpoint every 2–3 files.** Stop, summarize what was built, ask if anything is unclear BEFORE proceeding.
5. **Show alternatives honestly.** When he asks "why this way vs another", show the alternative and the real tradeoff. Don't just defend the current choice.
6. **No jargon without teaching.** Never say "parameterize via Pydantic discriminated unions" without first saying "severity will be one of four fixed words — we call that a discriminated union."
7. **Pace:** 5–8 files per day across 5 days. Do not rush.
8. **Never silently improve.** If there's a better way than the build prompt specifies, TELL HIM before changing course. Never silently deviate.
9. **Ask when in doubt.** He'd rather answer 10 questions than unwind a wrong decision.

---

## 3. What GhostCheck is

**One-paragraph summary:** A Claude Code skill pack that audits why senior job seekers get silently ghosted. 11 specialized agents each judge one dimension of silence risk (seniority mismatch, IT-services discount, stale JD posting, missing Google footprint, etc.), each produces an `AgentVerdict`, and a deterministic aggregator combines them into a callback probability with ranked silence drivers and fix hints. Users fork the repo, open in Claude Code, run `/ghostcheck audit --cv <file> --jd <file>`, and get `applications/<date_slug>/audit.md`.

**What makes it different from a CV checker:** Traditional CV checkers score the artifact (keywords, ATS). GhostCheck simulates the whole funnel — including invisible failure modes no existing tool catches: the Google test (what recruiters see before your CV), bucket mismatch, IT-services discount, theater postings, channel mismatch.

**Why a skill pack, not a Python app:** Fork-and-go beats pip-install for distribution; markdown-first is portable; V1 ships in 5 days with no server; reference product `santifer/career-ops` hit 37K+ stars on the same shape. Stays Pydantic-ready so a FastAPI wrap in V2 is additive, not a rewrite.

**The moat is the harness** — not any single agent. Shared `AgentVerdict` contract across all 11 agents, agents blind to each other (no groupthink), deterministic aggregator (math, not another LLM), fail-closed semantics, markdown-first portability. This goes in `docs/HARNESS_ENGINEERING.md`.

---

## 4. Hard architectural constraints (do not negotiate these silently)

1. **V1 has no Python.** No FastAPI, LangGraph, SQLite, or Playwright. Every agent is a `.md` file under `.claude/skills/agents/`.
2. **Every agent skill has YAML frontmatter** declaring inputs, outputs, capabilities, severity model — a future Python runtime must parse these without a rewrite.
3. **Data schemas live in `docs/SCHEMAS.md`** as Pydantic-ready specs.
4. **Fail-closed:** if an agent's output fails schema validation, it returns `UNKNOWN`, not a hallucinated verdict.
5. **Agents cannot call each other.** They only produce verdicts. Agents cannot write to user files.
6. **Temperature 0.1 for all agents** except `bucket-classifier` and `hm-deep-read` (0.3 — more nuanced).
7. **Max tokens per agent: 800.**
8. **No opinion without evidence** citation from CV, JD, or ExternalContext.

---

## 5. Vocabulary Santosh has already been taught (do not re-teach unless he asks)

From the Multi-Agent Solutions 101 preamble on 2026-04-21:

- **Agent** — LLM with a narrow job and a fixed I/O shape.
- **Skill** — markdown file that defines an agent, with YAML frontmatter + prompt body.
- **Skill pack** — folder of skill files; the project IS the pack.
- **Orchestration** — deciding which agent runs when (Claude Code does this for us).
- **Fan-out** — one input → many parallel agents → one aggregator; keeps agents blind to each other.
- **AgentVerdict** — the fixed-shape output every agent must return (`agent_id`, `severity`, `score`, `verdict`, `evidence`, `fix_hints`).
- **Aggregator** — deterministic math (not an LLM) combining verdicts into a final score.
- **Fail-closed** — on any failure, return `UNKNOWN`; never hallucinate.
- **Harness** — the whole system of contracts, orchestration, and schemas. The harness is the moat.

Terms he may still want deeper treatment on (he hasn't asked yet, but watch for it): YAML frontmatter internals, Pydantic discriminated unions, logistic regression / weighted logistic aggregator math, temperature, MarkItDown.

---

## 6. What was done on 2026-04-21 (previous session)

- Read the build prompt end-to-end.
- Delivered the Multi-Agent Solutions 101 preamble (section 9.d of the build prompt) — Santosh has not yet said "continue" past this point.
- Confirmed understanding of project scope (3-sentence summary, differentiation, skill-pack rationale, harness moat framing).
- Initialized the git repo, wrote `.gitignore` (excludes `old_req/`, `.DS_Store`, `.claude/settings.local.json`), committed, and pushed to `https://github.com/santosha86/Ghost_Check.git` on the `main` branch.
- **No project files built yet.** Day 1 foundation work has not started.

Files currently in the repo:
- `GhostCheck_Build_Prompt.md` (+ .pdf) — the binding spec
- `.gitignore`
- `SESSION_RESUME.md` (this file)

Files kept local, not in git (by user decision):
- `old_req/` — contains internal PRD and Publishing Playbook

---

## 7. Next step when session resumes

We are at **Day 1, file 1 of ~8**. Day 1 file order (from build prompt section 11):

1. `README.md` — fork-and-go instructions **← START HERE**
2. `LICENSE` — MIT
3. `CLAUDE.md` — project-wide instructions for Claude Code
4. `docs/SCHEMAS.md` — the AgentVerdict contract (most important file of the week)
5. `docs/HARNESS_ENGINEERING.md` — the moat doc
6. `docs/ZERO_TRUST.md` — security principles
7. `profile/cv.example.md`, `profile/style.example.md`
8. `config/profile.example.yml`, `config/weights.yml`

**Per Teach Mode, the flow for each file is:**
1. Explain in chat: what it is, why it exists, how it fits.
2. Wait for Santosh to say "continue" or ask follow-ups.
3. Write the file.
4. Write a short `docs/TEACH_NOTES/XX-<topic>.md` capturing the key learning.
5. After every 2–3 files, checkpoint: summarize + ask if anything is unclear.

---

## 8. When Santosh resumes, your first message should be roughly this shape

> Hi Santosh — I've read `SESSION_RESUME.md` and `GhostCheck_Build_Prompt.md`. Quick recap of where we are:
>
> - Preamble complete (you've covered agent / skill / orchestration / fan-out / framework-vs-skill-pack / AgentVerdict / fail-closed / harness).
> - Repo pushed to GitHub on `main`, no project files yet.
> - Next up: **Day 1, file 1 — `README.md`** (fork-and-go instructions).
>
> Teach Mode is on. Before I write README.md, here's what it is and why it exists first... [proceed with file-1 explanation]
>
> Want me to start with that explanation now, or is there anything from yesterday you want to revisit first?

Do NOT start writing files until he says to proceed.

---

## 9. How to update this file each session (pattern to preserve)

At the end of every meaningful work session:
1. Update the "Last updated" date at top.
2. Update section 6 ("What was done") with what got built.
3. Update section 7 ("Next step") to reflect the next file/task.
4. Update section 5 ("Vocabulary taught") if new terms were introduced.
5. Commit + push with a message like `Update SESSION_RESUME for <date>`.

This file is the persistent brain of the project across machines. Keep it honest and current.
