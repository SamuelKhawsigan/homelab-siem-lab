# 06 — Attack Simulation

## Overview

This document covers the attack simulation phase using Kali Linux as the attacker against the Ubuntu victim VM. Two ATT&CK-mapped techniques were simulated: network reconnaissance (T1046) and SSH brute force (T1110.001). All attacks were conducted within the isolated lab network (`10.10.10.0/24`) with no impact on the office network.

---

## Attacker VM — Kali Linux

| Setting | Value |
|---|---|
| VM ID | 103 |
| Name | kali-attacker |
| OS | Kali Linux (live ISO, no install) |
| RAM | 3072MB |
| Network | vmbr1 only (lab network) |
| IP | 10.10.10.138/24 (DHCP from OPNsense) |

### Setup

Kali was booted from the live ISO. SSH was started manually to allow remote access via jump host:

```bash
sudo systemctl start ssh
```

Remote access from MSI laptop via Proxmox jump host:

```bash
ssh -J root@100.70.88.30 kali@10.10.10.138
```

---

## Attack 1 — Network Service Discovery (T1046)

### Objective
Identify open ports and services on the victim before attacking.

### Tool
Nmap 7.98

### Commands

**Initial SYN port scan:**
```bash
nmap -sS -p 22,80,443,3389,8080 10.10.10.121
```

**Service version and OS fingerprinting:**
```bash
nmap -sV -O 10.10.10.121
```

### Output

```
PORT    STATE  SERVICE  VERSION
22/tcp  open   ssh      OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)

Device type: general purpose
OS details: Linux 4.15 - 5.19
Network Distance: 1 hop
```

### Finding
Only port 22 (SSH) is open. The victim is running OpenSSH 9.6p1 on Ubuntu. This narrows the attack surface to SSH — the only available entry point.

---

## Attack 2 — SSH Brute Force (T1110.001)

### Objective
Attempt to gain unauthorized access by guessing the SSH password for user `sammy`.

### Tool
Hydra v9.6

### Phase 1 — Small wordlist (10 passwords)

Create a custom wordlist:

```bash
cat > /tmp/passwords.txt << EOF
password
123456
admin
root
sammy
letmein
welcome
qwerty
password123
toor
