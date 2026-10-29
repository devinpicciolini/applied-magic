# AI features, agents and automation

Tiers: **[MUST]** non-negotiable · **[SHOULD]** strong default · **[EXTREME]** for money, identity, health or family data. Part of [applied-magic](../README.md).

Models in products and agents doing work. A model is an untrusted, expensive, confidently wrong component: cap its
spend, bound its reach, check its output. How to run a coding agent day to day is in [operating.md](operating.md); how
agents plan, review and ship code is in [engineering.md](engineering.md).

## The non-negotiables

- [MUST] Instructions found in web pages, files, emails or tool output are data, not commands: untrusted text is delimited and labelled, the model gets no action tools while reading it, and its output is schema-validated.
- [MUST] Actions with real-world effects (sends, calls, payments, posts) never fire from model output without human approval; agent-written messages default to drafts, sends only on explicit approval for that batch at a human-set throttle, and pauses on any formatting or fact defect.
- [MUST] Hard caps per call (max tokens) and per day/month/job (dollars, or share of a subscription quota), with a ledger row per call and alerts when hit; on a cap, degrade to a cheaper or local model or stop, never keep spending.
- [MUST] Every agent loop or scheduler has a two-part kill switch (a stop that persists across reboots + a DB `enabled` flag re-read every pass), new schedulers ship disabled, and a pause survives every watchdog, auto-deploy and restart script: list the resurrectors and confirm each skips it before calling anything "dormant".
- [MUST] Models only write proposals: code or a human commits facts, scores, prices and numbers, and proposals never reach later prompts as fact.
- [MUST] Agents never type passwords, paste secrets into chat or permanently delete data; a human runs those commands.
- [MUST] Each agent has its own least-scope grant (e.g. draft-only mail), refresh tokens are never copied between agents, and chat-bot interfaces allow-list chat IDs and fail closed.
- [MUST] When a safety classifier or permission check blocks an action, no other agent, tool, host or account performs it instead.

## Spend, loops and schedulers

- [MUST] AI loops have a minimum interval, low default concurrency, a daily run cap, change-gating (run only when inputs change) and staggered first runs.
- [MUST] Every automated AI call is tagged with its app or job, and usage breaks down per tag and per interactive session across all machines, bucketed by each message's own timestamp (a log file's modified time over-counts long sessions), so a spike is traced to its source within minutes; on a spike, attribute before killing, and pause rather than delete.
- [MUST] A job that fails on an item records a per-item failure count and backs off exponentially, never re-queuing the same item at once.
- [MUST] A subscription or provider quota shared by several agents is a single point of failure: quota errors alert, and background agents use a credential separate from the human's interactive one.
- [MUST] Configured model IDs are checked as available on the credential at boot or in the health check, AI token failures alert at once, and alerts say whether the cause is auth (an expired token is the usual "AI is down"), quota or the provider.
- [MUST] Customer-facing traffic runs on a metered API credential, never a personal or team chat subscription unless the provider's terms allow it, and nothing impersonates another client ("you are <vendor's tool>") to obtain access.
- [SHOULD] Long pipelines have a lock against overlapping runs, heartbeats that a long model call can't block, a durable queue with bounded retries and a dead-letter state, and every run ends in a notification; a watchdog revives stuck runs, and a watchdog restart never re-fires jobs that already ran (that pairing caused a crash loop of paid calls).
- [SHOULD] Once-per-period jobs write a durable marker before any side effect, and schedule guards read DB state, never derived filenames.
- [SHOULD] Agent tick scripts move the ticket's state themselves (claimed → in progress → in review) before working, or the next tick re-claims it; chat/queue pollers advance their cursor before processing and dedupe on exact message IDs, never float-timestamp arithmetic (it caused an infinite duplicate-reply loop).
- [SHOULD] If a query or arithmetic can answer it, no model is called; model choice and call shape are routed by task class in one module (the strongest model for reasoning and review, cheaper ones for chat, routine and scheduled work, no web search or tools for a task that needs none).
- [SHOULD] A bulk AI job or paid lookup gets a cost estimate first, takes explicit IDs or a date window (an "everything unprocessed" default is a finding), runs free filters and grading before spending paid credits, and starts with the smallest highest-value slice.

## Model output and prompts

