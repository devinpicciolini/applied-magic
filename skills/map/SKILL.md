---
name: map
description: Map a project or server from live checks into the docs an agent needs so nobody has to say "go look at X" again — a CLAUDE.md with real commands and architecture, plus an inventory of what runs, where, and with what data. Use for "map this", "learn this repo", "onboard onto this project", "write the CLAUDE.md", or before harden on an unfamiliar codebase.
---

# Map

Read *Source-of-truth docs* in `../../standards/operating.md` and *Backend layering* in `architecture.md`.

1. **Verify, don't trust.** Read the README, existing `CLAUDE.md`, package/manifests, CI config, deploy config, env example and migrations. Then confirm what you can live: run install, typecheck, lint and test (read-only; no migrations, no deploys); for a server, `ss -ltnp`, the process manager, cron/timers, nginx/tunnel config, disk. Anything you couldn't confirm is marked *unverified*.
2. **Inventory:** apps and entry points; how they talk (HTTP, queues, cron, webhooks in and out); the database(s) and the tables that hold PII, money, health or credentials; vendors and which keys they need (names only, never values); auth and roles; where it runs, how it deploys, how it's backed up and restored; who owns it.
3. **Write `CLAUDE.md`** at the repo root (or `/srv/CLAUDE.md` on a server), current state only: what this is (two lines) · the real commands (install, dev, build, lint, typecheck, test, migrate, deploy, health URL), each one you ran marked as verified · architecture and layering rules · where data lives · what CI runs · gotchas you hit · "Never touch" list. Keep it under ~150 lines; link out for detail. If one exists, edit it in place; don't append a second version.
4. **Flag, don't fix.** List problems found along the way (secrets in the repo, an open port, no backups, failing tests) ranked by harm, with the skill that fixes each (usually `check`).

Don't change code. Finish with the doc path, the verified/unverified split, and the ranked problem list.
