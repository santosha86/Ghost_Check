# SCHEMAS.md — Data Contracts for GhostCheck

This file is the single source of truth for every data shape used in the project. If you are modifying an agent, the aggregator, an enrichment skill, or the router, **read this file first**. The shapes defined here are load-bearing — a future Python runtime (V2) will map them 1:1 to Pydantic models. Do not invent fields, rename fields, or omit required fields.

---

## Core principles (read before skimming the shapes)

1. **Fail-closed.** Any agent output that fails schema validation is coerced to `UNKNOWN` with an `unknown_reason`. A malformed verdict never reaches the aggregator.
2. **No opinion without citation.** Every non-UNKNOWN verdict must carry at least one `Evidence` item pointing to a line of the CV, the JD, or a field of `ExternalContext`.
3. **UNKNOWN is a first-class outcome, not a failure.** Aggregation handles it deterministically (see "Weight renormalization" below).
4. **Pydantic-ready.** Types, constraints, and enums are declared so a Python wrap is additive, not a rewrite.

---

## 1. `AgentVerdict` — the contract every agent returns

Every one of the 11 agents produces exactly one `AgentVerdict` per audit run.

| Field          | Type                          | Required | Constraint                                                                  |
|----------------|-------------------------------|----------|-----------------------------------------------------------------------------|
| `agent_id`     | `AgentId` (enum, 11 values)   | yes      | One of the known agent IDs (see section 2).                                        |
| `severity`     | `Severity` (enum, 5 values)   | yes      | `CRITICAL` \| `HIGH` \| `MEDIUM` \| `LOW` \| `UNKNOWN`.                     |
| `score`        | `float`                       | yes      | `0.0` ≤ score ≤ `1.0`. Ignored by the aggregator when severity is UNKNOWN. |
| `verdict`      | `string` (markdown)           | yes      | 1–2 sentences. Human-readable summary.                                      |
| `evidence`     | `list[Evidence]`              | yes      | `min_length = 1` unless severity is `UNKNOWN`, then may be empty.           |
| `fix_hints`    | `list[string]` (markdown)     | yes      | `min_length = 0`. Empty for UNKNOWN verdicts.                               |
| `unknown_reason` | `string`                    | conditional | Required when and only when `severity = UNKNOWN`.                         |

### Example — a valid `CRITICAL` verdict

```json
{
  "agent_id": "bucket-classifier",
  "severity": "CRITICAL",
  "score": 0.82,
  "verdict": "CV reads as Staff engineer; target role is Director.",
  "evidence": [
    {
      "source": "cv",
      "location": "cv:line 34",
      "text": "Owned the AI platform roadmap across three product teams."
    },
    {
      "source": "jd",
      "location": "jd:line 8",
      "text": "You will own P&L for the AI division, ~80 engineers across four orgs."
    }
  ],
  "fix_hints": [
    "Re-scope two bullets to show owned P&L, not just roadmap ownership.",
    "Add an org-size metric (headcount, budget) to the last three roles."
  ]
}
```

### Example — a valid `UNKNOWN` verdict

```json
{
  "agent_id": "stale-detector",
  "severity": "UNKNOWN",
  "score": 0.0,
  "verdict": "Posting age could not be determined.",
  "evidence": [],
  "fix_hints": [],
  "unknown_reason": "No posting date in JD frontmatter, no date phrase in JD body, no source URL provided."
}
```

---

## 2. `AgentId` — the enum of all valid agent IDs

```
google-test
posting-decoder
bucket-classifier
it-services-discount
headline-filter
funnel-math
channel-mix
stale-detector
ats-simulator
recruiter-30sec
hm-deep-read
```

Exactly 11 values. Any `AgentVerdict` with an `agent_id` outside this set fails validation and is coerced to `UNKNOWN`.

---

## 3. `Severity` — the enum for verdict severity

```
CRITICAL    Likely the single biggest silence driver for this CV+JD pair.
HIGH        Material risk; should be addressed before re-applying.
MEDIUM      Noticeable issue; worth fixing if easy.
LOW         Minor polish; nice to have.
UNKNOWN     Agent could not assess — see unknown_reason.
```

