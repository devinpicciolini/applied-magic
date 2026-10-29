# Application security: secrets, auth, web, data, money and integrations

Tiers: **[MUST]** non-negotiable · **[SHOULD]** strong default · **[EXTREME]** for money, identity, health or family data. Part of [applied-magic](../README.md).

The mechanisms that held up in audits of real fintech, family-data and consumer apps. Copy them. Each attack class and
its probe is indexed in [attacks.md](attacks.md); this file holds the controls in depth.

## The non-negotiables

- [MUST] No secrets in git (history too: `git log -p`, a secret scanner), URLs, logs, error messages, analytics or crash reports; a secret ever printed to a terminal, chat, ticket or AI session, or found in history however old, is revoked and rotated (rewriting history never replaces revocation).
- [MUST] Required secrets (auth secret, webhook signing secret, encryption keys) are checked at boot and the app refuses to start if any is missing or empty; no code-level fallbacks (`'dev-only'`, `'changeme'`), and any `*_STUB`/auth-bypass flag crashes a production boot.
- [MUST] Passwords are hashed with argon2id (or bcrypt/scrypt), and login, sign-up, reset and OTP are rate-limited per **real client IP** read through one trusted helper that takes only the value your outermost proxy sets and overwrites (e.g. `CF-Connecting-IP`, or the right-most `X-Forwarded-For` hop your proxy appended), never a client-supplied header or the left-most XFF entry, or every user shares one bucket (or each attacker gets a fresh one).
- [MUST] Deny by default at the framework level: a global auth guard with an explicit `@Public()`/allow-list opt-out, plus a CI test that enumerates every route, server action and RPC and fails on any without a guard decision.
- [MUST] Tenant/owner is derived from the credential (API key → org, session → active org), never the request body; every fetched record is checked against it, and cross-tenant tests write as one tenant and read as another against a running server.
- [MUST] Webhooks verify the signature over the **raw body** with a constant-time compare and a pinned algorithm, and reject (404/401) when the secret isn't configured; no unsigned-test flag works in production.
- [MUST] Money movements are idempotent with keys derived from business IDs (payment ID, agreement ID + step), never a per-attempt random value, and a test proves two identical requests make one record.
- [MUST] Every automated message checks unsubscribe/opt-out **at send time** on the normalized address and fails closed if the check errors; jobs that send or charge run only on the confirmed primary.
- [MUST] PII columns and third-party credentials (OAuth refresh tokens, bank/aggregator tokens, user API keys) are encrypted at rest, and production refuses to start or write if the key is missing.
- [MUST] Databases listen only on localhost or the private network, use `scram-sha-256` with one role and password per app and data checksums on, no app connects as superuser, and unused roles are dropped with their `pg_hba` lines.

## Secrets and keys

- [MUST] Every secret can be rotated in one step: you know every consumer, swap the value, make one real call through each consumer (scope errors, like an org key with no project, fail every call silently), then revoke the old one.
- [MUST] Rotation tooling keys values by app + name (two apps can share a name with different values), refuses empty values, reads the value back after writing and fails loudly.
- [MUST] Secrets never travel as command-line arguments (`ps`, shell history), `sed` replacement strings (`&` and delimiters corrupt them), clipboard commands handed to a human, or container image layers and build args; use env, stdin or `600` files.
- [MUST] Long-running services use their own scoped token stored `600`, never a human's short-lived CLI or session token.
- [MUST] Browser-automation session files and profiles (cookies, storage state) are secrets: `600`, gitignored before first use, backed up encrypted.
- [MUST] Irreplaceable signing material (upload keystores, one-time-download sign-in and push keys) is escrowed off-machine the moment it's created with platform signing recovery on, and copies travel only over encrypted channels and are checksummed.
- [MUST] If a home or workspace directory ever reached public hosting, everything in it is inventoried and rotated; deploy roots are never home or workspace directories.
- [SHOULD] One list says where every secret lives (name, file, host, owner, never the value), including copies shared across apps; retiring an app removes its secrets everywhere and deletes its keys at the provider.
- [SHOULD] Keys are scoped to the minimum (send-only mail, zone-scoped DNS, read-only per-repo deploy keys), and paid keys have spend caps and alerts.
- [SHOULD] Every expiring credential (OAuth refresh tokens and test-mode clients, Apple client secrets, signing certs, profiles, subscription tokens) is listed with its expiry and alerts before it lapses, and before declaring a key lost every store is searched (OS keychains, password managers, escrow bundles).
- [SHOULD] Workstation-to-server syncs exclude env and secret files, throwaway scripts or temp env files holding a token are deleted right after use, and TOTP seeds and recovery codes never sit in a mailbox or anywhere an agent can read.
- [EXTREME] Keys for the most sensitive data live outside the DB and never enter the code host (no repo, CI variable or hosted secret store): only the owning server plus an offline password-manager escrow, kept in an immutable bundle (`chflags uchg`) only its own script unlocks.

