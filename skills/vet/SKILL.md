---
name: vet
description: Review the current diff or PR against the applied-magic standards that match the files it touches (auth, payments, permissions, encryption, migrations, uploads, webhooks, outbound messaging, infra, CI). Use before merging, or when asked for a security review of changes.
---

# Security review

1. Get the diff against the merge-base of a freshly fetched base branch (or the PR), including untracked source files. List the touched areas, then read beyond the diff: callers of every changed function and its tests.
2. Map the areas to the standards in `../../standards/` (start with each file's *The non-negotiables*):
   - secrets, auth, sessions, OAuth, permissions, uploads, webhooks, money, messaging, integrations, encryption → `app-security.md`
   - any new endpoint, form, input, file handling, redirect or fetch → `attacks.md`
   - migrations, schemas, IDs, money/time types, imports, bulk changes → `build.md` (*Data model*, *Data pipelines and outbound pools*, *Data integrity and bulk changes*) + `delivery.md` (*Migrations*)
   - CI, deploy, dependencies, lockfiles, mobile release → `delivery.md`
   - infra, Docker, nginx, DNS, backups → `infrastructure.md`
   - AI calls, agents, prompts over untrusted content → `ai.md`
   - any test, gate or error-handling change → `engineering.md` (*Code review*, *Testing*, *Errors and failure signalling*)
3. For every relevant check, mark PASS / FAIL / N/A against **the diff**, with file:line evidence. Check what the diff *should* have added too (missing tests, a missing auth check on a new route, a missing index or constraint), and treat each finding as a defect class: grep sibling entry points for the same pattern.
4. Flag every gate-relaxing hunk (skipped tests, raised timeouts, suppressions, `continue-on-error`) as needing a named human decision.
5. Open with one line: how many findings by severity, and whether it's safe to merge. Then findings most severe first: what an attacker or bug could do, in plain words, tier, file:line, the concrete failure scenario, the smallest fix. Data loss, trust-boundary bypass and uncapped spend are critical on sight. No style nits. End with "Not checked:".

Don't modify code unless asked. If something needs a human decision (from "Stop and ask the human first" in `../check/SKILL.md`), say so.
