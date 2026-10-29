---
name: magic
description: The front door to applied-magic. Works out what the human wants (start something new, check and fix what exists, review changes, check for attacks, ship, map a project, turn an incident into a lesson) and runs the right skill. Use when the human types /applied-magic:magic or /magic, says "magic", "help me build this right", "where do I start", or isn't sure which skill they need.
---

# Applied Magic

You route; the other skills do the work. Never make the human remember a command.

## 1. Read the room (no questions yet)
Look before asking: is the working directory a repo (`git rev-parse`), does it have code, a `CLAUDE.md`, a README, a diff (`git status`), a deploy config (`vercel.json`, `supabase/`, `netlify.toml`, `fly.toml`, Dockerfile)? Is `~/.claude/CLAUDE.md` thin or missing? What the human typed after the command is the strongest signal; if it names a goal, skip to step 3.

## 2. Ask one click-through question
Use AskUserQuestion with the 4 options that best fit what you saw, recommended one first (for example, an empty folder puts "Start something new" first; a dirty diff puts "Review my changes" first):

| Option | Runs |
|---|---|
| Start something new | `start`, after the intake below |
| Check this project | `map` if there's no `CLAUDE.md`, then `check` |
| Review my changes | `vet` |
| Ship it | `ship` |
| Check it for attacks | `attacks` |
| Map this project | `map` |
| Something broke / never again | `lesson` |

"Other" free text: match it to a row; if nothing fits, say what applied-magic covers in one line and stop.

## 3. Intake for anything new
One AskUserQuestion call, up to 4 questions, each with a recommended default first:
1. **What is it?** Web app · Mobile app · Both · API/backend only.
2. **Who uses it?** Just me · A team or client · The public · Paying customers.
3. **What data?** Nothing sensitive · Personal info · Money or payments · Health, identity or kids' data (EXTREME tier).
4. **Where does it run?** Own a box (recommended: one VM, Postgres on it, tunnel, push-deploy) · Client's cloud account · Must use a managed platform (say which).

Ask the idea itself in plain text only if the human hasn't said it. Sign-in is implied: none for "just me", Better Auth for the rest, enterprise SSO only if they say so.

## 4. Propose, then confirm
Show the stack as a short table: piece, choice, one-line trade-off (from *Default stack* in `../../standards/build.md`, adjusted for the answers; e.g. money → Stripe + webhooks + idempotency; mobile → the API is the contract, see `architecture.md`). Then AskUserQuestion: **Accept** · **Change something** · **Explain the trade-offs**. On Accept, write it as the decision record in the README and run the skill. On Change, ask only about that piece.

## Rules
Plain language, no jargon in the questions. One question round per step; don't interview. Everything in the routed skill's "Stop and ask the human first" still applies.