## Authentication

- [MUST] Password rules follow NIST SP 800-63B: at least 8 characters (15 when the password is the only factor), at least 64 allowed, any Unicode and paste allowed, no composition rules or forced rotation, and every new password checked against a breached/common-password list (k-anonymity range lookup).
- [MUST] Password rules and rate limits are enforced server-side at every entry point that sets a password (sign-up, reset, invite accept, admin create), e.g. an auth-library before-hook, not the form; bcrypt inputs over 72 bytes are refused or pre-hashed, never silently truncated.
- [MUST] Every account-creation path (sign-up, team invite, customer link, OAuth, API) calls one shared validator (burner email, VoIP phone, your own staff domains unless via staff invite), a test enumerates the paths, and the auth library's raw self-registration and unused endpoints return 403 in production.
- [MUST] Rate-limit and lockout state is persistent and shared (DB upsert or Redis, hashed keys); per-account lockout is keyed by an HMAC of the normalized identifier in two tiers (e.g. 5 fails/15 min → 15 min; 20/24 h → 24 h), cleared on success, and keyed by account + IP or gated by a challenge so nobody can lock a victim out; in-memory limiters are only a second layer.
- [MUST] Sign-up, login, reset and invite never reveal whether an account exists: same status, body and timing (compare against a dummy hash), with the real outcome sent by email.
- [MUST] Reset tokens are single-use, short-lived and bound to the user, a reset revokes every session and never disables or skips 2FA, unverified accounts can't sign in, and production mode is on (`NODE_ENV=production`), since some auth libraries disable rate limits and checks in dev.
- [MUST] Changing the password, email, phone or second factor requires re-authentication (current password or passkey), notifies the old address, and revokes every other session.
- [MUST] A password exists only inside the request that checks it: multi-step login carries a short-lived, single-use, server-side challenge ID, never the password, in page state, storage, cookies or hidden fields.
- [MUST] 2FA challenges, codes and attempt counters live in the DB (codes hashed) so a restart can't reset attempts (a test restarts mid-login); OTPs expire in minutes, are attempt-capped and never appear in logs, subjects or sent-mail archives.
- [MUST] Staff and admin roles have 2FA they can't turn off, the second factor is enforced server-side on every privileged path, and when an external IdP owns login its MFA policy is enforced there and kept as code/config.
- [SHOULD] Auth-library upgrades are reviewed for IP handling, linking rules, schema and deprecated options and never auto-merged alone; afterwards a test signs in from two client IPs and asserts separate rate-limit buckets, and CI exercises every auth transport shipped clients use (bearer for mobile, cookies for web).
- [SHOULD] Disposable email is refused (MX + burner denylist), phones only when a carrier lookup says mobile/landline (failing closed on lookup errors), and every sign-up records IP, an ASN/VPN/datacenter flag and user agent, alerting on VPN or placeholder-name sign-ups in money and identity flows.
- [SHOULD] A bot challenge on public forms once abuse appears, and abuse watchers (failed logins per account/IP, bulk sign-ups or resets, repeated 429s, bad-token probes) alerting a chat channel with PII masked.
- [SHOULD] Offer passkeys/TOTP plus single-use recovery codes stored hashed (email or SMS OTP is the weakest factor), require step-up for payout details, role changes, exports and key creation, and bound "trust this device" (e.g. 30 days, revoked on reset).
- [SHOULD] Tightening a rule (e.g. requiring verified email) is tested against existing accounts, grandfathered by timestamp.
- [EXTREME] Staff and admin sign in only with phishing-resistant factors (passkeys or hardware security keys), no SMS, email or TOTP fallback.

## Sessions, tokens and devices

