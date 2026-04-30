---
name: markitdown-parse
description: Two-phase parser. Phase 1 converts any supported document (PDF, MD, DOCX, PPTX, HTML, TXT) to markdown text — Read tool for PDF/MD (zero install), MarkItDown CLI for the others. Phase 2 extracts structured profile fields (CandidateProfile or JobProfile per docs/SCHEMAS.md sections 15-16) from the markdown. The router invokes the parser once per CV and once per JD with the appropriate profile_type. Returns a structured YAML object.
allowed-tools:
  - Read
  - Bash
---

# Parser skill — file to structured profile

When invoked with a file path and a `profile_type` (`cv` or `jd`), produce a structured profile object conforming to `docs/SCHEMAS.md` section 15 (`CandidateProfile`) or section 16 (`JobProfile`). Two phases: convert file to markdown, then extract structured fields. Fail closed on either phase: surface a clear error and halt.

## Inputs

The router passes:

- `file_path` — path to the CV or JD file.
- `profile_type` — one of `cv` or `jd`. Determines which schema to extract against.

## Phase 1 — File to markdown

### Step 1A — Determine the file extension

Lowercase the extension. If absent, halt: `"Cannot parse: file at <path> has no extension. Halt parsing."`

### Step 1B — Route based on extension

**Native path (no install needed): `.md` and `.pdf`**

For `.md`: Read the file with the Read tool. Contents are already markdown.
For `.pdf`: Read with the Read tool. For PDFs over 10 pages, use the `pages` parameter in chunks (max 20 per request); concatenate.

**MarkItDown path: `.docx`, `.pptx`, `.html`, `.txt`**

1. Check installation: `which markitdown` via Bash. If not found, halt with the install message:

   ```
   This document type requires MarkItDown with format-specific extras.
   Install once with: pip install 'markitdown[all]'

   Note the single-quotes — required on macOS zsh which treats unquoted
   square brackets as a glob pattern. The bare `pip install markitdown`
   does NOT include PPTX, DOCX, or HTML extras.

   Then re-run /ghostcheck audit.

   PDF and Markdown work without any install.
   ```

2. Invoke `markitdown "<path>"` via Bash. Capture stdout as the markdown text.

3. Handle MarkItDown errors. Surface stderr to the user.

### Unsupported extensions

If outside the six supported (`.md`, `.pdf`, `.docx`, `.pptx`, `.html`, `.txt`), halt with: `"Unsupported file type: <ext>. Halt parsing."`

The output of Phase 1 is a single markdown string. Hold it as `markdown_text`.

## Phase 2 — Markdown to structured profile

The structured-extraction step itself uses Claude's reasoning to walk the markdown and populate the schema fields. Branch on `profile_type`.

### If `profile_type == "cv"` — produce a `CandidateProfile`

Extract these fields from `markdown_text` per `docs/SCHEMAS.md` section 15:

**Required fields:**

- `name` — read from CV's H1 heading (`# <name>`) or the topmost line.
- `current_title` — line below the name; or the first listed role's title if that is more accurate.
- `current_company` — most-recent employer name from the first role block.
- `competencies` — bullet items from "Core Competencies", "Key Skills", or equivalent section. Empty list `[]` if section absent.
- `technologies` — extract specific tools, frameworks, languages mentioned: from competencies sub-bullets, from Technical Toolkit section if present, and from role-block bullet bodies. Deduplicate. Empty list if none extractable.
- `recent_roles` — list of `Role` objects (per section 15 sub-schema) for roles within the last seven years.
- `extraction_sources` — dict marking each populated field's source (`direct_extract` if read verbatim, `inferred` if computed, `default` if fallback).

**Optional fields:**

- `location` — usually below the title line; "Riyadh, Saudi Arabia" / "London, UK" / "remote".
- `total_years` — sum the role durations. If dates absent, leave null.
- `summary` — the "Professional Summary" / "Profile" paragraph.
- `earlier_roles` — `Role` objects for roles older than seven years.
- `key_projects` — `Project` objects from "Key Architectural Projects", "Selected Projects", or equivalent section.
- `education` — `Education` objects for degrees and credentials.
- `certifications` — standalone certifications.

**For each `Role` block, extract:**

