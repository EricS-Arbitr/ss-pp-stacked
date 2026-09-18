# CLAUDE.md — ss-pp-stacked onboarding guide

Guidance for Claude Code (claude.ai/code) and for humans picking up this
range overlay. Read this first.

## What this project is

This directory (`ss-pp-stacked/`) is a **range-specific Ansible overlay** for
the **PowerPlant** cyber-range scenario (`voltgrid.com`) on the **SimSpace NG**
platform. It is NOT a standalone playbook — it layers on the customer's shared
platform repo at `../range-development-ansible/`.

**What makes this range different from its siblings: it runs BOTH SIEMs.** A
distributed Splunk cluster and a Security Onion 2.4 grid, on the same 78 hosts,
so the same telemetry can be worked in either tool. `ss-pp-so` is the
Security-Onion-only range; `ss-pp-ab` is the Splunk-only one.

`range-development-ansible` ships base roles. `ss-pp-stacked` ships the range's
inventory, host_vars, group_vars, custom roles, the baseline playbook
(`arbitr_pp_playbook.yaml`) and the Security Onion phases (`playbooks/`).
`build_tarball.sh` bundles selected base roles plus the overlays into
`ab_pp.tgz`, which extracts to `/etc/ansible` on the controller where
`deploy.sh` runs `site.yml`.

## Repo layout

```
ss-pp-stacked/
├── CLAUDE.md                    ← you are here
├── README.md                    ← what this repo is, build/deploy in brief
├── site.yml                     ← ★ the entry point
├── arbitr_pp_playbook.yaml      ← the range baseline AND the whole Splunk build
├── playbooks/                   ← 05-time 10-mirror 20-vyos 30-prereqs
│                                  40-manager 50-nodes 60-verify 75-endpoint
├── hosts                        ← inventory
├── group_vars/                  ← all/ (main.yml, security_onion.yml) + per-group
├── host_vars/                   ← one yaml per managed host
├── roles/                       ← custom roles overriding or supplementing base roles
├── blueprints/                  ← the SimSpace blueprint this range is built from
├── build_tarball.sh             ← auto-discovers roles, validates, bundles ab_pp.tgz
├── deploy.sh                    ← runs site.yml (see the retry model below)
├── verify_vars.py               ← Jinja-var presence checker
├── verify_shell_args.py         ← ★ refuses plays Ansible's split_args() cannot parse
├── verify_so_inventory.py       ← refuses an SO host in [so_all] but not [linux]
├── verify_deployment.sh         ← post-deploy checks
├── collections/                 ← vendored pfsensible.core (never fetched at deploy time)
├── rules/                       ← bundled ETOPEN ruleset
└── ab_pp.tgz                    ← built artifact; rebuild with build_tarball.sh
```

Custom roles beyond the base repo, grouped by what they build:

| Role | Purpose |
|---|---|
| **Splunk** | |
| `splunk_cluster_manager` | pp-splunk-cm — indexer cluster manager and licence master |
| `splunk_indexer` | pp-splunk-idx01/02 — cluster peers, label `pp-idxc` |
| `splunk_search_head` | pp-splunk — search head against the cluster |
| `splunk_license_peer` | points the cluster manager, both peers and the search head at ONE license manager — four instances each using `license master self` is four violations, not one deployment |
| `splunk-forwarder` | universal forwarder on ~50 hosts |
| `splunk-es` | Enterprise Security content |
| `splunk-verify-user` | post-deploy proof that a user can actually log in and search |
| **Security Onion** | |
| `so_apt_mirror` | nginx on the controller serving the SO source, airgap detection content, ETOPEN rules, and the container-registry artifacts it builds with skopeo |
| `so_base` | prerequisites on every grid node |
| `so_manager` | answer file, registry seeding, `so-setup`, outcome verification |
| `so_search` | search node install + grid join |
| `so_sensor` | sensor install + grid join, GRE decap, Zeek/Suricata |
| `elastic_agent` | Elastic Agent install + Fleet enrolment (only via 75-endpoint, see below) |
| `vyos_mirror` | GRE tunnels + `tc` mirror rules feeding the sensors |
| **Range** | |
| `pfsense_firewall` | drives pfSense 2.8.1 via pfsensible.core plus `php -r` shims |
| `init` | waits for WinRM, then repairs and PINS the default-gateway ARP entry |
| `syslog_server` | pp-syslog as central rsyslog collector (UDP+TCP 514, per-host files) |
| `additional_dc` | promotes pp-dc02/03 into the voltgrid.com forest |
| `wordpress-pv`, `billing_site`, `voltgrid_site` | the web tier on pp-www |
| `is_inet_fix` | simulated-internet host repair |
| `strip_apipa` | removes 169.254.x.x left by a DHCP→static handoff |
| `network_discovery`, `disable_defender`, `global_dns` | Windows posture |

