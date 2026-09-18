# Netskope NPA Publisher — Disk Utilisation Monitoring (AWS)

Detect low disk space on every Netskope Publisher across every AWS account,
early enough to act, with no SSH keys, no bastion, no inbound firewall rules,
and no monitoring server to look after.

---

## The problem with the current TIG solution

The attached Telegraf/InfluxDB/Grafana design works, and for one or two
Publishers it is the right amount of engineering. It stops working for this
estate for six specific reasons:

| # | Limitation in the TIG design | Consequence at scale |
|---|---|---|
| 1 | `scp telegraf_install.sh` then SSH per Publisher, by hand | Linear human effort. 200 Publishers is 200 sessions, and no record of which ones were done. |
| 2 | Push model requires **inbound tcp/8086** from every Publisher to one server | Cross-account, cross-VPC, cross-region connectivity: peering or TGW, security-group changes, and a listening port on a box that must be reachable from every environment. |
| 3 | The monitoring server is a single self-managed VM | Single point of failure, and — pointedly — **the InfluxDB server is itself a VM that can run out of disk**. The monitoring system shares the failure mode it exists to detect. |
| 4 | Built by repurposing a Publisher OVA, destroying its config | Unsupported configuration, no clean upgrade path, and a documented warning that it will break a production Publisher if pointed at the wrong host. |
| 5 | No discovery. A Publisher is monitored only if a human remembered | Silent blind spots. An acquired account's Publishers are invisible until they cause an outage. |
| 6 | `admin`/`admin` Grafana, manual dashboard import, static SSH keys | No SSO, no audit trail of who looked at or changed what. |

This solution keeps what was good about it — Grafana, dashboards as code, a
real time-series view — and removes all six.

---

## Design in one paragraph

Ansible remains the configuration-management tool, but it stops being the
transport. Access runs over **AWS Systems Manager**, so the control node holds
no SSH keys and Publishers need no inbound port. Collection uses the
**CloudWatch agent**, installed and kept converged by a **State Manager
association that targets a tag rather than a host list**, which is what makes
fleet size irrelevant. Aggregation uses **CloudWatch cross-account
observability**: metrics stay in the account that produced them and the
monitoring account is granted a read view, so there is no shipping pipeline and
no central database. Detection uses **Metrics Insights alarms** — one alarm per
criticality tier, not one per instance — so a Publisher launched after the
alarm was created is covered automatically. Presentation is a CloudWatch
dashboard for on-call plus **Amazon Managed Grafana** for the familiar Grafana
experience without a VM. Onboarding a new account is a **CloudFormation
StackSet with AutoDeployment**, so an acquired account that joins the target OU
onboards itself.

![Architecture](docs/architecture.svg)

---

## Why cloud-native here, given the preference for the existing stack

Leadership asked for cloud-native services only where they give substantial
benefit. Three do, and the benefit is specific rather than general:

**SSM instead of SSH.** This is not a convenience. The TIG design's SSH+scp
approach requires a key distribution mechanism, a bastion or public IPs, and an
inbound rule on every Publisher. SSM removes all three: the agent dials out,
IAM authorises, and every command lands in CloudTrail with the caller's
identity. For an appliance carrying zero-trust network access traffic, removing
inbound reachability is a security outcome, not a convenience. Ansible still
owns what gets configured — only the transport changed.

**CloudWatch cross-account observability instead of a central InfluxDB.** The
alternative designs all require moving metrics across account and VPC
boundaries. Cross-account observability grants a read view instead of copying
data: no pipeline, no egress, no queue, no database to size, and nothing that
can itself fill up.

**StackSets with AutoDeployment instead of an onboarding runbook.** The company
grows by acquisition. Any design where onboarding an account is a task someone
performs will drift. Making OU membership the trigger removes the human step
entirely.

Everything else stays Ansible. Where a managed service added cost without
removing toil — Amazon Managed Prometheus, for instance, which would have meant
running exporters and a remote-write path — it was rejected. See
`docs/runbook.md` for the full option comparison.

---

## Repository layout