- [MUST] Model output that picks items may choose only from server-provided candidate IDs, with a deterministic fallback when the model fails or the budget runs out; a test forces the cap and asserts the fallback answers.
- [MUST] Generated emails, URLs and names are validated before persisting or sending, structured output is shape-checked before DB writes, generated URLs are allow-listed before fetching, and generated code never runs outside a sandbox.
- [MUST] AI data tools connect as a dedicated read-only DB role (`NOSUPERUSER NOBYPASSRLS`, connection limit, `default_transaction_read_only`, `statement_timeout`) whose `search_path` holds only curated views without hashes, tokens or user-written content; every query runs `BEGIN READ ONLY` with a row `LIMIT` and an external cancel watchdog, and file tools refuse paths that escape the repo after `realpath`.
- [MUST] Money, scores and other figures are computed in code and handed to the model as facts; the prompt forbids recomputing them, so every number shown is testable.
- [SHOULD] Send the model only the fields it needs (sender, subject, short snippet), use 1-indexed lists, state any length limit once, list only tools actually wired up, and give thinking models an explicit budget.
- [SHOULD] Malformed JSON, empty output, a valid-looking wrong answer and a refusal are four distinct failure modes, each handled.
- [SHOULD] User text passed to a shelled-out agent goes through an env var or file, never interpolated into a shell string.
- [SHOULD] Extraction (from mail, documents, statements) returns literal values with where each was found, and "not found" instead of a guess.

## Agent scope and permissions

- [MUST] Agents that edit shared resources (calendars, docs, lists) touch only items they created, tracked by stored IDs and a provenance tag.
- [MUST] Agents never send messages on a person's behalf without explicit approval of the exact text and recipient, and never accept terms, create accounts or toggle the user's VPN or exit node; when a message must come from the human (it should read as theirs, or a system only reacts to humans), the agent hands them the exact text, since a connector post carries app attribution.
- [MUST] Agent skill files, hooks, plugins and setup scripts are audited as code (install/update → network → credential access → shell → external writes) and installed only from their official source, pinned; a familiar publisher is not a verdict.
- [MUST] A hook or plugin that injects rules does so for subagents too (on start, resume and compaction), and a probe proves a subagent sees them.
- [SHOULD] Headless agents run with a strict, explicit tool/MCP config so a broken user-level connector can't hijack them, and after changing agent behaviour you confirm the runtime actually loads the changed file, then run a real probe.
- [SHOULD] An audit receipt is written before any agent sends data off the machine, and CI fails on a new unreceipted egress sink.
- [SHOULD] Self-modifying agents are bounded hard: no secrets, money or auth in scope, a confidence floor, a small nightly cap, a typecheck/test gate that reverts the whole batch on failure, every change committed.
- [SHOULD] Autonomy rolls out by risk tier: low-risk tickets first, PR-gated (agents never merge or deploy), money paths and decisions about a person (credit, fraud, eligibility) human-only, with the model stating signals and likelihood, never intent or accusation.
- [SHOULD] An agent that drafts external writes (comments, tickets, posts) holds no credential for them: it writes to an outbox that a script with the credential checks and delivers.
- [EXTREME] Money-moving or signing actions prepared by agents need a biometric confirmation on the owner's device for each decision.

## AI-written messages, public claims and agent state

- [MUST] Every AI-personalized message is checked before sending: the greeting name matches the person on the address, every claim matches your current docs, and the recipient passes the opt-out check.
- [MUST] Outbound AI copy passes a guard on every field (subject, preview, body) that rejects banned phrases and any number the code didn't compute; public agent posts never claim unverified features or results, and the context fed to a posting agent marks each self-claim verified or planned.
- [MUST] Agent memory follows [operating.md](operating.md) *Memory*: each fact carries its source (user said / agent did / data showed) and an absolute date, and the agent's actions or inferences are never attributed to the user.
- [SHOULD] Technical documents for outside readers label every API and feature claim LIVE or PLANNED, each LIVE claim checked against the current spec, the code and open PRs with a reference.
- [SHOULD] Long agent-run jobs keep their state and send log in a durable file or DB, not only in the session.
- [SHOULD] Intentional removals are documented in the repo with a restore tag, so agents don't bring them back.

## Evals and model data

- [SHOULD] Every model/task pair has a canary eval and a job returning empty for every input alerts; ambiguous classifications are cached and frozen (answers shift with batch composition), and embedding rows store model and version.
- [SHOULD] Eval candidates never see eval vocabulary or fixture-looking paths, judges grade from what the transcript shows was opened (never the candidate's self-report), and held-out sets are never read.
- [SHOULD] Paid eval runs go: zero-cost dry run on real rows → priced pilot of ~3 items per arm against a locked spend ledger → projection with a 95% band, stopping automatically if a bar is missed or the projection exceeds the cap.

## Private and local AI

- [SHOULD] After each upgrade, local AI tools are verified to make no outbound connections: telemetry and cloud features off, set in a launch config the package manager won't rewrite.
- [EXTREME] AI on sensitive data (summaries, transcription) runs on hardware you control, not a box someone else administers.
