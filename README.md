# ss-pp-stacked

Range-specific Ansible overlay for the **PowerPlant** cyber-range scenario
(`voltgrid.com`) on the SimSpace NG platform, carrying **both SIEMs on one
range**: a distributed Splunk cluster and a Security Onion 2.4 grid.

That is what distinguishes this repo from its siblings. `ss-pp-so` is Security
Onion only; this one runs Splunk alongside it so the same telemetry can be
worked in either tool.

This is an overlay, not a standalone playbook. It layers on the customer's
shared platform repo at `../range-development-ansible/`: that repo ships base
roles, this one ships the range's inventory, variables, custom roles and
playbooks. `build_tarball.sh` combines the two into `ab_pp.tgz`, which is
extracted to `/etc/ansible` on the range's Ansible controller.

## Build and deploy

```bash
./build_tarball.sh          # -> ab_pp.tgz  (validates before it writes)
# copy ab_pp.tgz to the controller, extract to /etc/ansible, then:
./deploy.sh                 # site.yml, up to 3 attempts
BOOT_DELAY=0 ./deploy.sh    # skip the boot wait on an already-up range
```

## The range

78 hosts — 45 Windows, 12 Linux, 5 Security Onion nodes, plus the network
devices.

**Splunk** is a distributed cluster, not a single box:

| Host | Role |
|---|---|
| `pp-splunk-cm` | cluster manager, and the license manager for the whole deployment |
| `pp-splunk-idx01`, `pp-splunk-idx02` | indexer peers, cluster label `pp-idxc` |
| `pp-splunk` | search head |

Universal forwarders run on ~50 hosts — every Windows endpoint, the Linux
servers, and the SO sensors.

**Security Onion** is a distributed grid: `so-manager`, `so-search`, and three
sensors (`so-sensor-corp`, `so-sensor-edge`, `so-sensor-ot`) fed by GRE mirrors
from `pp-corp-router`, `pp-isp-router` and `pp-ot-router`.

## What gets deployed

`site.yml` is the entry point:

| Phase | Does |
|---|---|
| `10-mirror` | nginx on the controller serving the SO source, detection content and container registry artifacts |
| `playbooks/00-baseline.yml` | the range baseline — network, AD, hosts, services, **and the whole Splunk cluster** |
| `05-time` | Windows clock correction, DC-first |
| `20-vyos` | GRE tunnels + `tc` mirror rules to the sensors |
| `30-prereqs` | `so_base` on every grid node |
| `40-manager` | waits for the registry artifacts, then installs the manager |
| `50-nodes` | search + sensors join the grid |
| `60-verify` | grid health |
| `75-endpoint` | Sysmon, then Elastic Agent enrolment into SO's Fleet |

**Both SIEMs receive endpoint data.** Every endpoint outside `[unmanaged]` and
`[so_all]` carries a Splunk universal forwarder and an Elastic Agent, so the
same process, file, registry and Windows event telemetry is workable in either
tool. That includes the Splunk cluster hosts themselves, with a path-scoped
Elastic Defend exclusion for `/opt/splunk/var` so their bucket churn does not
swamp the dataset.

Security Onion does not yet read the `pp-syslog` store — pfSense, VyOS, nginx
and squid still reach Splunk only. `ss-pp-so`'s `so_fleet_integrations` role
covers that if it is wanted here.

## Documentation

| File | For |
|---|---|
| `CLAUDE.md` | working on this repo — layout, conventions, pitfalls, recipes |
| `blueprints/` | the SimSpace blueprint this range is built from |

This repo has no `UPSTREAM_FIXES.md` of its own. Customer-repo bugs and the
overlay workarounds for them are logged in `../ss-pp-so/UPSTREAM_FIXES.md` and
`../ss-pp-ab/UPSTREAM_FIXES.md`; most apply here too, because the base roles
and the platform are the same.
