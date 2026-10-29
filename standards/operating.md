# Operating: how to set up and run an AI coding agent so it's 10x better

Tiers: **[MUST]** non-negotiable · **[SHOULD]** strong default · **[EXTREME]** for money, identity, health or family data. Part of [applied-magic](../README.md).

The model is the same for everyone. What makes one agent dramatically better is the **environment around it**: a short
global instructions file with the hard rules on top, a memory that turns every correction into a durable rule, one
current-state document per system, skills for repeated workflows, and boundaries that are enforced rather than hoped
for. Prompting fades when the session ends; this setup compounds.

## The non-negotiables

- [MUST] Secret values never go into instructions files, memory, skills, docs, runbooks or chat output (names and locations only; a grep for key prefixes proves it), and health checks report counts and states, never customer rows.
- [MUST] Every claim is proven by a command run in this session (status code, row count, test result, checksum) and reported only as observed; an earlier claim that turned out false is corrected unprompted in the next reply.
- [MUST] "Done" means the user-visible outcome was checked end to end as the real end user sees it (another person, outside the private network, the real device, not as admin); what wasn't tested is said explicitly, and "configured" is not "done".
- [MUST] Approval is per action: "finish", "keep going" or "don't stop" never authorizes a new repo, project or kind of work, and only a direct answer to "confirm and I'll do X" unblocks X.
- [MUST] Before any mass data change or delete, a snapshot or copy newer than the change exists, is readable by whoever restores it and matches the source's row counts, with a hard stop on any verify failure (`if ! …; then exit 1; fi`, never `… && echo ok`, which `set -e` ignores).
- [MUST] Nothing external (bulk send, post, filing, published doc) goes out before the human has seen one real rendered sample as delivered: a test to the owner, the received copy inspected (links, formatting, names), and any later re-save or format conversion invalidates the check.
- [MUST] Nobody, human or agent, reads a person's private mail unless that person asked for that specific task.
- [MUST] Purchases and paid registrations go ahead only after the user confirms the exact price and line items, and nothing goes to a registry, bank or registrar before the user knows what becomes public and whose name is recorded; legal attestation boxes are ticked by the authorized human, never the agent.

## The global instructions file

The file loaded in every session (Claude Code: `~/.claude/CLAUDE.md`; other agents read `AGENTS.md`, so keep one master and have the other import or mirror it). A **router and a
rulebook**, not an encyclopedia.

