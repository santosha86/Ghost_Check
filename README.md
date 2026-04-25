# GhostCheck

**Audit why senior applications get silence — not rejection.**

Senior candidates apply to the right roles, tailor the CV, hit Submit, and then... nothing. Not a rejection. Silence. Existing CV checkers score keywords and ATS compatibility, but the real reasons for silence are usually invisible to them — wrong channel, no external proof (the Google test), bucket mismatch (your CV reads one seniority level below the target), stale postings, IT-services discount, theater postings.

GhostCheck simulates the entire screening funnel using 11 narrow, specialised agents. Each agent inspects one dimension of silence risk and produces a verdict in a fixed, evidence-cited shape. A deterministic aggregator combines those verdicts into a callback probability with ranked silence drivers and specific fix suggestions.

---

## What makes GhostCheck different

Every CV checker you've tried scores the **artifact**. GhostCheck audits the **funnel**, including the invisible failure modes no existing tool catches:

- **The Google test** — what a recruiter sees about you *before* they open your CV.
- **Bucket mismatch** — CVs that read one seniority level below the target role.
- **IT-services discount** — the silent downgrade triggered by tenure at large services firms.
- **Theater postings** — JDs that are already pre-filled and exist only for compliance.
- **Channel mismatch** — applying through Easy Apply for roles that only move via referral or DM.
- **Stale JDs** — roles that have been open so long the shortlist is already locked.

Each of the 11 agents has a narrow mandate, cites evidence from your CV / the JD / external context for every claim, and refuses to produce a verdict it can't back up (fail-closed).

---

## Quick Start — from zero to your first audit

### 1. Prerequisites

**Required for every audit:**

