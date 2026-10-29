# Attack classes: the quick reference

Tiers: **[MUST]** non-negotiable · **[SHOULD]** strong default · **[EXTREME]** for money, identity, health or family data. Part of [applied-magic](../README.md).

Each line: the attack → the defense to prove. If the app has the surface, it needs the defense, shown with code
(file:line), config or a probe against dev/staging. Mechanisms in depth live in [app-security.md](app-security.md).

## The non-negotiables

- [MUST] **Middleware-only auth bypass** (e.g. a spoofed `x-middleware-subrequest`): auth is re-checked in every route handler, server action and API route (each is a public HTTP endpoint), the framework is patched, and internal headers are stripped at the edge.
- [MUST] **Broken object/function-level authorization (IDOR/BOLA)**: every object lookup is scoped to the caller and every admin/internal function checks the role server-side; probe with another user's ID and a non-admin session.
- [MUST] **Price or total from the browser**: the server computes price, discounts, tax, limits and currency from its own data; the client sends IDs and quantities only.
- [MUST] **Race conditions / double spend** (coupon twice, withdraw twice, invite reused, oversold stock): row locks or atomic conditional updates (`UPDATE … WHERE balance >= x`) plus unique constraints, tested with parallel requests.
- [MUST] **SQL, NoSQL and command injection**: parameterized queries everywhere including admin tools and scripts, no user-controlled operators (`$where`, `$ne`), argv arrays and never shell strings with user input.
- [MUST] **SSRF**: user-supplied URLs are fetched with http(s) only, every resolved address checked against private, loopback, link-local, metadata, CGNAT/VPN, multicast and IPv6 ULA ranges (after normalizing IPv4-mapped IPv6 and decimal/octal forms), redirects followed manually and re-checked per hop (max ~5), the resolved IP pinned, timeouts and byte caps set, and an egress firewall as the backstop.
- [MUST] **Exposed `.git`, `.env`, source maps, backups and debug tooling**: request `/.git/HEAD`, `/.env`, `*.map`, `.sql`/`.zip`/`.bak`/`.DS_Store`, `/api/debug`, `phpinfo`, GraphQL playground in production and expect 404; directory listing off.
- [MUST] **Failing open** (OWASP 2025 A10): an exception or timeout in an auth, permission, payment, rate-limit or send-gate check denies the request; no `catch` returns success, an empty filter or a default user.
- [MUST] **Denial of wallet**: every endpoint that costs money per call (AI, SMS, email, maps, enrichment) has per-user and global caps with alerts.
- [MUST] **Exposed hole found in production**: contain it at the edge first (a proxy deny rule, minutes), ship the reviewed code fix through CI, then check logs and accounts for prior use; report contained / prior use checked / fixed as three separate states.

## Framework and platform traps

- [MUST] **Public env prefixes** (`NEXT_PUBLIC_`, `VITE_`, `EXPO_PUBLIC_`): nothing secret gets one; the built bundle is scanned.
- [MUST] **Preview/staging deployments** are behind auth or an IP allow-list and never connected to production data or keys.
- [MUST] **BaaS storage and rules** (Supabase, Firebase, S3): private by default, RLS/storage rules on **every** table and bucket, never `allow read, write: if true`; call the API directly with the public anon key to prove it can only do what an anonymous user should.
- [MUST] **Open databases and admin panels**: no DB, Redis, Elasticsearch or admin UI reachable from the internet, no default passwords.
- [MUST] **Decompiled clients**: every shipped mobile or desktop app is assumed reverse-engineered, so it holds no secrets, privileged endpoints or business decisions, and the server re-authorizes everything a client can call.

## Business logic and fraud

