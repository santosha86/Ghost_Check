# ZERO_TRUST.md — Trust and Security Posture

*Why GhostCheck treats its own agents with the same suspicion it would treat an external API, and what that means in practice.*

---

## The thesis in one paragraph

Classical Zero Trust says: don't trust anything implicitly, even things inside your perimeter — verify every access, grant the minimum privileges needed, and design as if an attacker is already inside. That principle was born for corporate networks (Forrester, 2010; codified in NIST SP 800-207). In March 2026, Microsoft extended it to multi-agent AI systems under the label *Zero Trust for AI*. The three principles stay the same — **Verify explicitly**, **Apply least privilege**, **Assume breach** — but the actors change. The thing you now refuse to trust implicitly is not the firewall or the session token. It is your own agent. Every rule in this document exists because we applied those three principles to an 11-agent pipeline and worked backwards from them.

---

## Zero Trust for AI, in one paragraph

In classical Zero Trust you do not trust a packet because it is inside the firewall. In Zero Trust for AI you do not trust a verdict because it came from an agent you wrote. Your own agent can be manipulated through a cleverly-worded JD (indirect prompt injection, or XPIA — Cross-Prompt Injection Attack). It can be overprivileged, reaching a data source it should not touch. It can "leak" opinions into another agent's reasoning if you let them see each other's outputs, a multi-agent analog of lateral movement. Microsoft names this class of failure **"double agents"** — agents that look like they work for you but are actually acting on someone else's inputs. Zero Trust for AI is how you keep your agents from quietly becoming double agents.

---

## The three principles, applied to GhostCheck

Microsoft's three principles map onto the four concrete rules already listed in `CLAUDE.md` and `SCHEMAS.md`. This section shows the mapping plainly, so any contributor can see which principle a rule defends and which failure it prevents.

| GhostCheck rule                                                  | Microsoft principle it implements                 | Failure mode it prevents                                                 |
|------------------------------------------------------------------|---------------------------------------------------|--------------------------------------------------------------------------|
| Schema validation + fail-closed coercion on every verdict        | **Verify explicitly**                             | Confidently-wrong output from a manipulated or misaligned agent.         |
| Capability declarations in YAML frontmatter (per agent)           | **Apply least privilege**                         | Unauthorized data exposure; agents reaching tools they were never granted. |
| Agents cannot call each other; fan-out only                      | **Assume breach**                                 | Lateral movement — one agent's opinion leaking into another's reasoning. |
| Fail-closed on schema violation (UNKNOWN, not hallucinated)      | **Assume breach** + **Verify explicitly**         | Indirect prompt injection (XPIA) producing plausible-but-invalid output. |

Each principle is expanded below.

---

### Principle 1 — Verify explicitly

Every verdict produced by every agent is validated against the `AgentVerdict` schema in `SCHEMAS.md` before it is allowed to reach the aggregator. The verification is structural (required fields, correct types, score within range), typed (severity must be one of the five enum values), and evidential (non-UNKNOWN verdicts must carry at least one `Evidence` item). A verdict that fails any of these checks is not "repaired" by the harness — it is coerced to `UNKNOWN` with a schema-violation reason, exactly as specified in `SCHEMAS.md section 12`.

The significance of this is that the harness refuses to distinguish between a malicious agent and a buggy one. Both produce verdicts that fail validation. Both are handled identically: UNKNOWN, with the reason surfaced to the user in the audit's "Not assessed" section. No special casing. No silent recovery. This is the point of verification-explicit: the system does not care whether a bad verdict came from an attacker or from an honest mistake — the response is the same.

> *A verdict is not trustworthy because it came from one of our own agents. It is trustworthy because it validates.*

---

### Principle 2 — Apply least privilege

Every agent declares its capabilities in YAML frontmatter at the top of its `.md` file. A capability is the permission to do something beyond reading its declared inputs — typically, to call an external enrichment skill. The declaration is explicit, readable, and load-bearing.

Concretely:

