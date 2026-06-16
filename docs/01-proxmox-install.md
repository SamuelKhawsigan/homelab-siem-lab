# 01 — Proxmox VE Install

## Overview

This document covers the bare-metal installation of Proxmox VE 9.1 on an SZBOX H14 mini PC (Intel i3-N305, 16GB RAM). Starting from flashing the ISO to a USB drive, through the installer, to first login on the web UI.

---

## Hardware

| Component | Detail |
|---|---|
| Device | SZBOX H14 Mini PC |
| CPU | Intel i3-N305 (8 cores) |
| RAM | 16GB |
| Network | 4x Intel i226 2.5GbE NICs |

---

## Step 1 — Download and Flash the ISO

Download the latest Proxmox VE ISO from the official site:

```
https://www.proxmox.com/en/downloads
```

Flash it to a USB drive (minimum 4GB) using Rufus in **DD mode**:

1. Open Rufus
2. Select the USB device
3. Select the Proxmox `.iso` file
4. When prompted, select **DD Image** mode (not ISO mode)
5. Click Start

> **Note:** DD mode is required. ISO mode will produce a non-bootable USB for Proxmox.

---

## Step 2 — Boot from USB

1. Plug the USB into the SZBOX
2. Power on and press **Delete** to enter BIOS (or **F7** for direct boot menu)
3. Set the USB drive as the first boot device
4. Save and exit

The SZBOX will boot into the Proxmox installer.

---

## Step 3 — Run the Installer

At the boot menu, select **Install Proxmox VE (Terminal UI)** and press Enter.

Work through the installer screens:

- **EULA** → Accept
- **Target disk** → select the internal NVMe/eMMC drive, leave defaults
- **Location** → Country: Malaysia, Timezone: Asia/Kuala_Lumpur, Keyboard: US
- **Password** → set a strong root password, enter any email address
- **Network** → fill in as below

### Network Configuration

| Field | Value |
|---|---|
| Management Interface | `nic0` (auto-selected) |
| Hostname (FQDN) | `proxmox.local` |
| IP Address (CIDR) | DHCP-assigned (noted post-install) |
| Gateway | `10.8.50.1` |
| DNS Server | `202.169.30.20` |

> **Note:** A static IP was attempted during install (`10.8.9.10/23`) but the office switch rejected the new MAC address. DHCP was used instead and the box was assigned `10.8.9.72/24`. A static DHCP reservation or IP fix will be applied in a later phase.

- **Summary** → review all values, click **Install**

Installation takes approximately 5–10 minutes.

---

## Step 4 — First Boot

When installation completes, the system reboots automatically. **Remove the USB drive** as soon as the screen goes black to prevent booting back into the installer.

On first boot, the console displays:

```
Welcome to Proxmox VE

Proxmox VE https://10.8.9.72:8006/
```

---

## Step 5 — Access the Web UI

From a machine on the same network, open a browser and navigate to:

```
https://10.8.9.72:8006
```

Accept the self-signed certificate warning (click Advanced → Proceed).

Log in with:
- **Username:** `root`
- **Realm:** `Linux PAM standard authentication`
- **Password:** (set during install)

The Proxmox dashboard confirms a successful installation.

---

## Troubleshooting Notes

### Office switch rejecting new MAC address
The office network uses MAC-based filtering or 802.1X. The SZBOX's new MAC (`60:be:b4:22:fa:b3`) was not recognized after reinstall, causing the switch to drop all traffic even at layer 2 (gateway ping failed). The fix was to use DHCP with default installer settings — the DHCP request uses the same MAC and the switch allowed it once a lease was issued.

### Proxmox enterprise repo 401 errors
Fresh Proxmox installs include the enterprise repository enabled by default, which requires a paid subscription. This causes `apt-get update` to fail with 401 Unauthorized. Fixed in Phase 2.

---

## Screenshots

| # | Description | File |
|---|---|---|
| 1 | Proxmox installer network config screen | `screenshots/01-network-config.jpg` |
| 2 | Console on first boot showing web UI URL | `screenshots/01-first-boot.jpg` |
| 3 | Proxmox web UI dashboard after first login | `screenshots/01-dashboard.png` |