- `title`, `company`, `location`, `start_date` (`YYYY` or `YYYY-MM`), `end_date` (`YYYY`, `YYYY-MM`, or `"Present"`).
- `duration_years` — computed: end - start in years. Use 0.5 for sub-year roles where end is "Present".
- `client` — if explicitly named (common in services-firm role blocks like "Client: SABIC").
- `bullets` — verbatim list of achievement bullets.
- `is_services_firm` — true if `company` matches the `it_services` list in `config/known_firms.yml` (see V1.1 work; for V1, hardcoded list in this prompt or in `config/known_firms.yml` if present at parse time).

### If `profile_type == "jd"` — produce a `JobProfile`

Extract these fields from `markdown_text` per `docs/SCHEMAS.md` section 16:

**Required fields:**

- `title` — JD's role title. Usually the first heading or labelled "Job profile:" / "Position:".
- `responsibilities` — bulleted items under "Responsibilities" / "What you'll do" / "Role description".
- `required_skills` — bulleted items under "Skills" / "Requirements" / "Must have". Empty list if absent.
- `market_leadership_required` — boolean. True if JD body mentions "thought leadership engagements", "external speaking", "industry visibility", "represent the firm externally", "client workshops", or similar language. False otherwise.
- `consulting_signals_present` — boolean. True if JD body mentions "client engagements", "delivery playbooks", "ecosystem partner management", "billable", "consulting practice", "advisory", or similar.
- `product_signals_present` — boolean. True if JD body mentions "product roadmap", "product-led", "users" (not "clients"), "platform" as the firm's product, or similar.
- `services_signals_present` — boolean. True if JD body mentions "staff augmentation", "managed services", "client deliverables", "team augmentation", "delivery resource".
- `extraction_sources` — dict marking each populated field's source.

**Optional fields:**

- `company_name` — extract from JD frontmatter (`company:`) if present, OR from JD's first heading or "About us" section. Null if JD references the hiring firm only as "the firm" / "our firm" / "our organisation" without naming.
- `location` — location requirement; "remote" if explicit; null if absent.
- `seniority_keyword` — extract from title or body: "Chief", "Director", "VP", "Principal", "Staff", "Senior", "Lead", etc. Null if absent.
- `summary` — summary-of-role paragraph.
- `preferred_skills` — bullets under "Nice to have" / "Preferred" / "Desirable".
- `years_required` — minimum years of experience stated, e.g. extract `15` from "15+ years of total experience".
- `education_required` — degree statement; e.g. "Bachelor's required, Master's preferred".
- `certifications_required` — required certs.

## Output

Return the structured profile object as JSON (or YAML — both shapes the schema accepts). The router places it into the audit context as `candidate` (if `profile_type == "cv"`) or `target` (if `profile_type == "jd"`).

If structured extraction fails for a critical field (e.g., `name` cannot be extracted at all from the CV), halt the audit with: `"Phase 2 extraction failed for <profile_type>: <specific failure>. Halt audit."`

If extraction fails for an OPTIONAL field, set the field to `null`, mark its `extraction_sources` entry as `default`, and continue. Graceful degradation; the agents downstream handle null fields per their own rubrics.

## Behavioural rules

- **The parser does not classify documents** as CV-versus-JD; the router knows which is which and passes the correct `profile_type`.
- **The parser does not validate content quality.** Extracted fields may be sparse or imperfect; the agents decide what to do about that.
- **Verbatim extraction wherever possible.** A role's `title` field is the title as written in the CV. A JD's `responsibilities` are the bullets verbatim. The schema fields are STRUCTURAL — what they hold is the source's content as-written.
- **`extraction_sources` is non-optional.** Every populated field has a corresponding entry in `extraction_sources`. The audit reader uses these to weight confidence in agent verdicts.
- **No silent install of MarkItDown extras.** If a non-PDF/non-MD file requires MarkItDown extras and they are not installed, halt with the install message.
- **No retry on transient failures.** Read or MarkItDown errors halt the audit; the user re-runs.
- **Phase 1 and Phase 2 are sequential.** Phase 2 cannot run until Phase 1 produces `markdown_text`. A Phase 1 failure halts before Phase 2 begins.
