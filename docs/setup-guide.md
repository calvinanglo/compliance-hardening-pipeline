# Compliance Hardening Pipeline - Setup Guide

Step-by-step instructions for deploying the compliance hardening pipeline across Cisco, Linux, and Windows targets in the home lab.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Installing Ansible and Galaxy Collections](#2-installing-ansible-and-galaxy-collections)
3. [Configuring Inventory](#3-configuring-inventory)
4. [Running Cisco Audit](#4-running-cisco-audit)
5. [Running Linux Hardening](#5-running-linux-hardening)
6. [Running Windows Hardening](#6-running-windows-hardening)
7. [Generating the Compliance Report](#7-generating-the-compliance-report)
8. [ITIL Change Management Workflow](#8-itil-change-management-workflow)
9. [Verification](#9-verification)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Prerequisites

Before starting, the following must be in place:

**Projects 1 and 2 complete:**

- Project 1 (Network Infrastructure) -- Cisco switches and router are configured, VLANs are up, SSH is enabled on all network devices, and management VLAN (99) is reachable from the Ansible control node.
- Project 2 (Server and Workstation Deployment) -- Ubuntu servers and Windows workstations are deployed, have IP addresses assigned per the network plan, and are accessible over the network.

**Ansible control node setup:**

- A Linux machine (Ubuntu or similar) on the management network that can reach all target devices.
- Python 3.9 or later installed.
- SSH key pair generated for passwordless authentication to Linux servers and Cisco devices:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/ansible_key -C "ansible-control-node"
```

- SSH public key distributed to all Linux servers and Cisco devices:

```bash
ssh-copy-id -i ~/.ssh/ansible_key.pub ansible@10.10.20.11
ssh-copy-id -i ~/.ssh/ansible_key.pub ansible@10.10.20.12
```

- For Cisco devices, copy the public key via the console or existing SSH session:

```
SW-DIST(config)# ip ssh pubkey-chain
SW-DIST(conf-ssh-pubkey)# username admin
SW-DIST(conf-ssh-pubkey-user)# key-string
  <paste public key here>
SW-DIST(conf-ssh-pubkey-user)# exit
```

- Network connectivity verified from the control node to all targets:

```bash
ping 10.10.20.11    # ubuntu-server-01
ping 10.10.20.12    # ubuntu-server-02
ping 10.10.10.21    # win10-workstation-01
ping 10.10.10.22    # win10-workstation-02
ping 10.10.99.1     # SW-DIST
ping 10.10.99.11    # SW-ACC-1
ping 10.10.99.12    # SW-ACC-2
ping 10.0.1.1       # R1-CORE
```

> Screenshot: [Terminal showing successful ping results to all 8 target hosts]
> Save as: docs/screenshots/01-ping-all-targets.png

---

## 2. Installing Ansible and Galaxy Collections

**Install Ansible via pip:**

```bash
python3 -m venv ~/ansible-venv
source ~/ansible-venv/bin/activate
pip install ansible pywinrm
```

Verify the installation:

```bash
ansible --version
```

> Screenshot: [Terminal output of ansible --version showing version 2.15+ and Python path]
> Save as: docs/screenshots/02-ansible-version.png

**Install required Galaxy collections:**

From the project root directory:

```bash
cd /path/to/compliance-hardening-pipeline
ansible-galaxy install -r requirements.yml
```

This installs:

- `cisco.ios` (>= 4.0.0) -- for managing Cisco IOS devices via network_cli
- `community.windows` (>= 1.0.0) -- for Windows management modules (win_regedit, win_service, etc.)
- `community.general` (>= 6.0.0) -- general-purpose modules

Verify collections installed correctly:

```bash
ansible-galaxy collection list | grep -E "cisco|community"
```

> Screenshot: [Terminal output showing all three Galaxy collections installed with versions]
> Save as: docs/screenshots/03-galaxy-collections.png

---

## 3. Configuring Inventory

Edit `inventory/hosts.yml` with the correct IP addresses, usernames, and credential paths for your environment.

**Linux servers section:**

Update `ansible_host` to match each server's actual IP. Update `ansible_user` if you use a different service account. Update `ansible_ssh_private_key_file` to point to your SSH key:

```yaml
linux_servers:
  hosts:
    ubuntu-server-01:
      ansible_host: 10.10.20.11          # <-- your server IP
      ansible_user: ansible               # <-- your service account
      ansible_ssh_private_key_file: ~/.ssh/ansible_key
```

**Windows workstations section:**

Update `ansible_host` for each workstation. Set `ansible_user` and provide the password. The default transport is NTLM over HTTP (port 5985), which works for local accounts in a home lab without Active Directory:

```yaml
windows_workstations:
  vars:
    ansible_winrm_transport: ntlm
    ansible_winrm_server_cert_validation: ignore
    ansible_port: 5985
    ansible_connection: winrm
  hosts:
    win10-workstation-01:
      ansible_host: 10.10.10.21          # <-- your workstation IP
      ansible_user: Administrator          # <-- your admin account
```

Set the Windows password using an environment variable (do not put passwords in the inventory file):

```bash
export ANSIBLE_WINRM_PASSWORD='YourPasswordHere'
```

Or use `--ask-pass` at runtime:

```bash
ansible-playbook playbooks/windows-hardening.yml --ask-pass
```

**Cisco devices section:**

Update `ansible_host` for each switch and router. The connection type is `network_cli` with `cisco.ios.ios` as the network OS:

```yaml
cisco_devices:
  vars:
    ansible_network_os: cisco.ios.ios
    ansible_connection: network_cli
    ansible_user: admin
    ansible_ssh_private_key_file: ~/.ssh/ansible_key
  hosts:
    SW-DIST:
      ansible_host: 10.10.99.1           # <-- your switch IP
```

**Test connectivity to all groups:**

```bash
ansible linux_servers -m ping
ansible windows_workstations -m win_ping
ansible cisco_devices -m cisco.ios.ios_command -a "commands='show version'"
```

> Screenshot: [Terminal output showing successful ping/win_ping results for all three groups]
> Save as: docs/screenshots/04-inventory-connectivity-test.png

---

## 4. Running Cisco Audit

The Cisco audit playbook (`playbooks/cisco-audit.yml`) checks hardening posture without making any changes. It audits: SSH version 2, SNMPv3 configuration, NTP synchronization, syslog forwarding, HTTP server disabled, and CDP disabled.

**Step 4a: Run in check mode first (dry run)**

Even though this playbook is read-only (it only runs `show` commands), running in check mode is good practice to verify connectivity and catch any issues before the real run:

```bash
ansible-playbook playbooks/cisco-audit.yml --check
```

> Screenshot: [Terminal output of Cisco audit in --check mode showing task list and connection success]
> Save as: docs/screenshots/05-cisco-audit-check-mode.png

**Step 4b: Run the full audit**

```bash
ansible-playbook playbooks/cisco-audit.yml
```

The playbook will:
1. Connect to each device in the `cisco_devices` group (SW-DIST, SW-ACC-1, SW-ACC-2, R1-CORE).
2. Run `show` commands to check each hardening item.
3. Write per-device audit reports to `/tmp/audit_reports/`.
4. Print a PASS/FAIL summary for each device.

> Screenshot: [Terminal output showing the full Cisco audit run with PASS/FAIL summary per device]
> Save as: docs/screenshots/06-cisco-audit-full-run.png

Review the reports:

```bash
ls /tmp/audit_reports/
cat /tmp/audit_reports/SW-DIST-audit-*.txt
```

> Screenshot: [Contents of a Cisco audit report file showing PASS/FAIL for each check]
> Save as: docs/screenshots/07-cisco-audit-report-content.png

---

## 5. Running Linux Hardening

The Linux hardening playbook (`playbooks/linux-hardening.yml`) applies CIS Ubuntu Linux Benchmark controls. It is organized with tags for targeted runs: `cis-level-1`, `cis-level-2`, `ssh`, `network`, `logging`, `filesystem`, `accounts`, `services`, `packages`.

**Step 5a: Run in check mode with diff (preview all changes)**

Always preview before applying:

```bash
ansible-playbook playbooks/linux-hardening.yml --check --diff
```

This shows exactly what files would change and what the diffs look like, without making any modifications.

> Screenshot: [Terminal output of Linux hardening in --check --diff mode showing proposed changes]
> Save as: docs/screenshots/08-linux-hardening-check-mode.png

**Step 5b: Apply CIS Level 1 controls first**

Level 1 controls are lower risk and cover the baseline: filesystem hardening, network sysctl parameters, SSH configuration, account policies, service disabling, and package management.

```bash
ansible-playbook playbooks/linux-hardening.yml --tags cis-level-1
```

> Screenshot: [Terminal output of CIS Level 1 hardening run showing changed/ok task counts]
> Save as: docs/screenshots/09-linux-level1-run.png

**Step 5c: Verify Level 1 applied cleanly**

Re-run in check mode. If everything applied correctly, you should see zero changes:

```bash
ansible-playbook playbooks/linux-hardening.yml --tags cis-level-1 --check
```

**Step 5d: Apply CIS Level 2 controls**

Level 2 adds auditd installation, audit rules for privileged commands, and rsyslog forwarding to the Wazuh server. These are higher-impact changes -- file a Normal change record if your environment requires it (see Section 8).

```bash
ansible-playbook playbooks/linux-hardening.yml --tags cis-level-2
```

> Screenshot: [Terminal output of CIS Level 2 hardening run showing auditd and logging tasks]
> Save as: docs/screenshots/10-linux-level2-run.png

**Step 5e: Verify Level 2 applied cleanly**

```bash
ansible-playbook playbooks/linux-hardening.yml --tags cis-level-2 --check
```

---

## 6. Running Windows Hardening

The Windows hardening playbook (`playbooks/windows-hardening.yml`) applies CIS-based controls for Windows 10/11 workstations: UAC settings, firewall, RDP, account policies, service hardening, Defender configuration, audit policies, and registry hardening.

### WinRM Prerequisites

Before Ansible can manage Windows hosts, WinRM must be enabled on each workstation.

**On each Windows target, open an elevated PowerShell prompt and run:**

```powershell
# Enable WinRM
Enable-PSRemoting -Force

# Set WinRM to allow unencrypted traffic (home lab only - use HTTPS in production)
winrm set winrm/config/service '@{AllowUnencrypted="true"}'

# Ensure the WinRM service starts automatically
Set-Service -Name WinRM -StartupType Automatic

# Open firewall for WinRM
New-NetFirewallRule -Name "WinRM-HTTP" -DisplayName "WinRM over HTTP" -Enabled True -Direction Inbound -Protocol TCP -LocalPort 5985 -Action Allow
```

Verify from the Ansible control node:

```bash
ansible windows_workstations -m win_ping --ask-pass
```

> Screenshot: [Terminal output showing successful win_ping to both Windows workstations]
> Save as: docs/screenshots/11-winrm-connectivity.png

**Step 6a: Run in check mode (preview changes)**

```bash
ansible-playbook playbooks/windows-hardening.yml --check --ask-pass
```

> Screenshot: [Terminal output of Windows hardening in --check mode showing proposed changes]
> Save as: docs/screenshots/12-windows-hardening-check-mode.png

**Step 6b: Run the full hardening playbook**

```bash
ansible-playbook playbooks/windows-hardening.yml --ask-pass
```

The playbook applies changes across these categories:
- UAC set to highest level with secure desktop prompt
- Windows Firewall enabled on all profiles, inbound blocked by default
- RDP disabled (or NLA required if RDP is in use)
- Account lockout threshold (5 attempts), minimum password length (14 characters)
- Print Spooler, Remote Registry, and Telnet disabled
- Windows Defender real-time protection, cloud protection, and controlled folder access enabled
- Audit policies for logon, privilege use, and process creation
- Registry hardening (autorun disabled, anonymous access restricted, SEHOP enabled)
- WinRM restricted to management VLAN (10.10.99.0/24)

> Screenshot: [Terminal output of full Windows hardening run showing task results]
> Save as: docs/screenshots/13-windows-hardening-full-run.png

**Step 6c: Verify changes applied**

```bash
ansible-playbook playbooks/windows-hardening.yml --check --ask-pass
```

Expect zero changes if everything applied correctly.

---

## 7. Generating the Compliance Report

The compliance report playbook (`playbooks/compliance-report.yml`) scans all three platform groups and generates individual and aggregate reports.

It checks:
- **Linux:** auditd running, AppArmor enabled, SSH root login disabled, no empty password accounts, firewall active, AIDE installed, syslog forwarding to Wazuh.
- **Cisco:** SSH v2, SNMPv3, NTP sync, syslog forwarding, HTTP server off, CDP off.
- **Windows:** UAC enabled, firewall on, RDP disabled, account lockout set, Defender real-time protection on.

**Run the report:**

```bash
ansible-playbook playbooks/compliance-report.yml --ask-pass
```

> Screenshot: [Terminal output of compliance report playbook run completing across all three host groups]
> Save as: docs/screenshots/14-compliance-report-run.png

**View individual reports:**

```bash
ls /tmp/compliance-reports/
cat /tmp/compliance-reports/ubuntu-server-01-compliance-*.txt
cat /tmp/compliance-reports/SW-DIST-compliance-*.txt
cat /tmp/compliance-reports/win10-workstation-01-compliance-*.txt
```

**View the aggregate report:**

```bash
cat /tmp/compliance-reports/aggregate-compliance-*.txt
```

> Screenshot: [Contents of the aggregate compliance report showing PASS/FAIL for all hosts]
> Save as: docs/screenshots/15-aggregate-compliance-report.png

---

## 8. ITIL Change Management Workflow

Every hardening run that modifies systems requires a change record. The full process is documented in `docs/itil-change-process.md`. Here is the practical workflow to follow before each playbook run.

**Before running any hardening playbook:**

1. **Run the playbook in `--check --diff` mode** to see exactly what will change.

2. **Create a change record** using the template in `docs/itil-change-process.md`. Save it in the `changes/` directory:

```bash
cp docs/itil-change-process.md changes/CHG-XXX-description.md
```

Edit the file and fill out:
- Category (Standard, Normal, or Emergency)
- Target systems and playbook command with tags
- List of specific configuration changes
- Risk level and impact assessment
- Verification steps
- Rollback procedure

3. **For Standard changes** (most CIS Level 1 hardening): self-approve and proceed.

4. **For Normal changes** (kernel parameters, disabling user-facing services, Cisco production changes): schedule a maintenance window and get CAB sign-off before running.

5. **Run the playbook.**

6. **Re-run in `--check` mode** to confirm zero changes remain.

7. **Document results** in the Post-review section of the change record.

**Example -- filing a change for CIS Level 1 SSH hardening:**

```
# CHG-015: CIS Level 1 SSH Hardening - Linux Servers

Category:       Standard Change
Requested By:   Calvin
Date:           2026-04-13
Scheduled:      2026-04-13 02:00 AM
Approval:       SELF-APPROVED (Standard)

## What's Changing

Target systems: linux_servers group
Ansible playbook: linux-hardening.yml --tags ssh,cis-level-1

Changes applied:
- PermitRootLogin: no
- MaxAuthTries: 4
- PermitEmptyPasswords: no
- X11Forwarding: no

## Verification

1. SSH to each server, confirm sshd_config values
2. Re-run: ansible-playbook playbooks/linux-hardening.yml --tags ssh --check
   Expected: 0 changes

## Rollback

1. Console access via out-of-band VLAN 99 jump box
2. Manually revert /etc/ssh/sshd_config
3. systemctl restart sshd
```

> Screenshot: [Example change record file open in a text editor with all fields filled out]
> Save as: docs/screenshots/16-change-record-example.png

---

## 9. Verification

After completing all hardening runs, verify the results across each platform.

### 9a. Cisco Audit Report Output

Run the audit one more time to confirm all checks pass:

```bash
ansible-playbook playbooks/cisco-audit.yml
```

Expected output per device:

```
SW-DIST - PASS: 6 / 6
SW-ACC-1 - PASS: 6 / 6
SW-ACC-2 - PASS: 6 / 6
R1-CORE - PASS: 6 / 6
```

Review the detailed report files in `/tmp/audit_reports/` and confirm each check shows `[PASS]`:
- SSH version 2
- SNMPv3 user wazuh-monitor
- NTP synchronized
- Syslog to 10.10.20.10
- HTTP server disabled
- CDP disabled

> Screenshot: [Terminal output showing all four Cisco devices with 6/6 PASS results]
> Save as: docs/screenshots/17-cisco-audit-verification.png

> Screenshot: [Contents of a Cisco audit report file showing all PASS results]
> Save as: docs/screenshots/18-cisco-audit-report-detail.png

### 9b. Linux Hardening Changes Applied

Run the full hardening playbook in check mode to confirm idempotency (zero changes):

```bash
ansible-playbook playbooks/linux-hardening.yml --check
```

Expected: all tasks show `ok`, none show `changed`.

Spot-check key items manually:

```bash
# SSH hardening
ansible linux_servers -m command -a "sshd -T" | grep -E "permitrootlogin|maxauthtries|x11forwarding"

# Sysctl network hardening
ansible linux_servers -m command -a "sysctl net.ipv4.ip_forward"

# auditd running
ansible linux_servers -m command -a "systemctl is-active auditd"

# rsyslog forwarding
ansible linux_servers -m command -a "grep 10.10.20.10 /etc/rsyslog.conf"
```

> Screenshot: [Terminal output of Linux hardening --check mode showing 0 changed tasks]
> Save as: docs/screenshots/19-linux-hardening-verification.png

### 9c. Windows Hardening Changes Applied

Run in check mode to confirm zero changes:

```bash
ansible-playbook playbooks/windows-hardening.yml --check --ask-pass
```

Spot-check key items:

```bash
# UAC status
ansible windows_workstations -m win_reg_stat -a "path='HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' name=EnableLUA" --ask-pass

# Firewall status
ansible windows_workstations -m win_shell -a "Get-NetFirewallProfile | Select-Object Name,Enabled" --ask-pass

# Account lockout
ansible windows_workstations -m win_shell -a "net accounts" --ask-pass
```

> Screenshot: [Terminal output of Windows hardening --check mode showing 0 changed tasks]
> Save as: docs/screenshots/20-windows-hardening-verification.png

### 9d. Aggregate Compliance Report

Generate a final compliance report covering all platforms:

```bash
ansible-playbook playbooks/compliance-report.yml --ask-pass
```

Review the aggregate report:

```bash
cat /tmp/compliance-reports/aggregate-compliance-*.txt
```

The report should show PASS for all checks across Linux servers, Cisco devices, and Windows workstations. Any FAIL or WARN items need investigation.

> Screenshot: [Full aggregate compliance report showing PASS across all hosts and all checks]
> Save as: docs/screenshots/21-final-aggregate-report.png

---

## 10. Troubleshooting

### Ansible cannot reach Linux servers

```
fatal: [ubuntu-server-01]: UNREACHABLE! => {"msg": "Failed to connect to the host via ssh"}
```

- Verify the SSH key is correct: `ssh -i ~/.ssh/ansible_key ansible@10.10.20.11`
- Check that the `ansible` user exists on the target and has sudo privileges.
- Confirm `host_key_checking = False` is set in `ansible.cfg` (it is by default in this project).
- Check network routing -- can the control node reach the server VLAN (10.10.20.0/24)?

### Ansible cannot reach Windows hosts

```
fatal: [win10-workstation-01]: UNREACHABLE! => {"msg": "winrm connection error"}
```

- Verify WinRM is running on the target: open PowerShell on the Windows host and run `winrm enumerate winrm/config/listener`.
- Check that port 5985 is open: `Test-NetConnection -ComputerName localhost -Port 5985` from the Windows host.
- Confirm the firewall rule allows inbound on 5985.
- If using NTLM, make sure the `ansible_user` is a local administrator account.
- Install pywinrm on the control node if missing: `pip install pywinrm`.

### Ansible cannot reach Cisco devices

```
fatal: [SW-DIST]: UNREACHABLE! => {"msg": "unable to open shell"}
```

- Verify SSH works manually: `ssh -i ~/.ssh/ansible_key admin@10.10.99.1`
- Check that `ip ssh version 2` is configured on the device.
- Confirm `transport input ssh` is set on the VTY lines.
- The management VLAN (99) must be reachable from the control node.

### Playbook fails on a specific task

- Run with increased verbosity to get details: `ansible-playbook playbooks/linux-hardening.yml -vvv`
- Run only the failing tag to isolate: `ansible-playbook playbooks/linux-hardening.yml --tags ssh`
- Check if the target host meets the task prerequisites (e.g., `apt` available, correct OS version).

### WinRM NTLM authentication fails

```
winrm error: the specified credentials were rejected by the server
```

- Verify the password is correct.
- Ensure the account is a local administrator.
- On the Windows host, check that NTLM is enabled: `winrm get winrm/config/service/auth`
- If Basic auth is needed instead, set `ansible_winrm_transport: basic` in the inventory and enable it on the host: `winrm set winrm/config/service/auth '@{Basic="true"}'`

### Compliance report shows FAIL for a check

- Review the specific FAIL item in the report file.
- Re-run the corresponding hardening playbook with `--tags` for that area.
- If the check involves a service (auditd, ufw), SSH to the host and check the service status manually.
- Some checks (like AIDE database) require a one-time initialization step: `aide --init && mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db`

### Check mode shows changes after a successful run

- Some tasks are not fully idempotent (especially `win_shell` commands that use `Set-NetFirewallProfile`). These will always show as "changed" in check mode because Ansible cannot determine the current state through shell commands.
- For true idempotency verification on Windows, use the compliance report playbook instead -- it checks the actual state via registry reads and PowerShell queries.

### Permission denied on Linux targets

```
fatal: [ubuntu-server-01]: FAILED! => {"msg": "Missing sudo password"}
```

- The `ansible` user needs passwordless sudo. On the target, add: `echo 'ansible ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/ansible`
- Or run with `--ask-become-pass` to provide the sudo password at runtime.
