# Incident Report — IR-2026-004
## Multi-Stage SSH Intrusion with Persistence

| Field | Value |
|-------|-------|
| Report ID | IR-2026-004 |
| Date | June 2026 |
| Severity | Critical |
| Status | Resolved |
| Framework | NIST SP 800-61 Rev. 2 |
| Analyst | Tyrik Parker |
| Detection Source | Splunk SIEM |

---

## Executive Summary

A multi-stage intrusion was conducted against a monitored Linux endpoint (Raspberry Pi 5) in the security operations lab and successfully detected, contained, and eradicated using the established incident response process. The attacker performed network reconnaissance, brute forced SSH credentials against a user account, gained unauthorized shell access, executed enumeration commands, and established persistence via a cron job.

The Splunk SIEM detected each stage of the attack. The compromise was identified approximately 12 minutes after initial access, contained within 2 minutes of detection, and fully eradicated within approximately 25 minutes of the breach. The exploited weaknesses — weak password and password-based SSH authentication — were remediated during recovery.

This incident was executed as a controlled exercise to validate the lab's detection and response capability end to end.

---

## Incident Classification

| Attribute | Value |
|-----------|-------|
| Incident Type | Unauthorized access via credential brute force |
| Attack Vector | SSH (port 22) |
| Severity | Critical (confirmed access + persistence) |
| Affected System | Raspberry Pi 5 (monitored Linux endpoint) |
| Compromised Account | labtest (test account) |
| Outcome | Contained and eradicated, no lateral movement |

---

## Incident Timeline

| Time | Event | Phase |
|------|-------|-------|
| 11:34:39 | Attack initiated — reconnaissance and brute force begin | Attack |
| 11:34–11:52 | SSH brute force in progress (repeated failed logins) | Attack |
| 11:52:44 | Initial access — attacker authenticates as labtest | Attack |
| 11:52–11:55 | Enumeration commands executed (whoami, id, uname, etc.) | Attack |
| ~11:56 | Persistence established (cron job created) | Attack |
| ~12:05 | Compromise detected in Splunk (Accepted password event) | Detection |
| ~12:06 | Incident scoped — attacker IP, account, and persistence identified | Analysis |
| ~12:07 | Containment — attacker IP blocked via UFW, session terminated | Containment |
| ~12:10 | Eradication begins — cron persistence removed | Eradication |
| ~12:17 | Eradication complete — labtest account removed | Eradication |
| ~12:20 | Recovery — SSH hardened, fail2ban deployed | Recovery |

---

## Response Metrics

| Metric | Value | Description |
|--------|-------|-------------|
| Time to Compromise | ~18 minutes | Attack start to successful unauthorized access |
| Mean Time to Detect (MTTD) | ~12 minutes | Initial access to detection in SIEM |
| Mean Time to Respond (MTTR) | ~2 minutes | Detection to containment start |
| Containment to Eradication | ~10 minutes | Containment to full foothold removal |
| Total Dwell Time | ~25 minutes | Initial access to eradication complete |

A 2-minute MTTR reflects an efficient response once the compromise was identified. The ~12-minute MTTD represents the primary opportunity for improvement and is addressed in the recommendations.

---

## Attack Analysis (Stage by Stage)

### Stage 1 — Reconnaissance
The attacker performed an Nmap service scan against the endpoint to identify open ports and running services. The scan generated a high volume of UFW firewall block events from a single source IP, detected in Splunk.
- **MITRE ATT&CK:** T1046 — Network Service Discovery

### Stage 2 — Credential Brute Force
Using Hydra, the attacker conducted a dictionary attack against SSH for the labtest account. This produced a large number of failed password events in auth.log, all originating from the same source IP. The attack succeeded against the weak password.
- **MITRE ATT&CK:** T1110.001 — Brute Force: Password Guessing

### Stage 3 — Unauthorized Access & Execution
The attacker authenticated with the cracked credentials and ran enumeration commands to understand the system and identify privilege escalation paths. Splunk recorded the SSH session being opened for labtest.
- **MITRE ATT&CK:** T1078 — Valid Accounts; T1059 — Command and Scripting Interpreter

