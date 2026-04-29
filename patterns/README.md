# patterns/ — your personal silence signature (V1.1)

This folder is currently **empty by design**. It is reserved for the V1.1 batch pattern analysis output — the per-user "silence signature" that the V1.1 `pattern-detector` agent will produce by reading your accumulated audit history.

In V1, this folder contains only this README and a `.gitkeep`. No silence-signature artefacts are written here yet.

## What V1.1 will produce here

When `pattern-detector` (`.claude/agents/V1_1-pattern-detector.md` — currently a placeholder) is implemented and you have logged outcomes on at least 5 applications in the `applications/` folder, the V1.1 system runs `/ghostcheck pattern` (or equivalent) and writes:

```
patterns/<user>/silence-signature.md
```

The artefact summarises:

- **Dominant silence drivers** — which one or two agents fire HIGH or CRITICAL across most of your ghosted applications. These are your structural failure modes.
- **Discriminating signals** — agents whose verdicts predict outcome (HIGH on ghostings vs LOW on callbacks). These are the dimensions where targeted improvement has the highest expected callback impact.
- **Channel-outcome correlation** — your personal callback rate by channel. Population benchmarks tell what should work; this tells what does work for YOU.
- **Tier-target outcome correlation** — whether your CV reads at-tier for one target level but not the next.
- **Prioritised action list** — the top three changes that, if made, would have the highest expected impact on your future callback rate.

This is genuinely novel functionality. No existing CV checker offers cross-application pattern analysis because no existing CV checker stores audits as durable, schema-faithful, machine-readable artefacts. GhostCheck V1's `applications/<slug>/` folder structure is what makes V1.1's pattern detection possible.

## Why this is a separate folder from `applications/`

`applications/` holds raw per-audit data — verdicts, contexts, outcomes, channels. It is the input to pattern detection.

`patterns/` holds derived artefacts — silence signatures and longitudinal analyses. It is the output of pattern detection.

Keeping them separate means re-running pattern detection later (after more applications accumulate) does not pollute the per-audit history. You can also git-track the signature files (they are user-derived insight, not raw application data) once V1.1 ships, while `applications/*/` stays gitignored.

## Privacy posture for V1.1 silence signatures

By default, `patterns/<user>/` will be **user-private** — gitignored, on-disk only, never shared.

Optional opt-in (V1.2 placeholder, not yet implemented): users can choose to contribute anonymised signatures to a shared corpus that powers cross-user heuristics in V1.2's wiki layer (see `wiki/README.md`). The opt-in mechanism is `config/share_outcomes.yml` — currently a V1.2 placeholder, not active in V1 or V1.1.

## What to do in V1

Nothing — this folder is empty until V1.1 ships. You can ignore it.

When V1.1 lands, follow the activation criteria in `.claude/agents/V1_1-pattern-detector.md` (at least 5 applications with logged outcomes, at least 2 different outcomes). Until then, focus on logging outcomes and channels in `applications/<slug>/` per the convention in `applications/README.md`.

The `pattern-detector` placeholder file documents the V1.1 design intent in detail. Read it for the full specification.