- [MUST] Sessions are `HttpOnly`/`Secure`/`SameSite`, get a new ID at login and privilege change, are stored hashed server-side and revocable; logout kills them server-side, and the session lookup itself excludes disabled users.
- [MUST] Deactivating an account blocks new sessions (DB hook) and kills existing ones; closing it (on every path, admin deletion included) deletes credentials, releases the email and revokes provider refresh tokens (e.g. Apple).
- [MUST] Tokens you issue (API keys, invites, reset links, device tokens) are stored hashed and shown once; API keys carry a visible prefix, scope, expiry and last-used time.
- [MUST] Magic, customer and invite links are ≥256-bit random, stored as SHA-256, short-lived, single-use, valid only for the state they were issued for and bound to the invited identity; malformed tokens are rejected and alerted, and embedded flows swap a one-time code for the token instead of carrying it in the URL.
- [MUST] JWTs you issue pin the algorithm, check `iss`/`aud`/`exp`, use a separate secret per audience and schema-validate the payload; no hard-coded signing secrets (grep `SECRET_KEY`, `changeme`) and no dead token code.
- [MUST] Device credentials are per device: 256-bit random, stored hashed, unknown token → 401 (never a default user), listed and revocable with last-seen, required even on private-network ingest endpoints; pairing codes are ≤10 min, single-use, hashed and issued by an existing admin, and shared "owner" fallback tokens are retired once that works.
- [MUST] Device identity and E2E keys live in the OS keychain with an accessibility class that survives app updates; a release test installs the previous build, pairs, upgrades and asserts it's still paired.
- [SHOULD] Session cookie caches are ≤60 s and bypassed for privileged roles, sessions have an idle timeout (e.g. 30 min for staff and money flows) as well as an absolute lifetime (e.g. ≤30 days, ≤12 h for staff), and session IPs are stored as keyed hashes.
- [SHOULD] Bearer access tokens you issue are short-lived (≤15 min) with rotating refresh tokens; a reused refresh token revokes the whole family.

## OAuth, SSO and identity proxies

- [MUST] Pre-account hijack: an email/password account links to OAuth only after email verification, and if a verified OAuth identity auto-links into an unverified password account, a before-create hook replaces the password hash with random bytes, deletes every session and marks the email verified; unauthenticated email-keyed submissions never auto-attach to an account.
- [MUST] Identity tokens (Sign in with Apple/Google, your own OIDC) are verified server-side before any account match: pinned algorithm (reject `none` and HMAC where RS/ES is expected), JWKS by `kid`, `iss`, `aud` (every client ID listed), `exp`, `iat` skew, `sub`, and a one-time nonce stored hashed; accounts are keyed by (`iss`, `sub`), never by email alone.
- [MUST] An app that trusts an identity-aware proxy verifies the proxy's signed JWT (signature, audience, issuer, expiry), never a presence-only header, and the origin is reachable only through that proxy.
- [MUST] Each app has its own OAuth client and cloud project, and clients needing offline access are set to production publishing status before go-live (test-mode refresh tokens expire within days), with verified domains and minimum scopes.
- [MUST] Every public hostname is in both the framework's allowed origins and the auth library's trusted origins, and native clients send an explicit trusted `Origin`.
- [MUST] OAuth uses authorization code + PKCE with `state` (no implicit or password grant), exact redirect-URI matching and a callback host equal to the configured base URL; linking honours `email_verified` only from providers you trust to verify, and the rules are tested.
- [MUST] Staff and customer identities don't share one auth setup: staff reach internal tools through the company sign-in at an identity proxy, customers sign in to the app. If they must share, every provider-specific hook checks the exact provider on that callback, never "user has an account linked to X".
- [SHOULD] Internal/operator panels disable sign-up and use a separate database, auth instance and secret from the public app.

## Authorization and tenancy

- [MUST] Role checks are positive (`isStaff`), never negated (`!isApplicant`), and adding a role triggers a review of every negated check; guards are synchronous, or awaited and lint-enforced (`no-misused-promises`), since `!asyncGuard()` is always false and opens every route without a type error.
- [MUST] Admin routes are enforced by role server-side and tested with a non-admin session, route allow-lists for limited roles match exactly (prefix matching has opened whole APIs), and client and server admin allow-lists come from one source or a test proves they match.
- [MUST] Every route that takes an ID is tested with another user's ID across read, update, delete, list filters, exports, file URLs and download paths separately, re-checking the current role on every read (ownership survives role downgrades).
- [MUST] Isolation and role tests call the real HTTP route and assert on content (B's response holds none of A's IDs), not only status codes.
- [MUST] Staff paths (`/admin`, `/crm`, `/ops`) answer only on the internal hostname; the proxy 404s them on every customer-facing host, and a test requests each staff path on each public host.
- [MUST] After a tenancy leak is fixed, rows that leaked before the fix are found and repaired.
- [MUST] Staff and admin access is per person: no shared logins or shared admin accounts, so every audit row names one human.
- [SHOULD] A per-role probe scans pages and API responses for fields that role must never see, new sensitive columns default to an admin-only column set, hidden resources return 404 not 403, and 403/404/500 pages are checked for leaked names or counts.
- [SHOULD] Production-data access is an entitlement separate from sandbox access; admin is a distinct role or org type, and email-domain allow-lists also require an IdP-verified email.
- [SHOULD] Background jobs and webhooks run with the minimum privilege they need; read-only and analytics roles physically can't write and carry a statement timeout.
- [EXTREME] Row-level security in the database as a second line behind app checks for the most sensitive tables.