- [MUST] **Negative or extreme values**: negative quantities and amounts, zero-price tricks, overflow and absurd limits are rejected server-side.
- [MUST] **SMS pumping / toll fraud**: per-phone, per-IP and global SMS caps, a country allow-list, a carrier/line-type check and a bot challenge before any send, with alerts on volume spikes.
- [MUST] **Email pumping** via contact, invite and reset forms: rate limits and bot challenges, and users never choose the full content of mail you send to third parties.
- [MUST] **Workflow skipping**: the server enforces each record's state machine (no submit, approve, pay or ship from a state that doesn't allow it, no required step left empty); probe by calling each step out of order and with fields missing.
- [SHOULD] **Promo, trial and business-flow abuse** (API6): one coupon, referral or bonus per user/device/payment method, velocity limits on bulk buying, booking or holding inventory, and manual review above thresholds; trials tied to verified email + phone or a payment method, disposable domains detected.
- [SHOULD] **Fake sign-up clusters** (placeholder names, disposable email, virtual numbers, many identities from one network): sign-ups are linked by IP, device and phone, and the whole cluster is closed together.
- [SHOULD] **User-to-user abuse** (spam, harassment) wherever people see each other's content: block, report and a takedown path whose reports a named person reads.
- [SHOULD] **Unknown test or probe records in production** are investigated before they're touched (who, IP, what they touched, whether anything was sent), then closed and flagged so metrics and queues exclude them.

## Account takeover

- [MUST] **Credential stuffing**: rate limits per IP **and** per account, tiered lockout, breached-password checks, a bot challenge after failures, 2FA for privileged users, alerts on spikes.
- [MUST] **Password reset poisoning**: reset and verification links are built from a configured base URL, never the request's `Host`/`X-Forwarded-Host`.
- [MUST] **Reset/verify/OTP bombing**: per-account and per-IP send limits.
- [MUST] **Email change takeover**: re-authentication, confirmation to the **old** address, verification of the new one, and every session revoked afterwards.
- [MUST] **2FA bypass and session fixation**: the second factor is enforced server-side on every privileged path (calling the next endpoint directly or flipping a response field gets nothing), codes are attempt-capped, a reset never skips 2FA, and a new session ID is issued at login and privilege change.
- [MUST] **User enumeration**: identical status, body and timing for existing and unknown accounts on login, sign-up, reset and invite.
- [MUST] **Account pre-hijacking and OAuth flaws**: verify email before linking and retire the squatter's password on a verified OAuth link; PKCE + `state`, exact redirect URIs, ID tokens verified, accounts keyed by `iss` + `sub` (see app-security *OAuth*).
- [MUST] **Magic links eaten by mail scanners**: tokens are consumed by a POST from the landing page, not on GET.
- [SHOULD] **SMS 2FA weaknesses** (SIM swap, recycled numbers): prefer passkeys or TOTP; SMS is the weakest factor.

## Tokens and crypto

- [MUST] **Weak or confused JWTs**: HMAC secrets are ≥256 bits of CSPRNG output (never a word, default or tutorial value), asymmetric keys for tokens other services verify, the algorithm pinned (`alg: none` and confusion rejected), `kid` looked up only in a trusted key set, rotated on leak.
- [MUST] **Timing attacks and predictable tokens**: constant-time comparison (`timingSafeEqual`) for every secret, token, signature and hash, and CSPRNG only (`crypto.randomBytes`/`randomUUID`), never `Math.random`, timestamps or sequential IDs.

## Injection

- [MUST] **XSS** (stored, reflected, DOM): auto-escaping templates, an allow-list sanitizer for HTML/markdown, CSP; review every `innerHTML`, `dangerouslySetInnerHTML`, `v-html`, markdown renderer and SVG upload. Every field shown to another user or admin (names, notes, file names, sign-up fields) gets a stored-XSS probe, and junk accounts left by probes are found and removed.
- [MUST] **Server-side template injection**: user input is never rendered as a template.
- [MUST] **Prompt injection**: untrusted text is delimited data, the model gets no action tools while reading it, and outputs are schema-validated (see [ai.md](ai.md)).
- [SHOULD] **Parser attacks**: no deep-merge of untrusted JSON (prototype pollution: use `Object.create(null)` maps or schemas), no catastrophic regexes and length limits first (ReDoS), external entities off (XXE), and no `eval`/`pickle`/`unserialize`/full YAML load on untrusted data.
- [SHOULD] **Output-context injection**: exported cells starting with `= + - @` are escaped (CSV), and newlines and control characters are stripped or escaped in headers, redirects, mail headers and logged input (CRLF, log injection).

## Webhooks and callbacks

