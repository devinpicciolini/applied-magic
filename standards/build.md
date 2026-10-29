# Build: principles, the default stack, and what to build in from day one

Tiers: **[MUST]** non-negotiable · **[SHOULD]** strong default · **[EXTREME]** for money, identity, health or family data. Part of [applied-magic](../README.md).

Zero trust, the box we build on, and the defaults new code gets without being asked. Retrofitting any of this later
costs 10× and is usually incomplete.

## The non-negotiables

- [MUST] Every trust boundary is written down (what may talk to what) and everything else is denied; being on the private network, the same host or the office Wi-Fi grants nothing.
- [MUST] Each legal entity, business and personal life gets its own hosts, accounts, keys, networks and identities, with no credential shared across them.
- [MUST] Every destructive or irreversible action has a human in the loop and a verified backup taken first.
- [MUST] Public IDs are random (UUIDv4/v7 or prefixed like `usr_…`); auto-increment integers never appear in URLs, APIs, emails or file names, and an unguessable ID never replaces the authorization check.
- [MUST] Every table holding customer data has an owner/tenant column from the first migration, unique constraints include it (`UNIQUE (tenant_id, key)`), and every query filters by it in SQL, never after the fetch.
- [MUST] Money is integer minor units (or an exact decimal) with its currency stored alongside, never float, and every money-moving or message-sending operation takes an idempotency key.
- [MUST] Secrets, tokens, PII and session data never go in URLs or query strings, and GET never changes state (link scanners and prefetchers follow GETs).
- [MUST] Every external input (API, webhook, queue message, file import, env) is schema-validated at the boundary with explicit types, lengths and enums; unknown fields are rejected and user file names are never trusted.
- [MUST] Client-side checks are UX only: the server re-validates and re-authorizes everything, and nothing secret ships in the client bundle (anything with `NEXT_PUBLIC_`/`VITE_`/`EXPO_PUBLIC_` is public).
- [MUST] Bulk deletes and updates run as a dry run first (counts + samples), refuse above a hard row cap, and run for real only after human approval and a backup taken after the most recent change, set aside so rotation can't drop it.

## Zero-trust principles

- [MUST] Least privilege for every user, service, key, token, role and agent, re-checked whenever something moves.
- [MUST] Assume breach: one compromised app, key or machine can't reach the others.
- [MUST] Containers, WSL distros or VMs sharing a kernel or network namespace are not separate trust zones: loopback and ports are shared, a stop hook in one can break the others' networking, and load in one starves the rest.
- [SHOULD] Defense in depth: every critical control has an independent second one (firewall + kill switch, app auth + DB role, start guard + fence).
- [SHOULD] Every control has a check that proves from outside that it works.
- [EXTREME] Treat your past self as untrusted: re-verify old setups live instead of trusting the docs.

## Separate companies and clients (if you work for more than one)

Freelancers, agencies and anyone running more than one business: keep each one's code, data and accounts apart.

- [MUST] Before any connector, CLI or MCP call (calendar, chat, mail, docs, tickets, code host, DNS, tunnel), check which account, org or zone the credential belongs to; if that isn't the content's entity, make no call at all: no send, draft, search or quote.
- [MUST] A workspace (chat, calendar, mail, docs) belongs to one organization; another entity's or personal data never enters it, and a "private" or hidden flag inside it doesn't count as separation.
- [MUST] Personal data collected for one company or client isn't copied or moved to another without a documented legal basis; if another one needs it, give it a scoped, logged API, not a copy.
- [SHOULD] Each company owns its own code-host org, cloud, DNS, app-store, payments, mail and AI-provider accounts, so a client handoff or a sale moves accounts, not passwords.
- [SHOULD] Commits are authored with the right identity per directory automatically (`includeIf` in git config), and employment or invention-assignment agreements are checked before building something in a related field.

## Default stack (opinionated): own the box

The most secure system has the fewest vendors holding your data and the fewest doors open. For most projects,
**one Linux VM you control, with everything on it, is cheaper, simpler to secure and easier to hand off than a stack
of SaaS services.** Use managed services for a concrete reason (below), not by default.

