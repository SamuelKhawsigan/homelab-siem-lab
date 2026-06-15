# 08 — Defense: Active Response + Automated Blocking

## Overview

This document covers enabling Wazuh's active response capability to automatically block attackers when a brute force is detected. After this phase, the SIEM moves from passive detection to active defense — blocking the attacker's IP via iptables without any human intervention.

---

## How Wazuh Active Response Works

Wazuh active response is a built-in capability that executes scripts on monitored agents when specific rules fire. The flow is:

```
Attack detected → Rule fires → Manager sends command → Agent executes script → Attacker blocked
```

The `firewall-drop` script adds an iptables DROP rule on the victim for the attacker's source IP. After a configurable timeout, the block is automatically lifted.

---

## Step 1 — Configure Active Response on the Manager

Edit the Wazuh manager configuration:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add the following block inside `<ossec_config>`:

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5763</rules_id>
  <timeout>180</timeout>
</active-response>
```

| Parameter | Value | Meaning |
|---|---|---|
| `command` | firewall-drop | Script to execute on the agent |
| `location` | local | Run on the agent where the alert fired |
| `rules_id` | 5763 | Trigger on "Multiple authentication failures" |
| `timeout` | 180 | Block for 180 seconds then auto-unblock |

Rule 5763 is Wazuh's built-in brute force detection rule — it fires when multiple SSH authentication failures are detected from the same source IP.

Restart the manager to apply:

```bash
sudo systemctl restart wazuh-manager
```

---

## Step 2 — Verify Active Response Script on Victim

The `firewall-drop` script must exist on the agent:

```bash
ls /var/ossec/active-response/bin/
```

Confirm `firewall-drop` is in the list. It is installed automatically with the Wazuh agent.

---

## Step 3 — Re-run the Attack

From Kali, run the brute force again:

```bash
hydra -l sammy -P /usr/share/wordlists/rockyou.txt ssh://10.10.10.121 -t 4 -V
```

Simultaneously, monitor iptables on the victim in real time:

```bash
sudo watch -n 1 iptables -L -n
```

---

## Result — Automated Block

Within seconds of the brute force starting, Wazuh detected the pattern and executed `firewall-drop`. The iptables output on the victim showed:

```
Chain INPUT (policy ACCEPT)
target     prot opt source          destination
DROP       0    --  10.10.10.138    0.0.0.0/0

Chain FORWARD (policy ACCEPT)
target     prot opt source          destination
DROP       0    --  10.10.10.138    0.0.0.0/0
```

Kali's IP (`10.10.10.138`) was automatically added to the DROP chain. All further packets from the attacker were silently discarded. Hydra's attempts stalled — no connection refused, no response, just silence. This is intentional: a DROP rule gives the attacker no information about whether they've been blocked.

---

## Wazuh Active Response Alerts

The active response cycle is logged in Wazuh and visible in the dashboard:

| Rule ID | Description | Time |
|---|---|---|
| 651 | Host Blocked by firewall-drop Active Response | 17:11:13 |
| 652 | Host Unblocked by firewall-drop Active Response | 17:14:15 |
| 651 | Host Blocked by firewall-drop Active Response | 17:14:25 |
| 652 | Host Unblocked by firewall-drop Active Response | 17:17:25 |
| 651 | Host Blocked by firewall-drop Active Response | 17:17:33 |

The pattern shows the full defend loop:
1. Attack starts → Wazuh detects → blocks attacker (Rule 651)
2. 180 seconds pass → block lifted (Rule 652)
3. Attacker resumes → Wazuh detects again → blocks again (Rule 651)
4. Cycle repeats automatically with no human intervention

---

## Complete Attack-Detect-Respond Timeline

| Time | Event |
|---|---|
| 16:46 | Hydra small wordlist — 16 auth failures detected |
| 16:59 | Hydra rockyou — Level 10 brute force alert |
| 17:11 | Active response enabled — Kali blocked automatically |
| 17:14 | Block expired — Kali unblocked |
| 17:14 | Kali resumed attack — blocked again immediately |
| 17:17 | Block expired — Kali unblocked |
| 17:17 | Kali resumed attack — blocked again |

---

## Key Takeaways

1. **Zero human intervention.** From detection to block, the entire response was automated. The SIEM identified the threat and took action without an analyst needing to do anything.

2. **The attacker cannot tell they're blocked.** DROP (silently discard) vs REJECT (send back an error) — DROP gives the attacker no signal, making it harder for them to adapt.

3. **Auto-unblock prevents permanent lockout.** The 180-second timeout means a legitimate user who triggers false positives won't be permanently locked out. In production, this timeout would be tuned based on the environment.

4. **The loop is self-sustaining.** As long as the attacker keeps attacking, Wazuh keeps blocking. Each resumed attack immediately re-triggers the detection and response cycle.

---

## Screenshots

| # | Description | File |
|---|---|---|
| 1 | iptables showing Kali IP in DROP chain | `screenshots/08-active-response-block.png` |
| 2 | Hydra output stalled after block | `screenshots/08-hydra-blocked.png` |
| 3 | Wazuh dashboard showing Rule 651/652 cycle | `screenshots/08-wazuh-active-response-alert.png` |