- [Claude Code](https://www.anthropic.com/claude-code) installed (desktop, web, or a supported IDE extension).
- An active Claude subscription or API key.
- `git` installed locally.

**Optional — only needed if your CV or JD is in DOCX, PPTX, or HTML format:**

- [MarkItDown](https://github.com/microsoft/markitdown) installed via `pip install markitdown`.

PDF and Markdown files work out of the box — Claude Code's native file reading handles them without any install. The MarkItDown CLI is only needed when you have a Word document, PowerPoint deck, or HTML page that needs to be converted to markdown before the agents can read it. If you have a DOCX or PPTX you can also convert it to PDF first (most operating systems have built-in export to PDF) and avoid the install entirely.

### 2. Get the repo

Fork it on GitHub (recommended — keeps your audits private in your own fork), then clone your fork:

```bash
git clone https://github.com/<your-username>/Ghost_Check.git
cd Ghost_Check
```

Or clone directly if you just want to try it out:

```bash
git clone https://github.com/santosha86/Ghost_Check.git
cd Ghost_Check
```

### 3. Open it in Claude Code

Open the `Ghost_Check` folder as a Claude Code workspace. This lets Claude Code see the `.claude/skills/` folder and register the `/ghostcheck` command.

### 4. Drop in your CV

Copy the example CV and replace it with yours:

```bash
cp profile/cv.example.md profile/cv.md
```

Then open `profile/cv.md` and paste your CV. If your CV is a PDF or DOCX, put the file in `profile/` and GhostCheck will parse it for you when you run the audit.

### 5. Set your target profile

Copy the example profile config and fill in your target titles, locations, and seniority level:

```bash
cp config/profile.example.yml config/profile.yml
```

This takes one minute — it tells the agents what "target role" actually means for you.

### 6. Add the job description you want audited

Save the JD as a markdown or text file anywhere in the repo — for example `jobs/acme-senior-ai.md`. Paste the full JD text in the body.

**Optional but recommended** — add a short YAML header at the top of the JD file so the audit can use company context without a web lookup:

```yaml
---
company: Acme Fintech
size: 250
hq: Dubai, UAE
industry: fintech
funding_stage: Series B
posted_date: 2026-03-15
source_url: https://acme.com/careers/senior-ai-architect
---

# Senior AI Architect

<rest of the JD as plain markdown>
```

Every field is optional. If the header is missing or partial, GhostCheck fills gaps via web search (only the agents that declare `web_search` in their capabilities can do this). If both sources fail, the agents that need company context return `UNKNOWN` instead of guessing — fail-closed.

### 7. Run the audit

In your Claude Code session:

```
/ghostcheck audit --cv profile/cv.md --jd jobs/acme-senior-ai.md
```

### 8. Read your result

Your audit lands at `applications/YYYY-MM-DD_acme-senior-ai/audit.md`. Open it.

### What you get back

A markdown report with a **0–100% callback probability score**, the **top silence drivers** for this specific CV+JD pair (ranked by severity — usually two to five), and an **evidence-cited fix suggestion** for each. Every finding links back to the exact line of your CV, the JD, or the external context that triggered it.

Treat the findings as **prioritised hypotheses to investigate**, not guarantees. They are diagnostic signals from pattern-matching agents, grounded in cited evidence — not verdicts from an oracle. Your judgement is still the final step.

---

## Runtime

GhostCheck V1 runs end-to-end inside Claude Code, which is why the Quick Start above uses Claude Code commands. That is what the `/ghostcheck` router and the skill files expect today.

The architecture, however, is deliberately runtime-portable:

- Every agent is a plain markdown file with YAML frontmatter declaring its inputs, outputs, and capabilities.
- Data contracts in `docs/SCHEMAS.md` are Pydantic-ready.
- The aggregator is deterministic math, not another LLM call.

That means a future Python service, a different orchestrator, or a different LLM provider can drive the same 11 agents without rewriting any of them. The reasoning behind this design — and why it is the actual moat of the project — is documented in `docs/HARNESS_ENGINEERING.md`. The bring-your-own-LLM path is on the roadmap under V1.2.

---

## The 11 agents at a glance

The full prompt, inputs, outputs, and severity model for each agent live in its own `.md` file under `.claude/skills/agents/`. Short version:

| ID  | Agent                  | What it checks                                                           |
|-----|------------------------|--------------------------------------------------------------------------|
| A1  | google-test            | What a recruiter sees about you when they Google you before your CV.     |
| A2  | posting-decoder        | Whether the JD is a genuine opening or a theater (pre-filled) posting.   |
| A3  | bucket-classifier      | Whether your CV reads at the target seniority or one level below.        |
| A4  | it-services-discount   | Whether tenure at large IT-services firms triggers a silent downgrade.   |
| A5  | headline-filter        | Whether your one-line identity passes the six-second recruiter filter.   |
| B1  | funnel-math            | Whether your application-to-callback rate is normal or broken.           |
| B2  | channel-mix            | Whether you are applying through the right channel for this seniority.  |
| B3  | stale-detector         | Whether you applied too long after the JD was posted.                    |
| C1  | ats-simulator          | Keyword coverage, years match, title match (classic ATS score).          |
| C2  | recruiter-30sec        | First-scan impression: companies, tenure, trajectory.                    |
| C3  | hm-deep-read           | Whether your bullets show decisions owned or activities performed.       |

*Note: the `A` / `B` / `C` prefix marks the tier (and is the filename prefix — e.g. `.claude/skills/agents/A3-bucket-classifier.md`). The value stored in every `AgentVerdict.agent_id` is the second column — e.g. `bucket-classifier`, not `A3`.*

Every agent returns the same shape (`AgentVerdict`) defined in `docs/SCHEMAS.md`. Agents run blind to each other — no agent sees another agent's output — which keeps the aggregator's input independent.

---

## Example output (illustrative)

> This example is illustrative only. The first real audit lands end of Day 2 of the build and will replace this block.

```markdown
# Audit — Senior AI Architect @ Acme
Date: 2026-04-22
Callback probability: 34%

## Top silence drivers
1. HIGH — bucket-classifier: CV reads as Staff, JD targets Director.
   Evidence: no owned P&L line, no org-size metric in last 3 roles.
   Fix: re-scope two bullets to decision ownership + org-size metric.

2. HIGH — google-test: No discoverable surface for this title.
   Evidence: top 10 Google results for your name return internal-only posts.
   Fix: publish one 600-word post on the agentic architecture you led.

3. MEDIUM — channel-mix: 80% of your applications are Easy Apply.
   Evidence: applications/ log shows 12 of 15 via Easy Apply.
   Fix: route next 3 applications through referral or direct DM.
```

---

## Architecture in one paragraph

GhostCheck is a **skill pack**, not a Python application. Every agent is a markdown file with YAML frontmatter, Claude Code handles orchestration, and the aggregator is deterministic math — not another LLM. Nothing calls out to a server, nothing holds state beyond the files in this repo, and every file is human-readable. The reason this matters, and why it is the real differentiator of the project, is covered in `docs/HARNESS_ENGINEERING.md`.

---

## Trust and privacy

Your CV never leaves your machine unless you push it to a remote yourself. Agents cannot call each other, cannot write to your files, and declare their capabilities in frontmatter (for example, only the `google-test` agent is allowed to call web search). Any agent that cannot back a verdict with cited evidence returns `UNKNOWN` rather than inventing one — this is called fail-closed.

The full security posture lives in `docs/ZERO_TRUST.md`.

---

## Roadmap

- **V1 (now)** — single-application audit: 11 agents, deterministic aggregator, markdown report.
- **V1.1** — batch pattern analysis (the killer feature): feed five or more applications with outcomes and GhostCheck finds your **personal silence signature** — the pattern across your ghosting, not per-application.
- **V1.2** — calibration loop with real outcomes, bring-your-own-LLM provider abstraction.
- **V2** — optional Python/FastAPI runtime for teams and coaches.

Full detail in `docs/ROADMAP.md`.

---

## License

MIT. See `LICENSE`.

Built on top of [MarkItDown](https://github.com/microsoft/markitdown) for CV and JD parsing.