- `google-test` declares `capabilities: [web_search]`. It is the only agent that may call the `google-test-lookup` enrichment skill. When it runs, it does.
- `bucket-classifier` declares `capabilities: []`. It reads the CV, the JD, and `ExternalContext`. It may not call web search, may not call the company enricher, may not touch anything else. If the agent prompt somehow asks for a web search anyway — through prompt injection, or contributor error — the harness must refuse. In V1 this refusal is by convention (Claude Code honors the frontmatter); in V1.2+ a runtime policy engine enforces it unconditionally (see section 5 below).

Capabilities are *granted by enumeration*, not *denied by blacklist*. If a capability is not in the list, it is not available. The default is nothing. This matters because it inverts the common anti-pattern where agents have broad access "just in case" and then get their permissions trimmed later. GhostCheck trims nothing. Each agent starts with zero and is granted exactly what it needs, line by frontmatter line.

> *A capability declared in frontmatter is a capability granted explicitly. Everything else is denied by default. That is least privilege, applied to prompts.*

---

### Principle 3 — Assume breach

Two GhostCheck rules implement the "assume breach" posture.

**No agent-to-agent calls.** Agents in GhostCheck run fan-out, not chain. No agent ever sees another agent's output; each one receives only the CV, the JD, and the read-only `ExternalContext` enrichment blob. The aggregator is the only component that sees all verdicts, and it is deterministic math, not a reasoning step. This is lateral-movement prevention: if `bucket-classifier` is ever compromised by a crafted CV — say, a CV containing adversarial text designed to produce a misleading verdict — that compromise cannot propagate to `ats-simulator` or `recruiter-30sec`, because those agents never read `bucket-classifier`'s output. In a multi-agent system, *lateral movement isn't a network hop — it's one agent's opinion leaking into another's reasoning.* Fan-out breaks the path.

**Agents cannot write to user files.** Agents produce verdicts. That is all. They do not edit `profile/cv.md`, `profile/style.md`, `config/*`, or the JD file. They do not append to `applications/` directly. The aggregator alone decides what gets persisted under `applications/YYYY-MM-DD_<slug>/`. A compromised agent cannot silently modify the user's CV to "fix" what it reported as a bug. A misaligned agent cannot overwrite its own audit output. The data boundary is hard because the attack boundary must be assumed hard.

These two rules together mean: even if one agent is fully manipulated — by indirect prompt injection embedded in a JD, by a crafted CV line, by a contributor accidentally publishing a poisoned prompt — the blast radius is bounded. The damaged agent produces one damaged verdict. That verdict shows up in the final report, cited as evidence the user can see and question. It does not silently influence ten other agents and it does not write to disk.

> *Assume breach means: one agent going wrong should never quietly ruin the audit. It should show up in the report, cited, arguable.*

---

## V1 enforcement: design-time. V1.2+: runtime.

Be honest about this: GhostCheck V1 enforces these principles by **convention and schema validation**, not by a runtime policy engine.

What that means:

- **Frontmatter capabilities are honored by Claude Code reading the markdown.** A contributor who writes a rogue agent and grants themselves `capabilities: [web_search, filesystem_write, shell]` would self-document their own violation, but there is no runtime layer in V1 that would stop them from actually running with those capabilities if Claude Code chooses to honor them.
- **Schema validation happens at the aggregator boundary.** Any `AgentVerdict` that does not validate is coerced to UNKNOWN. This is the strongest V1 enforcement we have and it works at runtime.
- **Fan-out is a convention of the router** (`.claude/skills/ghostcheck/SKILL.md`). Nothing in V1 prevents a future router revision from daisy-chaining agents. The discipline is maintained by the contract, not by a kernel.

V1.2+ upgrades this to **runtime policy enforcement**:

- A capability-declaration policy engine that verifies, before any enrichment call, that the agent requesting the capability actually declared it. Unauthorized calls fail at execution time, not at code review time.
- An audit trail: every capability invocation logged with agent ID, timestamp, inputs hash, output hash. This is the provenance chain that Microsoft's observability guidance (April 2026) calls for.
- Schema version pinning on every verdict so a contract change in `SCHEMAS.md` does not silently break older agents.

