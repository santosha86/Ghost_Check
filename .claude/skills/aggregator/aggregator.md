---
name: aggregator
description: Combines validated AgentVerdicts from the GhostCheck agents into a callback probability, ranks the silence drivers, and writes the human-readable audit report plus the raw verdicts and context JSON to applications/YYYY-MM-DD_<slug>/. Deterministic math, no LLM calls. Invoked by the /ghostcheck router after all agents have returned and their verdicts have been validated against the AgentVerdict schema.
allowed-tools:
  - Read
  - Bash
  - Write
---

# Aggregator skill — combine verdicts, write audit

When invoked by the /ghostcheck router, follow these eight steps in order. Fail closed on any step: surface a clear error and halt. The aggregator is the system's anti-hallucination layer; it must stay deterministic at all times. Do not call any LLM, do not "interpret" or "smooth" the verdicts, do not invent silence drivers.

## Inputs

The router passes the following to the aggregator:

- The list of validated AgentVerdict objects (each conforming to `docs/SCHEMAS.md` section 1).
- The path to `config/weights.yml`.
- The audit context bundle: `cv_text`, `jd_text`, `user_profile`, `external_context`.
- The CV path (for the audit report header).
- The JD path (for the audit report header and slug derivation).

## Step 1 — Load and validate weights

Use the Read tool to load `config/weights.yml`. Parse it as YAML.

Validate:

- All eleven agent names are present as keys: `bucket-classifier`, `google-test`, `posting-decoder`, `it-services-discount`, `headline-filter`, `funnel-math`, `channel-mix`, `stale-detector`, `ats-simulator`, `recruiter-30sec`, `hm-deep-read`.
- Each value is a numeric weight in `[0.0, 1.0]`.
- The eleven weights sum to `1.0` within a tolerance of `0.001`.

If any check fails, halt with: `"weights.yml validation failed: <specific reason>. Halt audit."`

## Step 2 — Partition verdicts into S (non-UNKNOWN) and U (UNKNOWN)

Walk the list of verdicts:

- If `severity == "UNKNOWN"`, add to set U.
- Otherwise, add to set S.

If S is empty (all agents returned UNKNOWN), skip ahead to the audit-report write step but use the special probability marker `null` and the report text `"Insufficient data to produce a probability — every agent returned UNKNOWN."` Do not attempt to compute a number.

## Step 3 — Redistribute weights across non-UNKNOWN agents

For each agent in S, compute its effective weight:

```
effective_weight(agent) = weights[agent] / sum(weights[a] for a in S)
```

This redistributes the missing-agents' weight proportionally across the agents that did report. The effective weights across S sum to 1.0 by construction.

(See `docs/SCHEMAS.md` section 13 for the canonical specification of weight renormalisation on UNKNOWN.)

## Step 4 — Compute the weighted severity sum (z)

Map each verdict's severity to a numeric value per `docs/SCHEMAS.md` section 13:

| Severity | Value |
|---|---|
| CRITICAL | 1.0 |
| HIGH | 0.75 |
| MEDIUM | 0.5 |
| LOW | 0.25 |

Compute:

```
z = sum(effective_weight(agent_i) * severity_value(verdict_i))   for i in S
```

z lands in `[0.25, 1.0]` because the lowest possible severity per agent is `0.25` (the value for LOW), so the weighted sum cannot fall below `0.25` even when every agent says LOW.

## Step 5 — Apply the weighted-logistic to get callback probability

V1 formula and hyperparameters (locked 2026-04-25; tuned by V1.2 calibration loop, not redefined):

```
callback_probability = 1 / (1 + exp(k * (z - midpoint)))

k        = 4.0      # steepness
midpoint = 0.4      # severity-sum value where probability is 50%
```

Compute it via Bash with `bc -l` (no Python needed for this step):

```bash
z=<computed value of z, e.g. 0.5>
k=4.0
midpoint=0.4
prob=$(echo "scale=4; 1 / (1 + e($k * ($z - $midpoint)))" | bc -l)
```

Round the result to four decimal places. Convert to a percentage with one decimal for display in the audit (e.g. `0.4012` becomes `40.1%`).

What the formula yields at the four corners (sanity check):

| All-agent severity | z | callback_probability |
|---|---|---|
| All LOW | 0.25 | about 0.64 |
| All MEDIUM | 0.50 | about 0.40 |
| All HIGH | 0.75 | about 0.20 |
| All CRITICAL | 1.00 | about 0.08 |

