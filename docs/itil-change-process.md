# ITIL Change Management Process

How this pipeline integrates with ITIL 4 Change Enablement. Every hardening run that modifies a system needs a change record before it runs.

## Change Types

Standard Changes: Pre-approved, low risk, well-understood procedure. Most compliance hardening falls here. These go through a fast-track checklist, no full CAB meeting required. Examples: CIS Level 1 SSH hardening, enabling auditd, setting password policies.

Normal Changes: Higher risk or impact, require CAB review and scheduled maintenance window. Examples: Kernel parameter changes, disabling services that users depend on, any change to a Cisco device in production.

Emergency Changes: Unexpected critical vulnerability (zero-day), documented after the fact. Needs post-change review within 48 hours.

## Change Record Template

Copy this for each change. File in the changes/ directory before running any playbook.

```
# CHG-XXX: [Brief description]

Category:       Standard / Normal / Emergency
Requested By:   Calvin
Date:           YYYY-MM-DD
Scheduled:      YYYY-MM-DD HH:MM (maintenance window)
Approval:       [PENDING / APPROVED / REJECTED]

## What's Changing

Target systems: [list hosts or groups]
Ansible playbook: [playbook name + tags]

Changes applied:
- [List specific config changes]

## Impact Assessment

Risk Level: LOW / MEDIUM / HIGH
Affected users: [who might notice if something goes wrong]
Downtime: [None / X minutes]
Tested on: [dev clone, staging environment, etc]

## Verification

After the playbook runs:
1. [How to verify it worked]
2. Run playbook in --check mode again, expect no changes

## Rollback

How to undo this if something breaks:
1. [Step 1]
2. Takes approximately X minutes

## Sign-off

Approval:      ________________  Date: ________
CAB (if Normal):  ________________  Date: ________
Post-review:   ________________  Date: ________
```

## Practical Workflow

Before every hardening run:

1. Run the audit first in --check mode to see what will change
2. 2. Fill out the change template above
   3. 3. For Standard changes: self-approve and proceed
      4. 4. For Normal changes: schedule the maintenance window, get CAB sign-off
         5. 5. Run the playbook
            6. 6. Run audit again in --check mode to confirm no changes remain
               7. 7. Document results in the change record (Post-review section)
                 
                  8. ## Real Example: CHG-014
                 
                  9. ```
                     # CHG-014: CIS Level 1 SSH Hardening - Linux Servers

                     Category:       Standard Change
                     Requested By:   Calvin
                     Date:           2026-04-05
                     Scheduled:      2026-04-05 02:00 AM
                     Approval:       SELF-APPROVED (Standard)

                     ## What's Changing

                     Target systems: linux_servers group (ubuntu-dns01, ubuntu-fileshare01, ubuntu-auth01)
                     Ansible playbook: linux-hardening.yml --tags ssh,cis-level-1

                     Changes applied:
                     - PermitRootLogin: no
                     - MaxAuthTries: 4
                     - PermitEmptyPasswords: no
                     - X11Forwarding: no
                     - LoginGraceTime: 60
                     - Protocol: 2

                     ## Impact Assessment

                     Risk Level: LOW
                     No downtime. SSH changes apply on service restart.
                     Root logins were not in use anyway.
                     Tested: Verified on dev VM clone first, SSH still works after changes.

                     ## Verification

                     After run:
                     1. SSH to each server, check sshd_config values match above
                     2. Confirm SSH still works: ssh calvin@10.10.10.5
                     3. Re-run: ansible-playbook linux-hardening.yml --tags ssh --check
                        Expected: 0 tasks changed

                     ## Rollback

                     If SSH breaks after the change:
                     1. Console access to server (out-of-band, VLAN 30 jump box)
                     2. Edit /etc/ssh/sshd_config manually, revert changed values
                     3. systemctl restart sshd
                     4. Takes ~5 minutes

                     ## Sign-off

                     Approval:      Calvin   Date: 2026-04-05
                     Post-review:   Calvin   Date: 2026-04-05 - PASS, all servers accessible via SSH
                     ```

                     ## Tips Learned

                     The first time I ran this, I had PermitRootLogin set to no but one of the servers was using root for cron jobs pulling config from a backup server. Had to roll that back and switch the cron user to a service account before re-applying. That's why the dev testing step matters.

                     Also, running with --diff flag alongside --check is really useful. It shows you exactly what will change in the files before applying. Much better than just seeing "would change" with no detail.