```
.
├── group_vars/all.yml          # every tunable in one file
├── inventory/
│   ├── aws_ec2.yml             # dynamic inventory (tag-filtered, SSM transport)
│   └── generate_inventory.py   # one inventory source per account, from Organizations
├── playbooks/
│   ├── 00_platform_bootstrap.yml   # monitoring account: sink, SNS, params, SSM docs
│   ├── 10_account_onboard.yml      # StackSet across the Organisation
│   ├── 20_discover_publishers.yml  # discovery + reconcile against the Netskope tenant
│   ├── 30_enrol_publishers.yml     # instance profile, enrolment, verification
│   ├── 40_bootstrap_ssm_agent.yml  # SSH fallback for pre-SSM Publishers (retires itself)
│   ├── 50_alarms_dashboards.yml    # CloudWatch + Grafana dashboards
│   ├── 60_attribute_alert.yml      # on alert: name the offending instance
│   ├── 90_verify.yml               # end-to-end acceptance test
│   └── 99_remediate_disk.yml       # guarded reclaim, dry-run by default
├── files/
│   ├── cloudformation/publisher-monitoring-baseline.yml
│   ├── ssm-documents/Netskope-EnrolPublisherDiskMonitoring.yml
│   ├── ssm-documents/Netskope-PublisherDiskReclaim.yml
│   └── grafana/netskope-publisher-disk.json
├── roles/
│   ├── platform_monitoring_account/templates/cwagent-publisher-disk.json.j2
│   └── grafana_dashboards/templates/cw-fleet-dashboard.json.j2
└── docs/
    ├── architecture.svg
    └── runbook.md
```

---

## Minimal working demo (single account, ~25 minutes)

Proves data collection, aggregation, access management and the scalability
primitive end to end, without touching the wider estate.

```bash
# 0. Prerequisites on the control node - nothing is installed on a Publisher
make deps
aws sso login --profile monitoring          # short-lived credentials only

# 1. Edit the only file you need to edit
vi group_vars/all.yml                       # org_id, monitoring_account_id, regions, email

# 2. Build the central platform (no servers are created)
make platform
#    -> prints the OAM sink ARN; confirm the SNS subscription email

# 3. Onboard accounts. For the demo, scope to one OU containing one account.
ansible-playbook playbooks/10_account_onboard.yml -e oam_sink_arn=<from step 2>

# 4. Tag ONE Publisher and watch it enrol itself
aws ec2 create-tags --resources i-0123456789abcdef0 \
  --tags Key=Role,Value=netskope-publisher Key=CriticalityTier,Value=tier1

# 5. Discover and enrol (or simply wait 30 minutes for State Manager)
make discover
make enrol

# 6. Dashboards
make dashboards

# 7. Acceptance test - this is the gate
make verify
```

A green `make verify` asserts four things in order: every Publisher is
discoverable, every Publisher is reachable over SSM, disk metrics are arriving
in the monitoring account, and alarms exist and are not stuck without data.

### Demonstrating the scalability claim

The claim is not "the playbook runs fast". It is that fleet size does not
appear anywhere in the configuration. To prove it, tag a second Publisher and
run nothing at all:

```bash
aws ec2 create-tags --resources i-0fedcba987654321 \
  --tags Key=Role,Value=netskope-publisher Key=CriticalityTier,Value=tier2

# Do not run Ansible. Wait for the next State Manager interval (<= 30 min),
# or for the EventBridge rule if the instance was just launched.
aws ssm describe-instance-associations-status --instance-id i-0fedcba987654321
```

The Publisher installs the agent, reads its tier from its own tag, starts
reporting, appears on both dashboards, and falls under the existing tier-2
alarm. No playbook ran. No alarm was created. No dashboard was edited.

---

## Two things worth verifying in your environment

Stated plainly rather than buried, because both are assumptions about the
Netskope appliance rather than about AWS:

**1. SSM agent presence on the Publisher AMI.** Current Publisher images are
Ubuntu 22.04-based and Ubuntu's cloud images ship `amazon-ssm-agent` as a snap,
but the Publisher is a hardened appliance and older images in an estate grown
by acquisition may differ. Check one Publisher first:

