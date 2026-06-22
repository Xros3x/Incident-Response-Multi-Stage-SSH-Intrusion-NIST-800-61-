# Incident Response Plan
## Home Security Operations Lab

| Field | Value |
|-------|-------|
| Document Owner | Tyrik Parker |
| Version | 1.0 |
| Framework | NIST SP 800-61 Rev. 2 |
| Scope | Home SOC Lab Environment |
| Last Updated | June 2026 |

---

## 1. Purpose

This Incident Response Plan (IRP) defines how security incidents are detected, analyzed, contained, eradicated, and recovered from within the home security operations lab. It is modeled on the NIST SP 800-61 incident response lifecycle and is intended to demonstrate a repeatable, documented response capability for the lab environment.

The plan applies to all systems within the lab, including the Splunk SIEM, the Shuffle SOAR platform, monitored endpoints, and the network devices that connect them.

---

## 2. Scope

This plan covers security incidents affecting the following lab assets:

| Asset | Role | Monitoring |
|-------|------|-----------|
| Splunk Enterprise (Ubuntu VM) | SIEM / log aggregation | Self-monitored |
| Shuffle SOAR (Ubuntu VM) | Automated response | Workflow logs |
| Raspberry Pi 5 | Monitored Linux endpoint (victim) | UFW + Universal Forwarder |
| Windows Server 2022 (VM) | Monitored Windows endpoint | Windows Event Forwarding |
| Kali Linux (VM) | Authorized attack simulation host | N/A (controlled) |
| Proxmox VE | Hypervisor | Host logs |

Out of scope: production systems, third-party services, and any system outside the isolated lab network.

---

## 3. Incident Response Team (Lab Roles)

In this single-operator lab, one analyst performs all roles. In a production environment these would be distinct people; they are documented here to show understanding of the structure.

| Role | Responsibility |
|------|---------------|
| Incident Commander | Owns the incident, makes containment decisions |
| Analyst / Investigator | Performs detection, triage, and forensic analysis |
| Remediation Engineer | Executes containment, eradication, and recovery |
| Scribe | Documents the timeline and actions taken |

---

## 4. Incident Severity Classification

Incidents are classified to prioritize response effort.

| Severity | Definition | Example | Target Response |
|----------|-----------|---------|-----------------|
| Critical | Active compromise with confirmed attacker control or data loss | Attacker gained shell + persistence | Immediate |
| High | Confirmed intrusion attempt with partial success | Successful brute force login | Within 1 hour |
| Medium | Suspicious activity indicating targeting | Port scan + repeated failed logins | Within 4 hours |
| Low | Isolated anomaly with no confirmed impact | Single failed login | Same day |

---

## 5. The NIST 800-61 Incident Response Lifecycle

### Phase 1 — Preparation
Maintain readiness before an incident occurs.
- SIEM (Splunk) actively collecting endpoint and network logs
- SOAR (Shuffle) workflows configured for automated enrichment and alerting
- Detection rules and alerts defined for known attack patterns
- This IR plan and associated playbooks documented and accessible
- Baseline of normal activity established for comparison

### Phase 2 — Detection & Analysis
Identify that an incident is occurring and determine its scope.
- Alert fires in Splunk or notification arrives via Shuffle
- Analyst validates the alert is a true positive
- Determine affected systems, attack type, and entry point
- Classify severity using the table in Section 4
- Establish an incident timeline
- Preserve relevant logs and evidence

### Phase 3 — Containment, Eradication & Recovery
Limit damage, remove the threat, and restore normal operations.

Containment (short-term):
- Isolate the affected host from the network where appropriate
- Block the attacker source where feasible
- Preserve volatile evidence before making changes

Eradication:
- Remove attacker artifacts (malicious accounts, persistence mechanisms)
- Close the exploited vulnerability or misconfiguration
- Verify no additional footholds remain

Recovery:
- Restore the system to a known-good state
- Re-enable services and confirm normal operation
- Increase monitoring on the affected system temporarily

### Phase 4 — Post-Incident Activity
Learn from the incident to improve future response.
- Conduct a lessons-learned review
- Calculate response metrics (MTTD, MTTR, dwell time)
- Update detection rules and playbooks based on gaps found
- Produce the final incident report
- Map observed techniques to MITRE ATT&CK

---

## 6. Detection Sources

| Source | What It Detects |
|--------|----------------|
| Splunk SIEM | Failed logins, firewall blocks, Windows event anomalies |
| UFW firewall logs (Pi) | Port scans, blocked connection attempts |
| auth.log (Pi) | SSH authentication activity |
| Windows Security log | Failed logons, account creation, privilege use |
| Shuffle SOAR | Automated alert enrichment and notification |

---

## 7. Communication & Escalation

In the lab, escalation is documented as a process rather than involving multiple parties.

| Condition | Action |
|-----------|--------|
| Severity Critical or High | Document immediately, begin containment |
| Confirmed persistence | Escalate to full eradication workflow |
| Evidence of data exfiltration | Treat as Critical, preserve all evidence |

---

## 8. Evidence Handling

- Preserve logs before modifying any affected system
- Record timestamps in a consistent timezone
- Export relevant Splunk search results as evidence
- Capture packet evidence with Wireshark where applicable
- Maintain a chain-of-custody style record of what was collected and when

---

## 9. Tools Referenced

| Tool | Use in IR |
|------|-----------|
| Splunk Enterprise | Detection, log analysis, evidence export |
| Shuffle SOAR | Automated enrichment and notification |
| Wireshark | Packet-level evidence |
| AbuseIPDB / VirusTotal | Threat intelligence enrichment |
| UFW / iptables | Containment (blocking) |
| Native OS tools | Eradication (account/persistence removal) |

---

## 10. Plan Maintenance

This plan is reviewed and updated after each incident and whenever the lab environment changes significantly. Lessons learned from executed incidents are incorporated into both the plan and the associated playbooks.

---

*This document is part of a home security operations lab built to demonstrate hands-on SOC and incident response capability.*
