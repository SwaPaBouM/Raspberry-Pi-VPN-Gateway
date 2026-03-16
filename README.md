# Raspberry Pi VPN Gateway — VPN Provider + WireGuard

> Turn your Raspberry Pi into a **transparent VPN gateway** for your entire local network using your VPN provider and WireGuard. Every device connected to your router will automatically route its traffic through the VPN — no per-device configuration needed.

---

## Table of Contents

- [Hardware Requirements](#hardware-requirements)
- [Software Requirements](#software-requirements)
- [Architecture Overview](#architecture-overview)
- [Step 1 — System Preparation](#step-1--system-preparation)
- [Step 2 — WireGuard Configuration](#step-2--wireguard-configuration)
- [Step 3 — Enable IP Forwarding](#step-3--enable-ip-forwarding)
- [Step 4 — Configure NAT with iptables](#step-4--configure-nat-with-iptables)
- [Step 5 — Start & Enable WireGuard](#step-5--start--enable-wireguard)
- [Step 6 — Configure Your Router Gateway](#step-6--configure-your-router-gateway)
- [Step 7 — Verification](#step-7--verification)
- [Optional — dnsmasq DHCP Server](#optional--dnsmasq-dhcp-server-if-router-gateway-is-not-configurable)
- [Maintenance](#maintenance)

---

## Hardware Requirements

| Component | Minimum | Recommended |
|---|---|---|
| **Raspberry Pi** | Pi 3B+ | Pi 4 (2GB+ RAM) |
| **Storage** | 8 GB microSD | 16 GB+ microSD (Class 10) |
| **Network** | Wi-Fi or Ethernet | Ethernet (eth0) for stability |
| **Power Supply** | Official Pi PSU | Official Pi PSU |

> Ethernet connection to your router is strongly recommended for stability and performance.

---

## Software Requirements

- **OS**: Raspberry Pi OS (Debian-based, 64-bit recommended) or any Debian/Ubuntu derivative
- **VPN account**: Any paid plan that give a stable WireGuard access
- **Packages to install**:
  - `wireguard` + `wireguard-tools`
  - `resolvconf`
  - `iptables` + `iptables-persistent` / `netfilter-persistent`
  - `curl` (for testing)

---

## Architecture Overview

```
Internet
   │
   ▼
[ Router / ISP Box ]  ←── 192.168.1.1
   │
   ▼  (Ethernet)
[ Raspberry Pi ]      ←── 192.168.1.x  (static IP)
   │  WireGuard tunnel (UDP 51820)
   ▼
[ VPN Server ]  ←── Iceland / Switzerland (recommended)
   │
   ▼
Internet (anonymized)
```

All devices on the LAN use the Pi as their default gateway → all traffic exits through the VPN.

---

## Step 1 — System Preparation

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y wireguard wireguard-tools resolvconf curl iptables iptables-persistent
```

### Fix DNS (if resolution fails during install)

```bash
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

Make it persistent:

```bash
echo -e "nameserver 8.8.8.8\nnameserver 1.1.1.1" | sudo tee /etc/resolvconf/resolv.conf.d/head
sudo resolvconf -u
```

---

## Step 2 — WireGuard Configuration

### Download the WireGuard config from your VPN provider

1. Log in on your VPN provider account.
2. Go to **VPN → Downloads → WireGuard configuration** (*example*)
3. Select:
   - **Platform**: GNU/Linux
   - **Protocol**: WireGuard
   - **Country**: Iceland 🇮🇸 or Switzerland 🇨🇭 *(recommended for privacy — outside 14 Eyes jurisdiction)*
   - **Config type**: Standard or Secure Core
4. Download the `.conf` file

### Install the config

```bash
sudo cp /path/to/your-vpn-config.conf /etc/wireguard/wg0.conf
sudo chmod 600 /etc/wireguard/wg0.conf
```

### Verify the config

```bash
sudo cat /etc/wireguard/wg0.conf
```

Expected structure:

```ini
[Interface]
PrivateKey = <your_private_key>
Address = 10.2.0.2/32
DNS = 10.2.0.1

[Peer]
PublicKey = <server_public_key>
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = <server_ip>:51820
PersistentKeepalive = 25
```

---

## Step 3 — Enable IP Forwarding

```bash
# Enable immediately
sudo sysctl -w net.ipv4.ip_forward=1

# Make permanent
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

## Step 4 — Configure NAT with iptables

First, identify your local network interface (the one connected to your router):

```bash
ip route | grep default
# Example output: default via 192.168.1.1 dev eth0
# → your local interface is eth0
```

Apply the NAT rules (replace `eth0` with your interface if different):

```bash
sudo iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE
sudo iptables -A FORWARD -i eth0 -o wg0 -j ACCEPT
sudo iptables -A FORWARD -i wg0 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

Save the rules so they persist across reboots:

```bash
sudo netfilter-persistent save
```

Verify the rules are in place:

```bash
sudo iptables -t nat -L -v
sudo iptables -L FORWARD -v
```

---

## Step 5 — Start & Enable WireGuard

```bash
# Bring up the tunnel
sudo wg-quick up wg0

# Check the tunnel status
sudo wg show

# Enable at boot via systemd
sudo systemctl enable wg-quick@wg0

# Start via systemd (for consistent service management)
sudo systemctl start wg-quick@wg0
sudo systemctl status wg-quick@wg0
```

> `wg-quick` is a "oneshot" service — `active (exited)` is **normal** and means it ran successfully.

---

## Step 6 — Configure Your Router Gateway

In your router's admin interface, set the **default gateway** distributed by DHCP to the Pi's static IP:

```
Gateway : 192.168.1.x   ← your Pi's static IP
DNS     : 10.2.0.1      ← VPN DNS (via the tunnel)
DNS     : 8.8.8.8       ← fallback
```

All devices will automatically use the Pi as their router upon their next DHCP renewal.

> Make sure the Pi has a **static IP** on your network (either via router DHCP reservation by MAC address, or configured statically on the Pi itself).

---

## Step 7 — Verification

### Check the VPN tunnel

```bash
sudo wg show
```

Expected output includes:
- `latest handshake: X seconds ago`
- `transfer: X KiB received, X KiB sent`

### Check your public IP

```bash
curl https://ipinfo.io
```

The `ip`, `city`, and `country` fields should reflect the VPN server location (e.g., Iceland), **not** your real location.

### Check for DNS leaks

Visit [dnsleaktest.com](https://dnsleaktest.com) — all DNS servers listed should belong to your VPN provider and be located in the VPN country.

---

## Optional — dnsmasq DHCP Server (if router gateway is not configurable)

Some ISP routers do **not** allow changing the default gateway distributed via DHCP. In this case, run a DHCP server directly on the Pi to override this behavior.

### Install dnsmasq

```bash
sudo apt install -y dnsmasq
```

### Configure dnsmasq

```bash
sudo nano /etc/dnsmasq.conf
```

Add the following (adjust the IP range to avoid conflicts with your router's DHCP range):

```ini
# Listen only on the local network interface
interface=eth0
bind-interfaces

# DHCP range (adjust to avoid overlap with router's existing range)
dhcp-range=192.168.1.151,192.168.1.250,24h

# Default gateway = the Pi
dhcp-option=3,192.168.1.x

# DNS = VPN tunnel DNS + fallback
dhcp-option=6,10.2.0.1,8.8.8.8

# Use only the specified DNS servers
no-resolv
server=10.2.0.1
```

### Enable and start dnsmasq

```bash
sudo systemctl enable dnsmasq
sudo systemctl restart dnsmasq
sudo systemctl status dnsmasq
```

Verify it's handing out leases:

```bash
sudo journalctl -u dnsmasq -f
# Look for: DHCPDISCOVER / DHCPOFFER / DHCPACK lines
```

### Disable DHCP on your router

In your router's admin interface, **disable the built-in DHCP server**. The Pi's dnsmasq will now be the sole DHCP authority on the network.

> **Recovery**: If something goes wrong, simply re-enable DHCP on your router to restore normal connectivity instantly.

---

## Maintenance

### Useful commands

```bash
sudo wg show                          # Tunnel status
sudo wg-quick up wg0                  # Start tunnel manually
sudo wg-quick down wg0                # Stop tunnel
sudo systemctl restart wg-quick@wg0   # Restart via systemd
journalctl -u wg-quick@wg0 -f        # Live logs
```

### Auto-restart if tunnel drops

```bash
# Add a cron job to check every 5 minutes and restart if needed
(crontab -l; echo "*/5 * * * * sudo wg show wg0 | grep -q 'latest handshake' || sudo systemctl restart wg-quick@wg0") | crontab -
```

### Renewing your WireGuard config

VPN WireGuard keys may have an expiry. If the tunnel stops working:

1. Generate a new `.conf` from your VPN account
2. Stop the tunnel: `sudo wg-quick down wg0`
3. Replace the config: `sudo cp new-config.conf /etc/wireguard/wg0.conf && sudo chmod 600 /etc/wireguard/wg0.conf`
4. Restart: `sudo wg-quick up wg0`

---

## Recommended Countries for Privacy

| Country | Reason |
|---|---|
| 🇮🇸 **Iceland** | Outside 14 Eyes, strong privacy laws, no data retention requirements |
| 🇨🇭 **Switzerland** | Some VPN's home country, outside EU & 14 Eyes, excellent legal protections |

> **Avoid**: USA, UK, Canada, Australia (5 Eyes) — France, Germany (14 Eyes) !!

---

## Security Notes

- Keep your WireGuard `PrivateKey` **secret** — never commit it to a public repository
- This guide uses `8.8.8.8` (Google) as a fallback DNS only during setup change it to the DNS IP of your VPN provider 
- Disable IPv6 traffic

---

## Temporarily Bypassing the VPN

VPN throughput is typically limited (~2–5 MB/s). When you need full speed (large downloads, etc.), here are your options.

### Option A — Disable VPN for the entire network (fastest)

```bash
# Stop the tunnel → all devices route directly through the ISP
sudo systemctl stop wg-quick@wg0

# Re-enable when done
sudo systemctl start wg-quick@wg0
```

### Option B — Bypass on a single device only

Temporarily change the default gateway on one machine only, leaving the VPN active for all others:

```bash
# Switch to direct ISP routing on this device
sudo ip route replace default via 192.168.1.1

# Verify — should show your real ISP IP
curl https://ipinfo.io

# Restore VPN routing when done
sudo ip route replace default via 192.168.1.x   # ← your Pi's static IP
```

### Bonus — Handy aliases on the Pi

Add these shortcuts to `~/.bashrc` on the Pi for quick toggling over SSH:

```bash
echo "alias vpn-off='sudo systemctl stop wg-quick@wg0 && echo VPN OFF'" >> ~/.bashrc
echo "alias vpn-on='sudo systemctl start wg-quick@wg0 && echo VPN ON'" >> ~/.bashrc
source ~/.bashrc
```

Then from anywhere via SSH:

```bash
vpn-off   # full speed, direct ISP routing for the whole network
vpn-on    # back to VPN
```
