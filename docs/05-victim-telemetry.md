# 05 — Victim Setup + Wazuh Agent Telemetry

## Overview

This document covers enrolling the Ubuntu victim VM as a Wazuh agent and configuring auditd for rich telemetry. After this phase, every command execution, privilege escalation, file access, and network connection on the victim is logged and visible in the Wazuh dashboard.

---

## Step 1 — Deploy the Wazuh Agent

In the Wazuh dashboard, navigate to **Agents** → **Deploy new agent** and fill in:

- **Package:** DEB amd64 (Ubuntu uses Debian packages)
- **Server address:** `10.10.10.161`
- **Agent name:** `ubuntu-victim`

The dashboard generates the install command:

```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.5-1_amd64.deb && \
sudo WAZUH_MANAGER='10.10.10.161' WAZUH_AGENT_NAME='ubuntu-victim' dpkg -i ./wazuh-agent_4.14.5-1_amd64.deb
```

Run this on the Ubuntu victim VM.

> **Note:** Fix DNS before running wget — OPNsense DNS may not be responding. Set `nameserver 8.8.8.8` in `/etc/resolv.conf` temporarily.

---

## Step 2 — Start and Enable the Agent

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
sudo systemctl status wazuh-agent
```

Expected output shows `active (running)`.

The agent connects to the Wazuh manager at `10.10.10.161` on port 1514 and registers itself. Within 30–60 seconds the agent appears in the Wazuh dashboard as **Active**.

---

## Step 3 — Install auditd

The Linux Audit daemon (`auditd`) captures detailed system-level events — command executions, file access, privilege escalation, network connections. Wazuh reads auditd logs natively and maps them to MITRE ATT&CK techniques.

```bash
sudo apt-get install -y auditd audispd-plugins
sudo systemctl enable auditd
sudo systemctl start auditd
```

---

## Step 4 — Configure Audit Rules

Add rules to capture the most relevant events for attack detection:

```bash
sudo bash -c 'cat >> /etc/audit/rules.d/audit.rules << EOF
# Log all command executions (T1059)
-a always,exit -F arch=b64 -S execve -k command_execution

# Log privilege escalation attempts (T1548)
-w /etc/sudoers -p wa -k sudoers_changes

# Log credential file access (T1003)
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k shadow_changes

# Log SSH authentication activity (T1110)
-w /var/log/auth.log -p wa -k auth_log

# Log outbound network connections (T1071)
-a always,exit -F arch=b64 -S connect -k network_connect
EOF'
```

Reload and apply the rules:

```bash
sudo augenrules --load
sudo systemctl restart auditd
```

Verify rules are active:

```bash
sudo auditctl -l
```

---

## Verification

### Agent Status in Dashboard

In the Wazuh dashboard → **Agents**, `ubuntu-victim` appears with status **Active**, showing the agent IP (`10.10.10.121`), OS (Ubuntu 24.04), and Wazuh agent version.

### Events Flowing

In the Wazuh dashboard → **Threat Hunting**, filter by agent `ubuntu-victim` and set time range to last 15 minutes. After running a few sudo commands on the victim, events appear immediately:

| Rule ID | Description | Trigger |
|---|---|---|
| 5402 | Successful sudo to ROOT executed | `sudo` commands |
| 5501 | PAM: Login session opened | SSH login |
| 5502 | PAM: Login session closed | SSH logout |

318 events were observed within the first few minutes of agent enrollment, confirming the SIEM is receiving and processing telemetry correctly.

---

## ATT&CK Coverage from auditd

| Rule Key | ATT&CK Technique | Description |
|---|---|---|
| `command_execution` | T1059.004 | Unix Shell command execution |
| `sudoers_changes` | T1548.003 | Sudo and Sudo Caching |
| `passwd_changes` | T1003 | OS Credential Dumping |
| `shadow_changes` | T1003 | OS Credential Dumping |
| `auth_log` | T1110 | Brute Force |
| `network_connect` | T1071 | Application Layer Protocol |

---

## Screenshots

| # | Description | File |
|---|---|---|
| 1 | Wazuh agent install command from dashboard | `screenshots/05-agent-deploy-command.png` |
| 2 | Agent service running on ubuntu-victim | `screenshots/05-wazuh-agent-running.png` |
| 3 | Wazuh dashboard showing ubuntu-victim Active | `screenshots/05-agent-enrolled.png` |
| 4 | Agent detail page in dashboard | `screenshots/05-agent-detail.png` |
| 5 | auditd rules loaded on victim | `screenshots/05-audit-rules.png` |
| 6 | Wazuh Threat Hunting showing 318 events | `screenshots/05-events-flowing.png` |
