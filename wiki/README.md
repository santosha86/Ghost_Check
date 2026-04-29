# wiki/ — durable cross-session knowledge (V1.2)

This folder is currently **empty by design**. It is reserved for the V1.2 implementation of the Karpathy LLM Wiki pattern — durable, indexed, retrievable knowledge that compounds across audit sessions and (with explicit opt-in) across users.

In V1 and V1.1, this folder contains only this README and a `.gitkeep`. No wiki entries are written here yet.

## What V1.2 will produce here

When the V1.2 `wiki-compile` skill (`.claude/skills/wiki-compiler/wiki-compile.md` — currently a placeholder) is implemented, this folder accumulates three categories of entries:

### `wiki/<user>/`

User-private wiki entries. Each entry is a markdown file with structured frontmatter capturing a finding from the user's accumulated audit history:

- *"For you, IT-services-discount is the dominant silence driver — addressing it converts 5 of 7 ghosting patterns to callbacks."*
- *"For you, referral is the only viable channel above Director tier — Easy Apply has 0% callback rate across 12 attempts."*
- *"For you, your headline understates by one tier — every callback in your last 6 months had a headline rewritten before submission."*

Each entry has structured metadata: when it was created, when it was last updated, how many audits it is based on, the relevance count (how often the pattern fires).

Future audits surface these entries proactively: *"Per your wiki, the IT-services-discount fix is your highest-leverage change — verifying alignment with this audit's recommendations."*

### `wiki/shared/` (V1.2 + opt-in)

Anonymised cross-user patterns that generalise. Built from contributions of users who explicitly opt-in via `config/share_outcomes.yml`.

- *"At Director tier in consulting target firms, callback rates without referral are reliably under 5% across the user population (N=247 audits)."*
- *"Headlines containing 'Manager' are 30% less likely to convert at IC-target roles than headlines containing 'Architect' or 'Lead' (N=412)."*
- *"Three or more aggregator results in the top ten correlates with HIGH-or-CRITICAL `google-test` and is reliably present in ghosted-application traces (N=189)."*

These are population-level patterns. They feed back into V1.2 calibration of severity thresholds, weights, and aggregator hyperparameters.

### `wiki/domains/` (V1.2 + opt-in)

Domain-specific wiki entries contributed by fork users in different industries. Each tagged with its domain:

- `wiki/domains/electrical-engineering/ieee-presence-vs-callback.md`
- `wiki/domains/finance/compliance-signals-by-target-firm.md`
- `wiki/domains/medicine/board-cert-specificity-in-headline.md`

These entries surface only when an audit runs in that domain. They are how the harness pattern (a domain-agnostic architecture with domain-flavoured prompts) accumulates real domain knowledge over time.

## The Karpathy LLM Wiki pattern in one sentence

> *Most LLM agents recompute their context from scratch every session. A wiki agent durably stores what it has learned, indexes it, and retrieves the relevant entries when a similar situation arises. Over time, the wiki becomes the agent's externalised long-term memory.*

GhostCheck V1 is stateless across sessions (every audit starts fresh). GhostCheck V1.2 with wiki-compile is stateful across sessions and across users — the system gets sharper the more audits run on it.

This is what makes the V2 hosted multi-user version genuinely valuable rather than just a hosting wrapper. A career coach running GhostCheck on their cohort accumulates wiki entries over hundreds of candidates, and those entries make every future audit sharper. The wiki is the cohort's institutional memory.

## What to do in V1

Nothing — this folder is empty until V1.2 ships. You can ignore it.

When V1.2 lands, the `wiki-compile` skill runs after each audit and updates wiki entries automatically. The user does not write wiki entries by hand; the system compiles them from accumulated audit data.

The `wiki-compile` placeholder file documents the V1.2 design intent in detail. Read it for the full specification.

## Reference

- Andrej Karpathy's LLM Wiki pattern (search the public web for "Karpathy LLM Wiki" — the pattern is described across his recent talks and Twitter / X posts on agentic AI design).
- Implementation precedent in agentic AI: durable memory layers in CrewAI, LangGraph, and other multi-agent frameworks. GhostCheck V1.2's implementation will be markdown-first to match the rest of the repo's design.
