# Infrastructure: hosts, network, backups, failover, monitoring and people

Tiers: **[MUST]** non-negotiable · **[SHOULD]** strong default · **[EXTREME]** for money, identity, health or family data. Part of [applied-magic](../README.md).

Everything below the app: what runs where, who can reach it, how it's backed up, how it fails over and how you find out
when it breaks. Check live (`ss`, `iptables -S`, `sshd -T`, `stat`, a real request), never from memory or docs. The
target platform and the hardened-VM recipe are in [`build.md`](build.md) (*Default stack*); database auth and secrets
are in [`app-security.md`](app-security.md); deploy pipelines and CI runners are in [`delivery.md`](delivery.md).

## The non-negotiables

- [MUST] One current inventory lists every machine, VM, container, domain, tunnel, database, scheduled job and public URL and matches reality; every running process has an owner and a reason, and anything nobody can explain is stopped, then removed.
- [MUST] Nothing listens on a public interface unless it must: `ss -ltnup` (UDP too) on every host and VM, re-run after every migration and every upgrade of a network-facing service (new versions add all-interface listeners).
- [MUST] Public traffic enters through one front door (tunnel or reverse proxy) to services bound to `127.0.0.1`, behind a default-deny inbound firewall for IPv4 **and** IPv6 that allows only the exact management path.
- [MUST] SSH is keys-only (`PasswordAuthentication no`, `KbdInteractiveAuthentication no`, `PermitRootLogin no`) with a minimal `AllowUsers`, checked in the effective config with `sshd -T`: the first value read wins, and cloud images ship `sshd_config.d` drop-ins that can switch passwords back on.
- [MUST] Security updates install automatically (`unattended-upgrades`, including the third-party repos you depend on), and pending updates and `/var/run/reboot-required` are checked and cleared at least weekly.
- [MUST] Every database is backed up automatically, encrypted (e.g. `age`) to keys held in ≥2 places off the server, copied off the machine and off-site, with every zone in the off-site set (check the script, not the docs).
- [MUST] Restore drills actually restore into a scratch DB and compare row counts, after every backup-script change and on a schedule; a backup never restored is a hope.
- [MUST] Two copies never serve at once: a start guard refuses to start while the other side is primary, a fence stops this side if the other became primary, and background jobs (emails, charges) run only on the confirmed primary.
- [MUST] Before deleting a hosting project, app, tunnel, bucket or DNS target, check the platform's domain mapping for what it actually serves (look-alike twins have caused outages), and remove DNS records whose target is gone.
- [MUST] MFA on every provider (cloud, DNS, registrar, code host, email, payments, VPN admin), and access is removed the day a relationship ends.

## Inventory and ownership

- [MUST] Each business, entity or trust zone runs on its own host or VM, never "one more app" on a shared box.
- [MUST] Every project, prototype and site has its source in a git remote under the org that owns it, never only on one machine, a deployed non-git copy or a hosting platform's deployments.
- [SHOULD] The inventory is current state only (lines edited in place, no dated history or `.bak` copies), lists each scheduled job (cron, systemd timer, launchd, process-manager cron, CI schedule) with what it touches, keeps a port-owner list per host, and records each shared host's blast radius.
- [SHOULD] Strays are hunted down (old checkouts, data folders, DB dumps, leftover containers, half-migrated dirs, a second process manager as root, orphaned DB servers, forgotten tunnel connectors), and stale copies on non-production hosts are labelled or removed; compare row counts before treating any copy as a mirror.
- [EXTREME] A machine holds only what it serves; temporary clones, build dirs and simulators are reported with size and path when created and deleted when the task ends.

## Network exposure

