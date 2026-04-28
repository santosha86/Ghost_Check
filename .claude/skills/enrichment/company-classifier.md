---
name: company-classifier
description: Enrichment skill that builds the Company object in ExternalContext (per docs/SCHEMAS.md section 6). Classifies the hiring company by company_type (product, services, consulting, bigtech, startup, other, unknown) and fills supporting fields (name, size, hq, industry, funding_stage). Combines user-provided data from JD frontmatter with public web search to fill gaps. Tracks per-field provenance via the sources field. Used downstream by it-services-discount (Day 3) and channel-mix (Day 4).
allowed-tools:
  - WebSearch
---

# company-classifier — enrichment skill

When invoked by the /ghostcheck router with `jd_text` (the parsed JD as markdown), construct a `Company` object that conforms to the schema in `docs/SCHEMAS.md` section 6, with per-field provenance tracking via the `sources` map. Fail closed on missing data: any field that cannot be confidently filled is set to `null` (or to `"unknown"` for the `company_type` enum), with the `sources` entry tagging the gap.

## Inputs

- `jd_text` — the parsed JD as markdown. Includes any YAML frontmatter the user provided at the top of the JD file.

## Step 1 — Extract the company name

The `name` field is required if the `Company` object is to exist at all. Extraction sequence:

1. **Frontmatter:** look for `company: <Name>` in the JD's YAML frontmatter. If present, use it. Tag `sources.name = "user_provided"`.
2. **First heading or title block:** if no frontmatter, scan the JD body for the company name in the first heading (e.g. `# Acme Fintech — Senior AI Architect`) or in a "Company:" / "About us:" section near the top. Tag `sources.name = "text_inferred"` (note: the schema's enum for SourceTag is `user_provided | web_search | default`, so `text_inferred` is not allowed — fall back to `user_provided` if the text is structured like a frontmatter field, or skip the company object entirely if extraction is fuzzy).
3. **No real name found:** set `name = "[unspecified hiring firm — name not in JD]"` and tag `sources.name = "default"`. Continue to subsequent steps so `company_type` can still be inferred from JD body language. Many JDs (especially recruiter-distributed PowerPoint decks and consulting-firm postings) refer to the hiring firm only as "the firm" or "our firm" without ever naming it; in those cases the candidate still needs a verdict on company-type-driven discounts and channel choices, and the JD body usually has more than enough signal to classify the company-type even when the specific name is missing. The downstream agents (`it-services-discount`, `channel-mix`) check `company_type` directly and reason about target type without needing the specific company name.

Only if BOTH the name is unextractable AND the JD body has no usable company-type signals (Heuristic D in Step 3 returns nothing) should the entire Company object be `null`. The simple rule is: if we can fill in `company_type`, the Company object exists with that field plus the placeholder name; if even `company_type` cannot be inferred, return `null`.

## Step 2 — Read user-provided fields from frontmatter

If the JD has frontmatter, look for these fields and lift them into the Company object directly:

- `company` → `name` (already done in Step 1).
- `size` → `size` (e.g. `"200-500"` or `250`).
- `hq` → `hq` (e.g. `"Dubai, UAE"`).
- `industry` → `industry` (e.g. `"fintech"`).
- `funding_stage` → `funding_stage` (e.g. `"Series B"`).
- `company_type` → `company_type` (must be one of the enum values).

For each field actually present in frontmatter, tag `sources.<field> = "user_provided"`.

## Step 3 — Classify company_type via known-firm heuristics

If `company_type` was NOT user-provided, attempt classification via these heuristics, in this order:

**Heuristic A — known IT-services firms** (set `company_type = "services"`, tag `sources.company_type = "default"`):

Wipro, Tata Consultancy Services / TCS, Infosys, HCL Technologies / HCLTech, Cognizant, Tech Mahindra, Mphasis, L&T Infotech / LTI / LTIMindtree, Mindtree, Zensar, Hexaware, NIIT Technologies, Cyient, Persistent Systems, Mastek, Birlasoft, Sonata Software, Coforge / NIIT Technologies, KPIT, Hitachi Vantara, DXC Technology, Capgemini, Atos, Sopra Steria, NTT DATA, Fujitsu, Unisys, Cognizant, IBM Consulting (note: IBM's consulting arm is services, IBM Research is bigtech).

**Heuristic B — known consulting firms** (set `company_type = "consulting"`, tag `sources.company_type = "default"`):

McKinsey, BCG, Bain, Oliver Wyman, AT Kearney / Kearney, LEK, Roland Berger, Accenture, Deloitte, PwC / PricewaterhouseCoopers, EY / Ernst & Young, KPMG, Booz Allen Hamilton, Strategy& (PwC's strategy arm), Monitor Deloitte, ZS Associates, Parthenon-EY.

**Heuristic C — known big-tech firms** (set `company_type = "bigtech"`, tag `sources.company_type = "default"`):

Google / Alphabet, Meta / Facebook, Microsoft, Apple, Amazon, Netflix, Salesforce, IBM (the technology arm), Oracle, NVIDIA, Adobe, Tesla, SpaceX, OpenAI, Anthropic, ByteDance, Tencent, Alibaba.

**Heuristic D — JD-body context inference** (if no name match):

- Body mentions "Series A/B/C/D funding", "seed-stage", "growth-stage", "venture-backed", "founders" → `startup`.
- Body mentions "consulting practice", "client engagement", "billable hours", "delivery model", "advisory" → `consulting` or `services` depending on whether the firm name aligns with consulting (Heuristic B) or services (Heuristic A).
- Body mentions "product roadmap", "product-led growth", "users", "customers" (not "clients"), "platform" as a product — without consulting/services language → `product`.
- Body mentions "enterprise" without other distinguishing context → `other` (could be a captive enterprise IT shop, an industrial firm, or any of several types).

If a heuristic A/B/C name match is found, use it directly and skip D.

## Step 4 — Fill remaining fields via web search if available

For any of `size`, `hq`, `industry`, `funding_stage` still missing after Steps 2-3, attempt a web search to fill them — but only IF a real company name was extracted in Step 1. If `name` is the `"[unspecified hiring firm — name not in JD]"` placeholder, skip this entire step; there is no name to search for, and the supporting fields stay `null`.

When a real name exists, query template: `"<company name> company size hq industry"`.

Parse the top result for these fields. If found, tag `sources.<field> = "web_search"`. If the search fails or returns no usable data, leave the field as `null` and tag `sources.<field>` as absent (do not include it in the sources map).

For V1, web search is best-effort. The skill MUST NOT halt the audit if web search fails — it returns whatever fields it has and lets the agents reason with partial data.

**Important: LinkedIn is NOT used directly.** LinkedIn pages are gated behind login and unreliable to scrape. Rely on Google-cached snippets, the company's own About page (often surfaced in the search), and reputable third-party sources like Crunchbase, Bloomberg, or industry press.

## Step 5 — Default to `"unknown"` company_type if nothing matched

If after Steps 2-4 the `company_type` is still unset, set it to `"unknown"` and tag `sources.company_type = "default"`. The schema enum allows `"unknown"` as a valid value precisely for this fail-closed case.

## Output

Return a `Company` object matching `docs/SCHEMAS.md` section 6:

```json
{
  "name": "Acme Fintech",
  "size": "200-500",
  "hq": "Dubai, UAE",
  "industry": "fintech",
  "funding_stage": "Series B",
  "company_type": "product",
  "sources": {
    "name": "user_provided",
    "size": "web_search",
    "hq": "user_provided",
    "industry": "web_search",
    "funding_stage": "web_search",
    "company_type": "default"
  }
}
```

Or `null` if no company name could be extracted (see Step 1 fail-closed path).

The router places this into `ExternalContext.company` per `docs/SCHEMAS.md` section 5.

## Behavioural rules

- **Per-field provenance is non-optional.** Every populated field has a corresponding entry in `sources`. The audit's report and any human reviewer needs to know which fields came from where to weigh confidence appropriately.
- **The skill does not judge.** It fills facts. The `it-services-discount` and `channel-mix` agents reason over the structured Company object and produce verdicts.
- **`user_provided` always wins over `web_search`.** If frontmatter has a field, do not overwrite it with a web search result even if they conflict — log the conflict to `enrichment_errors` if it occurs.
- **Web search is best-effort, not blocking.** A web search failure does not halt the skill. Return whatever was filled deterministically and let the agents handle partial data.
- **No LinkedIn scraping.** Login-gated, unreliable, and Anthropic's WebSearch may not have access. Rely on Google-cached About pages and reputable third-party sources.
- **No PII gathering.** The skill collects company-level facts only. It does not fetch employees' names, contact info, or anything candidate-or-recruiter-specific. If a search incidentally surfaces such data, the skill discards it.
