---
name: google-test-lookup
description: Enrichment skill that runs the "Google test" — searches for the candidate's name on the public web and returns the top ten results classified by surface type (LinkedIn, GitHub, personal site, company, press, aggregator, other). Invoked by the /ghostcheck router during Step 5 (Build ExternalContext), before any agent reads the bundle. Provides the structured data the google-test agent reasons over.
allowed-tools:
  - WebSearch
---

# google-test-lookup — enrichment skill

When invoked by the /ghostcheck router with a `candidate_name` string, run a single web search on the candidate's name, parse the top ten results into the `GoogleResult` shape defined in `docs/SCHEMAS.md` section 7, classify each result by surface type, and return the list. Fail closed on any error: surface a clear message and return an empty list with an entry in `enrichment_errors` so the router can decide what to do.

## Inputs

The router passes a single argument:

- `candidate_name` — the full name of the candidate as a string. The router extracts it from the CV's H1 heading (the first `# <name>` line in the parsed markdown).

## Step 1 — Build the search query

Construct a single search query: the candidate's full name in quotation marks.

Example: `"Achanta Santosh Kumar"`.

The quotation marks force exact-phrase matching. Without them, Google can return results for partial-name matches (other people sharing a first or last name), which is noise for the google-test agent. With them, results are tightly scoped to people with this exact name.

V1 minimal stops here. V1.1 may add additional queries (name plus current employer, name plus role) to surface different signals; for V1, one focused name search is enough.

## Step 2 — Execute the WebSearch call

Use Claude Code's WebSearch tool with the constructed query. Cap results at ten — this is the typical recruiter scan depth before they either click through or move on.

If WebSearch returns fewer than ten results, take all of them. If it returns more than ten (rare with the cap), take the first ten by rank.

If WebSearch fails (no internet, rate limit, service error), halt the skill: return an empty list to the caller and surface a single string entry into the audit's `enrichment_errors` list of the form `"google-test-lookup failed: <error message>"`. The router puts an empty `google_results` into ExternalContext, and the google-test agent treats empty results as its own signal (typically severity HIGH or CRITICAL — absence of online surface is itself a finding).

## Step 3 — Parse each result into the GoogleResult shape

For each result the WebSearch tool returns, extract:

- `rank` — 1-indexed position in the results.
- `title` — the result's title as returned by the search engine.
- `url` — the canonical URL.
- `snippet` — the search-engine-provided excerpt of the page content.
- `domain` — extract the hostname from the URL (the part between `https://` and the next `/`). Lowercase it. Strip a leading `www.` if present.

These five fields are required by the `GoogleResult` shape (see `docs/SCHEMAS.md` section 7).

## Step 4 — Classify each result by surface

For each result, set the `surface` field to one of the seven enum values defined in the schema, by applying these heuristics in this order. The first matching rule wins.

| If the domain matches | Set surface to |
|---|---|
| `linkedin.com` (any subdomain or path) | `linkedin` |
| `github.com` or `*.github.io` | `github` |
| `medium.com`, `substack.com`, `*.medium.com`, `*.substack.com` | `personal_site` |
| Known personal-blog patterns: `*.dev`, `*.me`, `*.io` for individual people (heuristic: if the domain is short, like 2-3 segments, and contains the candidate's first or last name) | `personal_site` |
| Known press / publication domains: `techcrunch.com`, `forbes.com`, `bloomberg.com`, `wsj.com`, `economictimes.indiatimes.com`, `livemint.com`, `arabnews.com`, `gulfbusiness.com`, and similar industry-press domains | `press` |
| Known aggregator / scraping domains: `indeed.com`, `glassdoor.com`, `signalhire.com`, `contactout.com`, `rocketreach.co`, `apollo.io`, `zoominfo.com`, `lusha.com`, `findpeoplefast.net`, and similar people-data brokers | `aggregator` |
| Anything else | `other` |

The `company` surface value (per the schema) is reserved for results pointing at the candidate's own current or past employer's website (e.g. the candidate's bio on the firm's "About us" page). V1 does NOT populate `company` because doing so requires knowing the candidate's employer list, which lives in the CV. V1.1 can add this when the company-classifier enrichment skill provides the employer-domain list. For V1, employer-bio pages classify as `other`.

## Step 5 — Return the structured list

Return a list of `GoogleResult` objects to the router. Order by `rank` ascending (matches the order the search engine returned them).

The router appends this list to `ExternalContext.google_results` per `docs/SCHEMAS.md` section 5.

## Behavioural rules

- **The skill does not judge the results.** Severity, score, verdict — all of that is the `google-test` agent's job. This skill only fetches and classifies.
- **No synthesis.** If WebSearch returns no results, return an empty list. Do not invent results to fill the slate.
- **No caching in V1.** Each audit makes a fresh web search. If audit speed becomes an issue, V1.1 can add a per-name cache with a sensible TTL (e.g. 24 hours).
- **No deep crawling.** The skill captures what the search engine returns: title, snippet, URL, domain. It does NOT click into each result and parse the page content. The agent reasons over the surface-level signal alone, which is what a recruiter scanning the SERP also sees.
- **Privacy-respecting.** The search query is the candidate's own name, which the candidate has already published in their own CV. The skill does not search for sensitive personal data (DOB, address, family). If a future feature needs deeper search, it must be opt-in via a separate skill with its own capability declaration.