- [MUST] VM and container port forwarding to the host is off unless needed, for TCP **and UDP** (some hypervisors forward UDP on `0.0.0.0` by default), and containers publish only to `127.0.0.1` (daemon `"ip": "127.0.0.1"`): Docker-published ports bypass `ufw`.
- [MUST] `127.0.0.1` isn't private on a host running a userspace mesh VPN or proxy that can relay peers to loopback, and a root VPN daemon's relayed traffic bypasses kernel firewall rules: the mesh ACL enforces isolation and is tested, and containers behind a local proxy publish no port at all.
- [MUST] No default or catch-all server answers on 80/443: unknown `Host` headers are closed (nginx `return 444` on 80, `ssl_reject_handshake on` on 443).
- [SHOULD] Admin surfaces (dashboards, DB consoles, signing admin, metrics) are reachable only over the private network or behind an identity-aware proxy, never public behind just a password.
- [SHOULD] A self-hosted third-party app exposed publicly (e-sign, forms, file drop) passes only an allow-list of its public paths at the proxy and 404s the rest (setup, sign-in, admin, API), re-verified with a full outside run after every upgrade.
- [SHOULD] Mesh access is least-privilege per device/tag, policy has automated accept + deny tests that refuse a bad edit on save, and each trust zone is its own tag with no route to the others, proven by trying to connect.
- [SHOULD] An isolated zone's mesh policy is one-way (owner devices reach it on SSH only, it reaches nothing, exit-node LAN access off), tests cover both directions, and nodes hidden from the owner's device list are documented with how to reach them.
- [SHOULD] A runtime preload forces app servers to `listen()` on `127.0.0.1` even if the app asks for `0.0.0.0`, and local daemons use a `0600` Unix socket instead of a TCP port where possible.
- [SHOULD] IP allow-lists and "localhost needs no auth" aren't used behind NAT, masquerade or a proxy that collapses every client to one address.
- [SHOULD] VMs or distros that share one network namespace (e.g. WSL2) are treated as one host for ports, firewall rules and stop hooks.
- [SHOULD] Wildcard DNS plus wildcard tunnel ingress is documented as "every new vhost is public instantly", and each new vhost is reviewed for exposure and `noindex`.
- [SHOULD] On Wi-Fi or NAT'd VMs, tunnels pin transport to HTTP/2 (cloudflared `protocol: http2`): dropped UDP breaks QUIC tunnels and shows as 502s.
- [SHOULD] Mesh node key expiry is a per-node decision, with servers that have it off listed; firewall rules are deleted by content, never by rule number (numbers shift).
- [EXTREME] Hosts with the most sensitive data accept no inbound private-network traffic except one app port from named devices (everything else denied and tested), and DNS inside isolated VMs doesn't reveal which internal services exist.

## Egress

