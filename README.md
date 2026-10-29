# Applied Magic

Engineering standards as a **Claude Code plugin**, to stop people and their agents from building bad apps: secure by
default, zero trust, least code, easy to hand off. Every check came from real systems: production audits, incidents and
near-misses. **815 checks**, tiered **[MUST]** non-negotiable · **[SHOULD]** strong default · **[EXTREME]** for money,
identity, health or family data. Each file opens with **The non-negotiables**, the few MUSTs whose miss causes a breach,
data loss, an outage or money/messaging harm.

## Install

```
/plugin marketplace add devinpicciolini/applied-magic
/plugin install applied-magic@applied-magic
```

## Skills

Start with **`/applied-magic:magic`** (or just say what you want): it looks at where you are, asks one click-through question, and runs the right skill.
For something new it asks four quick questions (what, who, what data, where it runs) and proposes a stack you accept or change.

**You see it before it touches anything.** Every skill that changes things first shows a plain-English report: a score
(e.g. *31 of 48 must-haves passing*), what could hurt you right now with proof, what it will fix and what that gets you,
and what only you can do. Then you pick: fix the safe ones, pick which, or just keep the report. It ends with the score
before → after. Anything touching real users, DNS, data, money or secrets waits for you, with the exact step to run.

| Command | What it does |
|---|---|
| `/applied-magic:magic` | The front door: works out what you want and runs the right skill |
| `/applied-magic:check` | Score the project or server with proof, then fix what's safe once you say go |
| `/applied-magic:start` | Start a project right: plan small, default stack, day-one defaults, the full boilerplate kit |
| `/applied-magic:vet` | Review the current diff against the standards its files touch |
| `/applied-magic:attacks` | Walk every attack class and prove each defense |
| `/applied-magic:ship` | Pre-deploy gate: pick the lane, verify the gates, verify the live result |
| `/applied-magic:lesson` | Turn an incident into a new generic check in the right file |
| `/applied-magic:map` | Map a project or server from live checks into a `CLAUDE.md` with real commands |

## Standards

| File | What's in it |
|---|---|
| [`standards/build.md`](standards/build.md) | Zero trust, entity separation, the default stack (own the box), **which tool when, and why**, day-one data model, pipelines and bulk-change safety |
| [`standards/app-security.md`](standards/app-security.md) | Secrets, auth, sessions, OAuth, authorization, web hardening, encryption, money, messaging, integrations |
| [`standards/attacks.md`](standards/attacks.md) | Every attack class → the defense to prove |
| [`standards/infrastructure.md`](standards/infrastructure.md) | Inventory, network, egress, DNS, hosts, processes, backups, failover, monitoring, migrations, people |
| [`standards/delivery.md`](standards/delivery.md) | CI/CD lanes and gates, deploy verification, migrations, supply chain, mobile release |
| [`standards/ai.md`](standards/ai.md) | AI features and agents: spend, kill switches, model output, agent scope, AI-written messages, evals |
| [`standards/engineering.md`](standards/engineering.md) | Least code, planning, design and code review, testing, debugging, QA, performance, shipping |
| [`standards/operating.md`](standards/operating.md) | Running a coding agent: instructions file, memory, docs, skills, verification, approvals |
| [`standards/architecture.md`](standards/architecture.md) | Contract-first APIs: one schema → spec → typed clients; errors, pagination, IDs, layering, data layer, boundaries |

## How to use this with an agent

With the plugin, run the skills. Without it, paste this into the agent working on a project or server:

> Make this project/server meet the applied-magic standards (zero trust, least code). Work in five phases:
>
> 1. **Understand.** Map what actually exists, checked live rather than from docs: hosts, services, ports,
>    databases, data categories (PII, money, health, credentials), vendors, users and roles, secrets
>    (names only), backups, deploy path. Note the constraints: compliance, team, budget, traffic.
> 2. **Decide the platform.** Compare with *Default stack* in `build.md`. For every deviation, decide keep or
>    move using "When managed services ARE the right call". Write a decision record: choice, reason, risk accepted.
> 3. **Audit.** Start with each file's non-negotiables, then every section. Mark each item PASS / FAIL / N/A
>    with evidence (command + output, or file:line). An N/A needs a one-line reason.
> 4. **Report, then ask.** Change nothing yet. Show a score (MUSTs passing), what could hurt me now in plain
>    words with proof, what you'd fix and what needs me. Wait for my go.
> 5. **Harden.** Fix FAILs MUST → SHOULD → EXTREME, each by the least-code ladder in `engineering.md`. Do the
>    safe, reversible fixes yourself and verify each from outside. For anything destructive, outward-facing, or
>    touching money, secrets, DNS, auth for real users or production availability, stop and give the exact
>    command for a human to run.
>
> Rules: never print secret values; never delete data; never weaken a control or a gate to make something work;
> read before you change; every fix must survive a reboot. Finish with the decision record, a table of remaining
> FAILs with owner and next step, and the project's docs updated to the new current state.

**When to run what**
- **Day 0:** `start` adds the boilerplate kit (listed in `skills/check/SKILL.md`) in the first commits, before features.
- **First touch of an existing project:** the five phases once, then **one** PR adding every missing boilerplate item, plus a ranked to-do.
- **Every PR touching auth, payments, permissions, encryption, migrations, uploads, webhooks or outbound messaging:** `vet`, noted in the PR.
- **Before a release, sale or audit, or monthly:** the full run again, dated.
- **After any incident or near-miss:** `lesson`.

For a project's `CLAUDE.md`:
> Standards: applied-magic (zero trust, least code). New code follows `build.md` day-one defaults and the
> `engineering.md` ladder. Changes touching auth, payments, permissions, encryption, migrations, uploads, webhooks
> or outbound messaging re-check the related sections and note the result in the PR. Never weaken a control to
> make something work; ask instead.

## Quick pass (30 minutes)

1. `ss -ltnup` on every host: anything public that shouldn't be?
2. Request `/.git/HEAD`, `/.env`, `/*.map` on every public site.
3. Backups fresh, encrypted, off-site, and when was the last restore drill?
4. Any secret printed or committed recently? Rotate it.
5. Pending security updates or reboot-required on any host?
6. Is the login rate limit keyed on the real client IP?
7. Webhooks fail closed? Payments idempotent?
8. Any dangling DNS records?
9. Replication slots capped, replicas checked by LSN?
10. Anything running that nobody can explain?

## Contributing

Use `/applied-magic:lesson`, or by hand: one verifiable check per line, starting with its tier; no project names,
hosts, people or secrets; sharpen an existing line before adding a new one; generalize so the next person catches it
*before* it happens.

## License

MIT
