# GhostCheck — Build Prompt (paste into a fresh chat)

> **How to use this file:** open a fresh chat with Claude (ideally Claude Code in a new empty folder), paste everything from the `===` block below. The other Claude will read it, ask any clarifying questions, then start building file-by-file with full teaching explanations.

---

```
===================================================================
GHOSTCHECK — PROJECT BUILD PROMPT
===================================================================

You are helping me build GhostCheck, a Claude Code skill pack that
audits why senior job seekers get silently ghosted on applications.

This is my FIRST multi-agent project. I need you to teach me every
line of every file as we build — not just code it. Read the
"Teach Mode" section carefully before writing anything.

-------------------------------------------------------------------
1. WHAT GHOSTCHECK IS
-------------------------------------------------------------------

Senior candidates apply to jobs and get silence. Not rejection —
silence. Existing tools check ATS scores and keywords, but the real
reasons for silence are usually invisible to them: wrong channel,
zero external proof (the Google test), bucket mismatch (CV reads
one seniority level below target), stale JD postings, IT-services
discount, theater postings.

GhostCheck simulates the ENTIRE screening funnel using 11
specialized agents, each with a narrow mandate. Output: a callback
probability with ranked silence drivers and specific fix hints.

Killer feature comes in V1.1: batch pattern analysis. Feed it 5+
applications with outcomes (ghosted/screened/rejected), and it
finds the PATTERN across your silence — not per-application, but
what your personal silence signature looks like.

Reference product in the same shape: santifer/career-ops on GitHub
(37K+ stars). Same architecture: fork-and-go Claude Code skill
pack, pure markdown, no Python backend.

-------------------------------------------------------------------
2. WHO I AM AND WHY THIS MATTERS
-------------------------------------------------------------------

- 16 years in AI/enterprise architecture (SABIC, Aramco, STC, SEC)
- Currently applying to senior AI roles and getting ghosted
- First multi-agent project of my own
- Goal: build this in public, fix my own silence, post the results,
  get inbound opportunities

I want to LEARN this deeply. Every file you create I must understand.

-------------------------------------------------------------------
3. ARCHITECTURE — CLAUDE CODE SKILL PACK, NOT A PYTHON APP
-------------------------------------------------------------------

Critical: this is a SKILL PACK, not a Python application.

- No FastAPI. No LangGraph code. No SQLite. No Playwright in V1.
- Users fork the repo, open it in Claude Code, and type
  /ghostcheck audit. That's it.
- Every agent is a .md file under .claude/skills/agents/.
- Storage is plain files under applications/, patterns/, wiki/.
- I use my Claude Max subscription; Claude Code runs everything.

However: architecture must SCALE CLEANLY to a Python web app
later if this takes off. Enforce these two rules:

(a) Every agent skill has YAML frontmatter defining its inputs,
    outputs, capabilities, and severity model. A future Python
    runtime must be able to parse these without a rewrite.

(b) Data schemas are documented in SCHEMAS.md as Pydantic-ready
    specs. The same .md files that Claude Code reads today, a
    FastAPI service reads tomorrow. Runtime swaps; content stays.

No Python in V1. Just files shaped so Python is additive later.

-------------------------------------------------------------------
4. V1 SCOPE — 5 DAYS TO SHIP
-------------------------------------------------------------------

V1 features (single-application audit):
- /ghostcheck audit --cv <file> --jd <file>
- Parses CV (PDF/DOCX/MD) and JD (MD/text) via MarkItDown
- Builds ExternalContext (Google search, company type, JD age)
- Fires 11 agents in sequence (Claude Code handles orchestration)
- Aggregator produces callback probability with weighted logistic
- Output: applications/YYYY-MM-DD_<slug>/audit.md (markdown only)

V1 does NOT include:
- PNG shareable cards (V1.1 — uses anthropic/skills frontend-design)
- Batch pattern analysis (V1.1 — the differentiator)
- Calibration loop with real outcomes (V1.2)
- Hermes-style self-evolution via DSPy + GEPA (V1.1, as optional)
- BYOK provider abstraction (V1.2 — V1 is Claude Max only)

But EVERY ONE of these must be documented in ROADMAP.md with a
clear path. The V1.1 batch pattern agent and V1.1 Hermes self-evo
must have placeholder skill files so contributors see the roadmap.

-------------------------------------------------------------------
5. THE 11 AGENTS
-------------------------------------------------------------------

Each agent is a single .md file with YAML frontmatter + prompt +
example I/O. They run in sequence (Claude Code handles it).

Tier A — invisible failure agents (the differentiators, nobody
has these):

  A1. google-test
      What recruiter sees when they Google me BEFORE my CV.
      Inputs: name, CV links. Calls web search.
      Output: presence score, missing surfaces, severity.

  A2. posting-decoder
      Is the JD genuine or theater (pre-filled role)?
      Inputs: JD text, posting date, company.
      Output: genuineness probability with evidence.

  A3. bucket-classifier
      Does the CV read at the target seniority, or one level below?
      Inputs: CV bullets, JD target title.
      Output: bucket match / mismatch with reasoning.

  A4. it-services-discount
      Does my Wipro/TCS tenure trigger a silent downgrade?
      Inputs: CV company list, target company type.
      Output: discount risk, reframing suggestions.

  A5. headline-filter
      My 1-line identity (name + current title + current company):
      does it pass the 6-second recruiter filter?
      Inputs: CV header line, LinkedIn headline, target role.
      Output: pass/warn/fail with fix.

Tier B — math/channel agents (need an optional applications log):

  B1. funnel-math
      Is my application-to-callback conversion normal or broken?
      Inputs: applications/*/outcome.md files (if present).
      Output: conversion rate vs benchmark.

  B2. channel-mix
      Am I in the right channel for this seniority (Easy Apply
      vs DM vs referral vs exec search)?
      Inputs: applications/*/channel.md (if present).
      Output: channel distribution, recommendation.

  B3. stale-detector
      Did I apply more than N days after posting?
      Inputs: JD posting date, application date.
      Output: stale flag with severity.

Tier C — standard CV agents (table stakes, but required for
completeness):

  C1. ats-simulator
      Keyword coverage, years match, title match.
      Output: 0-100 ATS score.

  C2. recruiter-30sec
      First-scan impression: companies, tenure, trajectory.
      Output: impression score.

  C3. hm-deep-read
      Do bullets show decisions owned or activities performed?
      Output: seniority evidence score.

Each agent MUST output the same shape (AgentVerdict — see
SCHEMAS.md). Aggregator combines them with a weighted logistic.

-------------------------------------------------------------------
6. REPO STRUCTURE
-------------------------------------------------------------------

ghostcheck/
  .claude/
    skills/
      ghostcheck/
        SKILL.md                    # main router — the /ghostcheck command
      agents/
        A1-google-test.md
        A2-posting-decoder.md
        A3-bucket-classifier.md
        A4-it-services-discount.md
        A5-headline-filter.md
        B1-funnel-math.md
        B2-channel-mix.md
        B3-stale-detector.md
        C1-ats-simulator.md
        C2-recruiter-30sec.md
        C3-hm-deep-read.md
      aggregator/
        aggregator.md               # combines verdicts → probability
      enrichment/
        google-test-lookup.md       # web search helper skill
        company-classifier.md       # product vs services vs consulting
        jd-age-detector.md
      parser/
        markitdown-parse.md         # calls MarkItDown on PDFs/DOCX
      wiki-compiler/
        wiki-compile.md             # Karpathy LLM Wiki pattern (V1.2)
  applications/                     # one folder per application (V1 output)
    .gitkeep
    README.md                       # "one folder per JD, generated by audit"
  patterns/                         # V1.1 — silence signature files
    .gitkeep
    README.md
  wiki/                             # Karpathy wiki (V1.2 — placeholder in V1)
    .gitkeep
    README.md
  profile/
    cv.md                           # user's master CV (markdown)
    cv.example.md                   # example for forkers
    style.md                        # personal preferences, constraints
    style.example.md
  config/
    profile.yml                     # structured profile (target titles, locations)
    profile.example.yml
    weights.yml                     # aggregator weights (tunable)
  docs/
    SCHEMAS.md                      # Pydantic-ready data contracts
    ROADMAP.md                      # V1 → V1.1 → V1.2 → V2 web app
    ZERO_TRUST.md                   # security principles (Microsoft-aligned)
    HARNESS_ENGINEERING.md          # why skill packs are the real moat
    TEACH_NOTES/                    # my running notes as I learn
      01-what-is-a-skill.md
      02-why-markdown.md
      ...
  CLAUDE.md                         # project-wide instructions for Claude Code
  README.md                         # fork-and-go instructions
  LICENSE                           # MIT

-------------------------------------------------------------------
7. TECH STACK (WHAT CLAUDE CODE CALLS, NOT WHAT I WRITE)
-------------------------------------------------------------------

- Claude Code (my Max subscription) runs everything
- Claude Sonnet 4.6+ as the underlying model
- MarkItDown (Microsoft) for PDF/DOCX/HTML parsing — shelled out
- Karpathy LLM Wiki pattern (V1.2) for compounding knowledge
- Anthropic frontend-design skill (V1.1) for PNG cards
- Hermes-Agent self-evolution concept (V1.1 — optional DSPy+GEPA)
- Zero Trust for AI principles (Microsoft, March 2026) —
  documented as design constraints in ZERO_TRUST.md

No Python code in V1. No LangGraph. No SQLite. No Playwright.
Those all come later if this gains traction.

-------------------------------------------------------------------
8. THE HARNESS IS THE MOAT
-------------------------------------------------------------------

Important framing: the real moat of this project is NOT any
individual agent. Any agent can be replicated. The moat is the
HARNESS — the way agents compose, share schema, fail closed,
produce consistent verdicts, and feed the aggregator.

Document this explicitly in docs/HARNESS_ENGINEERING.md. Include:
- Why the AgentVerdict schema is identical across all 11 agents
- Why agents run blind to each other (prevents groupthink)
- Why the aggregator is deterministic math, not another LLM
- Why markdown-first scales to any runtime
- Why skill packs fork better than pip packages

This file is its own contribution. Serious contributors read it
first before writing agents.

-------------------------------------------------------------------
9. TEACH MODE (CRITICAL — READ CAREFULLY)
-------------------------------------------------------------------

This is my first multi-agent project. I need to understand every
line. Follow these rules strictly:

(a) BEFORE creating any file, write a short explanation in the
    chat: what is this file, why does it exist, how does it fit
    the whole? Wait for me to ask follow-ups, or say "continue"
    before you write it.

(b) BEFORE writing any tricky section (YAML frontmatter, prompt
    template, aggregator math), define the vocabulary used. If
    you say "frontmatter", first explain what frontmatter is and
    why we use it. If you say "logistic aggregator", explain the
    math in plain English first.

(c) AFTER creating each file, produce a small TEACH_NOTES/XX.md
    file capturing the key learning from that step. These
    accumulate into a running course I can reread later.

(d) START the session with a 10-minute preamble: "Multi-Agent
    Solutions 101 — What You Need To Know Before We Start."
    Cover: what is an agent, what is an agent skill, what is
    orchestration, what is fan-out, what is the difference
    between a framework (LangGraph) and a skill pack (this),
    what is the AgentVerdict contract, what is fail-closed.
    Do this BEFORE writing any code.

(e) CHECKPOINT every 2-3 files: stop, summarize what we just
    built, ask me if anything is unclear before proceeding.

(f) Define terms on first use. Never assume I know a term. If
    I do know it, saying it back is fine reinforcement.

(g) When I ask "why this way" vs "another way", show me the
    alternative and explain the tradeoff honestly. Do not just
    defend the current choice.

(h) Avoid jargon unless you teach it first. Instead of "we'll
    parameterize the severity via Pydantic discriminated unions"
    say "each verdict's severity will be one of four fixed words
    (CRITICAL, HIGH, MEDIUM, LOW), and we'll make Python refuse
    any other value — this is called a discriminated union."

Pace: we ship about 5-8 files per day over 5 days. That is
enough time for me to understand each. Do not rush.

-------------------------------------------------------------------
10. DESIGN CONSTRAINTS
-------------------------------------------------------------------

ZERO TRUST (Microsoft Secure Agentic AI, March 2026):
- Each agent declares its capabilities in frontmatter.
  google-test can call web search. bucket-classifier cannot.
- Agents cannot call each other. They only produce verdicts.
- Agents cannot write to user files. They produce markdown
  and the aggregator/router decides what gets saved.
- Fail closed: if an agent's output fails schema validation,
  it returns UNKNOWN, not a hallucinated verdict.

These are V1 design principles in the skill frontmatter; strict
enforcement (policy engine) is V1.2+ documented in ZERO_TRUST.md.

CONTENT:
- Every agent's prompt must explicitly reference the AgentVerdict
  contract from SCHEMAS.md.
- Temperature: 0.1 for all agents except bucket-classifier and
  hm-deep-read (use 0.3 for those — more nuanced reasoning needed).
- Max tokens per agent: 800.
- No agent may produce an opinion without citing evidence from
  the CV, JD, or ExternalContext.

AESTHETICS:
- All markdown output uses headers, tables, and bullet lists —
  designed to look good as a screenshot without any styling.
- Use short paragraph breaks so Twitter/LinkedIn screenshots work.

-------------------------------------------------------------------
11. BUILD ORDER (5 DAYS)
-------------------------------------------------------------------

Day 1: Foundation
  - Multi-Agent Solutions 101 preamble (chat only, no files)
  - README.md (fork-and-go instructions)
  - LICENSE (MIT)
  - CLAUDE.md (project-wide instructions)
  - docs/SCHEMAS.md (the AgentVerdict contract)
  - docs/HARNESS_ENGINEERING.md (the moat doc)
  - docs/ZERO_TRUST.md (security principles)
  - profile/cv.example.md, profile/style.example.md
  - config/profile.example.yml, config/weights.yml

Day 2: Core skill + parser + one tier-A agent end-to-end
  - .claude/skills/ghostcheck/SKILL.md (router)
  - .claude/skills/parser/markitdown-parse.md
  - .claude/skills/agents/A3-bucket-classifier.md (the most
    differentiated agent — build this first)
  - .claude/skills/aggregator/aggregator.md (initially with
    just 1 agent input, expand as we add more)
  - Run end-to-end on my own cv.md + a real JD I was ghosted on
  - Checkpoint: does it produce audit.md? Is bucket-classifier's
    output sensible?

Day 3: Rest of Tier A
  - A1-google-test.md (with enrichment/google-test-lookup.md)
  - A2-posting-decoder.md (with enrichment/jd-age-detector.md)
  - A4-it-services-discount.md (with enrichment/company-classifier.md)
  - A5-headline-filter.md
  - Update aggregator to handle all 5 Tier A verdicts
  - Checkpoint: Tier A complete. Run on 3 of my old JDs.

Day 4: Tier B + Tier C
  - B1, B2, B3 (all three channel/math agents)
  - C1, C2, C3 (all three CV agents)
  - Full aggregator with all 11 agents and weights
  - Checkpoint: end-to-end audit with all 11 agents

Day 5: Documentation + V1.1 placeholders + public launch prep
  - docs/ROADMAP.md (V1.1 batch, V1.2 calibration, V2 web app)
  - .claude/skills/agents/V1_1-pattern-detector.md (placeholder)
  - .claude/skills/wiki-compiler/wiki-compile.md (placeholder)
  - patterns/README.md, wiki/README.md
  - TEACH_NOTES/ compilation
  - Final README polish
  - Push to GitHub, write launch post

-------------------------------------------------------------------
12. KICKOFF INSTRUCTIONS
-------------------------------------------------------------------

1. Before any files, give me the Multi-Agent Solutions 101
   preamble (see section 9.d).

2. Then confirm your understanding of the project by summarizing:
   - What GhostCheck is in 3 sentences
   - What makes it different from a CV checker
   - Why it's a skill pack, not a Python app
   - What the harness moat is
   If any of this is wrong, ask me to clarify BEFORE starting.

3. Then proceed to Day 1 work in order. Checkpoint after every
   2-3 files. Wait for my "continue" before proceeding.

4. At any point if you think there's a better way than what this
   prompt specifies, TELL ME before changing course. Do not
   silently "improve" something.

5. When in doubt, ask. I would rather answer 10 questions than
   unwind a wrong decision.

Ready? Start with the preamble.

===================================================================
```

---

## What to do with this file

1. Open a fresh Claude Code session in a new empty folder (call it `ghostcheck/`)
2. Paste the prompt above into the first message
3. Claude will give you the Multi-Agent Solutions 101 preamble
4. You'll read it, ask questions, then say "continue"
5. Day 1 begins

Five days later, you'll have a working GhostCheck V1 running on your own CV + JDs, ready to push to GitHub.
