# Compliance Hardening Pipeline

Ansible automation for enforcing CIS benchmarks across network infrastructure and server environments. Audits Cisco device configs, hardens Linux/Windows hosts, and generates compliance reports with ITIL change control.

## Project Series

This is **Project 3 of 5** in a production enterprise environment build. Each project builds on the previous one.

| # | Project | What It Adds |
|---|---------|-------------|
| 1 | [Enterprise Network Segmentation](https://github.com/calvinanglo/enterprise-network-segmentation) | VLANs, OSPF, ACLs, pfSense firewall |
| 2 | [Wazuh SIEM Deployment](https://github.com/calvinanglo/wazuh-siem-deployment) | Centralized log collection, threat detection, incident response |
| 3 | [Compliance Hardening Pipeline](https://github.com/calvinanglo/compliance-hardening-pipeline) | Automated CIS benchmarks across all devices |
| 4 | [Network Monitoring Stack](https://github.com/calvinanglo/network-monitoring-stack) | Prometheus, Grafana, SNMP monitoring, SLA dashboards |
| 5 | [DR & BC Simulation](https://github.com/calvinanglo/dr-bc-simulation) | Disaster recovery testing, backup validation, RTO/RPO measurement |

### Prerequisites
- **Complete [Project 1](https://github.com/calvinanglo/enterprise-network-segmentation) first** — the Cisco devices and network topology must be in place
- **Complete [Project 2](https://github.com/calvinanglo/wazuh-siem-deployment) first** — the hardening playbooks forward logs to Wazuh at 10.10.20.10
- Ansible control node with Python 3 and SSH access to all devices
- Linux servers on VLAN 20, Windows workstations on VLAN 10

### What's Next
After completing this project, continue to [Project 4: Network Monitoring Stack](https://github.com/calvinanglo/network-monitoring-stack) to add Prometheus/Grafana observability. The monitoring stack uses the same SNMPv3 credentials configured in Project 1 and provides the SLA dashboards that complement the security monitoring from Project 2.

## What This Does

This pipeline automates security compliance enforcement and continuous auditing. Instead of manually checking whether SSH v2 is enabled on 50 servers, you run a playbook once and get a compliance report showing exactly which hosts fail and why.

The four main playbooks:
- **cisco-audit.yml** — Connects to network devices, grabs running configs, checks them against hardening standards, reports gaps
- **linux-hardening.yml** — Enforces CIS Level 1 + Level 2 controls on Ubuntu/Debian: SSH config, file permissions, auditd, kernel parameters
- **windows-hardening.yml** — GPO + PowerShell hardening for Windows 10/11 workstations: UAC, Windows Firewall, audit policies
- **compliance-report.yml** — Aggregates audit results from Linux, Cisco, and Windows hosts into a single compliance report

## How It Works

```
Ansible control node
    |
    ├─→ [Cisco L3 Switch]      ← SSH, check SSH v2, SNMP v3, NTP, ACLs
    │                            → Reports: pass/fail per control
    │
    ├─→ [Ubuntu servers]        ← Apply CIS hardening playbook
    │   ├─ SSH: PermitRootLogin=no, MaxAuthTries=4
    │   ├─ Auditd: enable, load login audit rules
    │   ├─ Sysctl: disable IPv6 forwarding, enable rp_filter
    │   └─ Reports: pass/fail per control with remediation notes
    │
    ├─→ [Windows workstations]  ← Apply PowerShell hardening
    │   ├─ UAC enforcement
    │   ├─ Windows Firewall rules
    │   └─ Reports: pass/fail per control
    │
    └─→ [Compliance Report]     ← Aggregated HTML/JSON report
        └─ Saved to reports/ directory with timestamp
```

## Quick Start

```bash
# clone and install dependencies
git clone https://github.com/calvinanglo/compliance-hardening-pipeline
cd compliance-hardening-pipeline
pip install ansible ansible-lint
ansible-galaxy collection install cisco.ios community.windows

# update inventory with your device IPs and credentials
vi inventory/hosts.yml

# dry run first - always check before applying
ansible-playbook playbooks/linux-hardening.yml -i inventory/hosts.yml --check

# run cisco audit
ansible-playbook playbooks/cisco-audit.yml -i inventory/hosts.yml

# harden linux servers (CIS Level 1 only)
ansible-playbook playbooks/linux-hardening.yml -i inventory/hosts.yml --tags cis-level-1

# harden windows workstations
ansible-playbook playbooks/windows-hardening.yml -i inventory/hosts.yml

# generate full compliance report across all hosts
ansible-playbook playbooks/compliance-report.yml -i inventory/hosts.yml
```

## Repo Structure

```
compliance-hardening-pipeline/
  playbooks/
    cisco-audit.yml              # Cisco device hardening audit
    linux-hardening.yml          # CIS Ubuntu Level 1 + Level 2
    windows-hardening.yml        # Windows 10/11 hardening
    compliance-report.yml        # Aggregate report generation
  inventory/
    hosts.yml                    # Device IPs, credentials, groups
  docs/
    itil-change-process.md       # Change management integration guide
```

## Key Playbooks

### cisco-audit.yml

Connects via SSH (network_cli) and runs show commands to verify hardening state. Checks SSH version, SNMP version, NTP sync, ACL presence, and unused service disablement. Outputs a per-device pass/fail report to /tmp/audit_reports/.

The audit is read-only — it doesn't change anything on the device. If you want to apply fixes, do that manually following the change management process and re-run the audit to verify.

### linux-hardening.yml

CIS Ubuntu Linux Benchmark Level 1 and Level 2. Split into tagged sections so you can run just what you need:

**Level 1 (low-risk, run first):**
- SSH: disable root login, disable password auth, set MaxAuthTries 4, require SSHv2
- Filesystem: disable cramfs, freevxfs, jffs2, usb-storage kernel modules
- Accounts: password max age 90 days, minimum age 7 days, warn at 14 days
- Services: disable avahi-daemon, cups, rpcbind

**Level 2 (higher impact, test first):**
- Sysctl: IP forwarding off, ICMP redirects rejected, TCP SYN cookies enabled, rp_filter on
- PAM: lockout after 5 failed attempts, history of 5 passwords
- Auditd: login events, sudo use, privileged command execution
- sudo logging to syslog

### windows-hardening.yml

Uses win_regedit and win_shell to enforce hardening. Key controls:
- UAC set to highest level, consent prompt on secure desktop
- Windows Firewall enabled on all profiles, default-deny inbound
- RDP disabled unless required; if enabled, NLA enforced
- Account lockout: 5 attempts, 30-minute lockout duration
- Audit policy: logon events, privilege use, process creation
- Security event log size: 1GB
- WinRM restricted to management VLAN (10.10.99.0/24)

Uses NTLM for WinRM transport since there's no AD in the home lab (no Kerberos).

### compliance-report.yml

Runs compliance checks across Linux servers, Cisco devices, and Windows workstations, then generates a timestamped aggregate report listing every check, its pass/fail status, and remediation guidance for failures. Individual and aggregate reports land in /tmp/compliance-reports/ on the control node.

## ITIL Change Management

Every hardening run that modifies a system needs a change record before it runs. See docs/itil-change-process.md for the full workflow.

Standard changes (pre-approved, low risk) — CIS Level 1 SSH hardening, enabling auditd, setting password policies. Fast-track checklist, no CAB meeting required.

Normal changes (higher risk) — CIS Level 2 sysctl changes, PAM modifications, WinRM reconfiguration. Full CAB review with impact assessment and rollback plan.

Emergency changes — only used if applying a critical patch outside the normal cycle. Must be documented within 2 hours of the change being made.

## Certifications this covers

CCNA 200-301 — Cisco IOS hardening via SSH, SNMP v3, NTP, ACL verification, understanding of show commands for compliance auditing

CompTIA Security+ — CIS benchmarks, attack surface reduction, defense in depth, compliance automation, evidence preservation via audit logs

ISC2 CC — Configuration management, access control enforcement, least privilege (SSH key auth, no root logins), organizational compliance

ITIL 4 — Automated testing reduces risk of failed changes; audit trail integrates with incident management; compliance dashboard supports problem management

## Troubleshooting

**Ansible can't connect to Cisco device:**
Make sure network_cli is set in the inventory and the device has SSH enabled. The playbook uses ios as the network OS. Test with `ansible cisco_devices -m ios_command -a "commands='show version'"`.

**Windows host unreachable:**
WinRM must be enabled: `Enable-PSRemoting -Force` on the target. The inventory uses NTLM auth — if you see 401 errors, verify the ansible_user and ansible_password are correct and the account has admin rights.

**False positives in the report:**
Some controls don't apply to every environment. If a check is flagging correctly configured settings as failures, review the variable definitions in the playbook vars section and adjust the expected values to match your baseline.
