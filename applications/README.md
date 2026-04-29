# applications/ — your audit history

This folder is where GhostCheck writes audit results — one folder per audit run. Everything inside subfolders (`applications/<slug>/*`) is **gitignored** because it contains your real CV+JD content and your real callback outcomes.

## What lives here

When you run `/ghostcheck audit --cv profile/cv.md --jd jobs/<slug>.md`, the aggregator creates a folder named:

```
applications/YYYY-MM-DD_<jd-slug>/
```

Where:

- `YYYY-MM-DD` is the ISO date of the audit run.
- `<jd-slug>` is derived from the JD filename — lowercase, alphanumeric only, hyphens for word boundaries.

For example: `applications/2026-04-29_strategy-ai-architect-chief/`.

If you re-run the audit on the same JD on the same day, the slug gets `_2`, `_3`, etc. appended (per `docs/SCHEMAS.md` section 10's slug-collision rule).

Each audit folder contains three machine-written files:

| File | Written by | Purpose |
|---|---|---|
| `audit.md` | aggregator | The human-readable audit report (per `docs/SCHEMAS.md` section 11). Top silence drivers, full per-agent verdicts, callback probability, fix hints. |
| `verdicts.json` | aggregator | The raw `AgentVerdict` objects from all eleven agents, schema-validated. Tooling-friendly. |
| `context.json` | aggregator | The `ExternalContext` snapshot used for this audit (company classification, JD age, google_results, etc.). Auditability — you can see exactly what context the agents reasoned over. |

## Optional user-written files (these activate funnel-math and channel-mix)

Two more files in each audit folder are **user-written** — you create them when you have data to log. They activate the Tier B agents (`funnel-math`, `channel-mix`) which would otherwise return UNKNOWN.

### `applications/<slug>/outcome.md`

A one-line file recording what happened with this application. Format:

```
outcome: <callback | screened | rejected | ghosted>
date: YYYY-MM-DD
notes: <optional one-line note about why or how>
```

Example:

```
outcome: ghosted
date: 2026-05-15
notes: No acknowledgement after 30 days from application. Followed up once via the recruiter's LinkedIn, no response.
```

Allowed `outcome` values:

- **`callback`** — a recruiter or hiring manager reached out for next steps (screen, interview, etc.).
- **`screened`** — you got a screening call but did not advance.
- **`rejected`** — explicit rejection email or message received.
- **`ghosted`** — no response of any kind after a reasonable wait (typically 30 days).

The `funnel-math` agent reads these across all your applications to compute callback rate vs senior-tier benchmarks. It returns UNKNOWN until you have at least 8 applications logged.

### `applications/<slug>/channel.md`

A one-line file recording how you applied. Format:

```
channel: <easy-apply | referral | dm | exec-search | recruiter-inbound | company-career-page>
date: YYYY-MM-DD
notes: <optional one-line context, e.g. who referred you>
```

Example:

```
channel: referral
date: 2026-04-15
notes: Referred by Priya R., now Director of AI at the firm; she submitted my CV through their internal portal.
```

Allowed `channel` values:

- **`easy-apply`** — LinkedIn Easy Apply or equivalent one-click submission.
- **`referral`** — someone inside the firm submitted or endorsed your application.
- **`dm`** — you direct-messaged a hiring manager, partner, or senior contact at the firm.
- **`exec-search`** — your application went via an executive search firm or named senior recruiter.
- **`recruiter-inbound`** — a recruiter reached out to you first; this application started with them.
- **`company-career-page`** — you applied via the company's own careers page (not Easy Apply, not referral).

The `channel-mix` agent reads these across all your applications to assess whether your channel distribution matches what works at your target seniority. Together with `funnel-math`, it provides the funnel-level diagnosis that single-application audits cannot.

## Logging rhythm — the practical advice

When you finish an application, take 30 seconds to drop in `channel.md` immediately. The channel is fresh in your mind. Logging it later is harder.

When you receive an outcome (or 30+ days have passed with no acknowledgement), drop in `outcome.md`. Don't wait — the agents only learn from logged data.

After 8-10 logged applications, your audits will start showing real funnel-level findings instead of the UNKNOWN-by-default state. After 15-20 applications, the V1.1 pattern-detector (when it ships) can identify your personal silence signature with statistical meaningfulness.

## Privacy

Everything in `applications/<slug>/` is gitignored — `audit.md`, `verdicts.json`, `context.json`, and the user-written `outcome.md` and `channel.md` files. Your audit history never leaves your machine unless you explicitly choose to share it.

The exception: `applications/.gitkeep` and `applications/README.md` (this file) ARE committed so the folder structure is visible to forkers cloning the repo.

## What V1.1 will add

When `pattern-detector` (V1.1) ships, it reads your accumulated `applications/*/audit.md`, `verdicts.json`, `outcome.md`, and `channel.md` files to identify cross-application patterns — your personal silence signature. The output lands at `patterns/<user>/silence-signature.md`. See `patterns/README.md` for what that looks like.

Until then, this folder is per-audit history; the patterns layer comes in V1.1.
