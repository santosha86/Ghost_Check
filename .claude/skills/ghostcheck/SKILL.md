---
name: ghostcheck
description: Audit why a senior CV is being silenced on a specific JD. Runs eleven independent agents in isolated subagent contexts to surface invisible failure modes (bucket mismatch, IT-services discount, theater postings, channel mismatch, ATS issues, and more), then combines them into a callback probability with ranked silence drivers and evidence-cited fix hints. Invoke when the user runs /ghostcheck audit --cv <path> --jd <path>.
user_invocable: true
args:
  cv:
    argument-hint: path-to-cv-file
    description: Path to the user's CV. Accepts .md, .pdf, .docx. Optional. Defaults to profile/cv.md, falls back to profile/cv.example.md with a warning.
  jd:
    argument-hint: path-to-jd-file
    description: Path to the JD being audited. Accepts .md, .pdf, .docx, .pptx, .txt. Required.
allowed-tools:
  - Read
  - Bash
  - Skill
  - Agent
---

# /ghostcheck audit — Router skill (V1 Day-6 cleanup version)

When the user runs `/ghostcheck audit --cv <path> --jd <path>`, follow the nine steps below in order. Fail closed at every step on any error: surface a clear message and stop. Do not guess, do not produce a partial audit, do not hallucinate a verdict.

The major architectural change from prior versions: the router invokes the parser to extract STRUCTURED profiles (`CandidateProfile` and `JobProfile` per `docs/SCHEMAS.md` sections 15-16) instead of passing raw `cv_text` and `jd_text` to agents. Every agent now reads named fields from the structured profiles, declared in the agent's frontmatter `inputs:` list as dotted-path references (e.g. `candidate.recent_roles`, `target.seniority_keyword`).

## Files this audit loads (Sources of Truth)

| File | Purpose | Fallback behaviour |
|---|---|---|
| `<cv path>` (from --cv) | The CV being audited | If --cv omitted, look for `profile/cv.md`. If absent, fall back to `profile/cv.example.md` with a warning. |
| `<jd path>` (from --jd) | The JD being audited | Required, no fallback. Halt if missing. |
| `config/profile.yml` | UserProfile schema (target titles, locations, seniority, company types, channels) | Fall back to `config/profile.example.yml` with a warning. |
| `profile/style.md` | UserStyle (free-form constraints and narrative) | Fall back to `profile/style.example.md` with a warning. |
| `config/weights.yml` | Per-agent weights for the aggregator | Required. Halt if missing. |
| `config/known_firms.yml` | Domain-specific firm name lists used by `it-services-discount` and `company-classifier` | Required. Halt if missing. (V1 cleanup adds this file.) |

## Step 1 — Validate input file extensions (Level 1 validation)

For both CV and JD paths:

- Confirm the file exists. If not, halt: `"File not found at <path>. Halt audit."`
- Confirm extension is in the allowed list:
  - CV: `.md`, `.pdf`, `.docx`
  - JD: `.md`, `.pdf`, `.docx`, `.pptx`, `.txt`
- If extension is outside the list, halt: `"Unsupported file type for <CV|JD> at <path>. Halt audit."`

If `--cv` is omitted, look for `profile/cv.md`. If absent, fall back to `profile/cv.example.md` with: `"Warning: using profile/cv.example.md (Rohan Mehta example persona). Set up your real profile/cv.md for accurate audits."`

## Step 2 — Parse to structured profiles (CV plus JD)

Invoke the parser skill at `.claude/skills/parser/markitdown-parse.md` TWICE:

1. **First call** with `file_path = <cv path>` and `profile_type = "cv"`. The parser produces `{markdown_text: <CV markdown>, profile: <CandidateProfile object>}`. Hold the profile as `candidate`. Hold the markdown as `cv_text_fallback`.

2. **Second call** with `file_path = <jd path>` and `profile_type = "jd"`. The parser produces `{markdown_text: <JD markdown>, profile: <JobProfile object>}`. Hold the profile as `target`. Hold the markdown as `jd_text_fallback`.

If parsing fails for either, surface the error and halt.

## Step 3 — Validate document type (Level 3 validation)

Make a single LLM classification call using `cv_text_fallback` and `jd_text_fallback`:

```
Below are two documents. Classify each as CV, JD, or other. Be conservative.

Document 1 (claimed to be a CV — first 2000 characters):
{cv_text_fallback first 2000 chars}

Document 2 (claimed to be a JD — first 2000 characters):
{jd_text_fallback first 2000 chars}

Return JSON:
{
  "doc1_type": "cv" | "jd" | "other",
  "doc1_confidence": 0.0 to 1.0,
  "doc2_type": "cv" | "jd" | "other",
  "doc2_confidence": 0.0 to 1.0,
  "reason": "brief explanation"
}
```

If `doc1_type` not `"cv"` with confidence above 0.7, halt with classifier verdict and reason.
If `doc2_type` not `"jd"` with confidence above 0.7, halt with classifier verdict and reason.

## Step 4 — Load user-layer files

- `config/profile.yml` — parse as YAML, validate against `UserProfile` schema (`docs/SCHEMAS.md` section 8). If absent, fall back to `config/profile.example.yml` with a warning.
- `profile/style.md` — read as markdown. If absent, fall back to `profile/style.example.md` with a warning.

Hold these as `user_profile` (parsed object) and `user_style` (raw markdown string).

