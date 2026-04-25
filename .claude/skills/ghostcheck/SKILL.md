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

# /ghostcheck audit — Router skill

When the user runs `/ghostcheck audit --cv <path> --jd <path>`, follow the eight steps below in order. Fail closed at every step on any error: surface a clear message and stop. Do not guess, do not produce a partial audit, do not hallucinate a verdict.

## Files this audit loads (Sources of Truth)

| File | Purpose | Fallback behaviour |
|---|---|---|
| `<cv path>` (from --cv) | The CV being audited | If --cv is omitted, look for `profile/cv.md`. If absent, fall back to `profile/cv.example.md` with a warning. |
| `<jd path>` (from --jd) | The JD being audited | Required, no fallback. Halt if missing. |
| `config/profile.yml` | UserProfile schema (target titles, locations, seniority, company types, channels) | Fall back to `config/profile.example.yml` with a warning. |
| `profile/style.md` | UserStyle (free-form constraints and narrative) | Fall back to `profile/style.example.md` with a warning. |
| `config/weights.yml` | Per-agent weights for the aggregator (sum = 1.0) | Required, no fallback. Halt if missing. |

The router reads each user-layer file once and bundles the contents per the Path 1 (Bundle-and-dispatch) pattern. Each agent receives only the inputs declared in its own frontmatter.

## Step 1 — Validate input file extensions (Level 1 validation)

For both CV and JD paths:

- Confirm the file exists at the given path. If not, halt: `"File not found at <path>. Halt audit."`
- Confirm the extension is in the allowed list:
  - CV allowed: `.md`, `.pdf`, `.docx`
  - JD allowed: `.md`, `.pdf`, `.docx`, `.pptx`, `.txt`
- If the extension is outside the list, halt: `"Unsupported file type for <CV|JD> at <path>. Halt audit."`

If `--cv` is omitted, look for `profile/cv.md`. If that does not exist, fall back to `profile/cv.example.md` and print: `"Warning: using profile/cv.example.md (Rohan Mehta example persona). Set up your real profile/cv.md for accurate audits."`

## Step 2 — Parse to markdown

For any non-markdown input (`.pdf`, `.docx`, `.pptx`, `.txt`), invoke the parser skill at `.claude/skills/parser/markitdown-parse.md` to convert the file to markdown text. If the input is already `.md`, read it directly with the Read tool.

The result of this step is two strings: `cv_text` and `jd_text`. Both are markdown.

If parsing fails for either, surface the parser's error and halt: `"Parser failed for <CV|JD> at <path>: <error>. Halt audit."`

## Step 3 — Validate document type (Level 3 validation)

Make a single LLM classification call with the following prompt:

```
Below are two documents. Classify each as CV, JD, or other. Be conservative — if a document looks ambiguous or could plausibly be the wrong type, return "other".

Document 1 (claimed to be a CV — first 2000 characters):
{cv_text first 2000 chars}

Document 2 (claimed to be a JD — first 2000 characters):
{jd_text first 2000 chars}

Return JSON exactly in this shape:
{
  "doc1_type": "cv" | "jd" | "other",
  "doc1_confidence": 0.0 to 1.0,
  "doc2_type": "cv" | "jd" | "other",
  "doc2_confidence": 0.0 to 1.0,
  "reason": "one-line explanation of why the verdicts are what they are"
}
```

If `doc1_type` is not `"cv"` with confidence above 0.7, halt: `"Document at <cv path> does not appear to be a CV (classifier verdict: <doc1_type>, confidence <doc1_confidence>, reason: <reason>). Halt audit."`

If `doc2_type` is not `"jd"` with confidence above 0.7, halt: `"Document at <jd path> does not appear to be a JD (classifier verdict: <doc2_type>, confidence <doc2_confidence>, reason: <reason>). Halt audit."`

## Step 4 — Load user-layer files

Read these files in order:

- `config/profile.yml` — parse as YAML, validate against the `UserProfile` schema in `docs/SCHEMAS.md` section 8. Required fields: `target_titles`, `target_locations`, `target_seniority`. If file missing, fall back to `config/profile.example.yml` and print `"Warning: using config/profile.example.yml (Rohan Mehta example targets). Set up your real config/profile.yml for accurate audits."`
- `profile/style.md` — read as markdown. If file missing, fall back to `profile/style.example.md` and print the equivalent warning.

Hold these as `user_profile` (parsed object) and `user_style` (raw markdown string).

## Step 5 — Build ExternalContext (V1 minimal)

Construct the `ExternalContext` object per `docs/SCHEMAS.md` section 5.