## Step 6 — Rank silence drivers

Sort the verdicts in S by these keys, in this order:

1. Severity value descending (CRITICAL first, then HIGH, then MEDIUM, then LOW).
2. Within the same severity, score descending.
3. Stable secondary tie-break by agent name alphabetically.

Take the top 3 to 5 entries (use 3 if there are exactly 3 or fewer above LOW; otherwise show up to 5). These become the "Top silence drivers" section in the audit.

## Step 7 — Derive audit folder path

Compute the slug from the JD filename:

1. Take the basename of the JD path.
2. Strip the extension.
3. Lowercase the entire string.
4. Replace any non-alphanumeric character with a hyphen.
5. Collapse consecutive hyphens to a single hyphen and strip leading or trailing hyphens.

Example: `jobs/Strategy & AI Architect - Chief.pptx` becomes `strategy-ai-architect-chief`.

Compute the date prefix: today's date in `YYYY-MM-DD` ISO format.

Compose the folder path: `applications/<date>_<slug>/`.

Use Bash to check if the folder already exists. If it does, append `_2`, `_3`, etc. until a free name is found (per `docs/SCHEMAS.md` section 10 slug-collision rule). Use Bash `mkdir -p` to create the folder.

## Step 8 — Write the three output files

### audit.md (the human-readable audit report)

Follow `docs/SCHEMAS.md` section 11 contract exactly. Use the Write tool. The structure:

```markdown
# Audit — <role title from JD> @ <company from JD>

**Date:** <YYYY-MM-DD>
**CV:** <CV path>
**JD:** <JD path>
**Callback probability:** <NN.N>%

## Top silence drivers

1. **<SEVERITY>** — <agent_id>: <verdict>
   - Evidence: <first evidence text> (<location>)
   - Fix: <first fix_hint>

2. ... (next driver)

## Not assessed

<For each verdict in U:>
- **<agent_id>** — <unknown_reason>

<If U is empty, write: "None — all agents produced verdicts.">

## Full verdicts (appendix)

<For each verdict, in S then U:>

### <agent_id>
- Severity: <SEVERITY>
- Score: <score>
- Verdict: <verdict text>
- Evidence:
  - "<text>" (<source>:<location>)
  - ... (each evidence item)
- Fix hints:
  - <fix_hint 1>
  - ... (each fix hint)
<For UNKNOWN verdicts include:>
- Unknown reason: <unknown_reason>

---

*Audit produced by GhostCheck V1. Callback probability computed via weighted-logistic over 11-agent verdicts: P = 1 / (1 + exp(4.0 * (z - 0.4))) where z is the weight-renormalised severity sum. Hyperparameters are V1 defaults; V1.2 calibration learns them from real callback outcomes.*
```

If the role title or company cannot be extracted from the JD frontmatter or first heading, use the JD filename slug as a fallback for the report header (without breaking the contract).

### verdicts.json

Use the Write tool to save the array of validated AgentVerdict objects as pretty-printed JSON (2-space indent). One file, all 11 verdicts (or however many were produced).

### context.json

Use the Write tool to save the ExternalContext object that was passed in, plus the CV and JD paths and today's date, as pretty-printed JSON. This is the auditability snapshot — anyone reviewing the audit can see exactly what context the agents reasoned over.

## Step 9 — Report to the user

Print exactly: `Audit complete: applications/<date>_<slug>/audit.md`

The router relays this to the user.

## Behavioural rules (invariants)

- **Deterministic always.** Same eleven verdicts plus same weights plus same date plus same JD filename always produce the same callback probability and the same files. No LLM calls, no randomness, no "interpretation."
- **The aggregator is the only writer to `applications/`.** No other skill or agent may create or modify files there.
- **No retry on agent failure.** UNKNOWN comes in, UNKNOWN stays. Aggregator does not re-prompt failed agents; that is outside its responsibility.
- **Schema-faithful audit.md.** The structure follows `docs/SCHEMAS.md` section 11 exactly. Section headers, ordering, and field labels match. Future tooling (the V1.1 PNG card renderer, the V2 web frontend) parses these reports by section header.
- **Hyperparameters live in this file.** If `k` or `midpoint` change across versions, the change is documented in the audit's footer note so audits stay reproducible from their own metadata.
- **No silent skip on missing fields.** If a verdict is missing required fields, halt — never extrapolate. Validation should have caught this at the router; the aggregator trusts the router's pre-validation but defends defensively.
