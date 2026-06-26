<div align="center">

# 🔀 Reverse Exit Tunnel for 3x-ui

**Route Xray inbound traffic through a remote node that has no public IP**

*WireGuard · SSH Reverse Tunnel · microsocks · socat · Docker · 3x-ui*

<br>

[![WireGuard](https://img.shields.io/badge/WireGuard-88171A?style=flat-square&logo=wireguard&logoColor=white)](https://www.wireguard.com/)
[![OpenSSH](https://img.shields.io/badge/OpenSSH-black?style=flat-square&logo=openssh&logoColor=white)](https://www.openssh.com/)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)](https://www.docker.com/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu_22.04+-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com/)
[![3x-ui](https://img.shields.io/badge/3x--ui-latest-blue?style=flat-square)](https://github.com/MHSanaei/3x-ui)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat-square)](https://github.com/)

<br>

🌐 **Language:** English · [Русский](README.ru.md)

</div>

---

## 📖 Overview

**Problem:** You run 3x-ui in Docker on a VPS (Entry Server). You want specific inbounds to exit the internet through a different machine (Exit Node) — which has **no static or public IP**.

**Solution:** The Exit Node initiates a tunnel *outward* to the Entry Server. The Entry Server's Xray routes selected inbounds through a `socat` relay → tunnel → `microsocks` on the Exit Node.

Two tunnel implementations are covered:

| | Solution A · WireGuard | Solution B · SSH Reverse |
|---|---|---|
| **Best for** | Home server, VPS, unrestricted network | Corporate network, NAT, strict firewall |
| **Protocol** | UDP | TCP |
| **Port on Entry Server** | 51820/UDP | 22/TCP (already open) |
| **Stability** | ★★★★★ | ★★★★☆ |
| **Setup complexity** | Medium | Low |
| **Firewall sensitivity** | High (UDP may be blocked) | Low (TCP 22 always passes) |

> [!TIP]
> **Not sure which to pick?** Try Solution A first. If the WireGuard handshake never establishes (0 packets received on Exit Node), your network blocks UDP — switch to Solution B.

---

## 🔀 Architecture

### Solution A — WireGuard

```
┌─────────────────────────────────────────────────────────┐
│                 Entry Server (VPS)                      │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │            Docker: 3x-ui / Xray                   │  │
│  │  [inbound] ──routing──► [remote-exit outbound]    │  │
│  │                         DOCKER_GATEWAY:10800       │  │
│  └──────────────────┬────────────────────────────────┘  │
│                     │ Docker bridge                     │
│                     ▼                                   │
│          socat relay (host)                             │
│          DOCKER_GATEWAY:10800 ──► 10.99.0.2:10800      │
│                                   │                     │
│          WireGuard wg-server      │                     │
│          listens :51820/udp ◄─────┘ (WG tunnel)        │
└───────────────────────────────────────────────────────┬─┘
                                                        │ WireGuard UDP
                                                        │ Exit Node initiates
                                                        ▼
                                        ┌───────────────────────┐
                                        │      Exit Node        │
                                        │  wg0: 10.99.0.2       │
                                        │  microsocks :10800     │
                                        └──────────┬────────────┘
                                                   │
                                                   ▼
                                           Internet (Exit Node IP)
```

### Solution B — SSH Reverse Tunnel

```
┌─────────────────────────────────────────────────────────┐
│                 Entry Server (VPS)                      │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │            Docker: 3x-ui / Xray                   │  │
│  │  [inbound] ──routing──► [remote-exit outbound]    │  │
│  │                         DOCKER_GATEWAY:10800       │  │
│  └──────────────────┬────────────────────────────────┘  │
│                     │ Docker bridge                     │
│                     ▼                                   │
│          socat relay (host)                             │
│          DOCKER_GATEWAY:10800 ──► 127.0.0.1:10800      │
│                                   │                     │
│          sshd opens 127.0.0.1:10800 ◄── SSH tunnel     │
└───────────────────────────────────────────────────────┬─┘
                                                        │ SSH TCP (port 22)
                                                        │ Exit Node initiates
                                                        ▼
                                        ┌───────────────────────┐
                                        │      Exit Node        │
                                        │  autossh reverse fwd  │
                                        │  microsocks :10800     │
                                        │  (127.0.0.1 only)     │
                                        └──────────┬────────────┘
                                                   │
                                                   ▼
                                           Internet (Exit Node IP)
```

---

## ✅ Prerequisites

| | Entry Server | Exit Node |
|---|---|---|
| **OS** | Ubuntu 22.04+ | Ubuntu 22.04+ |
| **Network** | Public IP, Docker + 3x-ui running | Internet access only |
| **Required open ports** | 22/TCP, 443/TCP + **51820/UDP** *(WG only)* | None |

---

## 📋 Variable Reference

| Variable | Description | How to find |
|----------|-------------|-------------|
| `YOUR_VPS_IP` | Entry Server public IP | `curl ifconfig.me` |
| `DOCKER_GATEWAY` | Docker bridge gateway | `docker exec <container> ip route \| grep default` → second field |
| `BR_IFACE` | Docker bridge interface | `ip link show \| grep br-` |
| `INBOUND_TAG` | Xray inbound tag in 3x-ui | Visible in Xray JSON, e.g. `inbound-1080` |

<details>
<summary>💡 How to find DOCKER_GATEWAY and BR_IFACE</summary>

```bash
# DOCKER_GATEWAY — run on Entry Server
docker exec <3x-ui-container-name> ip route | grep default
# Example: default via 172.19.0.1 dev eth0
#                      ^^^^^^^^^^ = DOCKER_GATEWAY

# BR_IFACE — match by network name
docker network ls                                          # find your compose network
docker network inspect <network-name> | grep Subnet        # confirm subnet
ip link show | grep br-                                    # br-XXXXXXXX = BR_IFACE
```

</details>

---

## 🛡️ Solution A: WireGuard

> Use when the Exit Node is on a home network, VPS, or any network without UDP filtering.

### A1 — Entry Server: WireGuard server

```bash
sudo apt update && sudo apt install -y wireguard

# Generate keys
sudo bash -c "
  cd /etc/wireguard
  wg genkey | tee server_private.key | wg pubkey > server_public.key
  chmod 600 server_private.key
"

# Save public key — needed for Exit Node (Step A2)
sudo cat /etc/wireguard/server_public.key
```

```bash
# Create config
SERVER_PRIVKEY=$(sudo cat /etc/wireguard/server_private.key)
sudo bash -c "cat > /etc/wireguard/wg-server.conf << EOF
[Interface]
PrivateKey = ${SERVER_PRIVKEY}
Address = 10.99.0.1/24
ListenPort = 51820

[Peer]
# Fill PublicKey after Step A2
PublicKey = PLACEHOLDER
AllowedIPs = 10.99.0.2/32
PersistentKeepalive = 25
EOF"

sudo ufw allow 51820/udp
# Don't start yet — need Exit Node public key
```

### A2 — Exit Node: WireGuard client + microsocks

```bash
sudo apt update && sudo apt install -y wireguard microsocks

# Generate keys
sudo bash -c "
  cd /etc/wireguard
  wg genkey | tee client_private.key | wg pubkey | tee client_public.key
  chmod 600 client_private.key
"

# Save public key — copy to Entry Server
sudo cat /etc/wireguard/client_public.key
```

```bash
# Create WireGuard config
CLIENT_PRIVKEY=$(sudo cat /etc/wireguard/client_private.key)
sudo bash -c "cat > /etc/wireguard/wg0.conf << EOF
[Interface]
PrivateKey = ${CLIENT_PRIVKEY}
Address = 10.99.0.2/24

[Peer]
PublicKey = ENTRY_SERVER_PUBLIC_KEY
Endpoint = YOUR_VPS_IP:51820
AllowedIPs = 10.99.0.1/32
PersistentKeepalive = 25
EOF"

sudo nano /etc/wireguard/wg0.conf  # fill ENTRY_SERVER_PUBLIC_KEY and YOUR_VPS_IP
```

```bash
# microsocks — listens on WireGuard interface
sudo tee /etc/systemd/system/microsocks.service << 'EOF'
[Unit]
Description=microsocks SOCKS5 proxy (WireGuard exit)
After=wg-quick@wg0.service
Requires=wg-quick@wg0.service

[Service]
ExecStart=/usr/bin/microsocks -i 10.99.0.2 -p 10800
Restart=always
RestartSec=5
User=nobody

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable wg-quick@wg0 microsocks
```

### A3 — Key Exchange

```bash
# On Entry Server — insert Exit Node public key
EXIT_PUBKEY="PASTE_EXIT_NODE_PUBLIC_KEY_HERE"
sudo sed -i "s|PublicKey = PLACEHOLDER|PublicKey = ${EXIT_PUBKEY}|" \
  /etc/wireguard/wg-server.conf
sudo cat /etc/wireguard/wg-server.conf  # verify
```

### A4 — Start & Verify Tunnel

```bash
# Entry Server
sudo systemctl enable --now wg-quick@wg-server

# Exit Node
sudo systemctl start wg-quick@wg0
sudo systemctl start microsocks
```

```bash
# Verify from Entry Server — all three must pass:
sudo wg show wg-server                            # latest handshake: X seconds ago ✓
ping -c3 10.99.0.2                               # may fail (ICMP blocked by ufw) — ok
curl --socks5 10.99.0.2:10800 https://api.ipify.org  # must return Exit Node IP ✓
```

> [!WARNING]
> If `wg show` shows handshake but `curl` times out and `tcpdump -i wg0` on Exit Node shows 0 packets — your network blocks incoming UDP. Switch to **Solution B**.

### A5 — socat Relay (Entry Server)

```bash
sudo apt install -y socat

# Replace 172.19.0.1 with your actual DOCKER_GATEWAY
sudo tee /etc/systemd/system/exit-relay.service << 'EOF'
[Unit]
Description=socat: Docker bridge → WireGuard → Exit Node microsocks
After=wg-quick@wg-server.service
Requires=wg-quick@wg-server.service

[Service]
ExecStart=/usr/bin/socat \
  TCP-LISTEN:10800,bind=172.19.0.1,fork,reuseaddr \
  TCP:10.99.0.2:10800
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now exit-relay

ss -tlnp | grep 10800  # must show 172.19.0.1:10800 LISTEN
```

---

## 🔑 Solution B: SSH Reverse Tunnel

> Use when the Exit Node is behind a corporate firewall, strict NAT, or any network that blocks UDP.

### B1 — Exit Node: SSH key

```bash
# Install packages
sudo apt update && sudo apt install -y autossh microsocks

# Generate dedicated SSH key for the tunnel
ssh-keygen -t ed25519 -f ~/.ssh/entry_tunnel -N ""

# Show public key — copy to Entry Server (Step B2)
cat ~/.ssh/entry_tunnel.pub
```

### B2 — Entry Server: Authorize key

```bash
# Add Exit Node public key to authorized_keys
echo "PASTE_EXIT_NODE_PUBLIC_KEY_HERE" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Verify SSH works from Exit Node before proceeding:
# ssh -i ~/.ssh/entry_tunnel ubuntu@YOUR_VPS_IP
```

> [!NOTE]
> `GatewayPorts` in sshd_config is **not required** — socat connects to `127.0.0.1:10800` on the same host where sshd runs.

### B3 — Exit Node: microsocks + autossh

```bash
# microsocks on localhost only (not exposed outside the tunnel)
sudo tee /etc/systemd/system/microsocks.service << 'EOF'
[Unit]
Description=microsocks SOCKS5 proxy (SSH tunnel exit)
After=network.target

[Service]
ExecStart=/usr/bin/microsocks -i 127.0.0.1 -p 10800
Restart=always
RestartSec=5
User=nobody

[Install]
WantedBy=multi-user.target
EOF
```

```bash
# autossh reverse tunnel
# Replace YOUR_VPS_IP and ubuntu with your Entry Server user
sudo tee /etc/systemd/system/ssh-tunnel.service << 'EOF'
[Unit]
Description=SSH Reverse Tunnel to Entry Server
After=network.target microsocks.service
Wants=microsocks.service

[Service]
User=ubuntu
ExecStart=/usr/bin/autossh -M 0 -N \
  -o "ServerAliveInterval=10" \
  -o "ServerAliveCountMax=3" \
  -o "ExitOnForwardFailure=yes" \
  -o "StrictHostKeyChecking=no" \
  -i /home/ubuntu/.ssh/entry_tunnel \
  -R 10800:127.0.0.1:10800 \
  ubuntu@YOUR_VPS_IP
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now microsocks
sudo systemctl enable --now ssh-tunnel
```

### B4 — Entry Server: socat relay

```bash
sudo apt install -y socat

# Replace 172.19.0.1 with your actual DOCKER_GATEWAY
# Note: target is 127.0.0.1 (SSH tunnel endpoint), not 10.99.0.2
sudo tee /etc/systemd/system/exit-relay.service << 'EOF'
[Unit]
Description=socat: Docker bridge → SSH tunnel → Exit Node microsocks
After=network.target

[Service]
ExecStart=/usr/bin/socat \
  TCP-LISTEN:10800,bind=172.19.0.1,fork,reuseaddr \
  TCP:127.0.0.1:10800
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now exit-relay

ss -tlnp | grep 10800  # must show 172.19.0.1:10800 LISTEN
```

### B5 — Verify SSH Tunnel

```bash
# Entry Server — SSH tunnel port should appear within seconds
ss -tlnp | grep 10800
# Expected: 127.0.0.1:10800  LISTEN  (opened by sshd)

# Full chain test from Entry Server
curl --socks5 127.0.0.1:10800 https://api.ipify.org
# ✓ Must return Exit Node IP
```

---

## ⚙️ Common: UFW Rule (both solutions)

Docker containers cannot reach `DOCKER_GATEWAY:10800` without an explicit UFW rule.

```bash
# Find Docker bridge interface
docker network ls
docker network inspect <your-compose-network> | grep Subnet
ip link show | grep br-
```

```bash
# Allow traffic from Docker network to socat relay
# Replace br-XXXXXXXX with your actual bridge interface name
sudo ufw allow in on br-XXXXXXXX to 172.19.0.1 port 10800 proto tcp

# Verify from inside container
docker exec <3x-ui-container> nc -zv 172.19.0.1 10800
# ✓ Expected: open
```

---

## ⚙️ Common: 3x-ui Configuration (both solutions)

The 3x-ui setup is **identical** for both solutions — the DOCKER_GATEWAY:10800 address never changes.

### Add SOCKS5 Outbound

**Outbounds → + Add Outbound**

| Field | Value |
|-------|-------|
| Protocol | `socks` |
| Tag | `remote-exit` |
| Address | `172.19.0.1` ← your `DOCKER_GATEWAY` |
| Port | `10800` |
| Username / Password | *(leave empty)* |

Click **Create** → **Save**.

<details>
<summary>JSON equivalent</summary>

```json
{
  "tag": "remote-exit",
  "protocol": "socks",
  "settings": {
    "servers": [{ "address": "172.19.0.1", "port": 10800, "users": [] }]
  }
}
```

</details>

### Add Routing Rule

**Xray Configs → Routing → + Add Rule**

| Field | Value |
|-------|-------|
| Inbound Tags | select your target inbound tag |
| Outbound Tag | select `remote-exit` |
| Everything else | *(leave empty)* |

Click **Create** → **Save** → Xray restarts.

> [!IMPORTANT]
> The new rule must be **above** `direct` and `block` in the routing list.

<details>
<summary>JSON equivalent</summary>

```json
{
  "type": "field",
  "inboundTag": ["inbound-1080"],
  "outboundTag": "remote-exit"
}
```

</details>

---

## ✅ Final Verification

```bash
# Entry Server — all services healthy
sudo systemctl status exit-relay

# Solution A only
sudo wg show wg-server          # latest handshake: recent ✓

# Solution B only
ss -tlnp | grep 10800           # 127.0.0.1:10800 LISTEN ✓

# Container can reach relay
docker exec <3x-ui-container> nc -zv 172.19.0.1 10800   # open ✓

# Full end-to-end test
curl --socks5 user:pass@YOUR_VPS_IP:INBOUND_PORT https://api.ipify.org
# ✓ Returns Exit Node IP, not Entry Server IP
```

---

## ♻️ Startup Order

### Solution A — WireGuard

```
Exit Node                          Entry Server
─────────────────────              ──────────────────────────────
wg-quick@wg0                       wg-quick@wg-server
  │ (Requires)                       │ (Requires)
  ▼                                  ▼
microsocks                         exit-relay
(10.99.0.2:10800)                  (172.19.0.1:10800 → 10.99.0.2:10800)
                                     │
                                     ▼
                                   Docker 3x-ui   (restart: unless-stopped)
```

### Solution B — SSH Reverse Tunnel

```
Exit Node                          Entry Server
─────────────────────              ──────────────────────────────
microsocks (127.0.0.1:10800)       sshd (always running)
  │ (Wants)                          │ ← SSH connection from Exit Node
  ▼                                  │   opens 127.0.0.1:10800
ssh-tunnel (autossh) ──────────────►│
                                     ▼
                                   exit-relay
                                   (172.19.0.1:10800 → 127.0.0.1:10800)
                                     │
                                     ▼
                                   Docker 3x-ui   (restart: unless-stopped)
```

---

## 🛡️ Watchdog (recommended for Solution A)

WireGuard under NAT can lose the session. This script restores it automatically:

```bash
# Exit Node
sudo tee /usr/local/bin/wg-watchdog.sh << 'EOF'
#!/bin/bash
LAST=$(sudo wg show wg0 latest-handshakes | awk '{print $2}')
DIFF=$(( $(date +%s) - LAST ))
if [ "$DIFF" -gt 180 ]; then
    logger "wg-watchdog: stale handshake (${DIFF}s), restarting"
    systemctl restart wg-quick@wg0
fi
EOF

sudo chmod +x /usr/local/bin/wg-watchdog.sh
(sudo crontab -l 2>/dev/null; echo "*/2 * * * * /usr/local/bin/wg-watchdog.sh") \
  | sudo crontab -
```

> Solution B (SSH) does not need a watchdog — autossh handles reconnection automatically.

---

## 🔄 Migration: WireGuard → SSH

If WireGuard UDP is blocked on your Exit Node network, clean up and switch to SSH:

### Cleanup on Entry Server

```bash
sudo systemctl stop wg-quick@wg-server exit-relay
sudo systemctl disable wg-quick@wg-server exit-relay
sudo ip link delete wg-server 2>/dev/null || true
sudo rm -f /etc/wireguard/wg-server.conf \
           /etc/wireguard/server_private.key \
           /etc/wireguard/server_public.key
sudo rm -f /etc/systemd/system/exit-relay.service
sudo ufw delete allow 51820/udp
sudo systemctl daemon-reload
```

### Cleanup on Exit Node

```bash
sudo systemctl stop wg-quick@wg0 microsocks
sudo systemctl disable wg-quick@wg0 microsocks
sudo ip link delete wg0 2>/dev/null || true
sudo rm -f /etc/wireguard/wg0.conf \
           /etc/wireguard/client_private.key \
           /etc/wireguard/client_public.key
sudo rm -f /etc/systemd/system/microsocks.service
sudo systemctl daemon-reload
```

Then follow **Solution B** steps B1–B5. The UFW rule and 3x-ui config remain unchanged.

---

## 🔧 Troubleshooting

<details>
<summary>❌ [WG] No handshake after several minutes</summary>

```bash
# Check UDP 51820 is open on Entry Server
sudo ufw status | grep 51820

# Check Exit Node can reach Entry Server
ping -c3 YOUR_VPS_IP

# Check both configs — keys must match cross-sides
sudo cat /etc/wireguard/wg-server.conf   # Entry Server
sudo cat /etc/wireguard/wg0.conf         # Exit Node

# Restart both
sudo systemctl restart wg-quick@wg-server   # Entry Server
sudo systemctl restart wg-quick@wg0         # Exit Node
```

</details>

<details>
<summary>❌ [WG] Handshake ok but curl times out (0 packets in tcpdump)</summary>

Your network blocks **incoming UDP**. The Exit Node can send but not receive.  
→ **Migrate to Solution B (SSH).**

</details>

<details>
<summary>❌ [SSH] 127.0.0.1:10800 not listening on Entry Server</summary>

```bash
# Check ssh-tunnel service on Exit Node
sudo systemctl status ssh-tunnel
sudo journalctl -u ssh-tunnel -n 30

# Test SSH connectivity manually from Exit Node
ssh -i ~/.ssh/entry_tunnel -N -R 10800:127.0.0.1:10800 ubuntu@YOUR_VPS_IP
# If this fails — check authorized_keys on Entry Server

# Check authorized_keys
cat ~/.ssh/authorized_keys | grep -c "ssh-"   # should be >= 1
```

</details>

<details>
<summary>❌ nc -zv DOCKER_GATEWAY 10800 returns "Host is unreachable" from container</summary>

```bash
# Missing UFW rule — add it
sudo ufw allow in on br-XXXXXXXX to DOCKER_GATEWAY port 10800 proto tcp

# Verify
docker exec <container> nc -zv DOCKER_GATEWAY 10800
```

</details>

<details>
<summary>❌ curl returns Entry Server IP instead of Exit Node IP</summary>

```bash
# Check routing rule order in 3x-ui
# Xray Configs → Routing
# Rule with your inbound → remote-exit must be ABOVE "direct" and "block"

# Verify outbound address is DOCKER_GATEWAY (not 10.99.0.2 or 127.0.0.1)
# Xray container can't reach those directly — socat relay is the bridge
```

</details>

---

## ❓ FAQ

**Q: Can I use multiple Exit Nodes with different inbounds?**  
A: Yes. Each Exit Node gets its own tunnel (WG peer or SSH tunnel on a different port), its own socat relay service (different bind port), its own outbound and routing rule in 3x-ui.

**Q: Can I use a protocol other than SOCKS5 for the outbound?**  
A: Yes. Once the tunnel is up, run any proxy on the Exit Node (Xray VLESS, Shadowsocks, HTTP CONNECT, etc.) and use the matching outbound type in 3x-ui. microsocks is recommended for simplicity — zero config.

**Q: Does this work without Docker (Xray as systemd)?**  
A: Yes, and it's simpler. Skip socat relay and the UFW step — point the SOCKS5 outbound directly to `10.99.0.2:10800` (WG) or `127.0.0.1:10800` (SSH) in your Xray config.

**Q: The Exit Node reboots and gets a new IP — does that break the tunnel?**  
A: No. The Exit Node always initiates outward to the Entry Server's fixed IP. The tunnel re-establishes on its own within seconds.

---

## 📁 File Map

```
Entry Server
├── /etc/systemd/system/exit-relay.service    socat relay (both solutions)
├── /etc/wireguard/wg-server.conf             WireGuard server config (Solution A)
└── ~/.ssh/authorized_keys                    Exit Node key (Solution B)

Exit Node
├── /etc/systemd/system/microsocks.service    microsocks SOCKS5 proxy
├── /etc/wireguard/wg0.conf                   WireGuard client config (Solution A)
├── /etc/systemd/system/ssh-tunnel.service    autossh reverse tunnel (Solution B)
├── ~/.ssh/entry_tunnel                        SSH private key (Solution B)
└── /usr/local/bin/wg-watchdog.sh             Watchdog cron script (Solution A)
```

---

<div align="center">

Made with ☕ · Pull requests welcome · [Open an Issue](https://github.com/)

</div>