## Step 5 — Build ExternalContext

Construct the `ExternalContext` object per `docs/SCHEMAS.md` section 5:

- `jd_age_days`, `jd_age_source` — invoke `.claude/skills/enrichment/jd-age-detector.md` with `jd_text_fallback`.
- `company` — invoke `.claude/skills/enrichment/company-classifier.md` with `jd_text_fallback`. If no company name extractable, the skill returns a placeholder name with `company_type` inferred from JD body language (per Day-3 fix).
- `google_results` — invoke `.claude/skills/enrichment/google-test-lookup.md` with `candidate.name`. Returns the top 10 search results classified by surface type.
- `enrichment_errors` — collect any non-fatal enrichment failures.
- `cv_text_fallback`, `jd_text_fallback` — the raw markdown strings from Step 2 are also placed here for any agent that explicitly declares them as inputs.

Hold this as `external_context`.

## Step 6 — Pre-aggregate applications_log

Walk `applications/*/` looking for `outcome.md` and `channel.md` files. For each subfolder that has at least one of those:

- Parse `outcome.md` (one of `callback | screened | rejected | ghosted`, plus date and optional notes).
- Parse `channel.md` (one of the six allowed channel values, plus date and optional notes).

Build `applications_log` as a list of objects:

```yaml
applications_log:
  - slug: <folder-name>
    audit_date: <YYYY-MM-DD>
    outcome: <enum>
    outcome_date: <YYYY-MM-DD>
    channel: <enum>
    channel_notes: <string or null>
```

If no logged outcomes exist (typical first-time use), `applications_log` is an empty list. The Tier B agents (`funnel-math`, `channel-mix`) handle empty list by returning UNKNOWN.

## Step 7 — Dispatch to agents (Pattern B subagent isolation, structured-field inputs)

For each agent in the active list, build a per-agent input bundle by resolving the agent's frontmatter `inputs:` declarations against the audit context. Each declared input is a dotted-path reference (e.g. `candidate.recent_roles`, `target.title`, `external_context.google_results`, `user_profile.target_seniority`, `applications_log`). Resolve each path; assemble the bundle.

Then invoke the Agent tool with:

- `subagent_type`: agent's name (matches the filename of `.claude/agents/<name>.md` without `.md`).
- Prompt body: the per-agent input bundle as YAML, plus any agent-specific framing the prompt body of the subagent needs.

Each subagent runs in its own isolated context window (Pattern B blind fan-out — preserved by Claude Code's subagent runtime). Subagents do NOT see each other's verdicts.

Active agent list (eleven agents in V1):

- `bucket-classifier`
- `google-test`
- `posting-decoder`
- `it-services-discount`
- `headline-filter`
- `funnel-math`
- `channel-mix`
- `stale-detector`
- `ats-simulator`
- `recruiter-30sec`
- `hm-deep-read`

For each agent:

1. Resolve the agent's `inputs:` paths against `{candidate, target, external_context, user_profile, user_style, applications_log}`.
2. Build the input bundle.
3. Invoke Agent tool with `subagent_type = <agent-name>` and the bundle.
4. Wait for the subagent's verdict.
5. Validate against the `AgentVerdict` schema (`docs/SCHEMAS.md` section 1).
6. If validation fails, coerce to UNKNOWN per section 12.

Collect all validated verdicts.

## Step 8 — Aggregate verdicts

Invoke the aggregator skill at `.claude/skills/aggregator/aggregator.md` with:

- The list of validated `AgentVerdict` objects.
- The path to `config/weights.yml`.
- The full audit context: `candidate`, `target`, `user_profile`, `external_context`.

The aggregator computes the callback probability via weighted-logistic over non-UNKNOWN verdicts (handling N for not-yet-implemented agents and U for UNKNOWN per `docs/SCHEMAS.md` section 13), then writes:

- `applications/YYYY-MM-DD_<jd_slug>/audit.md`
- `applications/YYYY-MM-DD_<jd_slug>/verdicts.json`
- `applications/YYYY-MM-DD_<jd_slug>/context.json`

Slug derivation: take basename of JD path, strip extension, lowercase, replace non-alphanumeric with hyphen, collapse consecutive hyphens. Date prefix: today in `YYYY-MM-DD`. Slug-collision rule: append `_2`, `_3`, etc. per `docs/SCHEMAS.md` section 10.

## Step 9 — Report

Print the path to `applications/YYYY-MM-DD_<jd_slug>/audit.md`.

If any prior step failed, the audit folder may be partial or absent. Surface what worked, what failed, and how the user can recover.

## Behavioural rules (invariants)

- **Fail closed.** Any step that cannot complete cleanly halts the audit with a clear message.
- **No agent-to-agent communication.** Agents run isolated. The router never replays one agent's verdict to another.
- **The aggregator is the sole writer to `applications/`.** The router does not write audit artefacts directly.
- **The router never modifies user files.** `profile/cv.md`, `profile/style.md`, `config/profile.yml`, `config/weights.yml`, JD files — all read-only from the router's perspective.
- **Capability declarations are honoured.** If an agent's frontmatter declares no `web_search` capability, the router does not pass it web-search results.
- **Honour the `AgentVerdict` schema absolutely.** Any verdict that does not validate becomes UNKNOWN.
- **Structured-profile fields are the primary interface to agents.** Raw markdown text is fallback only; agents declare structured-field paths in `inputs:`.