## Deploy structure

```yaml
- import_playbook: playbooks/10-mirror.yml    # FIRST — see below
- import_playbook: arbitr_pp_playbook.yaml    # baseline + the whole Splunk cluster
- import_playbook: playbooks/05-time.yml      # Windows clock, DC-first
- import_playbook: playbooks/20-vyos.yml      # GRE tunnels + tc mirror rules
- import_playbook: playbooks/30-prereqs.yml   # so_base on all SO nodes
- import_playbook: playbooks/40-manager.yml   # MUST complete before 50
- import_playbook: playbooks/50-nodes.yml     # search + sensor grid-join
- import_playbook: playbooks/60-verify.yml
```

**10-mirror runs ahead of the range baseline.** It fails fast — the mirror is a
hard prerequisite for phases 30/40/50, and finding it broken after a
multi-hour baseline wastes the whole run. And it overlaps: it kicks the 7.2 GB
container-image build off in the BACKGROUND and returns, so the baseline runs
while it downloads. `40-manager` blocks on the artifacts.

**Security Onion runs AFTER the range playbook, not interleaved.** The SO nodes
need routing, DNS and the mirror path live before they can do anything.

**`75-endpoint.yml` is deliberately NOT imported.** These endpoints already
carry a Splunk universal forwarder from the baseline; putting the Elastic Agent
on them as well is a separate decision, so it is explicit:

```bash
ansible-playbook -i hosts playbooks/75-endpoint.yml
```

That is the one structural difference from `ss-pp-so`, where endpoint enrolment
is part of every deploy. Anything you port between the two repos has to account
for it.

### The retry model

`deploy.sh` makes up to three attempts, and attempt 2 is **not** a success
condition:

1. full sweep
2. retry-file scope — a REPAIR PASS over the hosts that failed. It never breaks
   out of the loop however well it goes.
3. full sweep — this is what actually confirms the range

The retry file lists hosts that FAILED. Repairing them does not run the plays
whose targets were dropped when they failed, so a clean repair pass is not a
deployed range.

`FORKS` is DERIVED, not chosen: the largest single play target plus a small
margin, so the widest play runs in one batch. Recount when hosts are added.
`BOOT_DELAY` (default 180s, `BOOT_DELAY=0` to skip) lets a freshly provisioned
range finish booting — it covers hosts that do not exist yet, which
`wait_for_connection` cannot.

### Build-time gates

`build_tarball.sh` refuses to write an archive that would not deploy:

- `verify_shell_args.py` — an apostrophe in a PowerShell or shell comment
  inside a free-form module argument makes the PLAY FAIL TO LOAD. Ansible runs
  `split_args()` over those arguments and counts quotes; it does not know the
  script has comments. The file is still valid YAML, which is why this check is
  separate from `verify_vars.py`.
- `verify_so_inventory.py` — an SO host in `[so_all]` but not `[linux]` has
  roles to run and no way to log in. That cost three deploy attempts once.
- `verify_vars.py` — Jinja references with no definition. Advisory here, not a
  hard gate.

**Expected warning count against the STAGE is 8**, and most are false
positives: this repo carries the ORIGINAL 161-line `verify_vars.py`, while
`ss-pp-so` has a 330-line version that parses task-level `vars:` blocks, FQCN
`set_fact`, and role scope.

| Var | Real? |
|---|---|
| `billing_secret_key`, `pfsense_stale_gateways` | expected — intentional `\| default(...)` |
| `nat` | expected — base role, only present in the stage |
| `agent_targets`, `missing`, `unlisted` | false positives — task-level `vars:` in 75-endpoint |
| `range_utc`, `stored` | false positives — `set_fact` the old parser misses |

Porting `verify_vars.py` from `ss-pp-so` would cut this to the three real ones
and add role-scope checking, which catches a class that has failed at run time
three times across these repos. Not done yet.

Run it against the STAGE, not the repo root — the stage includes base roles
copied from `../range-development-ansible/`, so `nat` only appears there.

## Secrets

