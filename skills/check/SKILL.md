---
name: check
description: Audit and harden the current project or server against the applied-magic zero-trust standards. Maps what exists, decides the platform, checks every item with evidence, fixes what's safe and hands the human anything risky. Use when asked to harden, secure, audit or "make this secure".
---

# Harden

The standards live in `../../standards/`: `build.md`, `app-security.md`, `attacks.md`, `infrastructure.md`, `delivery.md`, `ai.md`, `engineering.md`, `operating.md`. Read the ones that apply (a backend-only repo skips mobile release; a server skips web UI). Every file opens with **The non-negotiables**: check those first, everywhere.

Tiers: **MUST** non-negotiable, **SHOULD** strong default, **EXTREME** wherever money, identity, health or family data is involved.

## Five phases

1. **Understand.** Map what actually exists, checked live rather than from docs: hosts, services, ports, databases, data categories (PII, money, health, credentials), vendors, users and roles, secrets (names only), backups and the deploy path. Note the constraints: compliance, team, budget, traffic. Read the project's `CLAUDE.md`/README for its real commands (`operating.md`, *Source-of-truth docs*); if they're missing, writing them is part of the job.
2. **Decide the platform.** Compare against *Default stack* in `build.md`. For each deviation (managed DB, serverless host, hosted auth, extra vendors), decide to keep or move it using "When managed services ARE the right call". Write a short decision record: choice, reason, risk accepted.
3. **Audit.** Go section by section. Mark each item PASS / FAIL / N/A with evidence (command + output, or file:line). An N/A needs a one-line reason.
4. **Show the report.** Change nothing before this. Use *The report* below, then always AskUserQuestion, even if the human said "fix it": **Fix the safe ones** (recommended) · **Let me pick** (multi-select of the fix list) · **Just the report**.
5. **Harden** what they chose. Fix FAILs in order MUST → SHOULD → EXTREME. Write each fix by the least-code ladder in `engineering.md` (reuse what exists; one runnable check per non-trivial fix; never weaken a gate). Do the safe, reversible fixes yourself and verify each from the outside. Add the chosen *boilerplate* items (below) in one PR.

## The report
Written for the owner, not a security engineer. Lead with the score, then the risks in plain words.

```
Score: 31 of 48 must-haves passing · 12 of 30 strong defaults
                                                      after the safe fixes → 44 of 48

What could hurt you now
1. Anyone can download your database password: /.env is served publicly.  (proof: curl → 200)
   Fix: block dotfiles in nginx. Safe, 1 line. I can do this.
2. Backups exist but have never been restored, so nobody knows they work.
   Fix: run a restore drill. Safe. I can do this.
3. Admin pages are on the open internet behind a password only.
   Fix: put them behind sign-in at the edge. Needs you: changes access for real users.

What I'll add (standard kit, missing today): security headers, log redaction, health check, CI gates, backup script
What needs you: 3 items (DNS, 2FA for users, rotating a leaked key), each with the exact step
Already good: 31 items (list on request)
```

Rules: every risk says who can do what to whom; every claim has its proof (command + output or file:line); "I can do this" only for items not in *Stop and ask*; no jargon without a five-word gloss. Keep the full PASS/FAIL table for the end or on request.

## Boilerplate kit (offered in the report, added on yes)
- `.gitignore`, `.env.example` (names only), one env module that validates config at boot and refuses to start on missing or empty secrets in production.
- Log redaction + its test, request IDs, a generic error envelope.
- Schema validation at every boundary; body size limits.
- Security headers (HSTS, nosniff, frame-ancestors, Referrer-Policy, Permissions-Policy); CSP in **report-only** first.
- Helpers: constant-time compare, CSPRNG tokens, hashed-token storage, AEAD seal/open with a key-version prefix + self-test, SSRF-safe fetch, webhook verification (raw body, timestamp window, fail closed).
- Random public IDs, timestamps, money in minor units, UTC.
- A health endpoint that checks the DB; a production smoke-test script.
- CI: frozen install, install scripts allow-listed, typecheck, lint, tests (a zero-test run fails), secret scan, dependency audit, client-bundle scan, auth-safety test; actions pinned by SHA; read-only tokens.
- Dependabot/Renovate, `CODEOWNERS` for sensitive paths, a PR template with a security checklist.
- Test skeletons: E2E for sign-up/sign-in/reset/core flow, a cross-tenant isolation test, a route-guard coverage test.
- README "Where it runs / deploy / restore"; a project `CLAUDE.md` with the real commands; `SECURITY.md` pointing to applied-magic.
- If the project owns a database: an encrypted backup script and a restore-drill script.

## Stop and ask the human first
- Turning on enforcement for real users: enforced CSP, mandatory 2FA, lockout or password rules, shorter sessions, new verification requirements.
- Auth library upgrades or provider switches; password-hash migrations.
- Encrypting existing data, backfills, key rotation, any migration that rewrites or deletes rows.
- DNS, domains, certificates, CDN/WAF, firewall, tunnel or private-network policy.
- Branch protection, CI runners, deleting branches or repos.
- Anything that sends real messages, charges money or calls paid APIs; creating accounts or adding vendors.
- Removing features, endpoints, public URLs or servers. Rotating or revoking secrets (prepare the steps; the human runs them).

## Rules
Never print secret values. Never delete data. Never weaken a control or a gate to make something work. Read before you change. Every fix must survive a reboot. Report only what you verified, per piece (`operating.md`, *Verification and reporting*).

## Finish with
The score before → after, each fix with how it was verified, the decision record; a table of remaining FAILs (tier, owner, next step, needs-approval yes/no); existing production breakage found along the way as its own ranked list; the project's docs updated to the new current state.
