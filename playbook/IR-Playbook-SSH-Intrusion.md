# Incident Response Playbook
## SSH Brute Force → Unauthorized Access → Persistence

| Field | Value |
|-------|-------|
| Playbook ID | PB-001 |
| Incident Type | External SSH intrusion with persistence |
| Framework | NIST SP 800-61 Rev. 2 |
| Applies To | Linux endpoints (Raspberry Pi 5) |
| Owner | Tyrik Parker |
| Version | 1.0 |

---

## Overview

This playbook provides the step-by-step response procedure for a multi-stage intrusion where an attacker performs reconnaissance, brute forces SSH credentials, gains unauthorized access, and establishes persistence on a Linux endpoint. It is designed to be followed in real time during the incident and maps each step to the NIST 800-61 lifecycle and relevant MITRE ATT&CK techniques.

---

## Attack Stages This Playbook Addresses

| Stage | Attacker Action | MITRE ATT&CK |
|-------|----------------|--------------|
| 1 | Network reconnaissance / port scan | T1046 Network Service Discovery |
| 2 | SSH brute force | T1110.001 Password Guessing |
| 3 | Unauthorized login / command execution | T1078 Valid Accounts, T1059 Command Execution |
| 4 | Persistence (new account or cron job) | T1136 Create Account, T1053.003 Cron |

---

## Phase 2 — Detection & Analysis

### Step 1: Validate the Alert
- Confirm the triggering alert in Splunk (SSH brute force, scan, or new account)
- Determine whether this is a true positive or expected lab activity
- Note the alert timestamp as the initial detection time (for MTTD)

### Step 2: Identify the Source and Target
Run in Splunk to confirm the attacker IP and targeted host:
```spl
index=main host="raspberrypi" "Failed password"
| rex field=_raw "from (?<src_ip>\d+\.\d+\.\d+\.\d+)"
| stats count by src_ip
| sort - count
```
- Record the top source IP (the attacker)
- Confirm the destination host

### Step 3: Determine Scope of Compromise
Check whether the brute force succeeded:
```spl
index=main host="raspberrypi" "Accepted password"
| rex field=_raw "from (?<src_ip>\d+\.\d+\.\d+\.\d+)"
| table _time user src_ip
```
- A successful login from the attacker IP confirms compromise → escalate severity to High/Critical

### Step 4: Look for Post-Access Activity
Check for command execution and privilege use after the login:
```spl
index=main host="raspberrypi" (sudo OR "session opened" OR useradd OR crontab)
| table _time _raw
```
- Evidence of account creation or cron modification confirms persistence → Critical

### Step 5: Classify Severity
Apply the severity table from the IR Plan:
- Failed attempts only → Medium
- Successful login → High
- Login + persistence → Critical

### Step 6: Preserve Evidence
- Export the relevant Splunk searches as evidence
- Note all timestamps in a consistent timezone
- Capture packet evidence in Wireshark if the attack is ongoing
- Begin the incident timeline document

---

## Phase 3 — Containment, Eradication & Recovery

### Containment (Short-Term)
Goal: stop the attacker from continuing while preserving evidence.

- Block the attacker source IP at the firewall:
```bash
sudo ufw deny from <ATTACKER-IP>
```
- If active session suspected, identify it:
```bash
who
last -a | head
```
- Consider isolating the host from the network if compromise is confirmed and spreading is a risk
- Preserve volatile data (active connections, running processes) before changes:
```bash
ss -tnp
ps aux --sort=-%cpu | head
```

### Eradication
Goal: remove the attacker's foothold completely.

- Identify and remove unauthorized accounts:
```bash
# Review recently created or suspicious accounts
cat /etc/passwd | sort -t: -k3 -n | tail
# Remove a malicious account
sudo userdel -r <MALICIOUS-USER>
```
- Identify and remove unauthorized persistence:
```bash
# Check all users' cron jobs
sudo crontab -l
for u in $(cut -f1 -d: /etc/passwd); do sudo crontab -l -u $u 2>/dev/null; done
# Check system cron locations
ls -la /etc/cron.* /etc/crontab
# Remove malicious cron entry
sudo crontab -r -u <USER>   # or edit the specific entry
```
- Verify no additional footholds (SSH keys, modified services):
```bash
cat /home/*/.ssh/authorized_keys 2>/dev/null
sudo systemctl list-unit-files --state=enabled
```

### Recovery
Goal: restore normal, secure operation.

- Reset credentials for the compromised account:
```bash
sudo passwd <USER>
```
- Harden SSH to prevent recurrence:
```bash
# In /etc/ssh/sshd_config:
# PermitRootLogin no
# PasswordAuthentication no   (key-based only)
sudo systemctl restart ssh
```
- Re-enable any services that were isolated
- Confirm the system is functioning normally
- Increase monitoring on the host temporarily (watch for re-compromise)

---

## Phase 4 — Post-Incident Activity

### Step 1: Calculate Metrics
| Metric | Definition |
|--------|-----------|
| MTTD (Mean Time to Detect) | Attack start → alert fired |
| MTTR (Mean Time to Respond) | Alert fired → containment complete |
| Dwell Time | Initial access → eradication complete |

### Step 2: Lessons Learned Review
Document and answer:
- How did the attacker get in?
- What detection worked? What was missed?
- How quickly was it contained?
- What would have prevented or limited this?

### Step 3: Improve Defenses
- Update or add Splunk detection rules for any gaps found
- Update this playbook with anything that was unclear during execution
- Apply hardening permanently (SSH config, account policy, fail2ban)

### Step 4: Produce the Incident Report
- Write the formal after-action report
- Map all observed techniques to MITRE ATT&CK
- Attach evidence (Splunk exports, screenshots, timeline)

---

## Quick Reference — Response Checklist

- [ ] Alert validated as true positive
- [ ] Attacker IP and target host identified
- [ ] Scope determined (did brute force succeed?)
- [ ] Persistence checked (accounts, cron, keys)
- [ ] Severity classified
- [ ] Evidence preserved
- [ ] Attacker IP blocked (containment)
- [ ] Malicious accounts removed (eradication)
- [ ] Persistence removed (eradication)
- [ ] Credentials reset, SSH hardened (recovery)
- [ ] System confirmed normal
- [ ] Metrics calculated (MTTD, MTTR, dwell time)
- [ ] Lessons learned documented
- [ ] Incident report produced

---

*Part of a home security operations lab built to demonstrate hands-on incident response capability.*