**This repo has no ansible-vault.** There is no `group_vars/all/vault.yml` and
no `ansible.cfg`; credentials sit in plaintext in `group_vars/`. `deploy.sh`
still carries vault-handling logic inherited from its sibling repos, and a
`.vault_pass` file may exist on disk with nothing to open.

**The repo is public.** Do not add credentials, keys or a vault password to it.
`.vault_pass`, `*.pem`, `*.p12`, `*.pfx`, `id_rsa*` and `.ssh/` are gitignored.

If a task embeds a credential in a shell command, give it `no_log: true`.
Ansible echoes `cmd` verbatim on failure and `deploy.sh` tees that to
`/var/log/playbook_run.log`, so without it a failure writes the credential to
disk in clear. Where the task's output is the diagnostic, keep
`failed_when: false` and surface stdout from a following task, never the
command. Credentials passed as MODULE PARAMETERS need nothing — the
`microsoft.ad` and `ansible.windows` collections mark those `no_log` in their
own argspec.

## Network topology

Same PowerPlant fabric as the sibling ranges:

- **eBGP only at the edge** — exactly one BGP session: `pp-isp-router`
  (AS 65002) ↔ `pp-external-firewall` (AS 65001).
- **OSPF between the two upstream firewalls** — `pp-external-firewall` ↔
  `pp-internal-firewall`, with `redistribute_ospf: true` so corp routes reach
  the ISP.
- **Static everywhere else** — `pp-corp-router`, `pp-internal-router` and
  `site-edge-router` carry `remove_vyos_bgp: true`. `pp-ot-firewall` has
  neither BGP nor OSPF (ESP boundary, default-deny). `pp-ot-router` is in
  `[vyos_routes_only]` because its image only accepts raw static routes.
- **Management plane** — every host has a SimSpace-assigned address in
  `10.255.240.0/20` on its first NIC. It is out-of-band and scenario traffic
  must not traverse it; Windows DDNS on that adapter is disabled and mgmt-IP A
  records are scrubbed from AD DNS.

## Inventory groups

- **Platform**: `windows`, `linux` (+ `ubuntu22`), `vyos`, `vyos_routes_only`,
  `pfsense`.
- **Splunk**: `splunk_cluster_manager`, `splunk_indexer`, `splunk_search_head`,
  `splunk` (the search head, by its older name), `splunk-forwarder`,
  `splunk_forwarder_exempt`.
- **Security Onion**: `so_manager`, `so_search`, `so_sensor`, `so_all:children`,
  `so_mirror_routers`, `ansible_controller` (the mirror host — deliberately NOT
  in `[linux]`).
- **Role**: `pdc`, `additional_dc`, `domain_controllers`, `file`, `proxy`,
  `syslog`, `members`, `corporate_servers`, `dmz`, `infrastructure`.
- **Posture**: `ae` (attack emulation), `aue` (attack-user-experience), `hunt`.
- **Special**: `unmanaged` — not targeted by any play; OT PLCs/HMIs that arrive
  pre-configured from their image.

**Group names are load-bearing.** The `so_*` roles address them by name via
`groups['so_manager']`, `groups['so_all']`, `groups['so_sensor']`, and the
Splunk roles do the same. Renaming one here without renaming it in the roles
breaks the build silently.

## Conventions

1. **Customer-role workarounds are plays, not role forks.** If a base role has
   a bug, add a compensating play after it. Fork into `roles/<name>/` only when
   the workaround cannot be expressed as extra tasks.

2. **`php -r` is the universal pfSense escape hatch.** pfsensible.core 0.7.x
   does not expose `<defaultgw4>`, `<nat><outbound><mode>`,
   `<installedpackages><frr>` or `<syslog>`. Those go through
   `ansible.builtin.shell: php -r '...'` with `require_once("/etc/inc/config.inc")`
   and `config_set_path()` + `write_config()`.

3. **Static-route comments cite the destination.** Every `next_hop` gets an
   inline comment naming the device at the other end. A /30 has two usable
   addresses and the next-hop is the OTHER one; this convention exists because
   that mistake has cost real time.

4. **Per-host opt-in cleanup lists.** `extra_static_routes_remove` on VyOS
   host_vars, `pfsense_stale_gateways` on pfSense host_vars.

5. **SimSpace image quirks are platform issues.** Work around them in overlay
   plays; do not try to fix the image from Ansible.

## Common pitfalls