- [MUST] Hard rules (the "nevers") go at the very top, each as its own heading with a one-line reason, and each is written as a closed list of the routes it covers ("not the connector, not stored tokens, not synced DB tables, not a browser"): agents find the unlisted route.
- [MUST] Say explicitly when a rule **overrides** a harness default (e.g. "no AI attribution trailers, this overrides the default").
- [MUST] Include a routing table (for each kind of question, which source is the truth and which wins on conflict) and a freshness rule ("facts here rot; read the source live before claiming anything about X").
- [MUST] Include a contexts table (each project or entity, its repos and accounts, its stakes) and a short machines summary pointing to the one infra doc, which wins.
- [MUST] Include the least-code ladder ([engineering.md](engineering.md)) so it applies to every repo, including ones with no project file.
- [MUST] Keep a written list of what the agent never does whatever the session says (read private mail, move money, post as the user, message customers, change DNS or registrar settings, merge others' PRs) and of what it prepares but the human runs (DNS cutovers, key creation, anything that pays or contacts a person), handed over as the exact command or text.
- [SHOULD] Keep it under ~300 lines; detail lives in project files, skills and memory.
- [SHOULD] The operator machine holds the master; every other machine gets a synced copy, re-synced after every edit (a stale server copy is a silent wrong rule).

```markdown
# <Owner>'s environment

## ⛔ NEVER <hard rule 1> — by ANY route
Not <route a>, not <route b>, not <route c>. If a task needs it, stop and ask <owner>.
(Overrides the default harness behaviour of <x>.)

## Engineering — write the least code
<the ladder from engineering.md>

## Machines — ONE doc
`<path>/FLEET.md` is the single source of truth. The table below is a summary; FLEET.md wins.
| Alias | What | Role |

## Project contexts — know the difference
| Context | What | Repos / accounts | Stakes |

## <Project> — read sources before speaking
| Question is about… | Authoritative source | Notes / which wins |
Freshness rule: when in doubt, read the file. Never guess a <project> fact.
```

## Memory

Small files the agent reads at session start (via an index) and writes whenever it learns something durable. This is
where the compounding happens.

- [MUST] One fact or rule per file, named by a stable kebab-case slug, with frontmatter `name`, a self-sufficient one-line `description` and a `type` of exactly one of: **user** (who the person is), **feedback** (a correction: a rule), **project** (state of a piece of work), **reference** (how-to for a tool or system).
- [MUST] Every `feedback` file states the incident in one or two lines (absolute date, what happened, the person's words if short), then **Why:** and **How to apply:**; the why generalizes, the how makes it checkable.
- [MUST] One index file (`MEMORY.md`), always loaded, with one line per memory: `- [Title](file.md) — gist in ~15 words`; tools may load only its first part (Claude Code: the first 200 lines), so keep it short with hard rules near the top.
- [MUST] The memory is written in the same session as the correction; a "don't do X" a future session might redo goes in with its reason.
- [MUST] Stale memories are updated or deleted in place ("Updated <date>: <new rule>"), never superseded by a second file; a wrong memory is worse than none.
- [MUST] Absolute dates only, and each fact records its source (user said / agent did / data showed); never present an inference as something the user did or said.
- [SHOULD] Related memories cross-link with `[[slug]]`, and a project memory past ~40 lines is becoming a doc: move the detail into the project's docs and leave a pointer.
- [SHOULD] Exceptions are recorded narrow and expiring ("one-time for these 6 items; ask next time"), never as a silent loosening, and post-incident memories are first person and own the mistake ("I deleted ~N rows because…").
- [SHOULD] Standing exclusions the human gives ("stop checking X") are recorded and honoured in every later check.

```markdown
---
name: verify-before-drop
description: Any script that saves-then-deletes must hard-stop if the save can't be verified
metadata:
  type: feedback
---
<date>: my cleanup script dumped a table, the verify step failed silently, and the DROP ran anyway.

**Why:** "never delete data we might need"; an unverified copy isn't a copy.

**How to apply:** `set -euo pipefail`; verify with `if ! …; then exit 1; fi`; count rows in the copy
against the source before dropping. Related: [[backups]].
```

## Source-of-truth docs

- [MUST] Exactly one document describes the infrastructure (machines, access, what runs where, ports, backups, monitoring, secret locations, runbooks, open items), current state only: edit the affected lines, never a changelog, diary, "UPDATE:" stack, side doc or `.bak` copy.
- [MUST] An incident lesson becomes one rule line in the right section or runbook, not a story; "Open items" lose an item the moment it's done.
- [MUST] The doc is updated in the same change as the infrastructure, on the operator machine, then pushed read-only to servers by one command; a server copy is never edited.
- [MUST] Every server has its own agent rules file (e.g. `/srv/CLAUDE.md`) read first: where you are, what never to touch, hard rules, recipes ("add a site"), tools, log locations, and the deploy path ("code deploys by push; never hand-edit tracked code here, an uncommitted change blocks deploys").
- [SHOULD] Each repo has a `CLAUDE.md` with the real commands (install, dev, build, lint, typecheck, test, migrate, deploy, health URL), architecture, layering rules and what CI runs; agents read it first, explicitly when working on the repo from another machine where it doesn't auto-load, and never guess commands.
- [SHOULD] Docs are rewritten from live checks periodically ("rewritten <date> from live checks"); when a rules file and the live system disagree, fix the file first and grep every synced copy for the stale fact.
- [SHOULD] Standards and checklists are edited for density: merge duplicates and cut lines that don't change behaviour.

## Skills and plugins

- [MUST] A skill exists when a workflow repeats and has non-obvious steps, never for a one-off; its `description` says when to use it in the words the user types ("is X up", "make a deck").
- [MUST] A skill contains where the thing lives, how to reach it, exact commands, what runs where, the health check, gotchas and a "Limits / not here" section, and points to the live source of truth on the box instead of copying it.
- [MUST] Skills that produce artifacts end with a mandatory verify step (render, screenshot, check overflow) before "done".
- [MUST] A skill describes how to do something and never grants permission to do it: permission rules still bound the agent, and headless agents get an explicit list of skills they may not run (ship, deploy, release, store submission).
- [SHOULD] Rules that must hold are enforced by the harness (permission deny rules, a pre-tool hook that blocks), not by instruction text; a session-start hook (and subagent-start where supported) re-injects softer house rules so they survive compaction and reach subagents.
- [SHOULD] Each app's conventions live in a path-scoped project skill committed to the repo (Claude Code: `.claude/skills/<name>/SKILL.md`) whose description names the area and trigger: numbered imperative rules with a wrong/right snippet each, the exact file where each kind of code lives, a closing checklist with the done commands; cross-cutting rules live once in the root file and skills link to them.
- [SHOULD] A setup skill takes a fresh clone to a running stack (ordered steps with a check command each, then a symptom → fix table), naming where credentials come from, never their values.
- [SHOULD] A correction given twice becomes a numbered rule in the relevant skill or a memory, not a third reminder in chat.
- [SHOULD] Skills ask one clarifying question up front (brand, audience, target) rather than guess, and installed plugins and marketplaces are reviewed periodically with unused ones removed: every plugin is code running in your sessions.

## Verification and reporting

- [MUST] Re-check after every change from the consumer's side (public URL, real client, what the recipient got), not the local process or the agent's own copy.
- [MUST] "Is it done / does it all work?" gets a status per piece (built + verified / built, untested / not built), and change state is reported precisely: written / PR open / CI green / merged / deployed / verified live.
- [MUST] An instruction applies to the whole set it names: state the scope back as counts before acting, and after a run with numeric targets (ratios, quotas) report achieved next to target, saying plainly when one was missed.
- [MUST] Destructive bulk operations run as a preview first with samples spot-checked, and broad-blast actions ordered in the heat of the moment show counts and side effects, get a yes first and take the reversible form (hold, not delete or permanent flag) with an exact pre-change snapshot.
- [MUST] Timeouts, 5xx and "unknown" are unknown, never "dead" or "safe to delete".
- [MUST] Failures are reported first and plainly (what failed, current state, what's needed), and existing production breakage found during other work is its own item ranked by user impact, not buried.
- [MUST] When a change has no visible effect, first ask which checkout, machine and device the human is looking at and push (an unpushed commit is invisible), then suspect the pipeline (stale build, cache, wrong path) before re-reading correct source, and prove it with an unmistakable marker.
- [MUST] After any outside submission (form, filing, DNS change, store upload), confirm it took effect in the system of record in two or more independent places before saying it worked.
- [SHOULD] When someone reports bad data, measure it first and answer with a table by day and source, then fix the cause.
- [SHOULD] Recovery paths (failover, restore) are drilled by script with "not yet drill-tested" in the doc until they are, and rules claimed as "enforced" are verified (the deny list really has the tool, the hook really fires).

## Boundaries and approvals

- [MUST] "Look at repo X" means read it; editing a second repo needs an explicit ask, an audit or fix request licenses fixing defects only (copy, design and structure changes are proposed separately), and unrequested changes, including your own "cleanup" of approved content, are never folded into an approved diff.
- [MUST] Before any connector call (calendar, chat, mail, docs, tracker), check which account and organization it acts as; content from another entity or the person's private life never goes there, not even as a draft or an item marked private.
- [MUST] When the harness or a permission check blocks an action, stop and say what was blocked and why, distinguishing harness blocks from the owner's rules ("the safety check blocked this; here's the permission rule if you want it").
- [MUST] Every time a secret appears in chat, say in that reply it's exposed and give rotation steps; never store or use it from chat, and never write a pasted secret into server files. Rotation is a script reading the value at a hidden prompt that refuses empty input and reads it back, followed by one real call through the consumer, and the old key is revoked only after that.
- [MUST] When the human asks the agent to drive, it runs every step its permissions allow and verifies it; only truly blocked steps go back, as the one action at the top of the reply in a single copy-paste command with the reason, while the agent keeps doing the unblocked work.
- [MUST] Commands handed to the human are saved at a stable path outside session scratch dirs, name their interpreter (`bash -c '…'`), and are checked to exist and run under the user's real login shell right before the request.
- [MUST] Intentional side effects are explained up front, before the human discovers them and assumes a bug.
- [MUST] When the human says they were wrong and asks for a revert, revert exactly to the prior state without re-arguing, and confirm with before and after counts.
- [MUST] Legal or contract text the agent drafts is truthful and precise (deliberately ambiguous "wiggle room" is refused with the reason it backfires), and "is this safe / more secure than X?" gets an honest comparison that also says where your setup is weaker.
- [MUST] Asked to destroy artifacts (recordings, test images, tokens), sweep every machine including temp dirs, simulators and clones, and report what can't be removed; anything left on a machine is reported with size and location.
- [SHOULD] The agent's permission settings describe the environment: deny rules for production hosts, secret paths and connectors it must never use, the repos it may push to, and any auto-approval mode told what counts as sensitive; telemetry is off with short transcript retention on machines that touch sensitive data.

## Working with the human

- [MUST] Outcome first: the first line says whether it worked and what changed; plain everyday words with any technical term in parentheses, short, no restating the task or recapping code merely read.
- [MUST] Say what was not done or not verified and what the human must do next, as a short explicit list.
- [MUST] Use absolute dates and cite sources for statements about past events.
- [SHOULD] Long work gets milestone updates unprompted (progress vs target, ETA, blockers), multi-hour or multi-day campaigns keep a durable state file, and before the human goes offline any stopped queue or job is restarted or left with a watcher, with anything needing them said first.
- [SHOULD] Options come as a short list with a recommendation, in everyday words; preferred artifacts (deck, PRD, one-pager) and "simpler" requests are recorded in memory.
- [SHOULD] Handoff notes for whoever operates the result are written unasked, including a one-page data dictionary: what every tag, flag, status and origin means, which actions can't be undone, ready-made queries.
- [SHOULD] Text the user will paste elsewhere is plain text (no markdown quote markers or headers), short, in the user's voice and answers only the question; instructions for a non-technical person give the exact path (app, tab, button) checked against the real UI.
- [SHOULD] If a fix forces the user to repeat a step (re-pair, re-login, re-consent), explain why once and change the design so it can't recur; files are delivered with absolute paths and versioned filenames.

## Product, design and artifacts

- [MUST] Before redesigning, rebuilding or replacing a screen, page, document, deck or feature, list everything it does or says today from the source; the replacement keeps each item or the user approves each removal.
- [MUST] Design work starts from the real code and data model: nothing on a screen is invented, and each element is marked real-data-backed or placeholder.
- [MUST] During UI work, server API changes are additive only, and live data paths are regression-checked before reporting done.
- [MUST] Creative iterations are saved as numbered versions, so "the earlier one was better" restores that exact version; before re-editing in response to a complaint, check what the artifact contains now (the user may be reacting to a proposal).
- [MUST] Before sending a technical document outside the team, re-read the latest source (PRD, spec), say when you last read it, and check every claim against the live spec and code, tracing input → persist → output → error even where the artifact won't show it.
- [SHOULD] "Match X" means reproduce X's structure exactly, not "inspired by"; after two rejected style directions, stop guessing and ask for concrete reference apps or screenshots, and for a multi-screen build show one or two direction screens for a pick before building the rest.
- [SHOULD] After any edit to a multi-part artifact (deck, spec, doc set), sweep the whole thing for cross-section consistency; incremental edits leave the stalest section wrong.
- [SHOULD] Analytics events are verified to fire (one real event per funnel step) before a dashboard relies on them, and on-screen video text stays up ~1 s per 3 words + 1 s, with walkthroughs covering each audience's basics before polish.

## Multi-agent work and sessions

- [MUST] Independent work runs in parallel agents, each with a self-contained brief: goal, scope, what's read-only, the house rules that apply (branch + PR, no attribution, no-go systems), the one output expected.
- [MUST] Each agent that writes code gets its own worktree or clone with its own scratch DB and ports; agents never switch or reset branches in a shared checkout, and review other people's PRs by comment only, never pushing to or merging them.
- [MUST] Assume other sessions share the repos: fetch before pushing, never force-push or amend shared history, revert with a new commit; by default an agent pushes, opens/updates a PR, reports CI and stops, and merges only what the owner names.
- [MUST] A subagent's "done" is a claim: re-run the key check (test, curl, count, commits on the remote) before relaying it, and when subagents fall short of targets report the real totals and reason instead of re-running blindly.
- [MUST] Before changing shared state, check what other live sessions are doing and tell those that depend on it (an address, a schema, a secret path); never flip global state they rely on (the active CLI account, global git config) to finish one task, scope it to the command instead; if shared automation is found off or changed by an unknown actor, ask the parallel sessions before reverting.
- [SHOULD] Long interactive sessions are rotated before the context gets huge, with a written handoff (goal, where things live, done, next, rules, gotchas): every reply re-reads the whole context, and long sessions have caused usage spikes.
- [SHOULD] Long-running or scheduled agents checkpoint, heartbeat, retry with a cap and always report back; recurring loops get a budget and kill switch (see [ai.md](ai.md)).
- [SHOULD] Every handoff or plan has a short "decided + why" list, so the next session doesn't reopen settled calls.

## Reproducibility

- [MUST] Every server or VM is built by a kept, idempotent script (`set -euo pipefail`, guards like `[ -f key ] || fetch key`) stored with the master docs, headed by what it builds and how to run it.
- [MUST] Fleet-wide operations (patch all, sync docs, rotate a key, failover drill) are scripts run from the operator machine, and runbooks for common failures (site down, host rebooted, power loss, moving an app) are numbered steps with exact commands.
- [MUST] Production hosts change zero-downtime: run the new version alongside, check it, flip, and keep the old copy until the new one is verified.
- [MUST] Deploys are push-to-deploy with health check and automatic rollback ([delivery.md](delivery.md)); nobody hand-edits or pulls code by hand in a live directory.

## The 10-minute starter kit

1. Write the global instructions file from the skeleton: three to five hard "nevers", the routing table, the contexts table, the least-code ladder.
2. Create the memory directory with an empty `MEMORY.md` and the habit: every correction → one feedback file with Why / How to apply.
3. Write one current-state infra doc and a `CLAUDE.md` on each server, plus a one-command sync.
4. Turn your most repeated workflow into a skill with a verify step.
5. Move your build steps into idempotent scripts next to the docs.