## Web hardening

- [MUST] Security headers on pages **and** API responses: HSTS (`max-age=31536000; includeSubDomains`), `nosniff`, `frame-ancestors`, `Referrer-Policy: strict-origin-when-cross-origin`, `Cross-Origin-Opener-Policy: same-origin`, Permissions-Policy; a proxy adds them only when the app didn't (`map` on `$upstream_http_*`) and re-includes the snippet in every nginx location with its own `add_header` (inherited headers are dropped there).
- [MUST] CSP is nonce- or hash-based with `strict-dynamic`, `object-src 'none'`, `base-uri 'none'`, `frame-ancestors`, plus `default-src`/`connect-src`/`style-src`, sent as a header (not `<meta>`, which ignores `frame-ancestors`) and rolled out report-only first.
- [MUST] CORS is a fixed allow-list shared with the auth library's trusted origins, never a reflected origin with credentials; cookie-authenticated state changes check `Origin`, and where cookies must be `SameSite=None`, every non-GET with a cookie from an untrusted origin gets 403.
- [MUST] Every endpoint has a body-size limit, and the proxy's limit matches the app's upload limit.
- [MUST] Internal, strategy or security documents on web hosts are token- or auth-gated with `noindex` (only intended sites are indexable), scheduled publishing jobs get the same approval gate as manual publishing, and marketing assets, screenshots and decks use seeded demo data, never real people's.
- [MUST] Built artifacts of decompilable clients (mobile, desktop, SPA bundles) are scanned against a banned-terms list (internal data-source and partner names, internal hostnames, sensitive field names), not only for secrets.
- [SHOULD] Redirect URLs are built from configured origins, never `Host`/`X-Forwarded-Host`, and the `trust proxy` hop count matches the real infrastructure.
- [SHOULD] Error pages leak no stack traces, versions or internal addresses (`server_tokens off`; `absolute_redirect off` behind a TLS-terminating tunnel so redirects neither downgrade to http nor leak `127.0.0.1:port`).
- [SHOULD] Static assets are versioned, and deploys purge the CDN for changed HTML or serve HTML `private, no-cache` (CDNs can override Cache-Control and hold pages for a year).
- [SHOULD] Static roots under a home directory use a traverse-only ACL for the worker user, never a world-readable home.
- [SHOULD] Placeholder constants (555 numbers, `example.com`, template entity names) are grepped before every release and on every legal page.
- [SHOULD] Removing data, a vendor or a dependency is verified by grepping the live build, the retained rollback build, logs, tmp and caches, and the rollback build is replaced with a clean copy.
- [EXTREME] Public surfaces that need only a few paths (a signing page, a webhook host) sit behind a proxy path allow-list matching the exact path forms the app uses (including suffixes like `.json`); everything else returns 404.

## Uploads and untrusted content

- [MUST] Uploads are typed by magic bytes, size-limited, stored outside the web root with random names and never executed; they're served as `attachment` with `nosniff`, `no-store` and `CSP: default-src 'none'; sandbox` from a separate origin, never with `Access-Control-Allow-Origin: *`.
- [MUST] Uploaded documents with PII are encrypted by the app per file (magic header + AEAD, mode `0600`) on every storage backend; object storage adds private ACLs and presigned URLs of ≤2 minutes.
- [MUST] Upload limits never silently cut a legitimate batch: refusals are logged from the first one, the client honours `Retry-After` and reports how many files landed, and per-request limits account for clients that send one file per request.
- [MUST] Untrusted HTML (mail, rich text) goes through an allow-list sanitizer library, never regexes, and renders with JavaScript off (WebView JS disabled, sandboxed iframe).
- [SHOULD] Remote images in untrusted HTML are proxied server-side through the egress proxy and SSRF guard (private and VPN hosts refused), external CSS is stripped, tracking pixels and SVG are refused, and failed images are dropped rather than left as live remote URLs.
- [SHOULD] Image/PDF processing runs in a constrained process (time + memory limits), images are re-encoded to strip metadata (EXIF GPS), files other people will download are malware-scanned first, and upload directories are `700` and in the backup set.