The aggregator maps `CRITICAL=1.0`, `HIGH=0.75`, `MEDIUM=0.5`, `LOW=0.25`, `UNKNOWN` = excluded (weights redistributed, section 11).

---

## 4. `Evidence` — the citation object

Every non-UNKNOWN verdict carries at least one of these.

| Field      | Type                        | Required | Constraint                                                            |
|------------|-----------------------------|----------|-----------------------------------------------------------------------|
| `source`   | enum: `cv` \| `jd` \| `external_context` | yes | Where the cited text came from.                                    |
| `location` | `string`                    | yes      | Line reference or field path (e.g., `cv:line 23`, `external_context.company.size`). |
| `text`     | `string`                    | yes      | `min_length = 20`. The actual quoted text, so the report is legible without the source files. |

---

## 5. `ExternalContext` — the enrichment blob agents consume

Built once per audit by the enrichment skills. Passed (read-only) to every agent.

| Field                | Type                       | Required | Notes                                                              |
|----------------------|----------------------------|----------|--------------------------------------------------------------------|
| `company`            | `Company` (object)         | optional | May be `null` if the JD has no company and no enrichment succeeded.|
| `jd_age_days`        | `int`                      | optional | Days since JD was posted. `null` if unknown.                       |
| `jd_age_source`      | enum: `user_provided` \| `text_inferred` \| `url_inferred` \| `unknown` | yes | Provenance of `jd_age_days`.                                       |
| `google_results`     | `list[GoogleResult]`       | yes      | May be empty. Populated by the `google-test-lookup` enrichment skill. |
| `enrichment_errors`  | `list[string]`             | yes      | Any non-fatal enrichment failures. Surfaced in the final audit.   |

---

## 6. `Company` — the hybrid enrichment object

Built by the `company-enricher` skill, which takes (a) any user-provided fields from the JD YAML header (section 10), and (b) fills gaps via web search on the public internet. **LinkedIn direct scraping is not used** — it is gated behind login and unreliable; we rely on Google-cached snippets, company About pages, and third-party sources.

| Field            | Type                                                                                     | Required | Notes                                                   |
|------------------|------------------------------------------------------------------------------------------|----------|---------------------------------------------------------|
| `name`           | `string`                                                                                 | yes      | Required if the `Company` object exists at all.         |
| `size`           | `string \| int`                                                                          | optional | Headcount band (`"200-500"`) or exact integer.          |
| `hq`             | `string`                                                                                 | optional | e.g., `"Dubai, UAE"`.                                   |
| `industry`       | `string`                                                                                 | optional | e.g., `"fintech"`.                                      |
| `funding_stage`  | `string`                                                                                 | optional | e.g., `"Series B"`, `"public"`, `"bootstrapped"`.       |
| `company_type`   | enum: `product` \| `services` \| `consulting` \| `bigtech` \| `startup` \| `other` \| `unknown` | yes | Output of the classifier step.                          |
| `sources`        | `dict[field_name to SourceTag]`                                                           | yes      | Per-field provenance. See below.                        |

### `SourceTag` enum (per-field provenance)

```
user_provided     — from the JD file's YAML header
web_search        — derived from public web search results
default           — classifier fallback when no info was found
```

Per-field provenance matters because a user may provide the company name themselves but let web search fill in size and funding stage — the audit must be able to say which.

### Example

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

---

## 7. `GoogleResult` — per-result shape for the google-test agent

| Field      | Type      | Required | Notes                                                               |
|------------|-----------|----------|---------------------------------------------------------------------|
| `rank`     | `int`     | yes      | 1-indexed position in the results.                                  |
| `title`    | `string`  | yes      |                                                                     |
| `url`      | `string`  | yes      |                                                                     |
| `snippet`  | `string`  | yes      | Search-engine-provided excerpt.                                     |
| `domain`   | `string`  | yes      | Extracted hostname (used to classify result type).                  |
| `surface`  | enum: `linkedin` \| `github` \| `personal_site` \| `company` \| `press` \| `aggregator` \| `other` | yes | Classified by the enrichment skill.                               |

