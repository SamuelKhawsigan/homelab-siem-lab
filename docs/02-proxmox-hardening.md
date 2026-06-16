# 02 — Proxmox Hardening + Tailscale Remote Access

## Overview

This document covers post-install hardening of Proxmox VE and setting up Tailscale for persistent remote access. After this phase, the SZBOX is reachable from anywhere via Tailscale regardless of office network restrictions, and survives reboots without losing remote access.

---

## Step 1 — Fix the Package Repositories

Fresh Proxmox installs ship with the enterprise repositories enabled by default. These require a paid subscription and cause `apt-get update` to fail with 401 Unauthorized errors. They must be disabled before any packages can be installed.

### Disable the enterprise repos

The enterprise sources use the newer `.sources` format. Appending `Enabled: no` disables them without deleting the files:

```bash
cat > /etc/apt/sources.list.d/ceph.sources << 'EOF'
Types: deb
URIs: https://enterprise.proxmox.com/debian/ceph-squid
Suites: trixie
Components: enterprise
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
Enabled: no
EOF
```

```bash
cat > /etc/apt/sources.list.d/pve-enterprise.sources << 'EOF'
Types: deb
URIs: https://enterprise.proxmox.com/debian/pve
Suites: trixie
Components: enterprise
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
Enabled: no
EOF
```

### Add the no-subscription repo

```bash
echo "deb http://download.proxmox.com/debian/pve trixie pve-no-subscription" > /etc/apt/sources.list.d/pve-no-sub.list
```

### Update package lists

```bash
apt-get update
```

The output should show no 401 errors — only hits from `deb.debian.org`, `security.debian.org`, `download.proxmox.com`, and `pkgs.tailscale.com`.

---

## Step 2 — Install Tailscale

Tailscale provides a persistent VPN tunnel that makes the SZBOX reachable from anywhere — bypassing office WiFi client isolation and surviving IP/DHCP changes.

### Install via the official script

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

This automatically detects Debian Trixie and adds the Tailscale apt repository.

### Enable and start the service

```bash
systemctl enable --now tailscaled
```

The `enable` flag ensures tailscaled starts automatically on every boot — critical for remote access after an unattended reboot.

### Authenticate to your tailnet

```bash
tailscale up
```

This prints a one-time login URL:

```
https://login.tailscale.com/a/xxxxxxxxxx
```

Open this URL on another device and authorize the node in your Tailscale account. The URL expires quickly — have your browser ready before running the command.

> **Security note:** Never share this URL publicly. It is a one-time auth token that grants access to your Tailscale account.

### Verify the connection

```bash
tailscale status
```

Expected output shows the node name, Tailscale IP (`100.x.y.z`), and connected peers.

---

## Step 3 — Confirm Remote Access

From the Tailscale admin console (`login.tailscale.com`), the SZBOX appears as `pve-1` with status **Connected** and Tailscale IP `100.70.88.30`.

Access the Proxmox web UI over Tailscale from any device on the tailnet:

```
https://100.70.88.30:8006
```

This works from home, mobile, or anywhere — regardless of the office network topology.

---

## Step 4 — Verify Boot Persistence

Confirm tailscaled is set to start on boot:

```bash
systemctl is-enabled tailscaled
```

Expected output:
```
enabled
```

This is the critical setting that prevents the SZBOX from going dark after a reboot. With this in place, any reboot (planned or unplanned) will automatically reconnect the node to the tailnet within ~30 seconds.

---

## Tailscale Node Details

| Field | Value |
|---|---|
| Node name | `pve-1` |
| Tailscale IP | `100.70.88.30` |
| OS | Linux 6.17.2-1-pve |
| Version | Tailscale 1.98.4 |
| Status | Connected |

---

## Screenshots

| # | Description | File |
|---|---|---|
| 1 | Clean `apt-get update` output — no 401 errors | `screenshots/02-apt-update-clean.png` |
| 2 | Tailscale admin console showing pve-1 Connected | `screenshots/02-tailscale-connected.png` |
| 3 | Proxmox web UI accessed via Tailscale IP | `screenshots/02-proxmox-via-tailscale.png` |

---

## Why This Matters

Without Tailscale enabled on boot, any reboot of the SZBOX breaks remote access entirely — as experienced during initial setup. The combination of `systemctl enable` (boot persistence) and Tailscale's encrypted tunnel means:

- Remote access works from anywhere, not just the office LAN
- Office WiFi client isolation is irrelevant
- IP address changes (DHCP) don't break access since Tailscale uses its own stable IP
- The box is always reachable as long as it has any internet connection
