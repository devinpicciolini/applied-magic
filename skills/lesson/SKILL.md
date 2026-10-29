---
name: lesson
description: Turn an incident, near-miss or newly learned rule into a generic, verifiable check and add it to the right applied-magic standards file. Use after something breaks, or when the human says "remember this", "add this to the list" or "never again".
---

# Add a lesson

1. Write the lesson as **one verifiable line**: `- [MUST|SHOULD|EXTREME] <check you can prove>`. One sentence where possible, two at most, with concrete nouns, commands or numbers ("X is bound to 127.0.0.1"), not advice ("be careful with X").
2. **Generalize it.** No project, company, product, person, host, IP, domain, account or secret names. It must help someone who's never heard of the incident catch it *before* it happens.
3. Choose the tier: MUST if a miss causes data loss, a breach, an outage or money/messaging harm; SHOULD for a strong default; EXTREME for money, identity, health or family data.
4. Pick the file: `build.md` (principles, stack, data model, pipelines, bulk changes), `app-security.md` (secrets, auth, web, encryption, money, messaging, integrations), `attacks.md` (attack → defense), `infrastructure.md` (hosts, network, DNS, backups, failover, monitoring, people), `delivery.md` (CI/CD, supply chain, mobile), `ai.md` (models and agents in products), `engineering.md` (least code, planning, review, testing, shipping), `operating.md` (running an agent day to day).
5. Search `../../standards/` for a check that already covers it. If one does, sharpen that line so it holds both nuances instead of adding a duplicate.
6. Add it to the right topic section, MUST before SHOULD before EXTREME. Put it in *The non-negotiables* only if it's a MUST whose miss causes a breach, data loss, an outage or money/messaging harm, and that section stays at 10 lines or fewer.
7. Update the check count in `README.md`, then commit with a short neutral message. No tool attribution in commits or PRs.