V1 minimal population:

- `jd_age_days`: parse from JD frontmatter `posted_date` if present; compute days from today. If absent, set to `null`.
- `jd_age_source`: `"user_provided"` if the JD frontmatter had `posted_date`, else `"unknown"`.
- `google_results`: empty list. (V1.1 will wire up the `google-test-lookup` enrichment skill.)
- `enrichment_errors`: empty list.
- `company`: `null`. (V1.1 will wire up the `company-enricher` skill that builds the hybrid Company object from JD frontmatter and web search.)

Hold this as `external_context`.

## Step 6 — Dispatch to agents (Pattern B: subagent isolation)

For each agent in the active list below, invoke the Agent tool with:

- `subagent_type`: the agent's name (matches the filename of the subagent file under `.claude/agents/`, without `.md`)
- Prompt body: the input bundle for that agent — only the inputs the agent declares in its YAML frontmatter `inputs:` field.

The agent's declared inputs are drawn from this fixed set: `cv_text`, `jd_text`, `user_profile`, `user_style`, `external_context`. Agents that do not need a particular input omit it from their frontmatter; the router passes only declared inputs.

Each agent runs in its own isolated context window. Agents must NOT see other agents' verdicts. This is the blind fan-out guarantee — preserved natively by Claude Code's subagent runtime when subagents are dispatched via the Agent tool.

**Active agent list (V1 starting state — only one agent exists; Days 3 and 4 add the rest):**

- `bucket-classifier`

For each agent:

1. Build the input bundle per the agent's frontmatter `inputs:` declaration.
2. Invoke the Agent tool with `subagent_type` set to the agent's name and the input bundle as the prompt.
3. Wait for the subagent to return its `AgentVerdict`.
4. Validate the returned verdict against the `AgentVerdict` schema in `docs/SCHEMAS.md` section 1.
5. If validation fails, coerce the verdict to UNKNOWN per `docs/SCHEMAS.md` section 12, with `unknown_reason: "schema_violation: <specific reason>"`.

Collect all validated verdicts into a list.

## Step 7 — Aggregate verdicts

Invoke the aggregator skill at `.claude/skills/aggregator/aggregator.md` with:

- The list of validated `AgentVerdict` objects from step 6.
- The path to `config/weights.yml` (the aggregator reads this to apply per-agent weights).
- The full audit context bundle: `cv_text`, `jd_text`, `user_profile`, `external_context`.

The aggregator computes the callback probability via weighted-logistic over the verdicts (handling UNKNOWN renormalisation per `docs/SCHEMAS.md` section 13), then writes:

- `applications/YYYY-MM-DD_<jd_slug>/audit.md` — the human-readable audit report
- `applications/YYYY-MM-DD_<jd_slug>/verdicts.json` — the raw `AgentVerdict` objects
- `applications/YYYY-MM-DD_<jd_slug>/context.json` — the `ExternalContext` used for this run

The slug derives from the JD filename: take the basename, strip the extension, lowercase, and replace any non-alphanumeric character with a hyphen. Example: `Strategy AI Architect.pptx` becomes `strategy-ai-architect`. The date prefix is today's date in ISO format (`YYYY-MM-DD`). If the resulting folder already exists (re-running an audit on the same JD on the same day), append `_2`, `_3`, etc., per `docs/SCHEMAS.md` section 10's slug-collision rule.

## Step 8 — Report to the user

Print the path to `applications/YYYY-MM-DD_<jd_slug>/audit.md` so the user can open it.

If any step before the aggregator failed, the audit folder may be absent or partial. In that case, surface what worked, what failed, and how the user can recover (e.g., fix the input file, set up `profile/cv.md`, install MarkItDown if the parser failed because of a missing dependency).

## Behavioural rules (invariants the router must respect)

- **Fail closed.** Any step that cannot complete cleanly halts the audit with a clear message. There are no partial audits in V1.
- **No agent-to-agent communication.** Agents run isolated. The router never replays one agent's verdict to another.
- **The aggregator is the sole writer to `applications/`.** The router does not write audit artefacts directly.
- **The router never modifies `profile/cv.md`, `profile/style.md`, `config/profile.yml`, `config/weights.yml`, or any JD file.** Those are user-owned inputs.
- **Capability declarations are honoured.** If an agent's frontmatter declares no `web_search` capability, the router does not pass it web-search results — even if the input bundle would technically contain them.
- **Honour the `AgentVerdict` schema absolutely.** Any verdict that does not validate becomes UNKNOWN; never patched, never repaired.