- **Compute:** one Ubuntu LTS VM per project / trust zone, built by a script: a cloud VM (in the client's account for a handoff) for anything with customers or an uptime promise; a VM on a Mac (Lima, OrbStack) or a home server only for personal and internal tools, since home power and ISP are single points of failure.
- **Database:** Postgres on the same VM, bound to `127.0.0.1`, scram auth, checksums on, one role per app. Same Postgres a host would run, with no database on the internet and no extra vendor.
- **Ingress:** Cloudflare Tunnel (outbound-only) → nginx on `127.0.0.1` → apps on `127.0.0.1`. **No open inbound ports.** An identity-aware proxy (Cloudflare Access) in front of every admin hostname that isn't private-network only. Trade-off: the tunnel's vendor terminates TLS and sees the traffic; where that's not acceptable, use the private network or your own TLS ingress with a firewall.
- **Private access:** Tailscale or WireGuard with tag-based ACLs **and ACL tests**; SSH keys only.
- **Egress:** a per-UID kill switch; apps leave only through a proxy (optionally a privacy VPN exit).
- **Auth:** a maintained library storing users in *your* DB (e.g. Better Auth), configured for production and kept updated. Hosted auth for enterprise SSO/SCIM, a compliance attestation, or when nobody will own auth upgrades.
- **Processes:** systemd units (or PM2 under systemd), memory-limited and sandboxed.
- **Deploys:** push-to-deploy with a read-only deploy key: stage dir → malware scan of the diff → swap → health check → automatic rollback. No CI vendor holds production credentials.
- **Secrets:** one `600` `.env` per app, backed up encrypted. A vault only when several people or systems need shared, audited access.
- **Backups:** `pg_dump` + file bundles encrypted with `age` to two offline keys → write-only relay off the box → off-site → restore drills.
- **Email:** a transactional provider on a dedicated sending subdomain with SPF/DKIM/DMARC and send-only keys.
- **Files:** local disk under `/srv/data/<app>`, backed up; object storage only when size or traffic needs it.
- **Private AI:** local models (Ollama, whisper.cpp) on hardware you control for anything touching personal data.
- **Monitoring:** a cron script checking a URL baseline, backups, replication and disk, alerting on change; a second machine watches the watcher.
- **Docs:** one current-state document per fleet plus a `CLAUDE.md`/README on each server. Handing off = handing over the box and that document.

**Why not the usual SaaS stack by default.** Serverless hosts keep your secrets in their dashboard, preview deploys can
expose unreleased branches and the data they connect to unless protection is on, long jobs and cron are awkward (fine
for a static site). Hosted Postgres is usually reachable from the internet unless you pay for private networking,
sits outside your egress controls and puts all your data with another vendor. Backend-as-a-service turns your security model into their config surface (RLS policies, public anon keys,
storage rules), which fails silently. Every vendor is one more account, key, data copy and bill.

**When managed services ARE the right call:** compliance needing attested infrastructure (SOC 2, PCI, bank or partner
audits); real multi-region HA or traffic one box can't carry; a partner mandating its platform for money movement; a team
that can't or won't patch, monitor and restore-drill a box (an unmaintained VM is worse than a managed platform);
enterprise SSO/SCIM. The rest of these standards still apply:
MFA, scoped keys, IP allow-lists, backups you control, restore drills.

**Spin up a hardened VM:** (1) Ubuntu LTS, app user with a fixed UID, no host mounts, no port forwards (TCP **and** UDP);
(2) `ufw` default deny, SSH keys only from the management address, `AllowUsers` set; (3) Postgres (localhost, scram,
checksums), nginx (localhost, default server `return 444`), the tunnel connector (sandboxed, own user), Tailscale;
(4) egress kill switch + proxy env in the process manager and login profile; (5) unattended upgrades, reboot window,
disk/memory/backup monitoring; (6) push-to-deploy with scan, health check, rollback; (7) encrypted backups off-site and a
restore drill **before** go-live; (8) the box README: what runs, ports, secret locations (names only), deploy, restore,
failover; (9) run these standards against it and record the results.

## Which one? The common choices, with the reason

Most teams don't fail from picking the wrong tool; they fail from picking **three** tools where one would do. Start with
the default; move up only when the "move up when" line is actually true today, not someday.

**Who can reach an internal tool** (admin panel, dashboard, Metabase, a script's web UI)
- **Just you, or a few technical people on your own devices:** a private network (Tailscale/WireGuard). No login page exists to attack, and it covers SSH, databases and phones too. Move up when someone can't or won't install a client.
- **A team or contractors using a browser:** an identity-aware proxy (e.g. Cloudflare Access, Google IAP) in front of the hostname, with **sign-in through the company Google/Microsoft accounts** restricted to your domain (plus named outside emails), 2FA enforced at that provider. Nobody installs anything, offboarding is one account switch-off, and the tool itself needs no login code. The origin must be reachable only through the proxy (a tunnel or firewall), and if the tool uses the proxy's user identity it verifies the proxy's signed token (e.g. the Access JWT), never a plain header. Use both together when the tool also needs SSH or DB access.
- **Customers:** app sign-in in your own DB (*Auth* above). Never put customers behind your company's Google sign-in, and never put an admin tool on the open internet behind only its own password.
- **Roles inside the tool** (some staff may edit, others only view): then the tool does need its own authorization, but still takes identity from the proxy or company SSO rather than its own passwords.
- Not: a hand-built login page for an internal tool; a shared admin password; making contractors join your private network for one web page; IP allow-lists as the only lock (IPs change and are shared); a separate admin app when a DB client over the private network or one guarded page would do.

**How customers sign in**
- **Business users:** Google/Microsoft sign-in plus email codes or passwords + 2FA as the fallback. Enterprise SSO (SAML/SCIM) only when a customer's contract asks for it.
- **Consumers:** passkeys or email one-time codes first; passwords only with 2FA available. On iOS, offering third-party sign-in means also offering a privacy-focused one such as Sign in with Apple (App Store guideline 4.8).
- **Hosted auth** (a vendor holding your users) for SSO/SCIM at enterprise scale, a compliance attestation you can't get yourself, or a team where nobody will own auth upgrades and security fixes; a stale self-hosted auth library is worse than a maintained vendor.

**Email**
- **People** use the company's Google/Microsoft accounts. **The app** sends through a transactional provider on a subdomain (`mail.` / `notify.`) with SPF/DKIM/DMARC. Never send app mail from a person's mailbox or SMTP password: one bounce storm or leak takes down the human's inbox too.
- Internal alerts go to a chat channel or push, not to email nobody reads.

**Where it runs**
- **App with a database, jobs or cron:** one box (*Default stack*). **Static marketing site or docs:** any static host is fine, including the serverless ones; there are no secrets or data there. When the box is genuinely busy (CPU or memory sustained above ~70%), first move to a bigger box; add boxes when uptime needs a hot standby or a second region, not before.
- **Environments:** local + production, with production's shape reproduced locally (same Postgres major, same env names). Add a staging box, built by the same script, when releases need testing against real integrations; never point staging or previews at production data.
- **Containers:** systemd is enough on one box. Docker when you need identical images across machines or a service that ships only as an image. Kubernetes only with a team whose job is running it.

**Data**
- **Database:** Postgres for anything multi-user. SQLite for a single-user tool or local app. Not a document DB "for flexibility": Postgres `jsonb` does that with constraints next to it.
- **Background work:** a systemd timer or cron → a Postgres-backed queue (`SELECT … FOR UPDATE SKIP LOCKED` or a library on it) → a dedicated broker only past what Postgres handles (roughly thousands of jobs per second). Don't add Redis just to hold jobs: it's one more stateful service to persist, secure and back up.
- **Search:** Postgres full-text and trigram first; a search engine when relevance tuning or volume demands it.
- **Files:** disk on the box (backed up) → object storage when files outgrow the disk or need a CDN.
- **Live updates:** polling every few seconds → server-sent events → WebSockets only for true two-way traffic (chat, collaboration).

**Code shape**
- **One repo, one deployable app plus workers.** Split into services only when two teams deploy on different schedules or one part needs to scale or be isolated on its own. A monorepo with a shared contract (`architecture.md`) beats several repos that drift.
- **Mobile:** a responsive web app first unless you need the store, push notifications, offline or device hardware; then native or one cross-platform framework over the same API, never a second backend or a per-client copy of business logic.
- **Feature flags:** an env var or a DB row; a flag vendor when non-engineers run experiments.
- **Payments:** the provider's hosted checkout and customer portal before any custom card form.
- **Secrets:** a `600` env file per app, backed up encrypted → a vault when several people or systems need audited, shared access.
- **AI:** hosted model APIs for data you'd show the vendor anyway; local models for personal, health or family data.

**Operations** (where managed usually wins)
- **Uptime checks:** a hosted checker from outside is fine and better than self-hosting it: a box can't report its own outage.
- **Error tracking:** a hosted tracker with PII scrubbing and request-body capture off is fine; self-host only when the data can't leave.
- **Off-site backups:** any object storage is a fine target once backups are encrypted before they leave the box.

## Data model

- [MUST] Database constraints enforce the rules (`NOT NULL`, `UNIQUE`, foreign keys, `CHECK`); multi-step writes run in one transaction, and concurrent edits use a version column or row locks.
- [MUST] Anything a user or the business may want back is soft-deleted (`deleted_at`) or archived, never hard-deleted by the app; unique indexes are partial (`WHERE deleted_at IS NULL`), and account deletion or a legal erasure request truly deletes, on a documented schedule that covers backups.
- [MUST] Email uniqueness is a unique index on the normalized email (trim, lowercase, NFC); before adding it, a dry run lists case-duplicate accounts and each is merged in one transaction that moves every foreign-key row.
- [MUST] Saving a settings or profile form never overwrites a stored credential (token, key, device secret) with an empty value: blank means unchanged, and a test proves it.
- [MUST] Instants are `timestamptz` (UTC) converted only for display; calendar dates (birthdays, due dates, "today") are `date` computed in the user's or business's named time zone, and comparisons happen in one place (the DB), never DB time against a client clock.
- [MUST] Each real-world thing has one identity key (normalized email, registration number, domain) checked before insert, and sign-up or import claims an existing record only through a verified identifier (a confirmed email), never through a shared link, prefill or typed name.
- [MUST] A status a user can set does what it says in the same transaction ("approved" grants the access, "cancelled" stops the billing); no status is a cosmetic label, and a test proves each transition's effect.
- [SHOULD] `created_at` / `updated_at` / `created_by` on every table.
- [SHOULD] The core nouns and their lifecycle are defined once in a README glossary before the first table (when a trial becomes a customer, what each status means, which ones can't be undone), and schema, API and UI use exactly those words.
- [SHOULD] PII lives in clearly named columns (easy to find, encrypt, export and delete), you store only what you need, phones are E.164 and usernames are checked for confusable look-alikes.
- [SHOULD] Ledgers (append-only entries + computed balances) instead of mutable balance columns.
- [SHOULD] A missing parameter (no location, no scope) never defaults to the widest result set, and time-bounded content (events, seasonal offers) has an end date enforced server-side.
- [EXTREME] Sensitive columns are encrypted from the first migration (see [app-security.md](app-security.md), *Encryption and privacy*).

## Requests, browser and client

- [MUST] Email links (verify, reset, magic login, invite) carry a single-use, short-lived token that the landing page POSTs to consume, so a mail scanner opening the link can't burn or use it.
- [MUST] HTTPS everywhere, no mixed content, no cleartext exceptions in mobile apps (headers: [app-security.md](app-security.md)).
- [MUST] Session and refresh tokens live only in `HttpOnly; Secure; SameSite` cookies (ideally `__Host-`), never in `localStorage`, `sessionStorage`, IndexedDB or JS-readable cookies.
- [MUST] Sensitive pages and API responses send `Cache-Control: no-store`.
- [MUST] No sensitive data in hidden inputs, `data-` attributes, HTML comments, page titles or `window.name`, and paywalled content is absent from the page source, not CSS-hidden.
- [MUST] `postMessage` names its target origin, and receivers compare `event.origin` exactly against an allow-list (never `includes()`/`endsWith()`) for every message type, including third-party connector modals.
- [SHOULD] Every outbound call has a timeout, bounded retries with backoff + jitter, and a response size cap (list limits: [architecture.md](architecture.md)).
- [SHOULD] Rate limits per user **and** per IP on every expensive or sensitive endpoint, sized from the largest legitimate use (a 200-file upload is 200 requests), kept in a store that survives restarts, and logged on every refusal; kill-switch flags for risky features (outbound messaging, payments, AI actions) from launch.
- [SHOULD] Sensitive client state (stores, caches, service-worker caches) is cleared on logout, and secrets or full PII never sit in long-lived globals that extensions or error reporters capture.
- [SHOULD] No PII in analytics, error reports, session replays or console logs; replay tools mask fields, and replay/analytics project IDs are separate per app.
- [SHOULD] No third-party scripts on login, payment or account pages, Subresource Integrity on any CDN script, sandboxed iframes, and correct `autocomplete` values (`new-password`, `one-time-code`; `off` on card security and backup codes).
- [EXTREME] Sensitive values are masked on screen by default (last 4), and mobile apps blur in the OS app switcher.

## Errors, logs and config

- [MUST] Clients get a generic error with a request ID; stack traces, SQL and internal hosts stay in server logs.
- [MUST] Structured logs redact known-sensitive fields automatically (password, token, authorization, cookie, card and bank numbers, SSN and government IDs, secret, key, DOB, postal code, address, IP, message bodies), the redactor has a unit test, and request bodies aren't logged by default.
- [MUST] A fresh clone runs with `cp .env.example .env` and zero edits: names match the local services, integrations not used locally are optional in the config schema (not filled with fake keys), and no step says "ask someone for a secret"; shared dev credentials come from one command, and local infra creates its own admin credentials (config validation: [architecture.md](architecture.md)).
- [MUST] Source lives in a repo you own from the first commit, and every running copy is built from a commit in it; nothing exists only in a hosting dashboard, a server folder or one laptop.
- [SHOULD] A correlation ID flows through logs, jobs and outbound calls, and the health endpoint checks the DB, not just "process up".
- [SHOULD] Runtime and package-manager versions live in one file CI also reads (`engines`/`packageManager`/`.nvmrc`), and dev ports avoid ones the OS already holds (5000/7000 on macOS) and can be overridden by env, so CI, dev and prod build the same thing and parallel checkouts don't collide.

## Imports, background jobs and outside data

- [MUST] New rows in anything that is shown to users or triggers actions default to hidden/held through a schema-level `DEFAULT` in the migration, so every writer (scrapers, imports, syncs) inherits it and a rebuild can't revert it; release happens only through an explicit rule.
- [MUST] Every imported record stores where it came from (source, `fetched_at`; a list if several sources merge), and personal data from outside sources is used only with a recorded lawful basis.
- [MUST] A worker that skips an ineligible queue item records the skip (status, reason or advanced cursor) so skipped items can't sit at the head and starve every run; a test puts N ineligible items first and asserts the batch still fills.
- [MUST] A job item's "done" state never depends on an optional or AI-generated extra (cover art, summary, thumbnail); a failed extra is recorded as missing and the core item completes.
- [SHOULD] Duplicate detection on imports compares several normalized identifiers (email, phone, domain, name), never one; shared domains (free mail, social networks) never count as a match.
- [SHOULD] Temporary exclusions use a reversible hold, separate from irreversible flags (opt-outs, bans), which are set only on confirmed cases; the docs list which flags can't be undone.
- [SHOULD] Imports validate and quarantine bad rows instead of failing half-way or silently dropping them.
- [SHOULD] Paid per-record API calls (data lookups, verification, AI) run after the free filters, against a local budget that stops cleanly at its cap or when the vendor reports credits exhausted.
- [SHOULD] Holding or retiring records upstream also withdraws the work already handed downstream (open tasks, scheduled jobs, queued messages), with a count of what was withdrawn.
- [SHOULD] The completeness of key fields in what users actually receive (per served item, per day) is tracked, and a drop alerts.

## Data integrity and bulk changes

- [MUST] Cleanup matching and purge scripts are proven against planted cases (the bad row deleted, the good row kept) and never run automatically on deploy: a loose match can delete real data at scale.
- [MUST] Deletion classifiers act only on definitive signals (NXDOMAIN, 404/410, an explicit "closed"); timeouts, 5xx, blocks and saturation are "unknown" and never delete anything or get stored as negative facts, and a vendor's "0 = unknown" is stored as NULL.
- [MUST] Re-imports and upserts merge and never overwrite human-edited fields (status, owner, notes).
- [MUST] Delete propagation from a replica first checks it is streaming with zero lag, and a sync that hit its page cap is marked truncated and never marks missing records deleted.
- [MUST] Backfills and one-off jobs that call paid APIs or send messages are scoped to explicit IDs or a date window and print a count first; "queue everything unchecked" is forbidden.
- [MUST] A panicked "delete/block everything" request gets the reversible minimum, with each row's prior state saved.
- [MUST] Data repairs (fixing bad rows) are their own preview-first command, unreachable from migrations, upgrades or startup; only the backfill a schema change needs lives in a migration. Flipping existing users' defaults or visibility needs the owner's explicit approval cited in the PR.
- [MUST] A fix for a tenancy or ownership bug ships with a query for the rows the bug already wrote and a repair (or a recorded "none found"); fixing the code alone leaves the leak in the data.
- [SHOULD] Every bulk job writes an audit row: who, when, filter, row count, how to undo.
- [SHOULD] Duplicate merges always point to the root record (no A→B→C chains, enforced by an invariant check), and no bulk merge runs over records with a pending manual repair.
- [SHOULD] Invariant checks are re-reviewed whenever the tenancy or ownership model changes (a backfill that can't converge exits non-zero and alerts), and exports depending on tenancy and exclusions read from one shared view that applies every exclusion flag, with a test.
- [EXTREME] Destructive SQL in production needs two people, or a human plus a separate verification run.