---

## 8. `UserProfile` — the shape of `config/profile.yml`

| Field                  | Type            | Required | Notes                                                  |
|------------------------|-----------------|----------|--------------------------------------------------------|
| `target_titles`        | `list[string]`  | yes      | `min_length = 1`. e.g., `["Director of AI", "VP AI"]`. |
| `target_locations`     | `list[string]`  | yes      | `min_length = 1`. `"remote"` allowed.                  |
| `target_seniority`     | enum: `mid` \| `senior` \| `staff` \| `principal` \| `director` \| `vp` | yes | Aligns the bucket-classifier.                          |
| `target_company_types` | `list[company_type]` | optional | Filters audit emphasis.                           |
| `preferred_channels`   | `list[string]`  | optional | e.g., `["referral", "dm"]`. Feeds channel-mix benchmarks. |

---

## 9. JD file frontmatter (optional, but recommended)

JD files may start with an optional YAML header. All fields are optional. The `company-enricher` skill reads this first, then fills gaps via web search.

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

If the header is absent, enrichment proceeds on the JD body alone. If present, user-supplied fields are always preferred over web-enriched ones.

---

## 10. The `applications/<date_slug>/` folder — audit artifact layout

Every audit run produces a folder at `applications/YYYY-MM-DD_<jd-slug>/`. Contents:

| File           | Written by | Required | Purpose                                                     |
|----------------|------------|----------|-------------------------------------------------------------|
| `audit.md`     | aggregator | yes      | The final human-readable audit. See section 11 for its contract.   |
| `verdicts.json`| aggregator | yes      | All 11 `AgentVerdict` objects, raw. For tooling / B-tier agents to re-read later. |
| `context.json` | aggregator | yes      | The `ExternalContext` used for this run. Auditability.     |
| `outcome.md`   | **user, optional** | no | One line: `callback`, `screened`, `rejected`, or `ghosted`, plus date. Input to B1 funnel-math in future audits. |
| `channel.md`   | **user, optional** | no | One line: `easy-apply`, `referral`, `dm`, `exec-search`, etc. Input to B2 channel-mix in future audits. |

### Slug collision rule

If two audits would produce the same folder (same JD on the same day), append `_2`, `_3`, … to the slug. Never overwrite.

---

## 11. `audit.md` — the final report contract

The aggregator writes a markdown file with exactly these sections in this order:

```markdown
# Audit — <role title> @ <company>

**Date:** YYYY-MM-DD
**CV:** <path>
**JD:** <path>
**Callback probability:** NN%

## Top silence drivers

1. **<SEVERITY>** — <agent_id>: <verdict>
   - Evidence: <first evidence `text`> (<location>)
   - Fix: <first fix_hint>

2. ...

## Not assessed

<For each UNKNOWN verdict:>
- **<agent_id>** — <unknown_reason>

## Full verdicts (appendix)

<For each of the 11 verdicts:>
### <agent_id>
- Severity: <SEVERITY>
- Score: <score>
- Verdict: <verdict>
- Evidence: <bullet list of all evidence items>
- Fix hints: <bullet list of all fix_hints>
```

The "Not assessed" section is required whenever any verdict is UNKNOWN — even if all 11 were assessed, include the header with "None" to make the contract visible.

---

## 12. Validation and fail-closed coercion

When the aggregator receives an `AgentVerdict` from an agent:

1. Validate against the schema in section 1.
2. On any validation failure (missing required field, wrong type, out-of-range score, empty evidence on non-UNKNOWN, etc.), coerce to:
   ```json
   {
     "agent_id": "<the failing agent's id>",
     "severity": "UNKNOWN",
     "score": 0.0,
     "verdict": "Agent output failed schema validation.",
     "evidence": [],
     "fix_hints": [],
     "unknown_reason": "schema_violation: <specific reason>"
   }
   ```
3. The coerced verdict proceeds into aggregation like any other UNKNOWN.

There is no malformed-verdict path. Either it validates, or it becomes UNKNOWN. This is the fail-closed rule made concrete.

