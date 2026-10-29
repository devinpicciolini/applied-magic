---
name: attacks
description: Walk every attack class in the applied-magic attack reference against the current app and prove each defense with evidence (requests, code, config). Use for a pre-launch check, a pentest-style pass, or "are we vulnerable to X".
---

# Attack check

Read `../../standards/attacks.md`, starting with *The non-negotiables*. For **each** attack class:

1. Decide whether the app has that surface (e.g. no GraphQL means N/A, with the reason).
2. If it does, prove the defense: code (file:line), config, or a live probe against a **dev/staging** URL you own (for example request `/.git/HEAD`, `/.env` and `*.map` and expect 404; send a reset request with a forged `Host`; replay a webhook; send two parallel coupon redemptions; a wrong-tenant ID). The mechanism to compare against is in `app-security.md`.
3. Mark PASS / FAIL / N/A. A timeout or an unobserved result is not a PASS.

Rules: never probe production destructively; no real messages, charges or paid calls; never print secrets; never use a real person's login. If you find an exposed hole in production, stop and hand the human the edge containment step first. Open with the score (e.g. "22 of 25 attacks blocked, 3 open") and the open ones in plain words (who can do what), then each FAIL with a concrete reproduction and the fix, then a one-line PASS list.
