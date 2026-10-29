---
name: ship
description: Pre-deploy gate for the current change using the applied-magic delivery standards. Picks the lane (normal, hotfix, website fast lane), runs or verifies every required gate, and confirms post-deploy checks. Use when asked to ship, deploy or release.
---

# Ship

Read `../../standards/delivery.md` and the *Shipping* section of `engineering.md`.

1. **Pick the lane.** Website/content-only goes in the fast lane. A production incident fix goes in the hotfix lane (same security gates). Everything else is the normal lane. Auth, payments, permissions, encryption, migrations, outbound messaging, `.github/`, deploy scripts or lockfiles always take the full normal lane plus a code-owner review.
2. **Prepare the branch.** Merge the latest base, run the full suite, and classify any failure in-branch vs pre-existing by running it on base (unsure = in-branch). Audit the plan: each item done / partial / not done / changed. Stage files by name; never `--no-verify`.
3. **Verify the gates** for that lane: typecheck, lint, tests (a lane that ran zero tests fails), secret scan, dependency audit, client-bundle scan, auth-safety and isolation tests, E2E for money and identity paths, migrations backward-compatible (expand, migrate, contract), a backup before any destructive migration, and no gate-relaxing hunk without a named human decision. If the diff touches `.github/`, deploy scripts or a lockfile, also check *Workflows and runners* and *Supply chain* (permissions, SHA pins, no PR code with secrets, install scripts, registry URLs).
4. **Merge and deploy** only the main tip after green CI on the exact head SHA, through the pipeline. Never from a laptop, a hand edit on a server or a workflow dispatched on a branch. Merge as its own step after checking the checks' exit code (never piped through `tail`); if a merge or deploy queue stops, restart it or say so before you finish.
5. **After deploy:** match the deploy run to the merge SHA, check the live build ID, run the smoke test (health incl. DB, sign-in page, one core read) and compare key pages to the pre-deploy baseline; make sure automatic rollback is armed. Without that evidence, report "deployed (unverified)".
6. **Mobile:** run *Mobile release* in `delivery.md`; the store can't roll back, so use phased or staged rollout and confirm the build reached testers and, after approval, the public storefront.

If any gate is red, stop and report. Never skip a gate to ship faster; make the gate faster instead.