---

## 13. Weight renormalization on UNKNOWN

Weights for each agent live in `config/weights.yml` and sum to `1.0` for the 11 configured agents.

When one or more verdicts come in as `UNKNOWN`:

1. Let `S` be the set of agents with a non-UNKNOWN verdict and `U` the set with UNKNOWN.
2. Let `W_total_assessed = Σ weights(S)`.
3. For each agent `a` in `S`, the effective weight for this audit is `weights(a) / W_total_assessed`.
4. The callback probability is the weighted logistic over agents in `S` only.
5. The audit's "Not assessed" section lists every agent in `U` with its `unknown_reason`.

This preserves a meaningful 0–100% callback probability (the score isn't artificially capped below 100% just because one agent couldn't assess) while keeping the user fully informed about what wasn't measured. If every one of the 11 agents goes UNKNOWN, the final callback probability is undefined and the audit states "Insufficient data to produce a probability."

---

## 14. Versioning and the V2 migration path

This file is versioned implicitly by git. When `SCHEMAS.md` changes in a backwards-incompatible way, bump the `schema_version` field referenced in every skill's YAML frontmatter (to be added in the agent skill files). V2 (the Python service) will read this file and auto-generate Pydantic models from the field tables above — that is why the tables follow a regular structure.

Fields not yet specified that V1.2 or V2 may add (documented here so forkers don't reinvent them): `confidence` (float) on `AgentVerdict`, structured `FixHint` object (target_field, before, after, rationale), `schema_version` string on every object.

---

## 15. `CandidateProfile` — structured representation of the CV

Built by the parser skill from the parsed CV markdown. Every agent that needs CV content reads `CandidateProfile` named fields rather than re-parsing raw `cv_text`. Single source of truth for the candidate's data; eliminates per-agent re-parsing variance.

| Field                | Type                         | Required | Notes                                                             |
|----------------------|------------------------------|----------|-------------------------------------------------------------------|
| `name`               | `string`                     | yes      | Full name as written in CV's H1 heading or top of page.           |
| `current_title`      | `string`                     | yes      | Title from CV's tagline or first-listed role.                     |
| `current_company`    | `string`                     | yes      | Most-recent employer name.                                        |
| `location`           | `string`                     | optional | City, country, or "remote" if explicit.                           |
| `total_years`        | `int`                        | optional | Sum of role-block durations; null if dates absent.                |
| `summary`            | `string`                     | optional | The "Professional Summary" / "Profile" paragraph.                 |
| `competencies`       | `list[string]`               | yes      | From "Core Competencies" / "Key Skills" section. May be empty.    |
| `technologies`       | `list[string]`               | yes      | Tools / frameworks mentioned in CV body. May be empty.            |
| `recent_roles`       | `list[Role]`                 | yes      | Roles within the last seven years. `min_length = 1` for non-empty CVs. |
| `earlier_roles`      | `list[Role]`                 | optional | Roles older than seven years.                                     |
| `key_projects`       | `list[Project]`              | optional | From "Key Architectural Projects" or equivalent section.          |
| `education`          | `list[Education]`            | optional | Degrees and credentials.                                          |
| `certifications`     | `list[string]`               | optional | Standalone certifications (CKAD, AWS SA, etc.).                   |
| `extraction_sources` | `dict[field_name, SourceTag]`| yes      | Per-field provenance; `direct_extract` for fields read straight from text, `inferred` for fields the parser had to compute, `default` for fallback. |

### `Role` — a single employment block

| Field            | Type                   | Required | Notes                                                       |
|------------------|------------------------|----------|-------------------------------------------------------------|
| `title`          | `string`               | yes      | Role title at this employer.                                |
| `company`        | `string`               | yes      | Employer name.                                              |
| `location`       | `string`               | optional | City / country / region.                                    |
| `start_date`     | `string`               | yes      | `YYYY` or `YYYY-MM` format.                                 |
| `end_date`       | `string`               | yes      | `YYYY`, `YYYY-MM`, or `"Present"`.                          |
| `duration_years` | `float`                | yes      | Computed from start and end. Used by `bucket-classifier`, `recruiter-30sec`, `it-services-discount`. |
| `client`         | `string`               | optional | Named client (common in services-firm CVs). Used by `it-services-discount`. |
| `bullets`        | `list[string]`         | yes      | Achievement bullets verbatim. Used by `hm-deep-read`, `bucket-classifier`. |
| `is_services_firm` | `bool`               | optional | Computed by parser via known-firms config. Used by `it-services-discount`. |

### `Project` — a key project block

| Field             | Type     | Required | Notes                                          |
|-------------------|----------|----------|------------------------------------------------|
| `name`            | `string` | yes      | Project name.                                  |
| `client`          | `string` | optional | Named client.                                  |
| `objective`       | `string` | optional | Project objective text.                        |
| `architecture`    | `string` | optional | Architecture description.                      |
| `business_impact` | `string` | optional | Business outcome / metrics.                    |

### `Education` — a single education block

| Field         | Type                         | Required | Notes                                          |
|---------------|------------------------------|----------|------------------------------------------------|
| `degree`      | `string`                     | yes      | Degree name (Bachelor, Master, Doctorate, etc.). |
| `field`       | `string`                     | optional | Field of study.                                |
| `institution` | `string`                     | optional | School / university name.                       |
| `year`        | `int`                        | optional | Year of completion.                            |
| `status`      | enum: `completed \| pursuing`| yes      | Used by `bucket-classifier`, `headline-filter` for credential-tier signals. |
| `country`     | `string`                     | optional | Country of institution.                        |

### Example — a valid `CandidateProfile`

```json
{
  "name": "Rohan Mehta",
  "current_title": "AI Solution Architect & Delivery Lead",
  "current_company": "Gulf AI Advisory",
  "location": "Riyadh, Saudi Arabia",
  "total_years": 16,
  "summary": "AI Solution Architect and Delivery Lead with 16+ years of experience...",
  "competencies": [
    "AI Solution Architecture",
    "GenAI / LLM Engineering",
    "Multi-Team Delivery Leadership"
  ],
  "technologies": ["LangGraph", "LangChain", "Ollama", "Llama3", "FAISS", "Milvus"],
  "recent_roles": [
    {
      "title": "Principal AI Solution Architect & Delivery Lead",
      "company": "Gulf AI Advisory",
      "location": "KSA",
      "start_date": "2024",
      "end_date": "Present",
      "duration_years": 1.0,
      "client": "a national utilities operator",
      "bullets": [
        "Served as design authority for SEC's enterprise AI programme...",
        "Led technical solutioning and presented solution architecture..."
      ],
      "is_services_firm": true
    }
  ],
  "education": [
    {
      "degree": "Doctorate",
      "field": "Emerging Technologies, Generative AI Specialization",
      "institution": "Golden Gate University",
      "country": "USA",
      "status": "pursuing"
    }
  ],
  "extraction_sources": {
    "name": "direct_extract",
    "current_title": "direct_extract",
    "total_years": "inferred",
    "is_services_firm": "inferred"
  }
}
```

---

## 16. `JobProfile` — structured representation of the JD

Built by the parser skill from the parsed JD markdown. Every agent that needs JD content reads `JobProfile` named fields rather than re-parsing raw `jd_text`.

| Field                              | Type                         | Required | Notes                                                       |
|------------------------------------|------------------------------|----------|-------------------------------------------------------------|
| `title`                            | `string`                     | yes      | JD's role title (often in the heading).                     |
| `company_name`                     | `string`                     | optional | Hiring firm if extractable; null when JD says only "the firm". |
| `location`                         | `string`                     | optional | Location requirement; "remote" if explicit.                 |
| `seniority_keyword`                | `string`                     | optional | "Chief" / "Director" / "Principal" / etc. extracted from title or body. |
| `summary`                          | `string`                     | optional | Role summary paragraph (often labelled "About the role").   |
| `responsibilities`                 | `list[string]`               | yes      | Bulleted responsibilities. May be empty.                    |
| `required_skills`                  | `list[string]`               | yes      | Hard requirements (technical skills, frameworks, tools).    |
| `preferred_skills`                 | `list[string]`               | optional | Nice-to-haves.                                              |
| `years_required`                   | `int`                        | optional | Minimum years of experience stated.                         |
| `education_required`               | `string`                     | optional | Degree requirement (e.g. "Bachelor's required, Master's preferred"). |
| `certifications_required`          | `list[string]`               | optional | Required certs.                                             |
| `market_leadership_required`       | `bool`                       | yes      | Whether JD asks for thought leadership, external speaking, opportunity origination. Used by `google-test`, `headline-filter`. |
| `consulting_signals_present`       | `bool`                       | yes      | Whether JD uses consulting language (client engagements, delivery playbooks, billable hours). Used by `it-services-discount`, `company-classifier`. |
| `product_signals_present`          | `bool`                       | yes      | Whether JD uses product language (product roadmap, users, platform). Used by same agents. |
| `services_signals_present`         | `bool`                       | yes      | Whether JD uses staff-augmentation / managed-services language. |
| `extraction_sources`               | `dict[field_name, SourceTag]`| yes      | Per-field provenance.                                       |

### Example — a valid `JobProfile`

```json
{
  "title": "AI Solution Architect – Chief",
  "company_name": null,
  "location": null,
  "seniority_keyword": "Chief",
  "summary": "The AI Solution Architect Chief is a senior technology and AI leader responsible for shaping, selling, and delivering enterprise-scale AI solutions...",
  "responsibilities": [
    "Define end-to-end AI solution architectures spanning data platforms, AI/ML pipelines, GenAI stacks...",
    "Act as design authority for complex, multi-vendor AI programs...",
    "Originate and shape AI-led opportunities across priority industries..."
  ],
  "required_skills": [
    "Enterprise AI solution architecture",
    "GenAI/LLM architectures",
    "MLOps",
    "Cloud security",
    "Executive communication"
  ],
  "preferred_skills": ["Cloud architecture certifications"],
  "years_required": 15,
  "education_required": "Bachelor's required, Master's or PhD preferred",
  "market_leadership_required": true,
  "consulting_signals_present": true,
  "product_signals_present": false,
  "services_signals_present": false,
  "extraction_sources": {
    "title": "direct_extract",
    "company_name": "default",
    "seniority_keyword": "direct_extract",
    "market_leadership_required": "inferred",
    "consulting_signals_present": "inferred"
  }
}
```

### `SourceTag` for `extraction_sources`

The per-field provenance tag values, distinct from the `Company.sources` enum (section 6) because the contexts differ:

```
direct_extract  — read verbatim from a CV/JD section (e.g. title from "AI Solution Architect & Delivery Lead" line)
inferred        — computed by the parser (e.g. total_years from summing role durations, is_services_firm from known-firms config)
default         — fallback used because no signal was found (e.g. company_name = null when no name extractable)
```

This mirrors the per-field provenance pattern already in use for `Company.sources` (section 6) — so the audit reader can see where every CandidateProfile and JobProfile field came from and weight it appropriately.

### How agents use these structures

Every V1 agent's frontmatter `inputs:` field references the structured profile fields directly. Examples:

- `bucket-classifier` declares `inputs: [candidate.recent_roles, candidate.summary, candidate.competencies, target.title, target.seniority_keyword, target.responsibilities, user_profile.target_seniority]`.
- `headline-filter` declares `inputs: [candidate.name, candidate.current_title, candidate.current_company, target.title, target.seniority_keyword, user_profile.target_titles, user_profile.target_seniority]`.
- `it-services-discount` declares `inputs: [candidate.recent_roles, candidate.earlier_roles, target.consulting_signals_present, target.services_signals_present, external_context.company.company_type]`.

The router resolves the dotted-path references at runtime and passes only the declared fields to each agent. Same Pattern-B subagent isolation as before; same blind fan-out.

The raw `cv_text` and `jd_text` strings remain available as fallback in `external_context.cv_text_fallback` and `external_context.jd_text_fallback` for cases where structured extraction fails partially — graceful degradation, never UNKNOWN-by-default.