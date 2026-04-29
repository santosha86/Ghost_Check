# GhostCheck — Session Resume

**Last updated:** 2026-04-25
**Next Claude:** read this file FIRST, then `GhostCheck_Build_Prompt.md`, then greet the user.

**Workflow constraint (important):** Santosh splits work between **two machines**, and Claude's behaviour must adapt:

- **Personal laptop** (Claude Code Desktop, macOS): `git push` works normally. File writes persist on disk. Real CV (`profile/cv.pdf`) and real JD (`jobs/strategy-ai-architect-chief.pptx`) live here, gitignored. This is where the audit pipeline actually runs.

- **Office laptop** (Claude Code on the web sandbox): `git push` returns 403. File writes in the sandbox are ephemeral — they exist only for the session and do not persist back to disk on Santosh's office machine. Read access to GitHub works (read-only review of what's already pushed). For any change to land in the repo, Santosh must mirror sandbox content into his personal Claude Project and eventually push from his personal laptop.

**Always confirm which environment the session is in before attempting `git push` or assuming file persistence.** Look at the current working directory and the surrounding signals: if it is `/Users/santosh_work/...` you are on the personal laptop and pushes work; if you are inside a sandboxed `/tmp/`-style path or the system reminders mention a sandbox, treat all writes as ephemeral and surface content to Santosh as copyable blocks rather than relying on the file system.

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
- **Skills vs Subagents** (Claude Code primitives, distinction discovered 2026-04-25). A Skill (under `.claude/skills/`) is a reusable prompt or workflow that runs in the MAIN conversation context — shares the parent's working memory. A Subagent (under `.claude/agents/`) runs in its own isolated context window with its own system prompt, tool restrictions, and model — invoked via the Task tool with `subagent_type` parameter, returns only its summary to the parent. Both are markdown files with YAML frontmatter, but their runtime behaviour is fundamentally different. GhostCheck uses Skills for the router, parser, aggregator, and enrichment helpers (they need to share context with each other), and Subagents for the 11 agents (they must NOT share context, to preserve blind fan-out).
- **Pattern B (subagent isolation per agent)** — locked architectural pattern where the router invokes each of the 11 agents via the Task tool with `subagent_type` matching the agent's name. Each agent runs in its own isolated context. In Claude Code V1 this uses the built-in Task tool; future runtimes (Python service, Gemini CLI, Ollama) replicate by making one separate API call per agent. Pattern A (sequential reads in shared context) was rejected because it produces dependent verdicts that look independent.

Terms he may still want deeper treatment on (he hasn't asked yet, but watch for it): YAML frontmatter internals, Pydantic discriminated unions, logistic regression / weighted logistic aggregator math, temperature, MarkItDown.

---

## 6. Session log (what was done, newest first)

### 2026-04-25 (later sessions — Day 3 build and validation)

- **README intro tightened for interviewer demo** (commit `4f7c170`). Replaced the longer "What makes GhostCheck different" framing with a sharper opening landing the architectural thesis in three sections; path fix on the harness-engineering link.
- **Day 3 file writes complete (7 files).** Pattern A — enrichment-then-agent in alternating order — was used so each agent had its enrichment data ready when written.
  - `.claude/skills/enrichment/google-test-lookup.md` — wraps WebSearch, classifies results into the GoogleResult shape from SCHEMAS.md section 7.
  - `.claude/agents/google-test.md` — judges online surface vs. JD's tier expectation. Tier-aware severity rubric.
  - `.claude/skills/enrichment/jd-age-detector.md` — three-strategy date inference (frontmatter > text > unknown). No web calls.
  - `.claude/agents/posting-decoder.md` — judges theater-posting risk via age × text combination.
  - `.claude/skills/enrichment/company-classifier.md` — builds the Company object with per-field provenance via the sources map.
  - `.claude/agents/it-services-discount.md` — judges silent-downgrade risk for IT-services-tenured candidates targeting consulting / bigtech / product firms. Asymmetric framing.
  - `.claude/agents/headline-filter.md` — judges six-second recruiter-scan tier-clarity.
- **End-of-Day-3 validation re-run on Santosh's real CV+JD pair.** Manual step-through (no Claude Code restart). Probability dropped from 64.6% (Day 2 close) → 41.5% (Day 3 strict, with two UNKNOWN agents) → 35.8% (Day 3 with company-classifier fix applied). The Day-3 audit lands at `applications/2026-04-25_strategy-ai-architect-chief_2/audit.md`. Top driver: `it-services-discount` HIGH (matched Santosh's hunch — confirmed services-firm tenure was a real silence driver).
- **Real design gap surfaced and fixed during validation.** Strict company-classifier returned null when no company name was extractable from the JD (common with PowerPoint deck JDs that reference "the firm" generically). This left it-services-discount UNKNOWN, breaking the most diagnostic agent on this exact case. Fix applied to two files:
  - `company-classifier.md` Step 1 relaxed to use a placeholder name (`"[unspecified hiring firm — name not in JD]"`) when no real name is extractable. Step 4 web-search skipped for placeholder names. Company object exists with `name=placeholder, company_type=inferred, sources.company_type=text_inferred` when JD body language gives a clear signal.
  - `it-services-discount.md` behavioural rule relaxed to accept any `company_type` set with `sources.company_type` of `text_inferred / web_search / user_provided` (rejecting only when set to `default` or when the whole Company object is null).
