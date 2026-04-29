---
name: V1_1-pattern-detector
description: V1.1 PLACEHOLDER (not active in V1). Will analyse the user's full applications/ folder — audits, verdicts, outcomes, channels — to identify cross-application patterns that single-audit agents cannot see. Produces the user's "personal silence signature" — the recurring failure mode across multiple ghosted applications. This is the killer feature flagged in the build prompt; activates when the user has logged 5+ applications with outcomes.
status: placeholder
target_version: V1.1
---

# V1.1 Pattern Detector — placeholder

This file is a placeholder. **The agent does not run in V1.** It is here to:

1. Document the V1.1 design intent so the agent's shape is visible to forkers and the roadmap is concrete rather than hand-wavy.
2. Reserve the file location (`.claude/agents/V1_1-pattern-detector.md`) so the V1.1 implementation lands at a predictable path.
3. Make the V1 to V1.1 transition surface explicit in the repo rather than buried in `docs/ROADMAP.md` alone.

The file's frontmatter `status: placeholder` and `target_version: V1.1` are non-standard fields for the GhostCheck schema; they exist only to signal placeholder status. The Claude Code runtime ignores unknown frontmatter fields.

---

## What pattern-detector will do (V1.1 design)

The V1 audit answers: *"Why was I ghosted on this specific application?"*

The V1.1 pattern-detector answers a different and more important question: *"What is my personal silence signature — the pattern across all my ghostings?"*

Specifically, when the user has logged at least five applications with outcomes (callback, screened, rejected, ghosted) in the `applications/` folder, the pattern-detector reads:

- All `applications/<slug>/audit.md` files (the per-audit verdicts).
- All `applications/<slug>/verdicts.json` files (the structured per-agent verdicts).
- All `applications/<slug>/outcome.md` files (user-logged outcomes).
- All `applications/<slug>/channel.md` files (user-logged channels).
- The `external_context.json` for each (the company-type and JD-age context per audit).

And identifies four kinds of cross-application patterns:

### Pattern 1 — Consistently-firing agents

Which agents return HIGH or CRITICAL severity across many ghosted applications but do not fire on callback applications? These are the candidate's structural silence drivers.

For example: `it-services-discount` HIGH on 6 of 8 ghosted applications and LOW on 0 of 2 callback applications would mean services-discount is the dominant cross-application driver for this candidate — addressing it is the single highest-leverage change.

### Pattern 2 — Discriminating signals

Which agents produce DIFFERENT severities between ghosted applications and callback applications? These are the agents whose verdicts predict outcomes for THIS specific candidate.

For example: `bucket-classifier` may be LOW on every application (the CV reads at-tier consistently) — that means bucket-classifier is uninformative for this candidate's funnel, and the silence is somewhere else. Compare to `channel-mix` showing CRITICAL when ghosted and LOW when callback-bound — channel-mix is highly discriminating.

### Pattern 3 — Channel-outcome correlation

What is the callback rate per channel for THIS candidate? Population-level benchmarks (from `funnel-math.md` and `channel-mix.md`) tell what should work; per-candidate observation tells what does work for THIS person.

For example: 5 out of 5 referrals converted to callbacks; 0 out of 12 Easy Apply submissions converted. The personal silence signature for this candidate is unambiguous — Easy Apply is wasted effort, referral is the only viable channel.

### Pattern 4 — Tier-target outcome correlation

Are Director-targeted applications converting differently from Principal-targeted? Sometimes a candidate's CV is genuinely at-tier for one level but not the next; pattern-detector surfaces that empirically.

---

## Output: silence-signature.md

The pattern-detector writes a `patterns/<user>/silence-signature.md` artefact summarising:

- The dominant silence driver(s) — which one or two agents drive the gap between ghosted and callback outcomes.
- The discriminating signals — what predicts outcome for this specific candidate.
- The channel pattern — what works, what does not.
- The tier pattern — where the candidate converts vs ghosts.
- A prioritised action list — top three changes that, if made, would have the highest expected impact on future callback rate.

This artefact is genuinely novel. No existing CV checker offers cross-application analysis because no existing CV checker stores audits as durable, schema-faithful artefacts.

---

## Why this is V1.1 and not V1

Two reasons:

1. **The harness must work end-to-end first.** V1 establishes the agent contracts, the aggregator math, the audit artefact shape. Pattern detection is built on top of those. Without V1's stable foundation, pattern-detector would be reading a moving target.

2. **The pattern-detector needs accumulated data.** The user must run V1 audits and log outcomes for at least 5-10 applications before patterns become statistically meaningful. That accumulation only happens after V1 ships and is used.

The V1.1 implementation reads `applications/*/` files (no schema change to V1). Adding pattern-detector is purely additive.

---

## Activation criteria (V1.1 spec, not V1)

The pattern-detector should activate when:

- At least 5 applications have logged outcomes (`outcome.md` files exist).
- At least 2 of those outcomes are different (e.g., 1 callback + 4 ghosted, or 2 callbacks + 3 ghosted). All-same-outcome data has no signal.
- At least 3 different agents fired non-UNKNOWN verdicts across the audited applications. (If everything is UNKNOWN, there is nothing to find patterns in.)

If activation criteria are not met, the pattern-detector returns: `"Insufficient logged history for pattern detection. Log outcomes (callback, screened, rejected, ghosted) on at least five applications, with at least two different outcomes among them, before re-running."`

---

## Implementation notes for V1.1 work

When V1.1 implementation begins, this file gets replaced with a real subagent definition. Key implementation considerations:

- The pattern-detector is a Subagent (lives at `.claude/agents/`) running in isolated context, but it reads MULTIPLE inputs (the full `applications/` folder), not a single CV+JD pair. The router invokes it differently from the per-audit agents.
- It produces a different output shape than `AgentVerdict` — it writes a `silence-signature.md` artefact, not a single severity-and-evidence verdict. This is the first agent that needs a non-AgentVerdict output schema. `SCHEMAS.md` will need a `SilenceSignature` shape added.
- It is invoked separately from `/ghostcheck audit` — likely as `/ghostcheck pattern` or similar slash command.
- Calibration of "what counts as a discriminating signal" benefits from V1.2's outcome-data-driven calibration. Pattern-detector and calibration are mutually reinforcing — together they are GhostCheck's compounding-value loop.
