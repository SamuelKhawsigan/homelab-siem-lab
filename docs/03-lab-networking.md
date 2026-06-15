# 03 — Lab Networking: Isolated Bridge + OPNsense Gateway

## Overview

This document covers the setup of an isolated lab network inside Proxmox using a internal-only bridge (`vmbr1`) and an OPNsense VM acting as the gateway, firewall, and DHCP server. After this phase, all lab VMs live on an isolated `10.10.10.0/24` network that can reach the internet through OPNsense NAT but cannot access the office LAN.

---

## Network Architecture

```
Internet
    |
Office LAN (10.8.9.0/24)
    |
OPNsense WAN (vtnet0) — 10.8.9.79/24 via DHCP
    |
OPNsense LAN (vtnet1) — 10.10.10.1/24
    |
vmbr1 (isolated internal bridge)
    |
Lab VMs (10.10.10.100–200 DHCP pool)
```

---

## Step 1 — Create the Isolated Bridge (`vmbr1`)

In the Proxmox web UI:

1. Node → **System** → **Network** → **Create** → **Linux Bridge**
2. Fill in:
   - **Name:** `vmbr1`
   - **IPv4/CIDR:** leave blank
   - **Gateway:** leave blank
   - **Bridge ports:** leave **empty** ← no physical NIC attached
   - **Autostart:** checked
3. Click **Create** → **Apply Configuration**

The empty Bridge ports field is the key — it means `vmbr1` has no path to the physical network. Traffic can only leave the lab through OPNsense, which we control.

---

## Step 2 — Deploy OPNsense VM

### VM Configuration

| Setting | Value |
|---|---|
| VM ID | 100 |
| Name | opnsense |
| ISO | OPNsense 26.1.6_2 amd64 dvd |
| Disk | 20GB VirtIO Block |
| CPU | 2 cores, host type |
| RAM | 3072MB (during install), 2048MB (after) |
| net0 | vmbr0 — WAN (office LAN) |
| net1 | vmbr1 — LAN (isolated lab) |

> **Note:** RAM was temporarily set to 3072MB during install. The OPNsense live image requires minimum 3GB RAM to copy the filesystem to disk. After installation completed, RAM was reduced back to 2048MB.

### Interface Assignment

At the OPNsense console prompt:

```
LAGGs? N
VLANs? N
WAN interface: vtnet0
LAN interface: vtnet1
Optional: (blank)
Proceed? y
```

### Install to Disk

- Keymap: default (US)
- Filesystem: UFS
- Target disk: vtbd0 (20GB virtual disk)
- Set root password
- Complete Install → reboot

Remove the ISO from the VM hardware before reboot to prevent booting back into the installer.

---

## Step 3 — Configure LAN Interface

After first boot, log in as root at the OPNsense console and select option **2** (Set interface IP address):

- Interface: **1 (LAN)**
- Configure via DHCP: **N**
- IPv4 address: `10.10.10.1`
- Subnet bits: `24`
- Upstream gateway: (blank)
- Configure IPv6: **N**
- Enable DHCP server: **Y**
- DHCP start: `10.10.10.100`
- DHCP end: `10.10.10.200`

OPNsense LAN is now `10.10.10.1/24` with a DHCP pool ready for lab VMs.

---

## Step 4 — Access OPNsense Web GUI

The OPNsense web GUI is only accessible from inside the lab network (`10.10.10.0/24`). To access it from the MSI laptop, SSH port forwarding is used via Proxmox as a jump host.

First, add Proxmox to the lab network temporarily:

```bash
# On Proxmox shell
ip addr add 10.10.10.254/24 dev vmbr1
```

Then from the MSI laptop (WSL):

```bash
ssh -L 8443:10.10.10.1:443 -J root@100.70.88.30 sammy@10.10.10.121
```

Access the GUI at `https://localhost:8443` and log in as root.

---

## Step 5 — Deploy Ubuntu Victim VM

### VM Configuration

| Setting | Value |
|---|---|
| VM ID | 101 |
| Name | ubuntu-victim |
| ISO | Ubuntu Server 24.04.1 LTS |
| Disk | 32GB VirtIO Block |
| CPU | 2 cores, host type |
| RAM | 2048MB |
| Network | vmbr1 only |

The Ubuntu VM connects only to `vmbr1` — it has no direct path to the office LAN, only through OPNsense.

### Install Settings

- Language: English
- Keyboard: English (US)
- Type: Ubuntu Server
- Storage: entire disk
- Username: `sammy`
- Hostname: `ubuntu-victim`
- OpenSSH server: **enabled**

On first boot the VM received `10.10.10.121/24` via DHCP from OPNsense — confirming the lab network is functioning.

---

## Step 6 — Add Isolation Firewall Rule

By default OPNsense allows all LAN traffic outbound, including to the office LAN via the WAN interface. A block rule is added to prevent lab VMs from accessing the office network.

In OPNsense web UI → **Firewall** → **Rules** → **LAN** → **Add** (at top):

| Field | Value |
|---|---|
| Action | Block |
| Interface | LAN |
| Direction | in |
| Protocol | any |
| Source | any |
| Destination | 10.8.0.0/13 |
| Log | ✅ enabled |
| Description | Block lab to office LAN |

> **Important:** This rule must be placed **above** the default allow rules. OPNsense evaluates rules top-to-bottom and stops at the first match. If the block rule is below the default allow, it will never be reached.

Click **Save** → **Apply Changes**.

---

## Verification

Three tests confirm the lab is working correctly:

```bash
# Test 1 — internet access via OPNsense NAT (should succeed)
ping -c 3 8.8.8.8

# Test 2 — OPNsense gateway reachable (should succeed)
ping -c 3 10.10.10.1

# Test 3 — office LAN blocked (should fail)
ping -c 3 10.8.9.1
```

| Test | Expected | Result |
|---|---|---|
| `ping 8.8.8.8` | ✅ replies | Internet works via NAT |
| `ping 10.10.10.1` | ✅ replies | Gateway reachable |
| `ping 10.8.9.1` | ❌ no reply | Lab isolated from office LAN |

---

## Lab Network Summary

| Host | IP | Role |
|---|---|---|
| OPNsense LAN | `10.10.10.1` | Gateway, firewall, DHCP |
| Ubuntu Victim | `10.10.10.121` | First lab VM, future Wazuh agent |
| Proxmox (temp) | `10.10.10.254` | Jump host for GUI access |
| DHCP Pool | `10.10.10.100–200` | Available for future VMs |

---

## Screenshots

| # | Description | File |
|---|---|---|
| 1 | Proxmox network list showing vmbr0 and vmbr1 | `screenshots/03-vmbr1-created.png` |
| 2 | OPNsense VM hardware showing net0 and net1 | `screenshots/03-opnsense-hardware.png` |
| 3 | OPNsense first boot console showing interface assignments | `screenshots/03-opnsense-booted.png` |
| 4 | OPNsense main menu showing LAN 10.10.10.1 and WAN | `screenshots/03-opnsense-interfaces.png` |
| 5 | Ubuntu VM network showing 10.10.10.121/24 from DHCP | `screenshots/03-ubuntu-network.png` |
| 6 | OPNsense web GUI dashboard | `screenshots/03-opnsense-dashboard.png` |
| 7 | Firewall rules showing block rule above allow rules | `screenshots/03-firewall-rules.png` |
| 8 | Final verification — internet works, office LAN blocked | `screenshots/03-final-verification.png` |