The V1 posture is therefore *"design principles enforced by convention, with schema validation at the boundary."* Call it what it is. The full runtime policy engine is documented in `ROADMAP.md` under V1.2. This file is honest about the gap because a Zero Trust posture that lies about its own enforcement level is not a Zero Trust posture — it is marketing.

> *Zero Trust is not a thing you *have*. It is a thing you *enforce*. V1 enforces it on the design surface. V1.2 enforces it on the execution surface.*

---

## What Zero Trust in GhostCheck does NOT promise

A Zero Trust claim without scope limits is a lie of omission. Four things this posture explicitly does not do:

**It does not hide your CV from Claude.** You send your CV text to the Claude API every time an audit runs. That is the contract of using an LLM-backed tool. Anthropic's privacy terms govern what happens to that text on their side. If the contents of your CV are sensitive beyond what Anthropic's terms cover, redact before you submit. Zero Trust here protects against agents misusing the CV. It does not protect the CV from being read by the underlying model.

**It does not fully defend against indirect prompt injection (XPIA).** A JD can contain text crafted to manipulate the agents reading it. V1's defences are partial but layered — five mechanisms that compound:

- **Fan-out architecture** — compromise of one agent cannot propagate; the other ten agents reading the same JD form independent verdicts.
- **Schema validation** — structurally malformed output is rejected at the aggregator boundary.
- **Fail-closed coercion** — any rejected verdict is converted to UNKNOWN with a reason; never patched up.
- **Capability declarations** — a compromised agent cannot reach tools its frontmatter does not grant.
- **Evidence-citation requirement** — every non-UNKNOWN verdict must carry at least one quoted `Evidence` item with source and location, raising the bar for forged verdicts.

These five together significantly reduce the blast radius, but do not eliminate the risk. A cleverly-crafted JD can still make one agent return a wrong but schema-valid verdict; that verdict will appear in the final audit, cited, and the user can see and dismiss it. **V1.1 will add evidence verification** — every cited quote must literally exist in the source it claims to come from, which catches the most common forged-quote attack. **V1.2+** adds input sanitisation and adversarial-text detection. V1 relies on the user's own reading of the evidence as the final defence.

**It does not protect against a malicious fork.** Anyone can fork GhostCheck, strip the frontmatter capability declarations, rewrite the aggregator to be an LLM, delete the schema validation step, and call the result "GhostCheck." The rules in this document govern *this repository*. A user who runs a fork is trusting whoever published the fork, not us. This is the inherent limit of open-source security posture and is the same limit every open-source project faces.

**It does not log or audit trust decisions.** V1 has no audit trail of which agents ran, which capabilities were invoked, or which verdicts were coerced. The `audit.md` shows the user what the aggregator concluded, and `verdicts.json` shows the raw per-agent output — but nothing logs the capability-invocation chain. Microsoft's April 2026 observability guidance (see `ROADMAP.md`) is the target for V2 when GhostCheck moves into shared/team usage.

---

## Standards chain

This posture is not novel. It is the industry convergence applied to one specific multi-agent system. The chain of references:

- **NIST SP 800-207** — the canonical Zero Trust Architecture specification. Classical networking origin.
- **CISA Zero Trust Maturity Model** — U.S. federal guidance on maturing a Zero Trust posture. Pillar structure.
- **CIS Controls** — baseline security controls referenced in Microsoft's assessment framework.
- **Microsoft Secure Future Initiative (SFI)** — Microsoft's enterprise program that grounds their agent-specific extensions.
- **Microsoft Zero Trust for AI** (March 2026) — the AI-agent adaptation of the three Zero Trust principles that this document directly aligns to.

Microsoft's *Zero Trust Assessment for AI* pillar is in development (summer 2026 per their announcement). When that ships, this file can be updated with concrete control-level mapping. Until then, GhostCheck aligns with the three *principles* — Verify explicitly, Apply least privilege, Assume breach — not with a specific control list that does not yet exist. That is the honest claim. Anything stronger would be fabricated compliance.

> *The prompts are leaves. The harness is the trunk. The Zero Trust posture is the fence around the tree.*
