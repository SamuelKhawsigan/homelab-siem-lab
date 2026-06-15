# 07 — Detection: Wazuh Rules + Dashboard Analysis

## Overview

This document covers the detection results from the attack simulation in Phase 6. It details which Wazuh rules fired, the alert levels, and how to read the dashboard evidence. The key outcome: Wazuh correctly identified an SSH brute force attack in real time and escalated from individual event logging to confirmed attack pattern detection.

---

## How Wazuh Detection Works

Wazuh uses a rule engine that evaluates incoming log events against a library of built-in and custom rules. Rules have levels from 0–15:

| Level | Severity | Meaning |
|---|---|---|
| 0–3 | Low | Informational, normal activity |
| 4–7 | Medium | Suspicious activity worth noting |
| 8–11 | High | Likely attack or policy violation |
| 12–15 | Critical | Confirmed attack, immediate action needed |

For SSH brute force specifically, Wazuh uses a **frequency-based escalation** approach:

1. Single failed SSH login → Rule 5710, Level 5
2. Multiple failures from same IP → Rule 5720, Level 8
3. Sustained failures exceeding threshold → Rule 5763, Level 10 ("Multiple authentication failures")

Rule 5763 is the key rule — it's what triggers active response in Phase 8.

---

## Detection Evidence

### Alert Summary

| Attack Phase | Total Alerts | Auth Failures | Highest Level | Key Rules |
|---|---|---|---|---|
| Small wordlist (10 passwords) | 16 | 16 | 8 | 5710, 5720 |
| rockyou.txt (sustained) | 56 | 56 | 10 | 5710, 5720, 5763 |

### Rule Groups Fired

| Rule Group | Description |
|---|---|
| `syslog` | Raw syslog events from SSH daemon |
| `sshd` | SSH-specific rules |
| `authentication_failed` | Failed authentication correlation |
| `pam` | PAM authentication module events |
| `access_control` | Access control policy events |
| `attacks` | Confirmed attack pattern (Level 10+) |

The progression from `authentication_failed` to `attacks` is the detection escalation — Wazuh moved from logging individual failures to classifying the activity as an active attack.

### Timeline

The Wazuh dashboard timeline shows a clear spike at the exact moment each Hydra run started:

- **16:46** — First Hydra run (small wordlist) — 16 alerts in ~10 seconds
- **16:59** — Second Hydra run (rockyou.txt) — 56 alerts, Level 10 reached

Zero alerts before the attack. Immediate spike during the attack. Clean return to zero after Hydra stopped. This is textbook SIEM detection behaviour.

---

## Key Rules

### Rule 5710 — SSH Authentication Failed
```
Level: 5
Group: syslog, sshd, authentication_failed
Description: sshd: authentication failed
```
Fires on every individual failed SSH login attempt.

### Rule 5720 — SSH Multiple Authentication Failures
```
Level: 8  
Group: syslog, sshd, authentication_failed, access_control
Description: sshd: Maximum authentication attempts exceeded
```
Fires when a single connection exceeds the SSH MaxAuthTries limit.

### Rule 5763 — Multiple Authentication Failures (Brute Force)
```
Level: 10
Group: syslog, sshd, authentication_failures, attacks
Description: sshd: brute force trying to get access to the system
Frequency: 8 events in 120 seconds from same source IP
```
This is the brute force correlation rule. It fires when Wazuh observes 8+ failed logins from the same source IP within a 2-minute window — confirming a brute force pattern rather than a user mistyping their password.

---

## Reading the Dashboard

### Threat Hunting View

Filters used:
- `manager.name: wazuh`
- `agent.id: 001` (ubuntu-victim)
- Time range: Last 10 minutes

Key columns:
- **timestamp** — when the event was logged
- **agent.name** — which host generated the event
- **rule.description** — human-readable alert description
- **rule.level** — severity (higher = more serious)
- **rule.id** — specific rule that fired

### Top 5 Alerts (rockyou.txt phase)
1. sshd: authentication failed
2. PAM: User login failed
3. Maximum authentication failures (brute force)
4. syslog: User authentication failed
5. syslog: User missed the login prompt

### Top 5 Rule Groups
1. syslog
2. authentication_failed
3. sshd
4. pam
5. access_control

### PCI DSS Mapping
Wazuh automatically maps alerts to compliance frameworks. The brute force alerts mapped to:
- **PCI DSS 10.2.4** — Invalid logical access attempts
- **PCI DSS 10.2.5** — Use of identification and authentication mechanisms
- **PCI DSS 11.4** — Use of intrusion detection techniques

This compliance mapping is valuable for organisations that need to demonstrate their SIEM covers regulatory requirements.

---

## Screenshots

| # | Description | File |
|---|---|---|
| 1 | Wazuh dashboard — first detection (16 alerts) | `screenshots/06-wazuh-brute-force-detected.png` |
| 2 | Wazuh dashboard — Level 10 escalation (56 alerts) | `screenshots/06-wazuh-rockyou-detected.png` |
| 3 | Wazuh active response alerts (Rule 651/652) | `screenshots/08-wazuh-active-response-alert.png` |
