# GhostCheck — Session Resume

**Last updated:** 2026-04-22
**Next Claude:** read this file FIRST, then `GhostCheck_Build_Prompt.md`, then greet the user.

**Workflow constraint (important):** Santosh works in Claude Code on the web from his company laptop. All `git push` operations happen **from his personal laptop** against GitHub, not from the sandbox. Do not attempt `git push` from the sandbox — it will return 403. Write files in the sandbox, give Santosh the content as copyable blocks, he mirrors them into his Claude Project and eventually pushes to GitHub from his personal laptop.

---

## Quick start — Santosh, copy-paste this into your next Claude Code session

When you open Claude Code (web or desktop) on a new machine and connect it to this repo, paste the block below as your very first message. Nothing else is needed.

```
I'm continuing a project called GhostCheck. This repo is connected.

Please:
1. Read SESSION_RESUME.md first — it has the full hand-off from the previous
   session (my profile, the binding Teach Mode rules, what's been done so
   far, and the exact next step).
2. Then read GhostCheck_Build_Prompt.md — the full build spec.
3. Greet me as instructed in section 8 of SESSION_RESUME.md, then wait for
   me to say "continue" before writing any files.

Do not start writing code yet.
```

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
3. **Learning notes stay personal, not in the repo.** There is no `docs/TEACH_NOTES/` folder. When a file's design reasoning is worth capturing for Santosh's learning, summarise the key lesson in chat so he can save it to his personal Claude Project — the public repo stays product-focused. (Decision taken 2026-04-22, replacing the original TEACH_NOTES-in-git plan.)
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

**Why a skill pack, not a Python app:** Fork-and-go beats pip-install for distribution; markdown-first is portable; V1 ships in 5 days with no server. Stays Pydantic-ready so a FastAPI wrap in V2 is additive, not a rewrite.

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
- **Fan-out** — one input to many parallel agents to one aggregator; keeps agents blind to each other.
- **AgentVerdict** — the fixed-shape output every agent must return (`agent_id`, `severity`, `score`, `verdict`, `evidence`, `fix_hints`).
- **Aggregator** — deterministic math (not an LLM) combining verdicts into a final score.
- **Fail-closed** — on any failure, return `UNKNOWN`; never hallucinate.
- **Harness** — the whole system of contracts, orchestration, and schemas. The harness is the moat.

**Added 2026-04-22 (later session — personal laptop):**