```bash
aws ssm describe-instance-information \
  --filters "Key=tag:Role,Values=netskope-publisher" \
  --query 'InstanceInformationList[].{Id:InstanceId,Ping:PingStatus,Agent:AgentVersion}'
```

Anything missing from that list needs `playbooks/40_bootstrap_ssm_agent.yml`
once, over SSH, after which it is managed keylessly like everything else. That
playbook exists to retire itself.

**2. Emitted metric dimension sets.** The Metrics Insights alarms pin
`SCHEMA("Netskope/Publisher", InstanceId, path, Tier)`. If the agent emits a
different dimension set than expected, the alarms sit in `INSUFFICIENT_DATA`
rather than failing loudly. Confirm once after the first enrolment and adjust
`aggregation_dimensions` in the agent config template if needed:

```bash
aws cloudwatch list-metrics --namespace Netskope/Publisher \
  --metric-name disk_used_percent --recently-active PT3H
```

`make verify` will warn you about exactly this case.

---

## Thresholds, and why percentages alone are wrong here

AWS Publishers launch with an **8 GiB root volume** (Netskope's own docs note
Azure needs 30 GiB "instead of the standard 8 GiB"). On an 8 GiB disk, 80% used
leaves 1.6 GiB — not reliably enough for a Publisher auto-update, which pulls
Docker images and runs `apt`. A percentage-only threshold therefore looks calm
right up to the point an update fails.

So the design alarms on whichever of these trips first:

| Signal | Threshold | Why |
|---|---|---|
| `disk_used_percent` | 70/80% tier 1, 75/85% tier 2, 80/90% tier 3 | Conventional utilisation warning |
| `disk_free_bytes` | < 1.5–2 GiB | Absolute headroom for an auto-update on a small root volume |
| `disk_inodes_free` | < 50,000 | Docker layer churn exhausts inodes; presents as "disk full" with bytes free |
| Projected days-to-full | < 14 days | Catches a slow leak days before it becomes an incident |
| `publishers_unmonitored` | > 0 | A blind spot outranks a threshold breach |

Adjust in `group_vars/all.yml`; nothing else needs touching.

---

## Cost

The dominant cost is custom metrics, and it is bounded by the explicit mount
point list — not by instance count alone. With 3 mount points and 5 measurements
per Publisher, plus the aggregation rollups:

| Item | Basis | 100 Publishers |
|---|---|---|
| Custom metrics | ~16 series/Publisher × $0.30/metric/month | ~$480/mo |
| Metrics Insights alarms | ~6 alarms/account × $0.10 | negligible |
| CloudWatch dashboard | 1 (first 3 free) | $0 |
| Amazon Managed Grafana | per active editor/viewer | ~$9/editor, $5/viewer |
| Cross-account observability | no charge for the link itself | $0 |

The single biggest cost lever is `monitored_paths`. Setting `resources: ["*"]`
in the agent config on a Docker host emits a series per overlay2 mount and can
multiply this bill by an order of magnitude — which is why the config uses an
explicit list and says so in a comment.

Compare against the retired design: one t3.small InfluxDB/Grafana VM is roughly
$15/month in compute, plus EBS, plus backups, plus the patching and on-call
burden of a server whose failure mode is the thing it monitors.

---

## Extending to Azure and GCP

The scenario scoped this to AWS, but the shape transfers because only two of
the six layers are AWS-specific:

| Layer | AWS | Azure | GCP |
|---|---|---|---|
| Keyless access | SSM Session Manager | Azure Arc / Run Command | OS Config / IAP tunnel |
| Discovery by tag | `aws_ec2` inventory | `azure_rm` inventory | `gcp_compute` inventory |
| Collector | CloudWatch agent | Azure Monitor Agent | Ops Agent |
| Aggregation | CloudWatch + OAM | Azure Monitor, Log Analytics | Cloud Monitoring |
| Presentation | **Managed Grafana** | same workspace | same workspace |
| Orchestration | **Ansible** | same playbooks | same playbooks |

The bottom two rows are shared, so a single Grafana workspace with three data
sources gives one pane of glass across all three clouds, driven by one Ansible
repository. The tagging contract (`Role`, `CriticalityTier`) is deliberately
cloud-neutral for this reason.
