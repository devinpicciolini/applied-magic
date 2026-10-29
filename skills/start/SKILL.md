---
name: start
description: Set up a new project the applied-magic way before writing features. Default stack, day-one defaults and the full boilerplate kit, with decisions surfaced for the human. Use when starting a new app, service or repo.
---

# New project

Read `../../standards/build.md` (*Default stack* and the day-one sections), `engineering.md` (*Least code*, *Planning and scope*), then `app-security.md`, `delivery.md` and `operating.md` (*Source-of-truth docs*).

1. **Plan small.** State the problem, the smallest version that does the core job and what's NOT in scope (`engineering.md`). The least-code ladder applies to the scaffold too: no framework, service or dependency the core job doesn't need.
2. **Pick the stack.** Use *Default stack* in `build.md` unless the human said otherwise or a "When managed services ARE the right call" reason applies. Write the choice and the reason in the README.
   Before writing files, show what you're about to set up and what each piece saves them, in one short list ("backups + a restore drill: you can lose the server and lose nothing"), and ask **Build it** · **Change something**. Skip the question if `/applied-magic` already got an Accept.
3. **Day-one defaults.** Apply `build.md` *Data model*, *Requests, browser and client*, *Errors, logs and config* and, if the app feeds people or outbound actions, *Data pipelines and outbound pools*: random public IDs, a tenant/owner column, DB constraints, UTC, money in minor units, idempotency keys, no secrets in URLs, tokens in HttpOnly cookies, a validated env module, log redaction, schema validation at boundaries.
4. **Boilerplate kit.** Add every item in the *Boilerplate* list of the `check` skill (`../check/SKILL.md`). One commit per area.
5. **Delivery.** Set up the normal lane from `delivery.md`: CI gates, deploy only main after green, post-deploy smoke test, automatic rollback. Add the website fast lane if there's a marketing site.
6. **Agent context.** Write the project `CLAUDE.md` with the real commands (install, dev, build, lint, typecheck, test, migrate, deploy, health URL), architecture and what CI runs.
7. **Decisions for the human.** Don't enable anything from "Stop and ask the human first" in `../check/SKILL.md`. List those as decisions with your recommendation.

Finish with the README sections "Where it runs / deploy / restore" filled in, and a short list of open decisions.