- [MUST] App users can't reach the internet directly: a per-UID kill switch (owner match) allows loopback only for IPv4 **and** IPv6, it's a boot-time unit verified after reboot (`iptables -S` and `ip6tables -S`), and each exception is the narrowest possible (one IP, one or two ports) and documented beside the rule.
- [SHOULD] Proxy env (`HTTP(S)_PROXY`, `ALL_PROXY`, `NO_PROXY`) is set in the process manager config **and** the login profile so apps, builds, git and package managers all use it; `NO_PROXY` lists exactly the internal addresses needed; and locked apps resolve DNS through the proxy (`socks5h://` or DNS-over-HTTPS via the proxy), proven by a real fetch and a build after a restart.
- [SHOULD] Containers get their own egress rules (e.g. `DOCKER-USER` drop except DNS and the one SMTP port).
- [EXTREME] Egress exits through a privacy VPN endpoint per trust zone (zones never share an exit) and fails closed: stop the VPN and verify nothing leaves, and check the exit with an IP-echo request through the real path (VPN exits often don't answer ping).

## DNS, domains and email

- [MUST] Every sending domain has SPF, DKIM and DMARC at `quarantine`/`reject`, tested with a mail tester; non-sending domains publish `v=spf1 -all`, a null MX (`0 .`) and DMARC `p=reject`.
- [MUST] Registrar and DNS accounts belong to the right entity, and DNS-provider wizards ("fix missing records", "add recommended records", email routing) are never accepted on an apex zone without reading the diff: they can repoint MX and move company mail.
- [MUST] Tunnel and DNS CLI changes confirm which account and zone the credential covers, then confirm the record landed there (CLIs silently create `newhost.otherzone`).
- [SHOULD] Transactional mail sends from a dedicated subdomain; DKIM and sending setups tied to a lost vendor account are removed and mail moves to a fresh subdomain.
- [SHOULD] CAA records restrict issuers; DNS changes use a scoped token, are logged and are verified after propagation.
- [SHOULD] New hostnames aren't queried before their record exists (resolvers negative-cache NXDOMAIN), and go-live is verified via a public resolver plus a pinned resolve.
- [SHOULD] Certificate expiry is monitored by you (CAs no longer email warnings and lifetimes keep shrinking), and if the CDN redirects HTTP to HTTPS the origin serves `/.well-known/acme-challenge/` on 443 too (or uses DNS-01), or renewal stops silently.
- [SHOULD] Registry status (`serverHold`, expiry) is monitored for every domain in live links, demos with third-party brands or look-alike pages never run on a production domain (one abuse report can hold every subdomain), new domains are submitted to the major web-filter vendors for categorization before launch, and every published contact mailbox exists and is monitored.
- [EXTREME] Registrar and transfer locks on every important domain, with auto-renew on a working card.

## Hosts and OS

- [MUST] No host folders are shared into VMs holding sensitive data (a mount lets the VM read the host's home).
- [MUST] Each VM has its own app user (fixed UID, no sudo) and its own DB roles and passwords, never reused across zones.
- [MUST] Secret files are `600` and secret dirs `700`, owned by the app user, checked with `stat`.
- [SHOULD] Every service (apps, proxy, DB, tunnel) runs as its own user with `Restart=on-failure` and sandboxing (`NoNewPrivileges`, `ProtectSystem=strict`, `ProtectHome`, `PrivateTmp`, `PrivateDevices`, empty `CapabilityBoundingSet`), scored with `systemd-analyze security <unit>`.
- [SHOULD] App groups, builds and co-located VMs have `MemoryMax` and `CPUWeight` so one can't starve the rest; disk (alert at 80%), memory, swap and load are watched, and swap pressure on a small host is chased down to what reserved the memory.
- [SHOULD] A reboot window exists (`Unattended-Upgrade::Automatic-Reboot` at a set time, or a weekly manual run), and desktop-OS servers never sleep on AC and can't force-restart for updates mid-day.
- [SHOULD] Time sync works (`timedatectl` shows synchronized; TLS, TOTP, logs and replication depend on it), servers and DB sessions run UTC, and any cron schedule in local time says so.
- [SHOULD] The journal is persistent and capped (`Storage=persistent`, `SystemMaxUse`), so the logs from before a crash or reboot survive.
- [SHOULD] Every host has a second, tested management path (e.g. mesh SSH when the hypervisor's shell relay wedges), and a wedged relay isn't mistaken for a dead guest.
- [SHOULD] No SSH agent forwarding (`-A`) to hosts others can be root on (use `ProxyJump`), and a changed host key is explained before connecting, never bypassed with `StrictHostKeyChecking no`.
- [SHOULD] Desktop indexers and sync daemons are off on servers, and helper scripts and venvs never live in shared `/tmp`, where they vanish at reboot and stray files can shadow stdlib modules.
- [SHOULD] Destroying a VM or distro is followed by a remnant sweep on the host (firewall rules, port proxies, scheduled tasks, disk images, logs, key files), and vulnerability news triggers a sweep of every server, VM and workstation, including dev tools bundling their own runtime, recording versions before and after.
- [EXTREME] Full-disk encryption on every host with recovery keys offline in two places, and unused packages, compilers and container runtimes removed from production VMs.

## Processes and scheduled jobs

- [MUST] Nobody builds, runs a dev server or `git pull`s in the directory a production process serves from or a push-to-deploy checkout; builds run one at a time in a fenced cgroup/slice at low priority, only while load is under a threshold.
- [MUST] Process managers run under the init system (systemd/launchd), not started by hand, so "stop the service" stops the apps; check for daemons whose start time doesn't match the unit.
- [MUST] Every managed app has a restart cap with backoff and a crash-loop alert; restart counts come from the full describe view (lists truncate) and are judged by uptime and whether they're climbing (lifetime counters never reset), and after any start you confirm it is actually online (unknown flags abort silently).
- [MUST] A deploy or reload confirms every process of the app runs the new build (uptime or build ID per process); some managers reload only the first of several names.
- [MUST] Apps load their env file at runtime, or the process is re-created from config, so a manager's first-start env snapshot can't keep a stale secret; restarts never import the operator's shell env (`--update-env`), which leaks one app's secrets into another.
- [MUST] Auto-restart logic restarts a named list of services one at a time, never "restart all" (which resurrects apps stopped on purpose), and health checks probe the service's own readiness endpoint with two consecutive failures before restarting.
- [MUST] Processes are killed by PID or listening port, never a broad `pkill -f` on a shared host (it can match other apps or the SSH session running it).
- [MUST] Cron, launchd and hook scripts are tested under their real minimal env (`env -i`, the cron PATH); empty or unreadable tool output is a failure, never "0 errors".
- [MUST] A root-launched proxy on :80/:443 restarts only through the root service manager, and `nginx -t` (or equivalent) passes before every reload.
- [MUST] Anyone who switches off shared automation (auto-deploy, timer, watchdog, scheduler) writes the reason, owner and expiry at the switch, and an alert fires if it stays off past the expiry; stopping an auto-start job also disables it so it doesn't return at reboot.
- [SHOULD] Custom settings on packaged services live in an override (systemd drop-in, your own launchd plist), never in the packaged unit, which upgrades rewrite.
- [SHOULD] Long jobs run as supervised units with checkpoints or resume, never `nohup … &`, GUI-driving jobs run in the user's GUI session, and orphaned simulators, headless browsers, shell loops and helpers are swept regularly.
- [SHOULD] Remote automation uses a login shell or absolute tool paths, and multi-line or JSON payloads travel as copied script files, not nested quoting.
- [SHOULD] Every service returns after an unclean power loss with no manual step (stale pidfiles cleared on start), tested by pulling the power.

## Containers

- [MUST] No privileged containers, no Docker socket mounted into apps, non-root where the image allows, and images pinned by digest with base images updated regularly.
- [SHOULD] No published ports when the proxy can reach the container's internal IP directly, and daemon log rotation is set (`"log-opts": {"max-size": "10m"}`).
- [SHOULD] Unused containers, volumes and images are cleaned (including anonymous volumes from CI service DBs), and container runtimes are removed from hosts that no longer run containers.

## Backups

- [MUST] Backup jobs discover every database (native and in every container) instead of pattern-matching names, dump names include the engine or container, retention prunes by age per database (never by total file count), and freshness is monitored so a stale backup alerts within hours.
- [MUST] Save-then-delete scripts hard-stop unless the copy is verified: `set -euo pipefail`, an explicit `if ! verify; then exit 1; fi` (a failing `a | grep -q x && echo ok` doesn't trip `set -e`), run as a user that can read the copy, with row counts matching the source.
- [SHOULD] Each database has a written RPO and RTO that the design meets: hourly dumps lose up to an hour, so anything tighter uses WAL archiving with a freshness alert (e.g. >15 min stale).
- [SHOULD] History keeps hourly copies for the last few hours **and** dailies for at least a week, as hard-link snapshots so unchanged files cost nothing.
- [SHOULD] The relay that moves backups off a host is write-only (`rrsync -wo`) and carries ciphertext only; off-site pruning works from your own manifest, since some cloud drives can't be listed by background jobs and "prune by listing" silently does nothing.
- [SHOULD] Env files are backed up encrypted alongside the DB (or the restore can't run), and whether file/upload data goes off-site is a recorded decision.
- [SHOULD] Re-keying backups re-encrypts the history and updates every recipients file, and introducing encryption re-encrypts or deletes every existing plaintext dump on every host.
- [EXTREME] Backups survive the loss of the primary cloud account (a second independent off-site), and signed legal documents with their audit trails are backed up nightly with longer retention.

## Replication and failover

- [MUST] Takeover needs several independent signals held for a sustained window: public endpoints down **and** the primary unreachable on the private network **and** the standby's own connectivity fine.
- [MUST] Physical replication slots are capped (`max_slot_wal_keep_size`) so a dead replica can't fill the primary's disk, and a retired replica's slot is dropped.
- [MUST] Replica connection strings set `connect_timeout` and TCP keepalives, or a replica booted before the network hangs and silently stops replicating.
- [MUST] Replication is monitored by **LSN position**, never "time since last transaction" (an idle primary looks like lag).
- [MUST] Failover and failback are drilled for real at least quarterly at a quiet hour by stopping the primary, timed and written down; the drill refuses to start mid-failover and confirms every tool the copy-back path needs exists on both sides.
- [SHOULD] A start guard waits for the private network at boot (a slow network must not read as "other side absent"), and failover paths never depend on sessions that expire (e.g. mesh SSH re-auth every ~12 h): use a dedicated key-only sshd on the private interface.
- [SHOULD] A rebooted standby resets to "stale" and must be re-seeded, failover is armed only once the target is fully seeded and verified, and failback makes the old primary a live replica that switches back only after replaying every WAL record; new data copies are built beside the old and swapped only when complete.
- [SHOULD] Any site marked must-stay-up has automatic failover to a second host plus automatic failback.
- [SHOULD] Health-check and copy tools are tested on relayed mesh links: some ping tools exit non-zero when the path isn't direct, and relayed throughput can be under 1 MB/s, so bulk seeds go over a direct path.

## Monitoring and alerting

- [MUST] Every public site is checked against an expected-status baseline, updated whenever sites are added or retired, and alerts fire on change (down/up) with repeats de-duplicated.
- [MUST] Health checks confirm the expected process, version and database answer: a version endpoint that also names the DB it's connected to, plus the process holding the port (`ss -ltnp`), since after a move another process or a fallback DB can answer.
- [MUST] A scheduled job's freshness threshold exceeds its cadence plus max runtime (≥30 h for a daily job), so normal runs never look stale and real misses always do.
- [MUST] Monitoring never reaches production through a machine that might be gone (a standby or jump host).
- [SHOULD] Every check is run once by hand after it's written (remote-command quoting bugs cause silent false alarms or passes), and checkers record the tool's real exit code before any pipe: exit 127, parse failures and empty runs are errors.
- [SHOULD] An hourly digest covers memory, disk, swap, load, power, backups, replication and deploys, deploy failures and rollbacks alert, and a dead-man's switch on a second machine alerts if the monitor disappears.
- [SHOULD] You know which ISP each machine uses, and a site failing from one network is re-checked from a second ISP before alerting (carriers SNI-block some hostnames).
- [SHOULD] If a watchdog auto-disables jobs that run faster than their cadence, check it before changing a cadence, and change DB-held cadence values in the DB.
- [EXTREME] Alerts include the exact command to diagnose or fix.

## Migrations and decommissioning

- [MUST] Cutover scripts begin with a read-only preflight that exits before any mutation, running as the same user, shell and environment as the real run, and staging copies built for cutover tests have outbound email, SMS and push verified off before the first test.
- [MUST] Before a migration or retirement, capture everything only the old host holds: uncommitted or unpushed code, stashes and branches, secrets kept only in the platform.
- [MUST] Copy every path the config points at (`grep -E 'DIR|PATH|UPLOAD|CACHE|KEY' .env`), compare file counts on both sides, and fix absolute paths into old host mounts (keys, certs, caches), which fail silently with ENOENT.
- [MUST] Before stopping an old instance, read its stop hooks (ExecStop, shutdown scripts) for effects on shared routes, firewall rules or DNS.
- [MUST] After the move, the old host loses its code, data, env, DB roles, DNS, mesh grants, monitoring baselines, watchdogs and restart scripts for the moved apps (or they resurrect duplicates); destroy it only after one clean business cycle and a passed restore drill on the new host.
- [SHOULD] At cutover, env files are re-synced and their key names diffed (a skeleton `.env` silently lacks keys), and duplicate keys in one env file fail a lint.
- [SHOULD] The checklist covers OS binaries the app shells out to, runtime and package-manager versions, DB extensions, memory limits for the new host, stale `git worktree` metadata, the process manager's stored cwd and launch agents.
- [SHOULD] Validation replays real request paths from recent access logs against old and new hosts and diffs status and body, explaining every diff; token-gated share links that changed are reissued, and the app is tested end to end as a user (sign in, core action, push/email).
- [SHOULD] Moving sensitive data proves every record round-trips (before/after fingerprints) before the source is cleared; repo moves verify byte-identical trees and every ref, export PRs and issues first and re-register CI runners.
- [SHOULD] Clients that hard-code the old address get a temporary relay with a dated removal plan, and exposure and backup checks are re-run on both hosts.

## Incident readiness

- [MUST] Runbooks exist for host down, site down, DB restore, key leak, failover and failback, and any single app can be restored from backup without its original host.
- [SHOULD] Provider contacts and fast support paths (DNS, registrar, payments, hosting) are written down, and every incident ends as one new rule line in the docs, not a narrative.
- [SHOULD] The admin workstation isn't a single point of failure: the fleet doc has a read-only copy on the servers, and every private key has an offline escrow copy you have restored from.
- [EXTREME] A twice-yearly tabletop: "this host is gone forever, go".

## People, accounts and workstations

- [MUST] Collaborators get scoped access (one machine, one repo, specific sudo commands), never root by default, and their user is fenced from the rest of the private network (owner-match firewall rule); a sudo rule that lets them become the app user equals reading every secret and is recorded as accepted risk.
- [MUST] Offboarding removes their SSH keys, mesh shares and device grants, sudoers entries, deploy keys, OAuth grants and provider seats, each verified by listing it afterwards.
- [MUST] SSH private keys on admin workstations have a passphrase or are hardware-backed; an unlocked laptop plus a bare key is root on every server.
- [SHOULD] Shared/team accounts are documented with who actually uses them, and recovery contacts and backup codes are current and offline.
- [SHOULD] Agents are read-only on registry, legal and filing accounts, and ownership changes are verified on the registry itself.
- [SHOULD] Workstations run the firewall in stealth mode, remote management and AirPlay-style receivers off, immediate screen lock, no setup-default passwords on remote desktops or consoles, and folders of sensitive personal and legal documents at `700` with files `600`.
- [SHOULD] A workstation VPN claims "always anonymous" only when traffic fails closed, and an exit node hides only the IP, never logged-in identities.
- [EXTREME] Admin accounts are separate from daily-use accounts.

## Physical and home lab

- [MUST] Home servers are on a UPS or adequate charger with lid-close sleep off and auto-restart after power loss on; laptop servers have charger wattage and on-battery state checked before heavy jobs and alert when discharging (an undersized charger causes crashes under load).
- [MUST] Shared machines (someone else's homelab) are untrusted: nothing sensitive lives there, and their config isn't changed without asking.
- [SHOULD] The home router has a changed admin password, remote admin and UPnP off and current firmware; cameras, TVs and IoT are private-network only with no cloud relay you didn't choose.
- [EXTREME] IoT sits on its own segment with no route to servers.