1. **VyOS images bake self-loop default routes.** FRR refuses to install a
   default whose next-hop resolves to a local interface, so `show ip route` has
   no `S>* 0.0.0.0/0`. Fix with `extra_static_routes_remove`.

2. **`next_hop` set to the host's own /30 address instead of the peer's.** The
   comments beside each value are not decoration.

3. **Windows DDNS registers both adapters into AD DNS**, so `ping pp-dc01` gets
   a mgmt IP half the time. The overlay play strips it.

4. **pfSense needs `ansible_user: admin`.** pfsensible.core writes via temp
   file + rename; `/tmp` and `/cf` are separate mounts, so the rename falls back
   to copy, and only `admin` can copy onto the root-owned config.

5. **pfSense FRR runtime dir does not auto-create.** `/var/run/frr/` is missing
   at boot and watchfrr fails with `Can't create pid lock file`. The role
   pre-creates it.

6. **An SO host missing from `[linux]`** has roles to run and no way to log in —
   every SO play reports `ok=0 unreachable=1`, indistinguishable from a VM the
   range never built. `verify_so_inventory.py` now refuses the build.

7. **`so-setup` exits 0 having failed.** Assert the OUTCOME — containers
   running, registry repository count — never the exit code.

## Verification

```bash
./verify_deployment.sh          # on the controller, after a deploy
```

Splunk, specifically:

```bash
# cluster health from the manager
ssh pp-splunk-cm "sudo /opt/splunk/bin/splunk show cluster-status -auth <user>:<pw>"
# a search head that cannot search is not a working SIEM
ansible splunk_search_head -m shell -a 'sudo /opt/splunk/bin/splunk status'
```

Security Onion:

```bash
ansible so_manager -b -m shell -a 'so-status'
ansible so_manager -b -m shell -a 'so-elasticsearch-query _cluster/health?pretty'
```

Routing:

```
show ip bgp summary       # VyOS: every neighbor Established
show ip route 0.0.0.0/0   # the default should be in the FIB
vtysh -c "show ip route"  # pfSense
```

"The config has the line" is not "the kernel installed the route."

## Working on this repo

1. **Read `../ss-pp-so/UPSTREAM_FIXES.md` before touching network, AD or SO
   plumbing.** This repo keeps no such log of its own, and most entries there
   apply here — same base roles, same platform, same images.

2. **Don't edit base roles to fix bugs.** Add an overlay role with the same
   name (`build_tarball.sh` prefers `./roles/<name>` over the base repo's), or
   add a compensating play.

3. **Don't edit the role list in `build_tarball.sh`.** Roles are discovered
   from the playbooks; add one by referencing it in a play.

4. **Rebuild the tarball with every commit that changes what deploys.** The
   controller deploys from the tarball, not from a git pull.

5. **Both SIEMs are in scope here.** A change that only considers one of them
   is half a change — Splunk forwarders and Elastic Agents read the same files
   on the same hosts, and the two can compete for a log source.

## Filesystem locations on a deployed controller

```
/etc/ansible/                          ← extracted tarball
/etc/ansible/site.yml                   ← what deploy.sh runs
/etc/ansible/arbitr_pp_playbook.yaml    ← baseline + Splunk
/etc/ansible/playbooks/                 ← the Security Onion phases
/etc/ansible/{host_vars,group_vars,roles}/
~/.ansible/ansible.log                  ← per-run log
~/.ansible/retry/                       ← failure retry hostlists
/var/log/playbook_run.log               ← full run transcript
```

## Current state

- **Both SIEMs are live on one range.** Splunk as a distributed cluster
  (manager + two indexer peers + search head, label `pp-idxc`) with universal
  forwarders on ~50 hosts; Security Onion as a five-node grid fed by GRE
  mirrors from three routers.
- **Container images come from the controller's mirror, not ghcr.io.** The
  controller builds the registry artifacts with skopeo and serves them;
  `so_manager` stages them before `so-setup`, which is the supported airgap
  path. No in-play system needs internet access.
- **Elastic Agent enrolment is opt-in**, not part of a deploy — the endpoints
  already carry a Splunk universal forwarder.
- **Three pfSense firewalls** on the 2.8.1 image; routing is eBGP-at-edge,
  OSPF between the upstream pair, static everywhere else.
- Syslog collects to pp-syslog's `/var/log/remote/` store from Linux, VyOS and
  pfSense.

If something here does not match a fresh deploy, the likely cause is an
upstream change in the customer repo or a new SimSpace image revision.
