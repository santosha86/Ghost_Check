---
name: jd-age-detector
description: Enrichment skill that infers how many days have passed since a JD was originally posted. Returns two fields — jd_age_days (integer or null) and jd_age_source (enum) — placed by the router into ExternalContext. Used downstream by the posting-decoder agent (theater-posting detection) and stale-detector agent (late-application flag). Pure text inference; no web calls, no external tools.
allowed-tools: []
---

# jd-age-detector — enrichment skill

When invoked by the /ghostcheck router with a `jd_text` string (the parsed JD as markdown), determine the JD's age in days and the source confidence-tag, then return both. Fail closed when no date signal exists: return `null` with `jd_age_source: "unknown"` rather than guessing.

## Inputs

The router passes one argument:

- `jd_text` — the parsed JD as markdown. Includes any YAML frontmatter the user provided at the top of the JD file, plus the JD body.

## Step 1 — Strategy 1: YAML frontmatter (`user_provided`, highest confidence)

Check whether `jd_text` starts with a YAML frontmatter block (delimited by `---` markers at the very top of the file). If yes, look for a `posted_date` field. The expected format is ISO 8601: `YYYY-MM-DD`.

Example frontmatter:

```yaml
---
company: Acme Fintech
posted_date: 2026-03-15
source_url: https://acme.com/careers/senior-ai-architect
---
```

If `posted_date` is present and parses as a valid ISO date:

- Compute `jd_age_days = today - posted_date` in calendar days.
- Set `jd_age_source = "user_provided"`.
- Return immediately. This is the highest-confidence signal.

## Step 2 — Strategy 2: text-pattern inference (`text_inferred`, medium confidence)

If frontmatter is absent or has no `posted_date`, scan the JD body for these patterns IN THIS ORDER. Stop at the first usable signal.

**Pattern A — relative phrases.** Phrases of the form *"posted N (days|weeks|months) ago"*, *"available for the past N months"*, *"listed N (days|weeks|months) back"*, *"X (day|week|month)s old"*. Convert the relative duration to days (week = 7 days, month = 30 days as approximation), then subtract from today.

**Pattern B — explicit posting dates in text.** Phrases of the form *"Posted: 15 March 2026"*, *"Listed on 2026-03-15"*, *"Originally posted on March 15, 2026"*, *"Date: 2026-03-15"*. Parse the date phrase to an ISO date, compute `today - that_date` in days.

**Pattern C — deadline-based weak inference.** Phrases of the form *"applications close on YYYY-MM-DD"*, *"deadline: YYYY-MM-DD"*, *"apply by YYYY-MM-DD"*. This does NOT directly give the posting date, but it constrains it:

- If the deadline is more than 30 days in the future, the JD was likely posted recently — set `jd_age_days` to a conservative estimate of 7.
- If the deadline is between today and 30 days in the future, the JD is mid-life — estimate `jd_age_days` as 21.
- If the deadline is in the past, the JD is stale — set `jd_age_days` to 60 as a conservative floor.
- Use cautiously; this is a weaker signal than patterns A and B.

If any of patterns A, B, or C yields a usable date or estimate:

- Set `jd_age_days` to the computed value.
- Set `jd_age_source = "text_inferred"`.
- Return.

## Step 3 — Strategy 3: URL-based inference (`url_inferred`) — skipped in V1

Some careers pages embed posting timestamps in their URL paths (e.g. `/careers/2026-03-15-senior-ai-architect/`). The `ExternalContext` schema reserves `url_inferred` as a valid `jd_age_source` value, but V1 does NOT implement this strategy because the pipeline does not reliably track JD URLs (they are not always supplied in the frontmatter). V1.1 adds URL inference when the company-enricher skill expands to include URL handling.

For V1, this strategy is a no-op.

## Step 4 — Default fallback (`unknown`)

If neither strategy 1 nor strategy 2 yields a usable date:

- Set `jd_age_days = null`.
- Set `jd_age_source = "unknown"`.
- Return.

The downstream agents (`posting-decoder`, `stale-detector`) read `jd_age_days = null` and respond by returning their own `UNKNOWN` verdicts. They do not guess.

## Output

Return a single JSON object:

```json
{
  "jd_age_days": 24,
  "jd_age_source": "user_provided"
}
```

`jd_age_days` is an integer or `null`. `jd_age_source` is exactly one of: `"user_provided"`, `"text_inferred"`, `"url_inferred"`, `"unknown"`.

The router places these into `ExternalContext.jd_age_days` and `ExternalContext.jd_age_source` per `docs/SCHEMAS.md` section 5.

## Edge cases the skill handles explicitly

- **Conflicting signals:** if frontmatter says one date and the body text says another (e.g. frontmatter `posted_date: 2026-03-01` but body says "posted last week"), frontmatter wins. Append an entry to `enrichment_errors` of the form `"jd-age-detector: conflicting date signals (frontmatter says X, text says Y); used frontmatter"`.
- **Future posting dates:** if the parsed date is later than today (e.g. "posted 2026-12-15" in an audit run on 2026-04-25), the date is malformed. Return `{jd_age_days: null, jd_age_source: "unknown"}` and log an enrichment error: `"jd-age-detector: posting date X is in the future; ignored"`.
- **Implausibly old dates:** if the parsed date is more than three years in the past (e.g. claims posted in 2021), still return the computed age but log a warning: `"jd-age-detector: posting date X is more than 3 years old; verify"`. Downstream agents may use this to flag a potentially mis-parsed date.
- **Multiple date signals in the body:** take the EARLIEST plausible date. The earliest matching date is most likely the original posting; later mentions are likely edits or deadlines.

## Behavioural rules

- **The skill makes no judgment about the age.** It only reports days and the source-tag. Judgment is done by `posting-decoder` and `stale-detector`, which read this output.
- **No web calls.** Pure text inference. The skill operates entirely on the `jd_text` argument it receives.
- **No silent guessing.** If no signal exists, return `null` with `jd_age_source: "unknown"`. Never fill in a default age like "30 days" when the signal is absent — that would corrupt downstream agents' reasoning.
- **Today's date** is the date of the audit run, not the date the user grabbed the JD. The router passes the audit date into `ExternalContext.audit_date` and the skill uses that as "today" for its math. (V1 minimal: if `audit_date` is not provided, the skill uses the current system date via Bash's `date` command — but `allowed-tools: []` blocks Bash, so the skill expects the router to pass the date explicitly. Document this contract in the agent prompt.)