## Encryption and privacy

- [MUST] You know where every category of personal data lives (tables, files, logs, backups, analytics, AI prompts), and PII goes to a third party (AI, analytics) only when intended, minimized and documented; vendor-side prompts and logs count as storage.
- [MUST] PII columns (DOB, IPs, government IDs, account numbers) use AEAD (AES-256-GCM, random 96-bit IV) through a typed column wrapper, and no data key is derived from a session or auth secret that might rotate.
- [MUST] Every raw-SQL path that touches sealed columns (coalesce, JSON builders, concat, jsonb patches) goes through seal/open in code, and seal skips values already carrying the cipher prefix.
- [MUST] Encryption backfills refuse to run until the release that reads ciphertext is live (reads tolerate legacy plaintext meanwhile), then verify zero plaintext rows plus a sample decrypt.
- [SHOULD] Every ciphertext carries a format + key-version prefix (`enc1:`, `v2:`) with a re-encrypt job for rotation; the table, column and row ID go in as associated data so ciphertext can't be swapped between rows; lookups use a blind index (HMAC-SHA256, separate HKDF-derived key, `h1:` prefix); a self-check proves round-trip, tamper detection and legacy reads.
- [SHOULD] Fields left plaintext on purpose are documented with the reason, chat notifications of business events carry no email or phone, retention rules exist and run, and each deletion records the date its copies age out of backups.
- [SHOULD] When a client engagement or a contractor ends, remove every copy and access you hold (accounts, keys, machines, data copies, AI transcripts, caches), after checking for any legal hold.
- [EXTREME] The most sensitive identity fields have no read route: sealed to an offline owner key, looked up by keyed hash, served only over mutual TLS.
- [EXTREME] E2E messaging keeps identity keys in hardware (Secure Enclave/StrongBox) and gives the server only public keys and ciphertext, using X3DH/PQXDH + Double Ratchet (or MLS for groups) with messages signed by the device identity and bound to thread and sender.
- [EXTREME] The E2E server rejects prekeys not signed by the device identity, hands out each one-time prekey once (`FOR UPDATE SKIP LOCKED`), refuses plaintext sends to E2E threads and envelopes missing a current device, and announces device-key changes; pushes say only "New message", and cross-platform interop tests run in CI.
- [EXTREME] Mobile apps holding sensitive data leave nothing readable at rest: memory-only HTTP cache (purge the on-disk cache at launch), `ThisDeviceOnly` keychain items and OS file protection on scratch files.

## Testing against real people's data

- [MUST] Tests and debugging never use a real person's login, session or token; they use temporary accounts created for the run and deleted after, and the run asserts zero test accounts, devices and uploads remain.
- [MUST] Features over private data (mail, chat, health) are tested with mock servers or throwaway accounts; checks against live personal data read only status codes, labels and counts, never content, and act only on items carrying a unique test tag. Synced mail and stored mail tokens fall under the same rule as the inbox.
- [SHOULD] Test artifacts holding credentials (pairing tokens, PIN files, simulators with a logged-in app, build clones carrying tokens) are deleted after the run, and a sweep of every machine confirms it.

## Logging, consent and evidence

- [MUST] Security events are logged (logins, failures, resets, role changes, admin and owner actions with who/when/IP, exports, money movements), logs are rotated and size-capped, and raw webhook payloads, IdP profiles and token query parameters are never logged unredacted.
- [MUST] Each consent stores the timestamp with time zone, the IP (encrypted), the user agent and the version/hash of the exact wording shown, with a separate record for each thing agreed to.
- [MUST] Data that needs consent (recordings, data shared with third parties) reaches no processor or vendor until the consent is recorded, and a refusal stops the upload.
- [SHOULD] Logs ship off the host for incident review, staff can export an evidence record (PDF with audit timeline), and every staff view of a sensitive document is logged.
- [EXTREME] Audit logs are append-only, and money audit rows are written in the same transaction as the action (fail closed if the audit write fails).

## Webhooks and side effects

