# 04 — Wazuh SIEM Deployment

## Overview

This document covers the deployment of Wazuh 4.14.5 as a single-node all-in-one installation on the lab network. Wazuh provides the SIEM, XDR, and active response capabilities for this project. After this phase, the Wazuh dashboard is accessible via SSH tunnel and ready to receive agent telemetry.

---

## VM Configuration

| Setting | Value |
|---|---|
| VM ID | 102 |
| Name | wazuh |
| OS | Ubuntu Server 24.04.1 LTS |
| Disk | 50GB VirtIO Block (expanded to 48GB usable via LVM) |
| CPU | 4 cores, host type |
| RAM | 4096MB |
| Network | vmbr1 only (lab network) |
| IP | 10.10.10.161/24 (DHCP from OPNsense) |

---

## Step 1 — Deploy Ubuntu VM

Install Ubuntu Server 24.04 on VM 102 using the same process as the victim VM (Phase 3), with these differences:

- Hostname: `wazuh`
- Username: `wazuh`
- Disk: 50GB
- CPU: 4 cores
- RAM: 4096MB

Enable OpenSSH during install for remote access.

---

## Step 2 — Fix DNS

After first boot, OPNsense's Unbound DNS resolver was not responding to queries from lab VMs (port 53 blocked by firewall). As a workaround, Google DNS was configured directly:

```bash
sudo bash -c 'echo "nameserver 8.8.8.8" > /etc/resolv.conf'
```

> **Note:** A proper fix requires adding a firewall rule in OPNsense to allow DNS (port 53 TCP/UDP) from LAN net to This Firewall. The direct DNS workaround is sufficient for installation purposes.

Verify DNS resolution works:

```bash
dig packages.wazuh.com
```

---

## Step 3 — Expand the LVM Volume

Ubuntu's default LVM setup only allocates approximately 24GB to the root logical volume, leaving the rest of the 50GB disk unallocated. The Wazuh dashboard package requires significant temporary disk space during extraction (~1GB). Expand the volume to use the full disk:

```bash
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
df -h /
```

Expected output shows root partition expanded to ~48GB with ~37GB free.

> **Why this matters:** The Wazuh dashboard package is 195MB compressed but expands to over 1GB during dpkg extraction. Combined with the apt cache (~1.5GB), the default 24GB allocation causes a "No space left on device" error during dashboard installation. This is a common gotcha on fresh Ubuntu VMs with default LVM sizing.

---

## Step 4 — Install Wazuh

Download the installation assistant and configuration file:

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.14/config.yml
```

Edit `config.yml` and replace all placeholder IPs with the Wazuh VM IP:

```bash
nano config.yml
```

Replace all instances of `<indexer-node-ip>`, `<wazuh-manager-ip>`, and `<dashboard-node-ip>` with `10.10.10.161`.

Run the all-in-one installation:

```bash
sudo bash wazuh-install.sh -a
```

The installer deploys three components in sequence:

1. **Wazuh indexer** — OpenSearch-based log storage and search engine
2. **Wazuh manager** — Core SIEM engine, rule evaluation, alert generation
3. **Wazuh dashboard** — Web UI for visualizing alerts and managing agents

Total installation time: approximately 5–8 minutes.

---

## Step 5 — Retrieve Credentials

When installation completes, the summary output displays the generated admin password:

```
INFO: --- Summary ---
INFO: You can access the web interface https://<wazuh-dashboard-ip>:443
      User: admin
      Password: <generated-password>
INFO: Installation finished.
```

**Screenshot this immediately** — the password is only shown once in plain text. Store it in a password manager or your Obsidian vault. Do not commit it to the repository.

---

## Step 6 — Access the Dashboard

The Wazuh dashboard is only accessible from inside the lab network. Use SSH port forwarding via Proxmox as a jump host to reach it from the MSI laptop:

```bash
ssh -L 8444:10.10.10.161:443 -J root@100.70.88.30 wazuh@10.10.10.161
```

Then open in browser:

```
https://localhost:8444
```

Log in with:
- **Username:** `admin`
- **Password:** (from install summary)

---

## Wazuh Services

Verify all three services are running:

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```

All three should show `active (running)`.

---

## Troubleshooting Notes

### Dashboard installation fails with "No space left on device"
The Wazuh dashboard package requires ~1.2GB of temporary space during dpkg extraction. On a default Ubuntu 24.04 LVM install, the root LV is too small. Fix: expand the LV with `lvextend` before installing.

### "Port 1515/55000 already in use" error on reinstall
A failed partial install leaves the Wazuh manager running but broken. Fix:
```bash
sudo systemctl stop wazuh-manager wazuh-indexer wazuh-dashboard filebeat
sudo pkill -f wazuh
```
Then blank the broken dpkg scripts and purge:
```bash
sudo bash -c 'echo "#!/bin/bash\nexit 0" > /var/lib/dpkg/info/wazuh-manager.prerm'
sudo bash -c 'echo "#!/bin/bash\nexit 0" > /var/lib/dpkg/info/wazuh-manager.postrm'
sudo dpkg --purge --force-all wazuh-manager
```

### DNS not resolving from lab VMs
OPNsense Unbound DNS does not respond to queries from LAN by default. Workaround: set DNS directly to `8.8.8.8` in `/etc/resolv.conf`. Proper fix: add OPNsense LAN firewall rule allowing TCP/UDP port 53 from LAN net to This Firewall.

---

## Wazuh Stack Summary

| Component | Service | Port |
|---|---|---|
| Wazuh indexer | wazuh-indexer | 9200 |
| Wazuh manager | wazuh-manager | 1514, 1515, 55000 |
| Wazuh dashboard | wazuh-dashboard | 443 |

---

## Screenshots

| # | Description | File |
|---|---|---|
| 1 | Wazuh install summary showing credentials | `screenshots/04-wazuh-install-complete.png` |
| 2 | Wazuh dashboard login page | `screenshots/04-wazuh-dashboard.png` |