- [MUST] **Forged or replayed webhooks**: signature over the raw body and a timestamp, constant-time compare, ±5 min window, processed-event-ID dedupe, and 404 when the secret is unset; probe with an unsigned, a re-signed-body and a replayed request.

## Files

- [MUST] **Path traversal**: every file read/serve route resolves the path (`realpath`) and verifies it stays inside the allowed root; you generate the file names.
- [SHOULD] **Zip slip and polyglot uploads**: extraction checks every entry path; uploads are magic-byte sniffed, SVG refused or sanitized, and served `nosniff` + sandbox CSP + `attachment` from a separate origin.

## Browser and cross-site

- [MUST] **Clickjacking**: `frame-ancestors 'none'` (or an allow-list) / `X-Frame-Options: DENY`.
- [MUST] **Open redirect**: every `next=`/`redirect=`/`returnTo=` accepts only same-site relative paths or an allow-list.
- [MUST] **CSRF**: `SameSite` cookies plus `Origin` (or `Sec-Fetch-Site`) checks on state-changing requests, no state change on GET; CSRF tokens where cookies must be cross-site.
- [MUST] **CORS misconfiguration**: no reflected origin with credentials, no `null` origin.
- [MUST] **Mass assignment and excessive data exposure**: explicit allow-lists of writable fields (role, provider IDs and verification flags are never client-settable), and explicit output schemas, never whole rows with hashes, tokens or internal fields.
- [SHOULD] **Cookie tossing and tabnabbing**: `__Host-` cookies and no untrusted content on subdomains of the app domain; `rel="noopener"` on links to user-supplied URLs.
- [SHOULD] **Web cache poisoning/deception**: CDN cache keys include everything responses vary on, authenticated pages are `no-store`, and paths like `/account.css` aren't cached.

## Supply chain

- [MUST] **Hijacked third-party script hosts** (the polyfill-CDN takeover): self-host third-party JS, or pin with SRI, keep an inventory of external script domains and remove abandoned ones.
- [MUST] **Malicious packages** (typosquats, dependency confusion, compromised maintainers, postinstall payloads): lockfiles, install scripts off by default, scoped registry config for internal names, dependency audit, review before adding.
- [MUST] **Malicious repos**: untrusted code is never opened in an IDE (workspace tasks run on open) and its scripts never run before being read (see [delivery.md](delivery.md)).
- [SHOULD] **Compromised CI actions**: third-party actions pinned by SHA; least-privilege tokens.

## Infrastructure and DNS

- [MUST] **Subdomain takeover**: DNS records are removed when their target (tunnel, bucket, PaaS app) is deleted, and CNAMEs are audited regularly.
- [MUST] **Locked-down surface still works**: after locking a public surface to a path allow-list or upgrading the app behind it, re-run the whole outside journey off-network with the network log open (link, assets, fonts, the action, completion page, links in emails, "send a copy"): zero 403/404/502, proxy logs clean, admin paths re-probed.
- [SHOULD] **Request smuggling and Host header attacks**: proxy and app servers patched and consistent, ambiguous `Content-Length`/`Transfer-Encoding` rejected, unknown hosts closed by the proxy's default server.
- [SHOULD] **DNS rebinding**: resolved IPs pinned for server-side fetches; local dev servers check `Host`.
- [SHOULD] **UID-matched firewall rules**: tested against every service sharing that UID (container port proxies can run as the same user); public paths re-checked right after applying, with automatic revert on failure.
- [EXTREME] **Fetcher pivot**: server-side fetchers run as a separate process/user with no route to internal services at all.

## Abuse and API surface

- [SHOULD] **Scraping and resource exhaustion**: rate limits, pagination caps, unguessable IDs, body and upload limits, query timeouts, job-queue limits.
- [SHOULD] **Realtime and GraphQL**: WebSockets authenticate on connect, authorize every subscription and re-check on token expiry; GraphQL has depth, complexity and batching limits, field-level authorization and introspection off in production.
- [SHOULD] **Improper inventory and unsafe consumption**: old API versions and forgotten endpoints are removed, and third-party API responses are validated like user input, with timeouts and no blind redirect following.
