# Delivery: CI/CD, supply chain and mobile release

Tiers: **[MUST]** non-negotiable · **[SHOULD]** strong default · **[EXTREME]** for money, identity, health or family data. Part of [applied-magic](../README.md).

"Why does deploying take 15 minutes? It should be instant." For a small team the pipeline **is** the senior engineer who
never tires, never skips the checklist at 11 pm and is still there when someone leaves. Speed comes from **lanes**, not
from removing gates.

**Why the gates pay for themselves.** A bad deploy to a money, auth or customer-data path costs days (incident, data
repair, apology, sometimes regulators); a 10-minute gate that catches one a quarter is the cheapest insurance you'll buy.
Small teams have no QA department or on-call: automated checks are the QA, automatic rollback is the on-call. Partners,
banks and enterprise buyers ask "how do you control changes to production?", and "reviewed, tested in CI, deployed from
main after green checks, smoke-tested and rolled back automatically" wins deals where "we push from laptops" loses them.
Trusted gates make people ship more often. A slow gate gets faster (caching, parallel jobs, more runners), never deleted.

## The non-negotiables

- [MUST] **Normal lane** (default for app code): PR → CI (typecheck, lint, unit + integration tests, security gates) → review → merge queue → deploy the main tip only after green CI → production smoke test → automatic rollback on failure.
- [MUST] Production is deployed only by the pipeline from the main tip after green CI: never from a laptop, a hand edit on a server or a workflow dispatched on a branch; deploys fast-forward only and refuse a dirty live checkout.
- [MUST] Main is protected by a ruleset or branch protection (PR required, required checks, no force-push or deletion, no bypass for admins); where the plan can't enforce that, the deploy refuses non-main or non-green commits and a versioned pre-push hook blocks pushes to main (`core.hooksPath` verified on every clone).
- [MUST] Deploy success is judged by the live URL and build ID, not an exit code: the deployed commit SHA is exposed and checked after every deploy, and a smoke test (health incl. DB, sign-in page, one core read) triggers automatic rollback.
- [MUST] Security gates run on every push: secret scanning (committed `.env` refused), dependency audit, a client-bundle scan (no server env names, no source maps) and an auth-safety test (passwords only hashed, cookies `httpOnly`, unverified users can't sign in, reset revokes sessions, rate limits survive restarts).
- [MUST] Migrations are backward-compatible (expand → migrate → contract) so code rolls back without schema rollback; destructive steps are separate, reviewed and preceded by a verified backup.
- [MUST] Lockfiles are committed, installs are frozen (`pnpm install --frozen-lockfile`, `npm ci`), and dependency install scripts are off (`ignore-scripts=true` in `.npmrc`; pnpm ≥10 blocks them by default) or allow-listed by name (pnpm `onlyBuiltDependencies`), never run through `eval`.
- [MUST] Test runners default to a scratch base URL and DB with a guard that refuses production; destructive scripts verify the target DB name, and scripts that change production or reviewer accounts live outside test dirs behind a guard.
- [MUST] Every workflow sets `permissions:` (top level `contents: read`, writes granted per job), third-party actions are pinned to a full commit SHA, secrets are scoped to the jobs that need them, and fork or outside PRs never run with secrets or a write token (`pull_request_target`/`workflow_run` jobs never check out, build or run PR code).

## Lanes

- [MUST] **Hotfix lane**, real incidents only: a small branch from main, the **same** required checks (tests may be scoped), one fast-tracked reviewer or a documented solo override reviewed within 24 h, deployed through the same pipeline; never a server edit or skipped security gate, and written up afterwards (what broke, the fix, the test that now prevents it).
- [SHOULD] **Website/content fast lane:** marketing sites, docs and copy live in a separate repo or path-filtered pipeline with light checks (build, link check, accessibility smoke, no secrets), deploy in a minute or two, and never share credentials or a pipeline with the app that touches money or customer data.
- [SHOULD] **Path-based risk tiers:** auth, payments, permissions, encryption, migrations, outbound messaging, `.github/`, deploy scripts and lockfiles are listed in `CODEOWNERS` with code-owner review required; copy- or style-only changes may run a lighter subset.
- [SHOULD] Feature flags let risky work merge and deploy dark, then switch on and off without a deploy.
- [SHOULD] The safe path is the easy one: one command or PR label for the hotfix lane, incident templates, review-reminder bots.
- [SHOULD] Small teams that maintain and sell keep one page describing SDLC and change control (lanes, reviews, gates, rollback, backups, access), which answers most security questionnaires, with evidence accruing automatically (CI logs, deploy history, PR reviews, incident write-ups), never recreated by hand before due diligence.

## Pipeline and gates

- [MUST] Deploy and merge scripts run with `set -euo pipefail`; a check's exit code is never piped through `tail` (a red check has merged that way), and verification is an explicit `if ! …; then exit 1`, run as a user that can read the file.
- [MUST] A test lane that ran zero tests fails: lanes pass on the runner's test count, not the exit code, and selectors matching nothing are rejected.
- [MUST] CI reads gate policy (required checks, thresholds, byte caps) from the base branch, never the PR's tree, so a PR can't relax its own gate; unknown policy keys fail closed.
- [MUST] A CI job flags gate-relaxing hunks (see [engineering.md](engineering.md), *Code review*) and requires a named human disposition in the PR.
- [MUST] After every web build, grep the client bundle for the production API host (≥1 hit) and localhost/dev URLs (0 hits); a domain change triggers a rebuild, since public env is baked in at build time.
- [MUST] When the deploy system rejects a push (e.g. a hostname another repo already claims), the pusher sees it as a commit status or message, not just a server log line.
- [SHOULD] A merge queue (or rebase + re-run before merge) so two green PRs can't break main; merges use CI results for the PR's current head SHA, new pushes dismiss stale approvals, and squash + stacked branches have a documented restack (`git rebase --onto origin/main <old parent head>`).
- [SHOULD] A stopped merge or deploy queue (conflict, red check) is restarted in the same session or alerts; it never idles overnight.
- [SHOULD] Deploy jobs share one `concurrency` group per environment with `cancel-in-progress: false` (a deploy is never killed mid-swap), while superseded PR and main CI runs are cancelled.
- [SHOULD] CI builds once and reuses the artifact, caches dependencies on enough runners that the queue isn't the bottleneck, doesn't re-run the identical suite on main after the merge queue tested the merge result, and targets under 10 minutes for the normal lane.
- [SHOULD] SAST runs on every PR, env access goes through one module (an AST check finds `process.env` nowhere else), log redaction and retention have tests, and size budgets fail CI (modules ~800 lines, functions ~150, generated prompt/doc files over a byte cap).
- [SHOULD] A credential-scanning pre-push hook is installed, and PR titles, bodies and comments are secret-scanned before posting.
- [SHOULD] Deploys are observable (who deployed which commit when) with a one-command rollback to the previous release.
- [SHOULD] Read-only reports and analytics run as saved queries or from a path excluded from deploy, never as product merges that trigger full CI and a production deploy.
- [SHOULD] CI service versions (DB, runtime) match production, tests run under at least two time zones, and processes run with `TZ=UTC`.

## Workflows and runners

- [MUST] Untrusted values (`github.event.*` titles, bodies, branch names, `inputs`) never appear as `${{ }}` inside `run:`; they pass through `env:` and are quoted, and `zizmor` plus `actionlint` run on every change to `.github/`.
- [MUST] `actions/checkout` sets `persist-credentials: false` unless the job pushes, and no step uploads the workspace or `.git` as an artifact.
- [MUST] Self-hosted runners never serve public repos or fork PRs, run each job on a fresh or reset environment (ephemeral runners or rebuilt containers), run as an unprivileged user that can't read the host owner's files or keys, and only the deploy runner can reach production.
- [MUST] Production credentials live only in a protected environment whose deployment branch policy allows `main` alone, with required reviewers for manual deploys where the plan supports them; repo- and org-level secrets hold nothing that can change production.
- [SHOULD] Cloud and registry access from CI uses OIDC federation (short-lived tokens, trust bound to repo plus branch or environment in the `sub` claim), not long-lived keys stored as secrets.
- [SHOULD] The org default workflow token is read-only, "Allow GitHub Actions to create and approve pull requests" is off, workflows from first-time and outside contributors need approval, and the allowed-actions policy lists GitHub-owned plus named publishers and requires full-SHA pinning (a pinned action still isn't a trusted one).
- [SHOULD] Pinned action SHAs carry a version comment and are bumped by Dependabot or Renovate; release and deploy jobs never restore caches that PR runs can write, and cache keys include the lockfile hash.
- [SHOULD] Runtimes on CI are pinned to the exact patched version (or `check-latest: true`), because self-hosted tool caches keep serving the highest cached, possibly vulnerable, patch.
- [EXTREME] CI runners are fenced: their own network, egress only through a gateway, no route to production or home/office networks, no long-lived production secrets.

## Deploys and post-deploy

- [MUST] Deploys build in a separate stage, health-check and roll back automatically; deploy keys are read-only and per repo, so the server can't push.
- [MUST] Code reaches build and deploy hosts only through git; never rsync a working tree over a checkout (it silently reverts generated files such as plists and clobbers `.env`).
- [SHOULD] Rollback is proven by deploying a deliberately broken build once, and again after any pipeline change: it must fail, restore, and keep serving.
- [SHOULD] Each deploy diff is scanned for malware patterns (obfuscated `eval`, `global.` assignments in config files, build configs over ~5 KB, `.gitignore` gaining unknown batch or JSON files, unexpected install scripts, binary blobs); a large config passes only if its sha256 is on a reviewed allow-list.
- [SHOULD] Deploys that `rsync --delete` target a code-only directory; uploads and data never live inside it.
- [SHOULD] The hostname → repo/service mapping lives in one file; a new repo for an existing site moves the mapping explicitly and turns off the old deploy, retiring a repo releases its hostnames, and auto-publish from repo config is limited to a sandbox domain, refuses host clashes, deny-lists core hosts and logs every registration.
- [SHOULD] A pre-deploy baseline per key page (screenshot, console errors, load time, 404s) is compared for ~10 minutes after deploy on the homepage plus the top ~5 pages, alerting only on changes that persist across 2+ checks (load failure critical, new console errors high, >2× load time medium, new 404s low).
- [SHOULD] Post-deploy depth matches the diff (docs/config → smoke; backend → errors + latency; frontend → full visual + console), production synthetic checks run the read-only parts of the money and identity flows every few minutes, and each deploy writes a report: PR, SHAs, stage durations, evidence, verdict.

## Migrations

- [MUST] Migrations are applied only by the migration tool, never by hand (hand runs have skipped `CREATE INDEX`); applied migrations are immutable (enforced in CI), and schema push/diff tools never run unattended against real databases.
- [MUST] When deploys don't run migrations, the pipeline blocks until the schema matches, and contract steps ship at least one release after the code stops using them.
- [SHOULD] Migrations set `lock_timeout` (a few seconds) and `statement_timeout`, so an ALTER waiting on a lock fails and retries instead of queueing every query behind it.
- [SHOULD] Large tables (~100k+ rows): indexes built concurrently, back-fills batched before `NOT NULL`, ALTERs combined into one lock, run off-peak; columns dropped only after a deprecation release, and new foreign keys indexed.

## Testing in the pipeline

- [MUST] Real-browser E2E (Playwright or similar) covers the money and identity paths: sign up, verify email, sign in with 2FA, reset password, the core flow that makes or moves money, and role A can't see role B's data.
- [MUST] Cross-tenant isolation tests run in CI against a running server.
- [MUST] Suites run against production clean up after themselves and assert nothing was left behind.
- [SHOULD] Visual and accessibility smoke tests on key pages, and a keyboard-only + screen-reader pass on sign-up, sign-in and the core flow before release (visible focus, labelled inputs, no focus traps, `aria-invalid`/`aria-describedby` on errors).
- [SHOULD] Public API and webhook contract tests come from generated schemas so partners don't break silently, and test data is synthetic or anonymized, with a guard checking the DB name and the outbound gate before tests run.

## Supply chain and repo safety

- [MUST] Dependency alerts are on, and critical runtime CVEs (Node, Bun, Python, OpenSSL) are patched within days on servers, CI runners and dev machines.
- [MUST] Untrusted repos are never opened in an IDE: workspace tasks (`.vscode/tasks.json` with `runOn: folderOpen`) execute on open, and `.vscode/settings.json` with `task.allowAutomaticTasks` removes the prompt; review in the browser or CLI first.
- [MUST] When a developer machine is found infected, revoke its GitHub sessions, tokens and SSH keys, then scan every branch of every repo it could push to before trusting any history (payloads have been force-pushed to all branches by amending the latest commit).
- [SHOULD] An auto-execution scanner runs on every outside code drop and new-collaborator merge: editor tasks and settings, devcontainer and IDE run configs, `.gitattributes` filters, `core.hooksPath`, git hooks, lifecycle scripts, binaries whose magic bytes contradict their extension (a "font" run by `node`), oversized config files; `git log -S` alone misses obfuscated variants.
- [SHOULD] Force-pushes to any branch alert, author/committer name and time-zone mismatches are flagged, and untrusted trees build only on an egress-locked host.
- [SHOULD] Dependency updates are automated (Dependabot or Renovate, grouped) with a cooldown of ≥3 days on new versions (Dependabot `cooldown`, Renovate or pnpm `minimumReleaseAge`), security fixes exempt, and auth, payment and framework bumps never auto-merged.
- [SHOULD] PRs run dependency review (`actions/dependency-review-action`) failing on new high/critical vulnerabilities, and lockfile diffs are checked so every resolved URL points at the expected registry (`lockfile-lint`); a lockfile change without a manifest change is flagged.
- [SHOULD] Supply-chain findings are rated by reachability and exposure (prod vs build, direct vs transitive, vulnerable function called), "dev-only" never caps severity because dev dependencies run with CI/publish credentials, and scanners run pinned by digest, offline, telemetry off, never executing the target's code.
- [SHOULD] Tools are installed from official releases only (repackaged builds have bundled vulnerable runtimes), and version managers and package managers are pinned (`packageManager`, `.nvmrc`) so a major jump can't silently change behaviour.
- [SHOULD] Published packages and images use trusted publishing (OIDC, no long-lived publish token) with provenance (`npm publish --provenance`, `actions/attest-build-provenance`), and registry accounts require 2FA.
- [EXTREME] New collaborators' history is checked for commit-rewrite or force-push malware before access, and outside contributions are diffed against the last known-good commit, not just the PR view.
- [EXTREME] Release artifacts are built by a reusable workflow on isolated runners (SLSA Build L3) and the deploy verifies the attestation (`gh attestation verify`) before switching traffic.

## Repository settings

- [MUST] Only owners create repos, change settings or add collaborators, and the org requires 2FA.
- [MUST] Secret scanning with push protection is on for every repo that supports it.
- [SHOULD] Release tags are protected by a tag ruleset (no update or delete), merged branches auto-delete, stale branches are pruned, and a CI hygiene job rejects unwanted tool-attribution lines and unexpected commit emails.
- [SHOULD] Every repo has CI running the check suite, `.env.example` (names only), a DB-checked health endpoint and a "where it runs / deploy / restore" README section.
- [SHOULD] Public repos have `SECURITY.md` with private vulnerability reporting on, and run OpenSSF Scorecard so regressions in these settings show up.

## Mobile release

- [MUST] No secret in an app build grants more than the user's own access, jailbreak/rooted-device assumptions are documented, and debug-only launch arguments and env hooks (auto sign-in, screen jumps, API base-URL overrides, admin flags) are compiled out of release builds, verified on the release binary.
- [MUST] The app's server address is a stable name, never a machine-specific address that breaks on migration.
- [MUST] Server APIs stay backward-compatible with shipped builds; contract tests cover the exact fields mobile models require (one missing field can drop a whole list), and a web route is removed only after confirming no shipped client uses it.
- [MUST] One-time client-side migrations (key moves, schema upgrades) run at every launch until a completion marker is set, and an upgrade test that skips a build (N-2 → N) passes before release.
- [MUST] Builds are signed at archive time (unsigned archives silently drop capabilities such as Sign in with Apple and push), and before upload the built Info.plist, privacy manifest and exported entitlements (usage strings including SDK-pulled ones, URL schemes, launch screen, sign-in, push) are checked and the store's own validation runs; after any capability change, delete and regenerate the provisioning profile.
- [MUST] Android release builds (minified, R8) run on a real device before upload, since shrinking breaks serialization that debug builds never exercise; Play App Signing is on and the upload keystore is backed up encrypted off the build machine.
- [MUST] Ship scripts exit non-zero on any failed step, poll processing and assert the build is attached to the version and available to testers (beta builds assigned to a tester group, or they're invisible).
- [MUST] Before submission: newest build attached, copyright and age-rating answers filled in, account deletion and report/block for user content exist, AI-data consent and privacy answers filled in, payment keys live, the review backend always on, demo credentials verified by signing in.
- [MUST] A first in-app purchase or subscription is added to the same review submission as the app version, and the review account has no comped entitlement, so the paywall is the first thing the reviewer reaches.
- [MUST] Availability is verified on the public storefront after approval ("ready for sale" can mean zero territories, and the management API can disagree with the store).
- [MUST] Simulator, dev and test builds can't write to real user records such as health data, and device IDs sent to servers are stable across reinstalls.
- [SHOULD] Releases use phased release (App Store) or staged rollout (Play) watched against crash-free rate, and the app checks a server-provided minimum supported version so a bad shipped build can be forced to update.
- [SHOULD] Builds run from the real path, not a symlink, and generated files (Info.plist, project files) are regenerated after every pull, never synced; the binary hash is compared when a change "doesn't show", and every build shows a visible build ID.
- [SHOULD] Version and build numbers have one source of truth checked against the store's latest build (parallel sessions ship too) and asserted in the built binary (generators can write literal values), major releases use manual release with reviewed notes, declared device families match what's tested, simulator-blind features get a real-device check, and builds meet the store's current SDK and target-API requirements.
- [SHOULD] Headless signing uses a dedicated CI keychain with a stored non-trivial password, and simulator builds are ad-hoc signed.
- [SHOULD] Media streams are tested on the real target player over the real network path with a 2-minute soak showing zero frame gaps (native players may reject low-latency modes over plain HTTP; overlay VPNs with ~1280 MTU stall large packets).
- [SHOULD] Config that changes at go-live (publishable keys) is fetched from the server, so going live needs no rebuild; push keys are stored server-side with restricted permissions and backed up encrypted.