- **Zero Trust for AI** — Microsoft's March 2026 adaptation of classical Zero Trust to multi-agent systems. Three principles: *Verify explicitly*, *Apply least privilege*, *Assume breach*.
- **Double agents** (Microsoft term) — overprivileged or manipulated agents that look like yours but act on someone else's inputs. The threat pattern that motivates capability declarations.
- **Lateral movement (in multi-agent context)** — one agent's opinion leaking into another's reasoning. Prevented by fan-out (agents never see each other's outputs).
- **XPIA / indirect prompt injection** — attacker-controlled text in inputs (e.g. a crafted JD) that manipulates the LLM. Partial V1 defences via schema validation + capability declarations; full input sanitisation is V1.2+.
- **Provenance** — chain of custody from input to output; per-field source tagging (e.g. `Company.sources` in `SCHEMAS.md section 6`) is the concrete expression.
- **DSPy + GEPA** — prompt-evolution stack. DSPy compiles prompt programs; GEPA (Genetic-Pareto Prompt Evolution, ICLR 2026 Oral) mutates prompt text against execution traces, reads *why* failures happened, keeps winners. The technique Hermes Agent uses for self-evolution; target for GhostCheck V1.1.
- **Sources of Truth table** — doc/code pattern: enumerate every input file with `{File | Path | When | Precedence}`. Santifer's `modes/_shared.md` is the canonical example. Useful as a general doc convention.
- **Bundle-and-dispatch vs two-hop loading** (Claude Code skill idioms) — router either (a) reads all user-layer files once and passes declared inputs to each agent, or (b) references a shared context file that in turn references user-layer files. GhostCheck uses (a); santifer uses (b). Both are valid.

Terms he may still want deeper treatment on (he hasn't asked yet, but watch for it): YAML frontmatter internals, Pydantic discriminated unions, logistic regression / weighted logistic aggregator math, temperature, MarkItDown.

---

## 6. Session log (what was done, newest first)

### 2026-04-25 (third session — personal laptop, Day 2 prep)

- Reviewed all Day 1 work in detail. Quality: high. No content rework needed.
- **Symbol sweep across all my files** — replaced section signs, arrows, approximation symbols, checkmarks, box-drawing characters, and the one stray emoji with plain English. Build prompt itself untouched (Santosh's spec). 40 replacements across 8 files. Driven by Santosh's feedback that AI-styling tics add reader friction in a plain-text project. Saved as a memory rule for future work.
- **Architectural decision locked: Pattern B (subagent isolation per agent)** for the router-to-agent dispatch. Each of the 11 agents runs in its own isolated context window. The router builds the per-agent input bundle once, then invokes each agent via Claude Code's Agent tool (V1) or via separate API calls in any future runtime (Python service, Gemini CLI, Ollama). Pattern A (sequential reads in a shared context window) was rejected because it semantically breaks blind fan-out: agent N+1 would see prior agents' verdicts in its context, producing dependent verdicts that look independent — the worst possible outcome.
- **Architectural decision locked: Input validation Levels 1 plus 3.** Level 1 = file-extension check at the start of an audit, rejecting unsupported types before parsing. Level 3 = LLM classification call after MarkItDown parsing, asking "is this a CV / is this a JD" with a confidence score, fail-closed if confidence is low. Level 2 (content-keyword heuristics) was rejected because Santosh's real JD is a PowerPoint deck, on which heuristic header-matching breaks. The LLM classifier handles decks and PDFs identically.
- **Real CV and JD organised into the audit-ready paths.** `CV_StrategyAnd_AI_Lead.pdf` moved to `profile/cv.pdf` (gitignored). `Strategy&_AI Solution Architect - JD - Chief.pptx` moved to `jobs/strategy-ai-architect-chief.pptx` (jobs directory gitignored). The PowerPoint format is real-world signal that the parser must handle .pptx via MarkItDown — confirmed MarkItDown supports it.
- **Slug naming convention adopted for filenames in `jobs/` and `applications/`** — lowercase, hyphens for word boundaries, no special characters. Avoids shell-quoting hassles, cross-platform issues, and downstream tool breakage.
- New feedback memory: explain alternatives deeply BEFORE asking Santosh to choose, not after. Each option presented as a full paragraph showing what it does mechanically and what its consequences are in practice.

**Files authored or updated this session (push-able from this machine):**

- `CLAUDE.md` — symbol sweep
- `docs/SCHEMAS.md` — symbol sweep
- `docs/HARNESS_ENGINEERING.md` — symbol sweep
- `docs/ZERO_TRUST.md` — symbol sweep AND XPIA "does NOT promise" paragraph expanded with five layered defences plus V1.1 evidence-verification roadmap mention
- `profile/style.example.md` — symbol sweep
- `config/profile.example.yml` — symbol sweep
- `config/weights.yml` — symbol sweep
- `SESSION_RESUME.md` — symbol sweep, this 2026-04-25 entry, and Pattern B / Input Validation locked-conventions added to section 7

**Files moved (gitignore-protected, not visible in git status):**

- `profile/cv.pdf` (Santosh's real CV — was at project root)
- `jobs/strategy-ai-architect-chief.pptx` (Santosh's real JD — was at project root)

### 2026-04-22 (later session — personal laptop)

- Resumed on personal laptop after the web-Claude session (same calendar date, different machine). Audited the 4 files authored in web Claude (`README.md`, `LICENSE`, `CLAUDE.md`, `docs/SCHEMAS.md`). Quality: high. Minor polish fixes applied: (a) `LICENSE` copyright to `Achanta Santosh Kumar` (full legal name); (b) `README.md` footnote clarifying that `A`/`B`/`C` are tier prefixes / filename prefixes, not the `agent_id` value stored in `AgentVerdict`.
- **Wrote `docs/HARNESS_ENGINEERING.md`** — 5-pillar structure (one schema / blind agents / deterministic aggregator / markdown-first / skill-pack distribution) + "What the harness is NOT" section + closing. Voice-forward (Santosh approved voice over dry). ~210 lines. Later extended Pillar 4 with a DSPy+GEPA paragraph making the "markdown-first scales to any runtime" claim concrete.
- **Wrote `docs/ZERO_TRUST.md`** — grounded in Microsoft's *Zero Trust for AI* framework (19 March 2026 — Santosh provided URL). Three Microsoft principles mapped to GhostCheck's four rules: *Verify explicitly* ↔ schema validation + fail-closed; *Apply least privilege* ↔ capability declarations; *Assume breach* ↔ no-agent-to-agent + no-user-file-writes. Standards chain cited (NIST + CISA + CIS + Microsoft SFI). Honest V1 vs V1.2+ split: design-time enforcement today, runtime policy engine later. Microsoft's *Zero Trust Assessment for AI* pillar is in development (summer 2026) — so we align with principles, not with yet-to-exist control IDs. ~175 lines.
- **Research: Nous Research *Hermes Agent Self-Evolution* (April 2026).** Real project, MIT-licensed, ~65K stars, ICLR 2026 Oral paper on GEPA. Architecture: DSPy + GEPA mutates skill prompts / tool descriptions / configs against execution traces; outperforms GRPO by 6-20% with ~35× fewer rollouts. **Key fit:** GhostCheck's `applications/<date_slug>/outcome.md` + `verdicts.json` are exactly the execution-trace shape GEPA expects — zero schema rework needed to integrate on V1.1.
- **Research: how santifer/career-ops actually loads user-layer files.** First round was inferential; Santosh pushed back; second round verified against source. Actual pattern is **two-hop**: `SKILL.md` to `modes/_shared.md` (a "Sources of Truth" table) to user-layer files (`cv.md`, `config/profile.yml`, `modes/_profile.md`, `article-digest.md`). Uses explicit imperative "Read X at evaluation time" instructions.
- **Decision: for GhostCheck, use Path 1 (Bundle-and-dispatch).** Router (`SKILL.md`) reads all user-layer files once and dispatches to each agent with exactly the inputs its frontmatter declares (`cv_text`, `jd_text`, `user_profile`, `user_style`, `external_context`). Cleaner for our "agents declare inputs in frontmatter" principle than santifer's two-hop. No change to `SCHEMAS.md` — `user_style` is opaque markdown string at agent-input level.
- **Updated `.gitignore`** — pre-emptively protects `profile/cv.md`, `profile/style.md`, `profile/cv.pdf`, `profile/cv.docx`, `applications/*/`, `jobs/` from accidental commits. `applications/README.md` + `.gitkeep` still allowed.
- **Wrote `profile/cv.example.md`** — fictional "Rohan Mehta" persona scrubbed from Santosh's real CV. Client names genericized (`SABIC to a Gulf petrochemical major`; `Aramco to a national oil & gas operator`; `SEC to a national utilities operator`; `STC to a national telecom operator`; `BSNL to an Indian telecom operator`). Employer names invented (`Gulf AI Advisory`, `Mideast Tech Partners`, `International Tech Services`). Dates shifted back 1 year for identifiability hedge. All bullets, technical vocabulary, and metrics preserved. HTML-comment preamble marks it as fictional.

**Design decisions locked this session (affect Day-2 work):**

- **Day-2 `SKILL.md` will have a "Files this audit loads" section** — listing CV, JD, `config/profile.yml`, `profile/style.md` (fall back to `.example.md` if user's real file missing) with load order and precedence. Inspired by santifer's `_shared.md` Sources of Truth table but inlined (GhostCheck has one audit flow, not 14 modes).
- **Agent frontmatter convention:** `inputs: [cv_text, jd_text, user_profile, user_style, external_context]`. Agents omit inputs they don't need (e.g. `ats-simulator` likely omits `user_style`).
- **HARNESS_ENGINEERING.md now references DSPy+GEPA** in Pillar 4 as concrete example of a second runtime reading the same markdown — makes the "markdown-first scales" claim verifiable rather than hand-wavy.

**For Day 5 (`ROADMAP.md` content — capture now so it's not forgotten):**

- **V1.1 self-evolution section:** reference `NousResearch/hermes-agent-self-evolution` and `gepa-ai/gepa` by name. Explain integration path: point GEPA at `.claude/skills/agents/*.md` + `applications/` folder (which already contains execution traces in the shape GEPA expects). Weights, prompts, and severity thresholds are the optimization targets; capability declarations and the aggregator math stay frozen (security / determinism boundaries).
- **V1.2 / V2 observability section:** reference Microsoft's *AI Steering Committee's 2026 Checklist: Observability* (16 April 2026). Four foundational questions — Inventory, Identity, Access, Outcomes — map to GhostCheck's registry of `.claude/skills/agents/*`, per-audit identity (just Santosh in V1), capability declarations in frontmatter, and `audit.md` outcomes. Enterprise dashboards are out of V1 scope but on the roadmap for V2 if GhostCheck moves into team/coach usage.

**Files authored or updated on personal laptop this session (push-able from this machine):**

- `docs/HARNESS_ENGINEERING.md` — NEW (+ GEPA paragraph added later)
- `docs/ZERO_TRUST.md` — NEW
- `profile/cv.example.md` — NEW
- `profile/style.example.md` — NEW
- `config/profile.example.yml` — NEW
- `config/weights.yml` — NEW
- `.gitignore` — UPDATED (profile/ + applications/ + jobs/ protection)
- `LICENSE` — UPDATED (copyright to full name)
- `README.md` — UPDATED (tier-label footnote)
- `SESSION_RESUME.md` — UPDATED (this file)

**Day 1 is COMPLETE (10 of 10 files done).** All eight file blocks from build prompt section 11 are in the repo.

### 2026-04-22 (early session — web sandbox, company laptop)

- Resumed the build on Claude Code web (new machine — company laptop). Santosh used the quick-start paste block from section 0 to hand off context.
- Decided **Option A — commit to `main`** (single canonical branch). No per-day feature branches.
- Discovered the sandbox's local git remote blocks `git push` with 403. Resolution: Santosh pushes from his personal laptop only; sandbox is for authoring. Captured this as a workflow constraint in section 0.
- Dropped `docs/TEACH_NOTES/` entirely. Learning notes live in Santosh's personal Claude Project, not the public repo. Section 2 rule 3 rewritten to match.

**Design decisions locked in this session (reflected in `docs/SCHEMAS.md`):**

- **Company enrichment: Option C (hybrid).** User-provided JD YAML header fills the `Company` object first; web search fills the gaps. LinkedIn direct scraping is not used (gated behind login). Per-field provenance tracked in `Company.sources` so the audit can show which fields came from the user vs web vs default.
- **Weight renormalization on UNKNOWN.** When one or more agents emit UNKNOWN, their weights are redistributed proportionally across the remaining agents so total stays 1.0. The audit surfaces unassessed dimensions in a dedicated "Not assessed" section.
- **`applications/<date_slug>/` folder contract.** Required: `audit.md`, `verdicts.json`, `context.json`. Optional user-written: `outcome.md`, `channel.md` (feed future B-tier agents — funnel-math, channel-mix — and enable V1.1 batch pattern analysis without a database).
- **CV parsing is stateless** — re-parse every audit run in V1. Parser cache is a V1.1 optimization (noted for `ROADMAP.md` when Day 5 arrives).
- **Severity enum: 5 values** — CRITICAL, HIGH, MEDIUM, LOW, UNKNOWN.
- **`HARNESS_ENGINEERING.md` framing**: engineering reasoning (alternatives + why rejected), not "why we're the moat." File not yet written — explain-before-write was delivered but Santosh wanted a plainer, example-driven version. Pick up next session.

**Files built this session (authored in sandbox, awaiting push from Santosh's personal laptop):**
  - `README.md` — fork-and-go front door. Eight-step Quick Start (now including the optional JD YAML header example in step 6), "What you get back" framed as evidence-cited hypotheses (not guaranteed reasons), a `Runtime` section stating Claude Code today / portable by design, 11-agent overview table.
  - `LICENSE` — MIT. Copyright line: `Copyright (c) 2026 Santosh` (Santosh can edit to full name anytime).
  - `CLAUDE.md` — the AI's operating manual for the project. Auto-read by Claude Code on repo open. Contains binding architectural constraints, model config defaults, repo layout, audit-run behaviour, development behaviour, "never do" stop-list, cross-references.
  - `docs/SCHEMAS.md` — full data contracts. 14 numbered sections: AgentVerdict, AgentId enum, Severity enum, Evidence, ExternalContext, Company (hybrid + provenance), GoogleResult, UserProfile, JD frontmatter, applications folder layout, audit.md report contract, validation & fail-closed coercion, weight renormalization, versioning / V2 migration notes.
  - No references to any external reference product anywhere in the repo (per Santosh's instruction).

### 2026-04-21 (previous session)

- Read the build prompt end-to-end.
- Delivered the Multi-Agent Solutions 101 preamble (section 9.d of the build prompt).
- Confirmed understanding of project scope (3-sentence summary, differentiation, skill-pack rationale, harness moat framing).
- Initialized the git repo, wrote `.gitignore` (excludes `old_req/`, `.DS_Store`, `.claude/settings.local.json`), committed, and pushed to `https://github.com/santosha86/Ghost_Check.git` on the `main` branch.
- No project files built.

### Files in the repo as of end of 2026-04-22 later session (pending push from Santosh's personal laptop)

- `GhostCheck_Build_Prompt.md` (+ .pdf) — the binding spec
- `.gitignore` — now also protects `profile/cv.md`, `profile/style.md`, `applications/*/`, `jobs/`
- `SESSION_RESUME.md` (this file)
- `README.md` — front door (+ tier-label footnote added in later session)
- `LICENSE` — MIT (copyright: Achanta Santosh Kumar)
- `CLAUDE.md` — the AI's operating manual
- `docs/SCHEMAS.md` — data contracts
- `docs/HARNESS_ENGINEERING.md` — the moat thesis (5 pillars + "What it is NOT" + GEPA/V1.1 forward reference)
- `docs/ZERO_TRUST.md` — trust + security posture, aligned with Microsoft *Zero Trust for AI* (March 2026)
- `profile/cv.example.md` — Rohan Mehta scrubbed-persona template

### Files kept local, not in git (by user decision)

- `old_req/` — internal PRD and Publishing Playbook
- Personal learning notes (previously planned as `docs/TEACH_NOTES/`) — live in Santosh's Claude Project only

---

## 7. Next step when session resumes

**Day 1 is COMPLETE (10 of 10 files done — 100%).** All documentation, examples, and config files from build prompt section 11 Day 1 are in the repo.

```
README.md                              (done)
LICENSE                                (done)
CLAUDE.md                              (done)
docs/SCHEMAS.md                        (done)
docs/HARNESS_ENGINEERING.md            (done)  (+ GEPA paragraph)
docs/ZERO_TRUST.md                     (done)
profile/cv.example.md                  (done)
profile/style.example.md               (done)
config/profile.example.yml             (done)
config/weights.yml                     (done)  (the only runtime-required Day-1 file)
```

**Next up: Day 2.** Build prompt section 11 Day 2 scope:

```
.claude/skills/ghostcheck/SKILL.md             — the /ghostcheck router
.claude/skills/parser/markitdown-parse.md      — MarkItDown parser skill
.claude/skills/agents/A3-bucket-classifier.md  — most differentiated agent, built first
.claude/skills/aggregator/aggregator.md        — deterministic combiner (starts with 1 input)
```

Then: **end-to-end test** — run `/ghostcheck audit` on Santosh's real `cv.md` + a real JD he was ghosted on. Checkpoint: does `audit.md` land in `applications/YYYY-MM-DD_<slug>/`? Is `bucket-classifier`'s output sensible?

**Per Teach Mode, the flow for each Day-2 file is:**
1. Explain in chat: what it is, why it exists, how it fits.
2. Wait for Santosh to say "continue" or ask follow-ups.
3. Write the file.
4. If there is a reusable design lesson, summarise it in chat so Santosh can save it to his personal Claude Project — no file in the repo.
5. After every 2–3 files, checkpoint: summarize + ask if anything is unclear.

**Load-bearing Day-2 conventions (locked in Day 1, must be honoured):**

- **`SKILL.md` must contain a "Files this audit loads" section** listing CV, JD, `config/profile.yml`, `profile/style.md` (fall back to `.example.md` if the real user file is missing), with load order and precedence (user file overrides example). Pattern inspired by santifer's `_shared.md` Sources of Truth table.
- **Agent frontmatter `inputs:` convention** — drawn from `{cv_text, jd_text, user_profile, user_style, external_context}`. Agents omit inputs they don't need (e.g. `ats-simulator` likely omits `user_style`). The router passes *only* declared inputs to each agent. This is Path 1 (Bundle-and-dispatch); it does NOT require a schema change to `SCHEMAS.md`.
- **`bucket-classifier` and `hm-deep-read` use temperature 0.3**; all other agents use 0.1. Max tokens per agent: 800. These values live in each agent's YAML frontmatter.
- **Santosh's real CV goes in `profile/cv.md`** (gitignored, never pushed). The example at `profile/cv.example.md` is a fallback only.
- **Agent invocation pattern: Pattern B (subagent isolation)**, locked 2026-04-25. The router invokes each agent in a fresh, isolated context window so no agent sees another agent's verdict. In Claude Code V1 this uses the built-in Agent tool; future runtimes (Python service, Gemini CLI, Ollama) replicate by making one separate API call per agent. Pattern A (sequential reads in a shared context) was rejected because it breaks blind fan-out: agent N+1 would see prior agents' verdicts in its context window, producing dependent verdicts that look independent. The semantic requirement of the SKILL.md is "invoke each agent in isolation"; the runtime supplies the implementation.
- **Input validation: Levels 1 + 3**, locked 2026-04-25. Level 1 (file-extension check) runs at the start of an audit and rejects unsupported types (e.g., `.png`, `.zip`) before parsing. Level 3 (LLM classification call) runs after MarkItDown parses the document and asks the model to confirm whether the input is a CV / a JD with a confidence score, fail-closed if confidence is low. Level 2 (content-keyword heuristics) was rejected because it false-positives on non-standard formats — Santosh's real JD is a PowerPoint deck, where heuristic header-matching breaks. The LLM classifier handles the deck and a normal PDF JD identically.

**Other deferred items:**

- **After Day 2 is stable**: create empty structural folders (`applications/`, `patterns/`, `wiki/`) with `.gitkeep` + `README.md` placeholders. Referenced in `CLAUDE.md section 4` but not yet present on disk.
- **Day 5 `ROADMAP.md` content is pre-captured** — see "For Day 5" block in section 6 above. Contains the full Hermes/GEPA V1.1 entry and the Microsoft Observability V1.2/V2 reference.

**Open taste / scope questions Santosh has not yet decided** (low urgency, flag at Day 2 kickoff):

- Does `bucket-classifier` explicitly reference the `UserProfile.target_seniority` enum value in its prompt, or does it infer from title patterns? (Affects prompt design.)
- When the user's real `profile/cv.md` is missing, does the router fall back silently to `cv.example.md`, or halt with an error asking the user to drop in their CV? Fail-closed instinct says halt; UX instinct says helpful warning + example. Santosh's call.

---

## 8. When Santosh resumes, your first message should be roughly this shape

> Hi Santosh — I've read `SESSION_RESUME.md` and `GhostCheck_Build_Prompt.md`. Quick recap of where we are:
>
> - Preamble complete (agent / skill / orchestration / fan-out / framework-vs-skill-pack / AgentVerdict / fail-closed / harness — all covered). Day 1 expanded vocab: Zero Trust for AI, double agents, lateral movement, XPIA, provenance, DSPy+GEPA, Sources of Truth table, bundle-and-dispatch.
> - **Day 1 is COMPLETE.** All 10 files in the repo: README, LICENSE, CLAUDE.md, `docs/SCHEMAS.md`, `docs/HARNESS_ENGINEERING.md`, `docs/ZERO_TRUST.md`, `profile/cv.example.md`, `profile/style.example.md`, `config/profile.example.yml`, `config/weights.yml`. Confirm they are on GitHub; if not, flag to Santosh.
> - Workflow: if in web sandbox, do not try `git push` — returns 403. On personal laptop, `git push` works.
> - Learning notes stay in Santosh's personal Claude Project (no `docs/TEACH_NOTES/` folder in the repo).
> - **Path-1 bundle-and-dispatch is locked.** Day-2 `SKILL.md` reads all user-layer files and dispatches to each agent with only the inputs its frontmatter declares. No schema change needed.
> - **Next up: Day 2** — `.claude/skills/ghostcheck/SKILL.md` (router) first, then `markitdown-parse.md`, then `A3-bucket-classifier.md`, then `aggregator.md`. End-of-Day-2 target: run a real end-to-end audit on Santosh's `cv.md` + one real ghosted JD.
>
> Teach Mode is on. Day 2 files are mostly runtime-required (not documentation), so explain-before-write matters more than ever. Ready to start with the router explanation, or do you want to revisit anything from Day 1 first?

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