- **New feedback memory:** "Empirical heuristics — write the best draft, calibrate from real audits." For agent severity rubrics and judgment heuristics where only real audits validate the choice, skip per-file approval gates. Architectural and schema decisions still require approval.

**Day 3 close design state:**

- 5 of 11 agents live (all 5 Tier A: bucket-classifier, google-test, posting-decoder, it-services-discount, headline-filter).
- 3 of 4 enrichment skills live (google-test-lookup, jd-age-detector, company-classifier; the V1.1 Hermes-evolution-trigger skill is not in scope).
- Architecture validated against real-world ground truth: probability moved meaningfully toward the actual ghosted outcome.

### 2026-04-25 (third session — personal laptop, Day 2 prep)

- Reviewed all Day 1 work in detail. Quality: high. No content rework needed.
- **Symbol sweep across all my files** — replaced section signs, arrows, approximation symbols, checkmarks, box-drawing characters, and the one stray emoji with plain English. Build prompt itself untouched (Santosh's spec). 40 replacements across 8 files. Driven by Santosh's feedback that AI-styling tics add reader friction in a plain-text project. Saved as a memory rule for future work.
- **Architectural decision locked: Pattern B (subagent isolation per agent)** for the router-to-agent dispatch. Each of the 11 agents runs in its own isolated context window. The router builds the per-agent input bundle once, then invokes each agent via Claude Code's Task tool with `subagent_type` parameter (V1) or via separate API calls in any future runtime (Python service, Gemini CLI, Ollama). Pattern A (sequential reads in a shared context window) was rejected because it semantically breaks blind fan-out: agent N+1 would see prior agents' verdicts in its context, producing dependent verdicts that look independent — the worst possible outcome.

- **Architectural decision locked: directory split between `.claude/skills/` (Skills) and `.claude/agents/` (Subagents).** Discovered while researching the Task tool: Claude Code distinguishes Skills (run in main conversation context) from Subagents (run in isolated context, invoked via Task tool with subagent_type). Subagent auto-discovery requires `.claude/agents/` exactly — the build prompt's `.claude/skills/agents/` path is invisible to the Task tool. So the router, parser, aggregator, and enrichment helpers stay under `.claude/skills/` (as the build prompt intends), but the eleven agents move to `.claude/agents/` as flat files (e.g. `bucket-classifier.md`, no tier prefix in filename — tier encoded in frontmatter). Build prompt section 6 stays untouched as Santosh's spec; the deviation is recorded in CLAUDE.md section 4 repo layout.
- **Architectural decision locked: Input validation Levels 1 plus 3.** Level 1 = file-extension check at the start of an audit, rejecting unsupported types before parsing. Level 3 = LLM classification call after MarkItDown parsing, asking "is this a CV / is this a JD" with a confidence score, fail-closed if confidence is low. Level 2 (content-keyword heuristics) was rejected because Santosh's real JD is a PowerPoint deck, on which heuristic header-matching breaks. The LLM classifier handles decks and PDFs identically.
- **Real CV and JD organised into the audit-ready paths.** `CV_StrategyAnd_AI_Lead.pdf` moved to `profile/cv.pdf` (gitignored). `Strategy&_AI Solution Architect - JD - Chief.pptx` moved to `jobs/strategy-ai-architect-chief.pptx` (jobs directory gitignored). The PowerPoint format is real-world signal that the parser must handle .pptx via MarkItDown — confirmed MarkItDown supports it.
- **Slug naming convention adopted for filenames in `jobs/` and `applications/`** — lowercase, hyphens for word boundaries, no special characters. Avoids shell-quoting hassles, cross-platform issues, and downstream tool breakage.
- New feedback memory: explain alternatives deeply BEFORE asking Santosh to choose, not after. Each option presented as a full paragraph showing what it does mechanically and what its consequences are in practice.
- **Slide deck pivot deferred** — was pulled forward from Day 5 launch prep then deferred again the same day because Santosh chose to prioritise Day 2 file writes over the install/restart cycle for the `frontend-slides` Claude Code skill. Returns to Day 5 launch prep as originally scoped.

- **Day 2 file writes complete (4 of 4).** All four files written in this session on personal laptop:
  - `.claude/skills/ghostcheck/SKILL.md` — the slash-command router. 8-step audit procedure, Files-this-audit-loads Sources of Truth table, Path-1 bundle-and-dispatch, Pattern-B subagent isolation via the Agent tool, fail-closed at every step.
  - `.claude/skills/parser/markitdown-parse.md` — smart parser. PDF and Markdown go through Claude Code's native Read tool (zero install). DOCX, PPTX, HTML, TXT shell to MarkItDown via Bash; halts with explicit install message if MarkItDown is missing.
  - `.claude/agents/bucket-classifier.md` — first subagent. Establishes the agent template every Day-3 and Day-4 agent will follow. `tools: []` (empty allowlist, least privilege). Embedded AgentVerdict schema in the prompt body. Severity rubric, evidence-citation rules, behavioural invariants.
  - `.claude/skills/aggregator/aggregator.md` — deterministic combiner. Reads weights from `config/weights.yml`, redistributes weight on UNKNOWN per SCHEMAS.md section 13, computes callback probability via weighted-logistic with V1 hyperparameters `k=4.0, midpoint=0.4` (using `bc -l` for the math, no Python dependency). Writes `audit.md`, `verdicts.json`, `context.json` to `applications/YYYY-MM-DD_<slug>/`.

- **Architectural decision locked: weighted-logistic formula in V1, hyperparameters tuned by V1.2 calibration.** Formula stays the same V1 to V2 — calibration only adjusts `k` and `midpoint`. Linear was considered and rejected because (a) extreme cases in linear are unrealistic (0% and 75% endpoints versus logistic's 8% and 64%), (b) build prompt and SCHEMAS.md both specify "weighted logistic," (c) Santosh's principle of formula consistency across versions matters more than V1 simplicity.

- **README.md prerequisites section updated** to reflect MarkItDown's optional status: required only for DOCX, PPTX, HTML inputs; PDF and Markdown work without any install.

**End-to-end validation completed in this session (manual step-through, since the new subagent could not be auto-discovered without a session restart).** The validation exercised the full SKILL.md flow on Santosh's real CV (`profile/cv.pdf`) and PowerPoint JD (`jobs/strategy-ai-architect-chief.pptx`):

- File extension validation passed (`.pdf` and `.pptx` both in allow-list).
- CV parsed via Claude Code Read tool (zero install). JD parsed via MarkItDown CLI — required `pip install 'markitdown[all]'` because the bare `pip install markitdown` does not include format extras. The parser's halt-with-install-message worked as designed; the user installed the extras and the JD parsed cleanly on retry.
- Document classification pass: doc1 confidence ~0.99 as CV, doc2 confidence ~0.98 as JD. Both above 0.7 threshold.
- User-layer files fell back to `.example.*` siblings (real `profile/style.md` and `config/profile.yml` not yet set up).
- ExternalContext built minimal per V1 spec (no Google search, no company enrichment, no JD age detection).
- bucket-classifier verdict (manually computed by following its system prompt verbatim, since the subagent was not auto-discovered): severity LOW, score 0.25. CV reads at the target Chief / Principal Solution Architect tier with multiple strong evidence signals (design authority, multi-team delivery, 16+ years, executive engagement). Schema validation passed.
- Aggregator math: with only bucket-classifier active, effective_weight redistributed to 1.0; z = 0.25; callback probability via `bc -l` of `1 / (1 + exp(4 * (0.25 - 0.4)))` = 0.6456 (64.6%).
- Output written to `applications/2026-04-25_strategy-ai-architect-chief/`: `audit.md`, `verdicts.json`, `context.json`. All three files conform to the `SCHEMAS.md` contract.

**Honest limitation surfaced by the validation: V1 with one agent live is too optimistic on this case.** Santosh was ghosted on this real application; V1 reports 64.6%. The mismatch is expected — the agents that would most likely catch this specific silence (it-services-discount, channel-mix, google-test, posting-decoder) are scheduled for Days 3 and 4. The V1 audit is intentionally narrow at this build state and the audit.md surfaces this honestly with a "V1 starting-state notice" callout at the top and an explicit "Not assessed (not yet implemented)" section listing the ten missing agents with one-line descriptions.

**Two real-world findings from the validation, fixed in this same session:**

1. README.md and the parser skill both updated to specify `pip install 'markitdown[all]'` (with the single-quotes for zsh) instead of the bare `pip install markitdown`. The bare install does not include PPTX, DOCX, or HTML extras; users hit the wall on first non-PDF input.
2. Aggregator skill updated to partition verdicts into THREE sets (S = non-UNKNOWN, U = UNKNOWN, N = not-yet-implemented) and surface N as "not yet implemented" in the audit's "Not assessed" section. Previously the spec only handled S and U; in V1 starting state where most of the eleven agents do not exist yet, the audit needs to honestly tell the user which agents were not run because they have not been written, not because they returned UNKNOWN.

**Day 2 is now closed.** Pipeline runs end-to-end, output structure is correct, formula is implemented and verified, real-world findings are applied, audit.md lands honestly with a clear V1-state notice. Day 3 begins with the remaining Tier A agents.

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

### ACTIVE PRIORITY: End-to-end testing then Day 6 cleanup pass

Day 5 closed 2026-04-29. All documentation, V1.1 / V1.2 placeholders, folder READMEs, and the ROADMAP have shipped. V1 is launch-ready by file-count and content; the architectural cleanup pass (per section 9 technical-debt list) happens after end-to-end testing on Day 5/6.

Files added in Day 5 (committed in the Day-5-close commit):

- `docs/ROADMAP.md` — V1 to V2 plan with V1.1 (Hermes/GEPA self-evolution, batch pattern analysis, sharable artefacts), V1.2 (calibration, BYOK, Karpathy LLM Wiki, runtime Zero Trust), and V2 (Python service, web frontend, Microsoft AI Observability framework). Honest calibration disclaimers included.
- `.claude/agents/V1_1-pattern-detector.md` — placeholder for the V1.1 batch pattern agent that produces the personal silence signature.
- `.claude/skills/wiki-compiler/wiki-compile.md` — placeholder for the V1.2 Karpathy LLM Wiki implementation.
- `applications/README.md` — folder convention plus `outcome.md` and `channel.md` logging format.
- `patterns/README.md`, `wiki/README.md` — placeholder READMEs for V1.1 and V1.2 output.
- `.gitkeep` files in all three structural folders so they exist for forkers.
- `README.md` — Roadmap section updated to point at `docs/ROADMAP.md` with sharper one-liners per version.

**Next step: end-to-end testing.** Execute the validation plan that Santosh requested before Day 6 cleanup begins. The intent is to surface issues that we have not seen yet so the cleanup pass addresses them in the same focused session as the section-9-A architectural debt.

End-to-end testing scope:

1. **Re-validate the Day-4 audit** on Santosh's real CV plus PowerPoint JD. Confirm the previous 40.1% probability still produces. Sanity check that no Day-5 file additions broke runtime behaviour (they should not — Day 5 added documentation only, no runtime logic).
2. **Cross-domain audit** on at least one non-AI / non-Santosh CV+JD pair. Recommended: write a scrubbed-persona electrical engineer or finance director CV with a matching JD; run `/ghostcheck audit` end-to-end; observe what the agents do with prompts that were written from AI-domain data. This is the test that validates the harness-portability thesis. Findings populate section 9-C of this file.
3. **Edge-case parser test** — run on a markdown CV (not PDF, not PPTX) and a plain-text JD. Confirm Read-tool path works for both. Should be smoke-test-only.
4. **Audit-folder log test** — write three example `applications/<slug>/outcome.md` and `channel.md` files for a fictional history; run `/ghostcheck audit` again; confirm `funnel-math` and `channel-mix` activate (move from UNKNOWN to a real verdict). This validates the section 9-B3 router pre-aggregation expectation.

After end-to-end testing surfaces issues:

- Populate section 9-C of this file with each issue, severity, and proposed fix.
- Begin the Day-6 cleanup pass executing section 9-D's plan: A1 first (structured-extraction architecture, 3-4 hours), A2 falls out of A1 mostly, A3 (cross-domain validation), B-items in priority order, C-items as the final sweep.

Estimated total Day-6 cleanup: 6-8 hours focused work. Could be one long session or split across two.

After cleanup, V1 is genuinely launch-ready (clean architecture, documented debt resolved, cross-domain-validated). V1.1 work begins per `docs/ROADMAP.md`.

### Ground-truth check (completed 2026-04-25)

Santosh confirmed during Day 3 that `it-services-discount` was the agent he most suspected was the silence driver on this specific application. End-of-Day-3 validation confirmed the hunch: the agent fired HIGH on his real CV+JD pair, contributed the largest single weight to the callback probability drop (64.6% → 35.8%), and produced specific reframing fix hints (delivery-language → ownership-language) that match the actual silence pattern at top-tier consulting firms.

The exercise procedure below is preserved in case the same check is useful for future audits on different CV+JD pairs.

### Original procedure (kept for reference)

Santosh was ghosted on this exact CV+JD pair. He has real-world knowledge about WHY that the audit cannot see directly. Before writing Day 3 agents, walk him through this short exercise:

**Step 1 — read the audit.** Open `applications/2026-04-25_strategy-ai-architect-chief/audit.md` (or its content if you are on web sandbox where that file is not visible — you can recreate it in chat from `verdicts.json` and `context.json` which Santosh can paste in if needed). The relevant section is "Not assessed (not yet implemented)" which lists ten agents.

**Step 2 — among those ten, four are most likely the real silence drivers for THIS specific application:**

- `it-services-discount` (planned Day 3) — would assess whether tenure at large IT-services firms (Wipro, TCS, Nor Consult in this CV) triggers a silent downgrade at top-tier consulting and product companies.
- `channel-mix` (planned Day 4) — would assess whether the application channel (Easy Apply, referral, direct DM, exec search) matches what works at the Chief level.
- `google-test` (planned Day 3) — would assess whether Googling the candidate's name surfaces enough thought-leadership signal to validate the JD's "thought leadership engagements" expectation.
- `posting-decoder` (planned Day 3) — would assess whether the JD is a genuine open requisition or a theater posting (already pre-filled internally).

**Step 3 — ask Santosh which one or two of these four he ACTUALLY believes was the silence driver for this specific application.** He has ground truth from his real experience: how he applied (which channel), whether anyone inside the firm signalled anything, what his online presence actually looks like vs. competitors at this seniority, whether the role was visibly filled internally before he applied. He does not have to share the answer publicly, but having it in his head shapes Day 3 priorities.

**Step 4 — use his answer to choose which Day-3 agent to write FIRST.** If he believes `it-services-discount` was the driver, write that agent first so the system's accuracy is testable fastest on the next audit run. Same logic for `google-test` or `posting-decoder`. (`channel-mix` is Day 4 by build prompt order, not Day 3, so cannot be first this round.)

**Step 5 — re-run `/ghostcheck audit` on the same CV+JD pair after the first Day-3 agent lands.** If the callback probability moves meaningfully (say from 64% toward 30-40%) and the new agent's verdict matches Santosh's real-world hypothesis, the system is on the right architectural track. If the probability does not move or the verdict misses the actual reason, that is a more important finding than any individual agent — it tells us the design needs adjustment, not just more agents.

**Why this exercise matters more than the audit's number:** at this build state, the 64.6% probability is almost meaningless because only one of eleven agents contributed. The real validation is whether the system is identifying the right HYPOTHESIS SPACE about why Santosh was ghosted. If the four agents called out match his real-world hypothesis, the architecture is sound. If they miss the actual reason entirely, the architecture needs rethinking. This exercise checks the architecture, not the math.

**Day 3 scope (per build prompt section 11):** the remaining Tier A invisible-failure agents.

```
.claude/agents/google-test.md           — A1 (online surface assessment)
.claude/agents/posting-decoder.md       — A2 (theater posting detection)
.claude/agents/it-services-discount.md  — A4 (services-firm tenure downgrade signal)
.claude/agents/headline-filter.md       — A5 (six-second recruiter scan)
```

Plus the enrichment skills they depend on:

```
.claude/skills/enrichment/google-test-lookup.md      — web search wrapper for A1
.claude/skills/enrichment/jd-age-detector.md         — date inference for A2 (and B3 stale-detector on Day 4)
.claude/skills/enrichment/company-classifier.md      — services vs product vs consulting for A4
```

**Each Day-3 agent follows the bucket-classifier template established in Day 2:**

- `name`, `description`, `model: sonnet`, `tools: []` (or `tools: [Read]` if the agent reads enrichment outputs from disk — most do not).
- `inputs:` field declares which of `{cv_text, jd_text, user_profile, user_style, external_context}` the agent needs.
- `capabilities:` field declares external tool permissions (e.g. `[web_search]` for `google-test`, `[]` for the others).
- AgentVerdict schema embedded directly in the prompt body.
- Severity rubric, evidence-citation rules, behavioural invariants — same shape as bucket-classifier.

**At end of Day 3,** re-run `/ghostcheck audit` on the same CV+JD pair. Expectation: the callback probability drops meaningfully because `it-services-discount` is now active and will likely flag HIGH severity for Santosh's Wipro+TCS+Nor Consult background versus a top-tier consulting Chief role.

**Day 4 then adds:** Tier B (funnel-math, channel-mix, stale-detector) and Tier C (ats-simulator, recruiter-30sec, hm-deep-read), at which point all eleven agents are live.

**Open architectural questions deferred to Day 3 kickoff (low urgency):**

- For `google-test`: which web search tool does the enrichment skill use — Claude Code's WebSearch native tool, an MCP-backed search, or a shelled-out CLI like ddgr or googler? Prefer native, fallback documented.
- For `it-services-discount`: how does the agent get the company-type classification of the JD's hiring company? In V1 the ExternalContext.company is null (V1 minimal); Day 3 likely needs the company-classifier enrichment skill to populate it. Sequence matters.
- For `posting-decoder`: theater-posting detection benefits from JD age. The jd-age-detector enrichment skill needs to come first OR we accept that posting-decoder returns UNKNOWN when JD age is null.

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
- **Agent invocation pattern: Pattern B (subagent isolation)**, locked 2026-04-25. The router invokes each agent in a fresh, isolated context window so no agent sees another agent's verdict. In Claude Code V1 this uses the built-in Task tool with `subagent_type` parameter pointing at named subagent files in `.claude/agents/`; future runtimes (Python service, Gemini CLI, Ollama) replicate by making one separate API call per agent. Pattern A (sequential reads in a shared context) was rejected because it breaks blind fan-out: agent N+1 would see prior agents' verdicts in its context window, producing dependent verdicts that look independent. The semantic requirement of the SKILL.md is "invoke each agent in isolation"; the runtime supplies the implementation.

- **Directory split: Skills under `.claude/skills/`, Subagents under `.claude/agents/`**, locked 2026-04-25. The build prompt section 6 originally placed all eleven agents under `.claude/skills/agents/`, but Claude Code's documented convention is that subagents (which is what the eleven agents are) live at `.claude/agents/` to be auto-discovered by the Task tool. Following Claude Code's convention is necessary for native blind-fan-out to work; placing agents at the build prompt's path would silently break the isolation guarantee. Concrete layout: the router (`ghostcheck/SKILL.md`), the parser (`parser/markitdown-parse.md`), the aggregator (`aggregator/aggregator.md`), and the enrichment helpers stay under `.claude/skills/` because they need to share context with each other and with the user. The eleven agents live as flat files under `.claude/agents/` (e.g. `bucket-classifier.md`, `google-test.md`) — no tier prefix in the filename, since the Task tool dispatches by subagent_type which matches the filename without `.md`. Tier (A / B / C) is encoded in each subagent's frontmatter for the aggregator's weighting logic. The build prompt itself stays untouched as Santosh's spec; the deviation is documented here and reflected in `CLAUDE.md` section 4 repo layout.
- **Input validation: Levels 1 + 3**, locked 2026-04-25. Level 1 (file-extension check) runs at the start of an audit and rejects unsupported types (e.g., `.png`, `.zip`) before parsing. Level 3 (LLM classification call) runs after MarkItDown parses the document and asks the model to confirm whether the input is a CV / a JD with a confidence score, fail-closed if confidence is low. Level 2 (content-keyword heuristics) was rejected because it false-positives on non-standard formats — Santosh's real JD is a PowerPoint deck, where heuristic header-matching breaks. The LLM classifier handles the deck and a normal PDF JD identically.

**Other deferred items:**

- **After Day 2 is stable**: create empty structural folders (`applications/`, `patterns/`, `wiki/`) with `.gitkeep` + `README.md` placeholders. Referenced in `CLAUDE.md section 4` but not yet present on disk.
- **Architecture slide deck (Path 3, deferred)**: pulled forward 2026-04-25 then deferred again the same day because Santosh chose to prioritise Day 2 file writes over the `frontend-slides` install/restart cycle. Returns to Day 5 launch prep as originally scoped. Locked 8-slide outline still applies when the deck is built. See the 2026-04-25 entry in section 6 for the outline.
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
> - **Days 2, 3, 4, and 5 are all CLOSED.** All eleven agents written, all four enrichment skills, all documentation, V1.1 / V1.2 placeholders, folder READMEs, ROADMAP. V1 is launch-ready by file count.
> - **Pending architectural cleanup.** Section 9 of this file catalogues V1 technical debt for a single focused cleanup pass after end-to-end testing. The structured-extraction architecture (A1) is the largest item — replaces hardcoded text in agent prompts with named-field references. Plus cross-domain validation (A3), config-driven firm lists (A2), router pre-aggregation of applications_log (B3), and several smaller items. Estimated 6-8 hours of focused work.
> - **ACTIVE: end-to-end testing.** Run the validation plan from section 7 of this file: re-confirm the Day-4 audit, run a cross-domain audit (electrical engineer or finance director), parser edge-case test, applications-log activation test. Capture findings in section 9-C. Then execute section 9-D cleanup plan in order.
>
> Teach Mode is on. The build prompt's five-day scope is functionally complete. What remains is the disciplined cleanup pass — exactly the kind of work that distinguishes a demo-quality build from a production-quality build.

Do NOT start writing files until he says to proceed.

---

## 9. Technical debt — fix in one focused pass after Day 5 + end-to-end testing

This section is the consolidated list of architectural and content debt accumulated during Days 2-4 that we deliberately did not fix mid-build, plus the issues we expect end-to-end testing on Day 5/6 to surface. Decision logged 2026-04-29: V1 ships with the debt for the interviewer demo (current state works for Santosh's profile and Santosh's CV+JD pair); fixes happen in one focused session after Day 5 documentation lands and end-to-end testing surfaces additional issues.

### A. Architectural debt (the big one)

**A1. No structured-extraction layer between parser and agents.**

Every agent currently re-parses raw `cv_text` and `jd_text` as markdown strings. The right architecture parses CV and JD ONCE into a structured YAML object with named fields (a `CandidateProfile` and a `JobProfile`), and every agent reads named fields rather than re-interpreting raw text. Concrete shape proposed in chat 2026-04-29:

```yaml
candidate:
  name, current_title, current_company, total_years, location
  recent_roles: [{title, company, start_date, end_date, client, bullets}]
  earlier_roles: [...]
  summary, competencies, technologies, education

target_role:
  title, company_name, seniority_keyword
  years_required
  required_skills, preferred_skills, responsibilities
  market_leadership_required, external_thought_leadership_required
```

Files affected: `docs/SCHEMAS.md` (add the two schemas), `.claude/skills/parser/markitdown-parse.md` (extend to do structured extraction), `.claude/skills/ghostcheck/SKILL.md` (router passes structured profiles instead of raw text), all 11 agent files (frontmatter `inputs:` references named fields; body references named fields; hardcoded examples disappear because there is nothing to hardcode).

Estimated work: 3-4 hours. Becomes the single largest item in the V1.1 cleanup pass.

**A2. Hardcoded Santosh-specific and AI-specific content in agent prompts.**

Three layers of violations identified during Day 4 review:

- **Identity-level violations** (in the agent's "## My job" or description):
  - `funnel-math.md` My-job paragraph 2 says "senior AI architect / principal / director-level candidate" — should reference `user_profile.target_seniority` dynamically.
  - `bucket-classifier.md` My-job paragraph 2 uses "Staff Engineer level when the JD targets Director" as a concrete example — should be tier-abstract.

- **Heuristic-level violations** (in "What I look for" sections):
  - `google-test.md` assumes GitHub is a relevant surface — true for software/AI, less true for electrical engineering academia (which uses IEEE / ResearchGate) or finance (which uses none of those). Should be parameterised by domain or moved to `config/online_surfaces_by_domain.yml`.
  - `it-services-discount.md` has the verbatim IT-services firm list (Wipro, TCS, Infosys, ...) baked into the prompt body — should live in `config/known_firms.yml` and be referenced by path.
  - `company-classifier.md` has verbatim consulting / services / bigtech firm lists baked in (Heuristics A/B/C) — same fix, move to `config/known_firms.yml`.

- **Example-level violations** (fix-hint examples reference Santosh's actual CV):
  - `google-test.md` fix-hint example mentions "the SABIC RAFT-vs-RAG decision".
  - `it-services-discount.md` fix-hint examples quote TCS bullets, mention SABIC clients.
  - `headline-filter.md` fix-hint examples rewrite Santosh's actual "AI Architect & Manager" headline.
  - `channel-mix.md` fix-hint example mentions "PGP at NIT Warangal" (Santosh's actual education).
  - `ats-simulator.md` fix-hint examples reference LangGraph, MLOps, RAFT (AI keywords).
  - `recruiter-30sec.md` fix-hint examples mention "Nor Consult Telematics" and "Saudi Electricity Company (SEC)".
  - `hm-deep-read.md` fix-hint examples are LITERALLY Santosh's CV bullets verbatim.

Right fix per layer:
- Identity-level: replace hardcoded tier names with references to `user_profile.target_seniority` / `target_titles`.
- Heuristic-level: move firm lists and domain assumptions to `config/*.yml` files. Agents read from config, forkers edit YAML not prompts.
- Example-level: replace Santosh-specific examples with explicit references to `cv.example.md`'s "Rohan Mehta" canonical persona, with an inline note "(illustrative — uses cv.example.md persona; the agent fills in your actual CV's content at runtime)".

Estimated work: subsumed by A1 above (structured-extraction architecture removes most of the example-level violations naturally).

**A3. Cross-domain validation gap.**

Day 3 and Day 4 validation passed on Santosh's CV+JD pair. That validation was partly coincidence: the heuristics were written FROM Santosh's data, so they naturally fit Santosh's data. A real cross-domain test would run the system on at least three different CV+JD pairs from genuinely different domains (e.g., a senior electrical engineer applying to a hardware-architect role; a senior product manager applying to a CPO role; a senior finance director applying to a CFO role) and confirm the verdicts are sensible.

Until that test happens, the system's "domain-agnostic" claim is theoretical, not validated. After A1 lands, the test is much more meaningful because the architecture itself is genuinely domain-agnostic.

Plan: alongside A1's structured-extraction work, write three scrubbed-persona example CVs and JDs from different domains, run audits on all three, confirm sensible verdicts, fix any agents whose heuristics break.

### B. Smaller debt items catalogued

**B1. Aggregator hyperparameters are V1 defaults, not calibrated.** `k = 4.0`, `midpoint = 0.4` in `aggregator.md`. V1.2 calibration loop will tune from real outcome data once enough audits accumulate. Document this status more prominently in `aggregator.md` and `audit.md` footers.

**B2. `weights.yml` per-agent weights are V1 defaults.** Same status as B1. The 0.14 / 0.12 / 0.10 / etc. weights were chosen by intuition; calibration will adjust.

**B3. Router does not pre-aggregate `applications_log` for funnel-math and channel-mix.** SKILL.md needs a step that walks `applications/*/outcome.md` and `applications/*/channel.md`, builds the `applications_log` array, and passes it. Currently those agents return UNKNOWN because the input is empty. Document the file convention in `applications/README.md` (Day 5 work).

**B4. Company-classifier name-extraction is brittle.** Day 3 fix relaxed the strict no-name-then-null rule, but the underlying name extraction (frontmatter or first heading) misses many real cases. V1.1 could use web search on JD content + role title to identify the hiring firm even when not named verbatim.

**B5. JD-frontmatter convention is documented but not encouraged.** README.md mentions optional YAML frontmatter for JDs (with `posted_date`, `source_url`, etc.), but the demo run on Santosh's PowerPoint JD showed how easy it is to skip. V1.1 could add a `--add-frontmatter` flag or a startup prompt that asks the user to fill in known fields before parsing.

**B6. Probability counter-intuitively went UP between Day 3 and Day 4.** Day 3 was 35.8%; Day 4 was 40.1%. Mathematically correct (Tier C agents found positive signals that diluted the Tier A negative-severity contribution) but visually surprising. Worth a note in `aggregator.md` body and in `audit.md` template explaining why "more agents active" does not always mean "lower probability."

**B7. Some Day 1-2 docs reference the .claude/skills/agents/ path that we deviated from on Day 2-prep.** The build prompt itself uses the old path; that is intentionally untouched. But cross-references within our own `docs/` and inside the agent prompts need a sweep. Risk: low. Confusion potential: small but real.

### C. Issues end-to-end testing surfaced

End-of-Day-5 smoke test (2026-04-29, Santosh's profile + JD) — runtime stability validation after Day 5 added documentation only. Findings:

**C1. Smoke test PASSED.** Pipeline runs end-to-end. All eleven V1 agents produce verdicts (or honest UNKNOWN). Aggregator math reproduces deterministically — z = 0.500, callback probability = 40.13%, identical to Day-4-close audit. Day 5's documentation additions made no runtime behaviour change. This is the expected outcome and confirms the system is stable to ship at the V1 architecture level.

**C2. Audit metadata could be sharper about "missing inputs cost."** Four agents (posting-decoder, funnel-math, channel-mix, stale-detector) returned UNKNOWN due to data-limited inputs. The audit's metadata footer says they are UNKNOWN but does not quantify how the probability would shift if those inputs were present. Future enhancement: surface "if you logged X applications, the probability would land in N-M range; if you added posted_date to JD frontmatter, the probability would shift by Y points."

**C3. The Day-3 company-classifier fix is load-bearing for Santosh's specific case.** Without the placeholder-name + text-inferred-company-type fallback, `it-services-discount` would have returned UNKNOWN — losing the most diagnostic agent on this exact CV+JD pair. The smoke test confirmed the fix works as designed. Lesson: validation cycles that surface design fixes during implementation (rather than after) are higher-value than post-launch fixes. Continue this practice on the Day-6 cleanup pass — re-run the audit after each significant fix to confirm the system still produces sensible verdicts.

**C4. Cross-domain test deferred but recommended.** The smoke test ran on Santosh's CV+JD only. Cross-domain validation (electrical engineer or finance director CV + matching JD) is the genuinely informative test — it would surface whether section 9-A2 (hardcoded content in prompts) actually breaks the system or merely produces slightly-awkward fix-hint examples. Worth doing before the cleanup pass executes A1, so cleanup priorities reflect what cross-domain testing reveals.

**C5. Performance: ~30-90 seconds for a manual step-through on Santosh's CV+JD.** Acceptable for single-user V1. With actual Claude Code Agent-tool dispatch (which requires session restart so subagents are auto-discovered), eleven parallel subagent contexts would run faster than the manual step-through. Worth confirming once the next real `/ghostcheck audit` invocation happens post-restart.

**C6. No parser edge-case tests run yet.** Markdown CV, plain-text JD, malformed inputs — all unspecified. Recommend: include a markdown-CV smoke test in the cross-domain test (write the scrubbed-persona CV as `.md`, not `.pdf`, to exercise the Read-tool path independently of MarkItDown).

**C7. No load test or batch test.** Outside V1 scope; mentioned for completeness.

**C8. Schema-validation under unusual agent output.** Not actively tested. The ats-simulator's coverage-based scoring relies on string matching that could conceivably produce edge-case scores; the hm-deep-read's bullet-level scoring depends on Claude correctly counting strong-vs-weak verbs across 30+ bullets. Both are areas where a deliberately adversarial test (a CV designed to fool the agent) might reveal brittleness. Not blocking V1 ship; flagged for the cleanup pass.

### D. Execution plan for the cleanup pass

Once Day 5 closes and end-to-end testing produces section C, execute in this order:

1. **A1 first (structured-extraction architecture).** Largest blast radius, highest leverage. Do this in a focused session of 3-4 hours. Updates SCHEMAS.md, parser, router, all 11 agents.
2. **A2 falls out of A1 mostly.** What remains (config-driven firm lists) is 30 minutes of YAML extraction.
3. **A3 (cross-domain validation).** Write three scrubbed-persona test CVs and JDs. Run audits. Fix any agent that produces nonsense verdicts on a non-AI domain.
4. **B-items in priority order.** B3 (router pre-aggregating applications_log) first because it unblocks two agents from UNKNOWN. Others as time permits.
5. **C-items as a sweep.** Whatever testing surfaces.

Estimated total cleanup work: 6-8 hours of focused effort. Could be one long session or split across two.

After this cleanup, V1.1 begins with a clean architecture: structured extraction, config-driven heuristics, domain-portable, cross-domain-validated, calibration-ready.

### E. Why we are not fixing this now

Three honest reasons logged for the record:

1. **Current state works for Santosh's profile and the interviewer demo.** Heuristics fit the data; verdicts are sensible; numbers are believable. Refactoring under demo time pressure introduces risk for no immediate user-visible benefit.
2. **End-to-end testing on Day 5/6 will surface MORE issues.** Doing the cleanup now means another cleanup pass after testing. Doing both together is more efficient.
3. **Day 5 documentation builds the launch artifact.** ROADMAP.md, README polish, V1.1 placeholders — these are best done with stable code beneath them, not while the architecture is shifting.

The cost of the decision is that V1 ships with hardcoded prompts and a fragile parse-many-times architecture. The mitigation is that this list exists, is on GitHub the moment SESSION_RESUME is committed, and the cleanup pass is scoped and ready.

---

## 10. How to update this file each session (pattern to preserve)

At the end of every meaningful work session:
1. Update the "Last updated" date at top.
2. Update section 6 ("What was done") with what got built.
3. Update section 7 ("Next step") to reflect the next file/task.
4. Update section 5 ("Vocabulary taught") if new terms were introduced.
5. Commit + push with a message like `Update SESSION_RESUME for <date>`.

This file is the persistent brain of the project across machines. Keep it honest and current.