### Stage 4 — Persistence
The attacker created a cron job to maintain access independent of the password. This persistence mechanism was detected in the cron logs.
- **MITRE ATT&CK:** T1053.003 — Scheduled Task/Job: Cron

---

## MITRE ATT&CK Summary

| Tactic | Technique | ID |
|--------|-----------|-----|
| Discovery | Network Service Discovery | T1046 |
| Credential Access | Brute Force: Password Guessing | T1110.001 |
| Initial Access / Defense Evasion | Valid Accounts | T1078 |
| Execution | Command and Scripting Interpreter | T1059 |
| Persistence | Scheduled Task/Job: Cron | T1053.003 |

---

## Detection Detail

The following SIEM detections fired or were used to scope the incident:

| Detection | Source | Purpose |
|-----------|--------|---------|
| UFW block spike from single IP | kern.log | Identified reconnaissance scan |
| Failed password volume | auth.log | Identified brute force in progress |
| Accepted password event | auth.log | Confirmed successful compromise |
| SSH session opened | auth.log | Confirmed unauthorized access |
| Cron modification | cron logs | Confirmed persistence |

The full attack chain was reconstructed in Splunk using a consolidated timeline search across all relevant event types.

---

## Response Actions Taken

### Containment
- Preserved volatile evidence (active sessions, connections, processes) before making changes
- Blocked the attacker source IP at the host firewall (UFW)
- Terminated the active attacker session

### Eradication
- Removed the cron job persistence mechanism
- Verified no additional persistence (SSH keys, other cron locations, rogue accounts)
- Removed the compromised labtest account entirely

### Recovery
- Hardened SSH configuration (disabled root login, reduced MaxAuthTries)
- Deployed fail2ban for automated brute force protection
- Verified services healthy and SIEM logging intact
- Continued elevated monitoring on the endpoint

---

## Root Cause Analysis

The incident was made possible by two weaknesses:

1. **Weak password** — the labtest account used a password present in common wordlists, allowing a successful dictionary attack.
2. **Password-based SSH authentication** — SSH accepted password authentication, exposing the service to brute force.

The host firewall (UFW) was active and logging, which provided the detection signal for the reconnaissance phase. Logging was correctly forwarding to the SIEM, which enabled full reconstruction of the attack.

---

## Lessons Learned

**What worked:**
- The SIEM detected every stage of the attack, from recon through persistence.
- Logging was correctly configured and forwarding, enabling complete incident reconstruction.
- Once detected, containment and eradication were fast and followed the playbook cleanly.

**What could be improved:**
- Detection latency (~12 minutes) was driven by manual review. An automated real-time alert on "Accepted password" from an IP with prior failed attempts would shorten MTTD significantly.
- The successful login itself should generate an immediate high-priority alert, not rely on analyst review.
- A correlation rule linking failed-then-successful logins from the same IP would have flagged the compromise at the moment it occurred.

---

## Recommendations

| Priority | Recommendation |
|----------|---------------|
| Critical | Enforce key-based SSH authentication; disable password auth where feasible |
| Critical | Enforce strong password policy on all accounts |
| High | Create a real-time Splunk alert for successful login following failed attempts from the same IP |
| High | Maintain fail2ban across all SSH-exposed endpoints |
| Medium | Integrate this detection with the Shuffle SOAR workflow for automated enrichment and notification |
| Medium | Implement least-privilege review to limit post-access options |
| Low | Schedule periodic credential audits |

---

## Conclusion

The lab successfully detected and responded to a complete multi-stage intrusion following the NIST 800-61 lifecycle. The attacker was identified, contained, and eradicated with no lateral movement or lasting foothold, and the exploited weaknesses were remediated. The primary improvement opportunity is reducing detection time through automated real-time alerting on successful compromise indicators — a change that would convert this from a manually-detected incident into an automatically-flagged one.

This exercise validated the end-to-end incident response capability of the lab: preparation (plan and playbook), detection and analysis (SIEM), containment, eradication, recovery, and post-incident review.

---

**Analyst:** Tyrik Parker
**Certification:** CompTIA CySA+
**Framework:** NIST SP 800-61 Rev. 2
