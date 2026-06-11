**Hardware:** ASUS N55S (old laptop) → headless server  
**OS:** Ubuntu Server 26.04 LTS (Minimized)

> **Note — this is one person's build.** Hardware-specific values below (interface
> names like `wlp3s0`/`enp5s0`, the AR9285 WiFi card and its 2.4 GHz-only quirk, MAC
> addresses, IPs, and the iPhone-hotspot network) are particular to this ASUS N55S
> setup. On other hardware, run `ip a` / `ip link` and substitute your own. The
> concepts and command flow are general-purpose.

---

## Installation Decisions

### Ubuntu Server vs Ubuntu Server Minimized

- **Minimized** = stripped-down image: no man-pages, fewer pre-installed packages, smaller footprint.
- Ideal for resource-constrained hardware where every MB of RAM matters (k3s + PostgreSQL will run on top).
- Anything missing → `apt install` later.

### Network Configuration

- **DHCP** — let the router assign an IP automatically. Static IP is configured later via **DHCP reservation** on the router (bind MAC address → fixed IP). This is more reliable than hardcoding a static IP in the OS.
- **Network bonding** — combining multiple NICs into one virtual interface for redundancy or bandwidth aggregation. Not needed for a single-NIC setup.
- **WiFi on Ubuntu Server** — the installer often doesn't expose WiFi. Skip network during install, configure via `netplan` post-install.

### DHCP Reservation (when you have a router)

- Router admin panel (usually `192.168.1.1`) → DHCP reservation section.
- Bind N55S MAC address → desired fixed IP. One-time setup.
- The server still uses DHCP internally — it just always gets the same IP from the router.
- If you change networks, the server adapts automatically (unlike a hardcoded static IP in the OS via netplan, which would break).

### Storage Layout

- **Use entire disk + LVM** — the right choice for a homelab server.
- **LVM (Logical Volume Manager)** — abstraction layer over physical disks. Allows resizing partitions, adding disks later without reinstalling the OS.
- **LUKS (Linux Unified Key Setup)** — full disk encryption. **Don't enable for headless servers** — every reboot requires a passphrase to unlock the disk, and there's no keyboard/monitor attached.
- **LUKS on headless servers is possible** — use **Dropbear SSH in initramfs** to enter the passphrase remotely before the disk unlocks. Other options: **Clevis + Tang** (network-bound auto-unlock), **Clevis + TPM2** (hardware-bound auto-unlock). Not needed for homelab, but good to know for production.

### SSH (Secure Shell)

- Encrypted protocol for remote access to a server's terminal over the network.
- **OpenSSH server** — must be installed on the server side to accept connections.
- **Password auth** — enable during install to allow initial connection. Disable later after setting up key-based auth.
- **Key-based auth** — more secure than passwords. A keypair (public + private) is generated on the client machine. Public key is copied to the server via `ssh-copy-id`. After that, disable password auth in `/etc/ssh/sshd_config` (`PasswordAuthentication no`).
- **Default port:** 22.

### SSH Key Setup (post-install)

```bash
# From workstation: copy public key to server
ssh-copy-id user@<server-ip>

# Verify: should connect without password
ssh user@<server-ip>

# On server: disable password auth
sudo sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

After this, only machines with the key can connect. Brute-force attacks become useless.

### UFW (Uncomplicated Firewall)

Simplified frontend for `iptables`/`nftables`. Controls which ports are open to incoming traffic.

```bash
sudo apt install -y ufw
sudo ufw allow OpenSSH      # allow SSH (port 22) before enabling!
sudo ufw enable              # activate firewall
sudo ufw status              # check rules
```

**Rule of thumb:** always `allow OpenSSH` before `enable`, or you'll lock yourself out of a headless server.

### Proxy Server

- **Forward proxy** — intermediary between your machine and the internet. Your request goes to the proxy first, then the proxy reaches out on your behalf.
- **Why it exists:** corporate networks use proxies to log, filter, cache, and control outbound traffic. Without proxy config in such environments, tools like `apt` can't reach the internet.
- **Home network** = direct internet access via router → no proxy needed.
- **Reverse proxy** (related but different) — sits in front of your _services_ and routes incoming requests. Traefik (built into k3s) is a reverse proxy. Relevant in Phase 1+.

### Headless: disable suspend on lid close

By default, closing the laptop lid triggers suspend. For a headless server, disable it:

```bash
sudo mkdir -p /etc/systemd/logind.conf.d
sudo bash -c 'cat > /etc/systemd/logind.conf.d/lid.conf << EOF
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
EOF'
sudo systemctl restart systemd-logind
```

---

## Post-Install: WiFi Setup

```bash
# 1. Find WiFi interface name
ip link
# Look for wlan0, wlp3s0, etc.

# 2. Create netplan config
sudo nano /etc/netplan/50-wifi.yaml
```

```yaml
network:
  version: 2
  wifis:
    wlan0:               # ← your interface name
      dhcp4: true
      access-points:
        "YourSSID":
          password: "YourPassword"
```

```bash
# 3. Apply
sudo netplan apply
```

---

## Key Concepts (Glossary)

|Term|Meaning|
|---|---|
|**Headless server**|Server without monitor/keyboard/GUI, managed remotely via SSH|
|**DHCP**|Dynamic Host Configuration Protocol — router auto-assigns IP addresses|
|**DHCP reservation**|Router always gives the same IP to a specific MAC address|
|**MAC address**|Unique hardware identifier of a network interface|
|**Proxy (forward)**|Middleman for outbound traffic (client → proxy → internet)|
|**Reverse proxy**|Middleman for inbound traffic (internet → proxy → your service)|
|**Netplan**|Ubuntu's network configuration tool (YAML-based)|
|**NIC**|Network Interface Card (сетевая карта) — physical or virtual network adapter. The Ethernet port or WiFi chip in your machine|
|**Bonding**|Combining multiple NICs into one logical interface|