- [MUST] The webhook signature covers a timestamp, requests outside ±5 min are rejected, and a processed-event-ID table blocks replays inside the window.
- [MUST] Cron and internal trigger endpoints require a bearer secret compared in constant time, and 404 when it's unset.
- [SHOULD] Decisions re-fetch authoritative state from the vendor API instead of trusting the payload, stale or out-of-order events are ignored, handlers acknowledge fast and process idempotently, return 5xx only to request a retry, and never echo verification details.
- [SHOULD] Webhooks you send are HMAC-signed over timestamp + body, go to customer URLs only through the SSRF guard, are rate-capped, carry idempotency keys and go through an outbox table, never fire-and-forget.
- [SHOULD] Every side-effecting job logs inputs, outcome and a correlation ID.

## Money

- [MUST] Card numbers never touch your servers or logs: the provider's hosted checkout or hosted fields collect them (keeping PCI scope at SAQ A).
- [MUST] Postings are balanced double-entry in integer minor units, with overdraft allowed only on named system accounts.
- [MUST] Provider callbacks correlate only through an ID you issued; anything without one is dropped and logged.
- [MUST] Test and live payment keys never mix: production keys are periodically checked as live-mode on the expected account ID, and live databases hold no test-mode provider IDs.
- [MUST] The first live webhook after a payment cutover is confirmed in the provider's delivery log, or by the smallest charge that can be cancelled.
- [MUST] Fee math includes processor fees, and a test asserts the platform fee can't go negative across amounts.
- [MUST] App-store receipts and server notifications (JWS) are verified (ES256, `x5c` chain to a pinned vendor root, validity dates), and Play purchases through the Play Developer API, before granting anything.
- [SHOULD] Ingest dedupe uses unique constraints (`ON CONFLICT DO NOTHING`), not read-then-write.

## Outbound messaging

- [MUST] A message row is written as `queued` before sending, so a crash neither double-sends nor loses the record; messages the gate refused are never auto-retried.
- [MUST] Unsubscribe links and SMS STOP replies take effect immediately (including unread STOPs), and every message type a user can turn off has a setting for it.
- [MUST] If the app messages people who didn't sign up (sales outreach, imported lists): opt-out lists apply at import, export and every share with a third party, an opt-out cancels that person's queued and scheduled messages, and you follow the anti-spam law of each recipient's country.
- [MUST] Turning on auto-send, or running a backfill or re-send, previews the recipient list for human approval and passes every item through the same gate.
- [MUST] Sent-mail archives never store links or codes that grant access ("link removed"), and such emails can't be resent.
- [SHOULD] Per-channel, per-actor send budgets are persistent; bulk sends have rolling caps, quiet hours in the recipient's time zone and dedupe by recipient.
- [SHOULD] Daily opt-out and complaint rates are monitored, and sending auto-pauses below the provider's suspension threshold.
- [SHOULD] The gate is tested with reserved sentinel recipients (`.invalid`, fictional numbers) that senders skip in production test runs, a missing key is recorded as "not delivered" (never silent success), and delivery is checked via the provider's API, not an inbox search.
- [SHOULD] A tested "stop all sending" switch covers every channel (email, SMS, push) and works without a deploy; for high-volume or outreach sending, a second gate outside the app (provider side) fails closed.

## Integrations

- [MUST] Every integration has its own scoped key and documented owner and uses the vendor's documented partner API with service credentials, never internal portal APIs driven by a scraped session or bot-protection workarounds (if a site blocks server fetches, do it client-side with consent).
- [MUST] OAuth refresh-token failures (`invalid_grant`, 401) alert at once, and a dead token never silently falls back to stale data.
- [MUST] A job that fans out to several channels reports partial or failed when any one fails, never "ok".
- [SHOULD] Integrations fail safely (an outage degrades a feature, never corrupts data or double-sends), last-successful-sync per connection is visible and alerts when stale, and sandbox and live credentials never cross into each other's data.
- [SHOULD] Connector syncs advance the cursor only after a successful write, upsert idempotently and keep the raw payload; re-linking a bank or aggregator connection removes the old item at the provider.
- [SHOULD] A pairing or consent step asks for every protocol and scope it will ever need at once, stores the result durably and backed up, and a restart/redeploy test proves no re-pairing is needed.
- [SHOULD] Each scheduled feature has a verified caller, every pipeline stage alerts when its input is empty or was never populated, a job with zero output for a day alerts, and every automated report lands where a human reads it.
