---
name: wiki-compile
description: V1.2 PLACEHOLDER (not active in V1 or V1.1). Will implement the Karpathy LLM Wiki pattern — durable knowledge accumulating across audit sessions and across users (with opt-in anonymised sharing). Compiles per-user silence-signature insights into structured wiki entries, indexed for retrieval during future audits. Turns single-shot audits into compounding institutional knowledge for the user and (optionally) the broader community.
status: placeholder
target_version: V1.2
---

# V1.2 Wiki Compiler — placeholder

This file is a placeholder. **The skill does not run in V1 or V1.1.** It exists to:

1. Document the V1.2 design intent inspired by Andrej Karpathy's "LLM Wiki" pattern — knowledge that persists and compounds rather than being recomputed every session.
2. Reserve the file location (`.claude/skills/wiki-compiler/wiki-compile.md`) for the V1.2 implementation.
3. Make the V1.2 milestone explicit in the repo so contributors and forkers see where the system is going.

The frontmatter `status: placeholder` and `target_version: V1.2` are non-standard signal fields; Claude Code ignores unknown fields.

---

## What wiki-compile will do (V1.2 design)

GhostCheck V1 produces per-application audits. V1.1 produces per-user silence signatures across multiple audits. V1.2's wiki-compile produces durable, retrievable knowledge entries that persist across sessions and (optionally) across users.

The pattern (from Karpathy):

> *"Most LLM agents recompute their context from scratch every session. A wiki agent durably stores what it has learned, indexes it, and retrieves the relevant entries when a similar situation arises. Over time, the wiki becomes the agent's externalised long-term memory."*

Applied to GhostCheck, three categories of wiki entries:

### Category 1 — Personal silence-signature wiki entries

When the V1.1 pattern-detector identifies a recurring silence driver for a specific user (e.g., "for Santosh, IT-services-discount is the dominant signal across 7 of 9 ghosted applications"), wiki-compile turns that finding into a durable wiki entry:

```yaml
---
name: it-services-discount-as-personal-driver
scope: user
user_id: <user>
created: YYYY-MM-DD
last_updated: YYYY-MM-DD
relevance_count: 7  # number of applications where this pattern fired
---

# IT-services discount is your dominant silence driver

Across 7 of your 9 ghosted senior-architect applications,
it-services-discount returned HIGH severity. The agent's reframing
fix hints (delivery-language to ownership-language) addressed it
on the 2 applications where you applied them; both converted to
callback. The pattern is empirically validated for your funnel.

## What to do
- Apply the reframing fix systematically to all CV bullets, not
  just the one or two highlighted in any single audit.
- Channel the message in cover letters: lead with "design
  authority" and "owned outcomes" rather than "delivered for
  client."
- For top-tier consulting targets specifically, consider one to
  three months of named external advisory work (board advisor,
  named industry-conference talks) to overcome the discount
  before re-applying.
```

Future audits surface this entry when they run on Santosh's data: *"Per your silence-signature wiki, the IT-services-discount fix is your highest-leverage change — verifying that this audit's recommendations align."*

### Category 2 — Cross-user generic patterns (opt-in, anonymised)

When users opt in to share their anonymised audit-outcome data, the wiki accumulates patterns that generalise:

- *"At Director tier in consulting target firms, callback rates without referral are reliably under 5% across the user population"* — a population benchmark, no individual identifying data.
- *"Headlines containing 'Manager' are 30% less likely to convert to callback at IC-target roles than headlines containing 'Architect' or 'Lead'"* — a heuristic learned from data.
- *"Three or more aggregator results in the top ten correlates with HIGH-or-CRITICAL google-test verdicts and is reliably present in ghosted-application traces"* — a confirmation of a V1 heuristic with empirical backing.

These population-level entries accumulate over time and feed back into V1.2 calibration of severity thresholds and per-agent weights.

### Category 3 — Industry-specific or domain-specific heuristics

When fork users contribute audit-outcome data from their specific domain (electrical engineering, finance, medicine, etc.), the wiki accumulates domain-specific knowledge:

- *"In electrical engineering at senior-architect level, IEEE peer-review presence correlates more strongly with callback than GitHub presence."*
- *"In finance at MD-level, FCA / SEC compliance signals in CV correlate with callback at large-bank targets but not at hedge-fund targets."*
- *"In medicine at attending-level, board-cert specificity in headline correlates with callback at academic-hospital targets."*

These entries are tagged with their domain and surface only when an audit runs in that domain.

---

## Why this is V1.2 and not V1 or V1.1

Three reasons:

1. **Wiki content needs accumulated audits.** Until users have run dozens or hundreds of audits with logged outcomes, there is no signal to compile into wiki entries. V1.1 provides the per-user signal accumulation; V1.2 builds on top.

2. **Cross-user sharing requires explicit opt-in machinery and anonymisation.** Both are infrastructure work that V2 (the hosted runtime) supports natively. V1.2 lays the groundwork; full cross-user wiki only really fires under V2 deployment.

3. **The Karpathy pattern is itself an active research area.** The right shape for "durable, retrievable, compounding knowledge for an LLM agent" is still being figured out. V1.2 implements a defensible first-pass; V2 may rework based on what the community learns by then.

---

## Implementation notes for V1.2 work

When V1.2 implementation begins, this placeholder gets replaced with a real skill definition. Key considerations:

- **Storage:** wiki entries live as markdown files under a `wiki/` folder at the repo root. Each entry has structured frontmatter (name, scope: user / population / domain, created, last_updated, relevance_count, tags) plus a markdown body. The folder is gitignored at the user-scope level (private user wikis), and a `wiki/shared/` sub-folder holds opt-in shared content.

- **Indexing:** maintain a `wiki/index.md` (or `wiki/index.yml`) that lists entries with their tags so the audit router can surface relevant entries. Future enhancement: vector-embed the entries for semantic retrieval, but markdown-tag-and-retrieve is the V1.2 baseline.

- **Compile cadence:** the wiki-compile skill runs after each audit (cheap; reads the new audit's verdicts, updates relevance_counts on existing entries, creates new entries when patterns reach a confidence threshold). It does NOT run in real-time during the audit — the audit's flow is unchanged.

- **Privacy:** scope is enforced at the file-system layer. User-scope entries live in their own folder; population-scope entries live in `wiki/shared/`; nothing is shared without explicit opt-in via `config/share_outcomes.yml` (V1.2 placeholder; not yet implemented).

- **Surface in audits:** during a V1.2 audit, the router queries the wiki for entries matching the current CV+JD context (by tag and by user) and surfaces them in the audit's `Wiki Notes` section. This is what makes the system feel like it is "remembering" what it has learned.

---

## Why the wiki pattern matters for GhostCheck specifically

Most CV checkers are stateless: every CV gets the same heuristic treatment regardless of how many audits the user has done. GhostCheck V1.2 is the first CV-audit system that **learns** — the wiki turns audits into compounding knowledge.

This is also what makes V2 (the hosted multi-user version) genuinely valuable rather than just a hosting wrapper. A career coach running GhostCheck on their cohort accumulates wiki entries over hundreds of candidates, and those entries make every future audit sharper. The wiki is the cohort's institutional memory.

The Karpathy pattern — durable, indexed, retrievable — is the right shape. The V1.2 implementation operationalises it for a specific high-stakes domain.
