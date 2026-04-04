# Compliance Hardening Pipeline

Ansible automation for enforcing CIS benchmarks across network infrastructure and server environments. Audits Cisco device configs, hardens Linux/Windows hosts, and generates compliance reports with ITIL change control.

## What This Does

This pipeline automates security compliance enforcement and continuous auditing. Instead of manually checking whether SSH v2 is enabled on 50 servers, you run a playbook once and get a compliance report showing exactly which hosts fail and why.

The three main playbooks:

- **cisco-audit.yml** — Connects to network devices, grabs running configs, checks them against hardening standards, reports gaps
- - **linux-hardening.yml** — Enforces CIS Level 1 + Level 2 controls on Ubuntu/Debian: SSH config, file permissions, auditd, kernel parameters
  - - **windows-hardening.yml** — GPO + PowerShell hardening for Windows 10/11 workstations: UAC, Windows Firewall, audit policies
    - - **compliance-report.yml** — Aggregates all audit results into HTML and JSON reports with risk scoring
     
      - ## What You Get
     
      - After running the pipeline, you have:
     
      - 1. **Audit Results** — HTML report showing what's compliant and what's not
        2. 2. **Hardened Systems** — All non-compliant configs automatically fixed
           3. 3. **Audit Trail** — YAML files tracking every change (integrates with Git for ITIL CAB approval)
              4. 4. **Risk Scoring** — CVSS-based severity on each finding
                 5. 5. **Ansible Facts** — Saved for trend analysis and historical compliance tracking
                   
                    6. ## Architecture
                   
                    7. ```
                       [Ansible Controller — VLAN 30]
                       │
                       ├─→ [Cisco L3 Switch]      ← SSH pull config, check SSH v2, encryption, SNMP, NTP
                       │                            → Reports: 8/10 checks pass, fix 2 failures
                       │
                       ├─→ [Ubuntu servers]        ← Apply CIS hardening playbook
                       │   ├─ SSH: PermitRootLogin=no, MaxAuthTries=4
                       │   ├─ Auditd: enable, audit rules for logins
                       │   ├─ Sysctl: disable IPv6, enable IP forwarding controls
                       │   └─ Reports: 42/45 controls pass, 3 manual items require review
                       │
                       ├─→ [Windows workstations]  ← Apply GPO + PowerShell hardening
                       │   ├─ UAC enforcement
                       │   ├─ Windows Firewall rules
                       │   ├─ BitLocker/encryption
                       │   └─ Reports: 38/40 controls pass
                       │
                       └─→ [Compliance Report Engine]
                           └─ Aggregates all results → HTML + JSON + JIRA tickets for gaps
                       ```

                       ## Quick Start

                       Prerequisites:

                       - Ansible 2.9+ on controller (VLAN 30 jump box)
                       - - SSH access to Cisco devices (IP SSH enabled, username/key in inventory)
                         - - WinRM or SSH access to Windows hosts (WinRM on port 5985)
                           - - sudo/root access on Linux systems
                            
                             - 1. **Clone and inventory:**
                              
                               2. ```bash
                                  git clone https://github.com/calvinanglo/compliance-hardening-pipeline.git
                                  cd compliance-hardening-pipeline
                                  # Edit inventory/hosts.yml with your device IPs and credentials
                                  ```

                                  2. **Run Cisco audit only:**
                                 
                                  3. ```bash
                                     ansible-playbook playbooks/cisco-audit.yml -i inventory/hosts.yml
                                     # Outputs: reports/cisco-audit-TIMESTAMP.html
                                     ```

                                     3. **Harden Linux servers:**
                                    
                                     4. ```bash
                                        ansible-playbook playbooks/linux-hardening.yml -i inventory/hosts.yml --tags cis-level-1
                                        # Applies SSH hardening, file permissions, auditd, kernel parameters
                                        ```

                                        4. **Full compliance report:**
                                       
                                        5. ```bash
                                           ansible-playbook playbooks/compliance-report.yml -i inventory/hosts.yml
                                           # Generates: reports/compliance-summary-TIMESTAMP.html
                                           ```

                                           ## File Structure

                                           ```
                                           ├── playbooks/
                                           │   ├── cisco-audit.yml              # Cisco config audit
                                           │   ├── linux-hardening.yml          # CIS Linux benchmarks
                                           │   ├── windows-hardening.yml        # Windows hardening (GPO + PowerShell)
                                           │   └── compliance-report.yml        # Report aggregation
                                           │
                                           ├── roles/
                                           │   ├── cisco-hardening/             # Tasks: SSH, SNMP, NTP, ACL checks
                                           │   ├── linux-cis/                   # Tasks: SSH, sysctl, auditd, file perms
                                           │   ├── windows-hardening/           # Tasks: UAC, firewall, audit
                                           │   └── audit-report/                # Jinja2 templates for HTML reports
                                           │
                                           ├── inventory/
                                           │   ├── hosts.yml                    # Device IPs, credentials, groups
                                           │   └── group_vars/                  # Per-group variables (versions, settings)
                                           │
                                           ├── templates/
                                           │   ├── audit-report.html.j2         # HTML report template
                                           │   ├── compliance-summary.json.j2   # JSON export for dashboards
                                           │   └── change-record.md.j2          # ITIL change template
                                           │
                                           ├── vars/
                                           │   ├── cis-linux-level-1.yml        # CIS control definitions
                                           │   ├── cis-cisco-hardening.yml      # Cisco hardening standards
                                           │   └── windows-controls.yml         # Windows security baselines
                                           │
                                           └── docs/
                                               ├── cisco-hardening-standard.md  # Cisco requirements
                                               ├── linux-cis-guide.md           # What each Linux control does
                                               ├── windows-baselines.md         # Windows security controls
                                               └── itil-change-process.md       # How to integrate with CAB
                                           ```

                                           ## Key Playbooks

                                           ### cisco-audit.yml

                                           Audits Cisco device running configs against hardening checklist:

                                           - SSH v2 enabled
                                           - - No default SNMP community strings (public/private)
                                             - - NTP configured
                                               - - No unused interfaces in UP state
                                                 - - Password encryption enabled
                                                   - - Logging configured
                                                     - - Access lists restrict management access
                                                      
                                                       - Reports pass/fail + remediation steps for each failure.
                                                      
                                                       - ### linux-hardening.yml
                                                      
                                                       - Implements CIS Benchmarks Level 1 and Level 2:
                                                      
                                                       - **Level 1 (6 hours to implement):**
                                                       - - Disable unused filesystems (cramfs, freevxfs, jffs2, hfs, hfsplus, udf)
                                                         - - Set SELinux/AppArmor to enforcing (if applicable)
                                                           - - SSH hardening: PermitRootLogin=no, MaxAuthTries=4, PermitEmptyPasswords=no, Protocol 2
                                                             - - Enable auditd and configure audit rules
                                                               - - Enable file integrity monitoring
                                                                
                                                                 - **Level 2 (4 hours):**
                                                                 - - Sysctl hardening: IP forwarding, ICMP redirects, TCP SYN cookies
                                                                   - - PAM (Pluggable Authentication Modules): lockout after 5 failed attempts
                                                                     - - Enable AIDE for file integrity
                                                                       - - Restrict SSH access by user/group
                                                                         - - Configure sudo logging to syslog
                                                                          
                                                                           - ### windows-hardening.yml
                                                                          
                                                                           - Applies security baselines to Windows 10/11:
                                                                          
                                                                           - - Enable UAC (User Account Control)
                                                                             - - Windows Firewall: enable inbound/outbound rules
                                                                               - - Disable unnecessary services (RDP, NetBIOS, LLMNR)
                                                                                 - - BitLocker encryption enforcement
                                                                                   - - Audit policy: log successful/failed logins
                                                                                     - - Windows Update: auto-download and schedule install
                                                                                      
                                                                                       - ## Compliance Report Output
                                                                                      
                                                                                       - After running `compliance-report.yml`, you get:
                                                                                      
                                                                                       - **HTML Summary:**
                                                                                      
                                                                                       - ```
                                                                                         COMPLIANCE SUMMARY — 2026-04-04
                                                                                         ================================
                                                                                         Environment: Production (10.10.0.0/16 network)
                                                                                         Report Generated: 2026-04-04 14:32:00
                                                                                         Next Audit: 2026-04-11

                                                                                         OVERALL COMPLIANCE: 87% (125 / 144 controls pass)

                                                                                         By Category:
                                                                                         - Network Devices:     90% (9 / 10 pass)   1 finding: SNMP v1 still enabled on old router
                                                                                         - Linux Servers:       88% (21 / 24 pass)  3 findings: 2 manual reviews, 1 AppArmor config
                                                                                         - Windows Workstns:    84% (8 / 10 pass)   2 findings: BitLocker not enforced on 2 machines

                                                                                         CRITICAL FINDINGS (fix immediately):
                                                                                         1. Router1 still using SNMP v1 — Risk: Easy credential disclosure
                                                                                            → Action: disable snmp-server community public
                                                                                            → Change ID: CHG-143 (pending CAB approval, scheduled 2026-04-10)

                                                                                         HIGH FINDINGS (fix this sprint):
                                                                                         2. WS-02 has BitLocker disabled — Risk: Data exposure if stolen
                                                                                            → Action: Enable BitLocker via GPO or manual powershell
                                                                                         3. Ubuntu-Auth01 auditd not logging logins — Risk: No forensics trail
                                                                                            → Action: Import audit rules, restart auditd

                                                                                         MEDIUM FINDINGS (address in next quarter):
                                                                                         4. SSH banners not customized on 3 servers
                                                                                         5. File permissions on /var/log/secure not restricted (644 vs 600)
                                                                                         ```

                                                                                         **JSON for Integration:**

                                                                                         ```json
                                                                                         {
                                                                                           "timestamp": "2026-04-04T14:32:00Z",
                                                                                           "environment": "production",
                                                                                           "overall_compliance": 0.87,
                                                                                           "findings": [
                                                                                             {
                                                                                               "id": "SNMP-v1-enabled",
                                                                                               "severity": "critical",
                                                                                               "cvss": 7.5,
                                                                                               "affected_systems": ["Router1"],
                                                                                               "description": "SNMPv1 community string 'public' detected",
                                                                                               "remediation": "Disable snmp-server community public; use SNMPv3",
                                                                                               "change_id": "CHG-143"
                                                                                             },
                                                                                             ...
                                                                                           ]
                                                                                         }
                                                                                         ```

                                                                                         ## ITIL Change Management Integration

                                                                                         Every hardening change creates a change record for CAB (Change Advisory Board) approval:

                                                                                         **Before running hardening:**

                                                                                         ```bash
                                                                                         # 1. Generate change record
                                                                                         ansible-playbook playbooks/generate-change-record.yml \
                                                                                           -e "change_type=Standard" \
                                                                                           -e "affected_systems=all-linux-servers" \
                                                                                           -e "description=CIS Level 1 hardening on Ubuntu servers"
                                                                                         # Outputs: changes/CHG-144-linux-hardening-2026-04-04.md

                                                                                         # 2. Review the record, submit to CAB
                                                                                         # 3. Get approval
                                                                                         # 4. Then run hardening
                                                                                         ```

                                                                                         **Change Record Template (auto-generated):**

                                                                                         ```markdown
                                                                                         # CHG-144: CIS Level 1 Hardening — Ubuntu Servers

                                                                                         Category:       Standard Change
                                                                                         Requested By:   Calvin
                                                                                         Date:           2026-04-04
                                                                                         Approval:       [PENDING]

                                                                                         ## What's Changing

                                                                                         Enforce CIS Benchmarks Level 1 on:
                                                                                         - Ubuntu-DNS01, Ubuntu-DNS02
                                                                                         - Ubuntu-FileShare01
                                                                                         - Ubuntu-Auth01

                                                                                         Changes include:
                                                                                         - SSH: PermitRootLogin=no, MaxAuthTries=4
                                                                                         - Auditd: Enable, load login audit rules
                                                                                         - Filesystems: Disable cramfs, freevxfs, jffs2, hfs, hfsplus, udf
                                                                                         - Sysctl: IP forwarding=0, ICMP redirects=0, SYN cookies=1

                                                                                         ## Impact Assessment

                                                                                         Risk Level: LOW
                                                                                         - Changes are defensive only (restrict bad behavior, not enable new services)
                                                                                         - Tested on dev clones first ✓
                                                                                         - Rollback plan: Restore from git backup (< 5 min per server)

                                                                                         Downtime: None (all changes applied without restart)

                                                                                         ## Verification

                                                                                         After playbook runs:
                                                                                         1. SSH to each server, verify PermitRootLogin=no in /etc/ssh/sshd_config
                                                                                         2. Check auditd status: systemctl status auditd
                                                                                         3. Run audit again: ansible-playbook playbooks/linux-hardening.yml --check
                                                                                            Expected: PASS (no changes needed)

                                                                                         ## Rollback

                                                                                         If any issues:
                                                                                         1. Revert from git: git checkout roles/linux-cis/
                                                                                         2. Run playbook in "undo" mode: ansible-playbook playbooks/linux-hardening.yml -e mode=rollback
                                                                                         3. Takes ~5 minutes per server

                                                                                         Approval:      ______________________   Date: __________
                                                                                         CAB Sign-off:  ______________________   Date: __________
                                                                                         ```

                                                                                         ## Certifications Demonstrated

                                                                                         **CCNA:** Cisco device audit, SSH hardening on network infrastructure, SNMP/NTP config validation, ACL verification

                                                                                         **CompTIA Security+:** CIS benchmarks, attack surface reduction, defense in depth, compliance automation, evidence preservation (audit logs)

                                                                                         **ISC2 CC:** Configuration management, access control enforcement, organizational compliance, least privilege (SSH key auth, no root logins)

                                                                                         **ITIL 4:** Automated testing reduces risk of failed changes; audit trail integrates with incident management; compliance dashboard supports problem management; change records enable release management

                                                                                         ## Troubleshooting

                                                                                         Ansible can't connect to Cisco device:

                                                                                         ```bash
                                                                                         # Test SSH connectivity:
                                                                                         ssh -i keys/cisco.key admin@10.10.30.1

                                                                                         # Check Ansible inventory syntax:
                                                                                         ansible-inventory -i inventory/hosts.yml --list

                                                                                         # Run with verbose debug:
                                                                                         ansible-playbook playbooks/cisco-audit.yml -v -vv

                                                                                         # Common issue: SSH key permissions
                                                                                         chmod 600 keys/cisco.key
                                                                                         ```

                                                                                         Windows hosts failing WinRM tasks:

                                                                                         ```powershell
                                                                                         # On Windows target, enable WinRM:
                                                                                         Enable-PSRemoting -Force
                                                                                         Set-NetFirewallRule -Name "WINRM-HTTP-In-TCP" -Enabled True
                                                                                         ```

                                                                                         Playbook reports pass but you know a control is wrong:

                                                                                         ```bash
                                                                                         # Check what Ansible sees on the target:
                                                                                         ansible windows_workstations -m setup | grep -i bitlocker

                                                                                         # Verify the control definition:
                                                                                         cat vars/windows-controls.yml | grep -A 5 bitlocker

                                                                                         # Test a single task:
                                                                                         ansible-playbook playbooks/windows-hardening.yml --tags bitlocker --check
                                                                                         ```

                                                                                         ## Next Steps

                                                                                         1. Customize `inventory/hosts.yml` with your environment
                                                                                         2. 2. Review CIS benchmarks in `vars/` — not all controls apply to every org
                                                                                            3. 3. Run in `--check` mode first to see what changes without applying
                                                                                               4. 4. Set up JIRA integration to auto-create tickets for high-severity findings
                                                                                                  5. 5. Schedule weekly audits via cron (run playbooks Monday 2 AM)
                                                                                                     6. 6. Integrate compliance dashboard with your monitoring stack (Project 4)
