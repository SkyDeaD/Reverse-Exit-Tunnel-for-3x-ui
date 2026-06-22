<div align="center">

# 🔀 Reverse Exit Tunnel for 3x-ui

**Route Xray inbound traffic through a remote node that has no public IP**

*WireGuard · microsocks · socat · Docker · 3x-ui*

<br>

[![WireGuard](https://img.shields.io/badge/WireGuard-88171A?style=flat-square&logo=wireguard&logoColor=white)](https://www.wireguard.com/)
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

**Problem:** You run 3x-ui inside Docker on a VPS, but want specific inbounds to exit the internet through a different machine — which has **no static or public IP**.

**Solution:** The exit node initiates a WireGuard tunnel *outward* to the entry server. The entry server's Xray routes selected inbounds through a `socat` relay → WireGuard → `microsocks` on the exit node.

```
Client → [Entry Server: 3x-ui inbound] → [socat relay] ──WireGuard──► [Exit Node: microsocks] → Internet
```

> [!NOTE]
> The exit node never needs to accept incoming connections from the outside.  
> It always initiates the tunnel itself — NAT, CGNAT, dynamic IP: all fine.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Entry Server (VPS)                             │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                  Docker: 3x-ui / Xray                      │     │
│  │                                                            │     │
│  │  [inbound: INBOUND_TAG] ──routing rule──► [uz-exit]        │     │
│  │                             (SOCKS5 outbound               │     │
│  │                              DOCKER_GATEWAY:RELAY_PORT)    │     │
│  └───────────────────────┬────────────────────────────────────┘     │
│                          │  Docker bridge network                   │
│                          ▼                                          │
│               socat relay (systemd)                                 │
│               DOCKER_GATEWAY:RELAY_PORT ─────────────┐              │
│                                                      │              │
│                                                      ▼              │
│                                             WireGuard (wg-server)   │
│                                             WG_SERVER_ADDR:51820    │
└─────────────────────────────────────────────────────────────────────┘
                                             │
                                  WireGuard tunnel (UDP)
                                  Exit node initiates connection
                                             │
                                             ▼
                             ┌──────────────────────────────┐
                             │         Exit Node            │
                             │                              │
                             │  wg0: WG_CLIENT_ADDR         │
                             │  microsocks: PROXY_PORT      │
                             └───────────────┬──────────────┘
                                             │
                                             ▼
                                    Internet (Exit Node IP)
```

### Traffic flow

```
Client
  │  connects to Entry Server inbound
  ▼
3x-ui inbound  (tag: INBOUND_TAG)
  │  routing rule → outbound tag: remote-exit
  ▼
SOCKS5 outbound → DOCKER_GATEWAY:RELAY_PORT
  │  Docker bridge network (layer 3 only)
  ▼
socat relay  (running on VPS host, not in Docker)
  │  WireGuard tunnel
  ▼
microsocks  (Exit Node: WG_CLIENT_ADDR:PROXY_PORT)
  │
  ▼
Internet  ← IP = Exit Node's IP ✓
```

---

## ✅ Prerequisites

| Component | Location | Requirements |
|-----------|----------|-------------|
| **Entry Server** | VPS with public IP | Ubuntu 22.04+, Docker, 3x-ui running |
| **Exit Node** | Any machine | Ubuntu 22.04+, internet access, **no public IP needed** |

> [!IMPORTANT]
> Firewall on Entry Server must allow **UDP port 51820** (WireGuard).

---

## 📋 Variable Reference

Replace these placeholders throughout the guide:

| Variable | Description | How to find |
|----------|-------------|-------------|
| `YOUR_VPS_IP` | Entry Server public IP | `curl ifconfig.me` on Entry Server |
| `DOCKER_GATEWAY` | Docker bridge gateway | `docker exec <container> ip route \| grep default` |
| `BR_IFACE` | Docker bridge interface | `ip link show \| grep br-` |
| `WG_SERVER_ADDR` | WireGuard IP on Entry Server | Use `10.99.0.1` (example) |
| `WG_CLIENT_ADDR` | WireGuard IP on Exit Node | Use `10.99.0.2` (example) |
| `PROXY_PORT` | microsocks listen port | Use `10800` (example) |
| `INBOUND_TAG` | Xray inbound tag in 3x-ui | Visible in Xray JSON config |
| `OUTBOUND_TAG` | Name for the new outbound | Use `remote-exit` (example) |

<details>
<summary>💡 How to find DOCKER_GATEWAY and BR_IFACE</summary>

```bash
# Find Docker gateway (run on Entry Server)
docker exec <your-3x-ui-container> ip route | grep default
# Example output: default via 172.19.0.1 dev eth0
#                                 ^^^^^^^^^^^ this is DOCKER_GATEWAY

# Find Docker bridge interface name
docker network ls
docker network inspect <your-compose-network> | grep Subnet
ip link show | grep br-
# Match the network ID from 'docker network ls' to br-XXXXXXXXX
```

</details>

---

## 🚀 Setup

### Step 1 — Entry Server: WireGuard

```bash
# Install WireGuard
sudo apt update && sudo apt install -y wireguard

# Generate keys
sudo bash -c "
  cd /etc/wireguard
  wg genkey | tee server_private.key | wg pubkey > server_public.key
  chmod 600 server_private.key
"

# Show public key — copy it, needed for Exit Node
sudo cat /etc/wireguard/server_public.key
```

Create `/etc/wireguard/wg-server.conf`:

```bash
SERVER_PRIVKEY=$(sudo cat /etc/wireguard/server_private.key)

sudo bash -c "cat > /etc/wireguard/wg-server.conf << EOF
[Interface]
PrivateKey = ${SERVER_PRIVKEY}
Address = 10.99.0.1/24
ListenPort = 51820

[Peer]
# Exit Node — fill PublicKey after Step 2
PublicKey = PLACEHOLDER
AllowedIPs = 10.99.0.2/32
PersistentKeepalive = 25
EOF"

# Verify
sudo cat /etc/wireguard/wg-server.conf
```

```bash
# Open firewall port
sudo ufw allow 51820/udp

# Don't start WireGuard yet — need Exit Node public key first
```

---

### Step 2 — Exit Node: WireGuard + microsocks

```bash
# Install packages
sudo apt update && sudo apt install -y wireguard microsocks

# Generate keys
sudo bash -c "
  cd /etc/wireguard
  wg genkey | tee client_private.key | wg pubkey | tee client_public.key
  chmod 600 client_private.key
"

# Show public key — copy it to Entry Server
sudo cat /etc/wireguard/client_public.key
```

Create `/etc/wireguard/wg0.conf`:

```bash
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

# Fill in real values
sudo nano /etc/wireguard/wg0.conf
# Replace: ENTRY_SERVER_PUBLIC_KEY and YOUR_VPS_IP
```

Create microsocks systemd service:

```bash
sudo tee /etc/systemd/system/microsocks.service << 'EOF'
[Unit]
Description=microsocks SOCKS5 proxy for reverse exit tunnel
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
sudo systemctl enable wg-quick@wg0
sudo systemctl enable microsocks
# Don't start yet — complete the key exchange first
```

---

### Step 3 — Key Exchange

**On Entry Server** — replace the placeholder with Exit Node's public key:

```bash
EXIT_PUBKEY="PASTE_EXIT_NODE_PUBLIC_KEY_HERE"

sudo sed -i \
  "s|PublicKey = PLACEHOLDER|PublicKey = ${EXIT_PUBKEY}|" \
  /etc/wireguard/wg-server.conf

# Verify the result
sudo cat /etc/wireguard/wg-server.conf
```

Expected final config on Entry Server:

```ini
[Interface]
PrivateKey = <server private key>
Address = 10.99.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <exit node public key>
AllowedIPs = 10.99.0.2/32
PersistentKeepalive = 25
```

---

### Step 4 — Start the Tunnel

**On Entry Server:**

```bash
sudo systemctl enable --now wg-quick@wg-server
sudo wg show wg-server
```

**On Exit Node:**

```bash
sudo systemctl start wg-quick@wg0
sudo systemctl start microsocks

# Verify WireGuard interface
ip addr show wg0
# Expected: inet 10.99.0.2/24

# Verify microsocks is listening
ss -tlnp | grep 10800
# Expected: 10.99.0.2:10800  LISTEN
```

**Verify tunnel health (from Entry Server):**

```bash
# 1. Check WireGuard handshake
sudo wg show wg-server
# Look for: latest handshake: X seconds ago ✓

# 2. Ping Exit Node
ping -c3 10.99.0.2

# 3. The key check — does traffic exit through Exit Node?
curl --socks5 10.99.0.2:10800 https://api.ipify.org
# ✓ Must return Exit Node's IP, not Entry Server's IP
```

> [!WARNING]
> If Step 3 fails, the tunnel is broken. Do not proceed.  
> See [Troubleshooting](#-troubleshooting) below.

---

### Step 5 — Entry Server: socat Relay

Bridges the Docker network (`DOCKER_GATEWAY`) and WireGuard (`WG_CLIENT_ADDR`).

```
Docker container → DOCKER_GATEWAY:10800 ──socat──► 10.99.0.2:10800 → microsocks
```

```bash
sudo apt install -y socat
```

```bash
# Replace 172.19.0.1 with your actual DOCKER_GATEWAY
sudo tee /etc/systemd/system/exit-relay.service << 'EOF'
[Unit]
Description=socat relay: Docker bridge → WireGuard → Exit Node microsocks
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

# Verify relay is listening
ss -tlnp | grep 10800
# Expected: 172.19.0.1:10800  LISTEN  socat
```

---

### Step 6 — UFW: Allow Docker → Relay

By default, UFW's `deny (incoming)` blocks the Docker container from reaching `DOCKER_GATEWAY:10800`.

```bash
# Find your Docker bridge interface
docker network ls
docker network inspect <your-compose-network-name> | grep Subnet
ip link show | grep br-
# Match network ID from 'docker network ls' to the br-XXXXXXXX interface
```

```bash
# Allow traffic from Docker network to the relay port
# Replace br-XXXXXXXX with your actual bridge interface
sudo ufw allow in on br-XXXXXXXX to 172.19.0.1 port 10800 proto tcp

# Verify
sudo ufw status verbose | grep 10800
```

```bash
# Final connectivity test from inside the container
docker exec <your-3x-ui-container> nc -zv 172.19.0.1 10800
# ✓ Expected: 172.19.0.1 (172.19.0.1:10800) open
```

---

### Step 7 — 3x-ui: Outbound + Routing Rule

#### 7.1 Add SOCKS5 Outbound

Navigate to **Outbounds → + Add Outbound**

| Field | Value |
|-------|-------|
| Protocol | `socks` |
| Tag | `remote-exit` |
| Address | `172.19.0.1` ← your `DOCKER_GATEWAY` |
| Port | `10800` |
| Username | *(leave empty)* |
| Password | *(leave empty)* |

Click **Create** → **Save**.

<details>
<summary>JSON equivalent</summary>

```json
{
  "tag": "remote-exit",
  "protocol": "socks",
  "settings": {
    "servers": [
      {
        "address": "172.19.0.1",
        "port": 10800,
        "users": []
      }
    ]
  }
}
```

</details>

#### 7.2 Add Routing Rule

Navigate to **Xray Configs → Routing → + Add Rule**

| Field | Value |
|-------|-------|
| Inbound Tags | select your target inbound (e.g. `inbound-1080`) |
| Outbound Tag | select `remote-exit` |
| Everything else | *(leave empty)* |

Click **Create** → **Save** → Xray restarts automatically.

> [!IMPORTANT]
> The new routing rule must be **above** `direct` and `block` rules in the list.  
> Drag it to the top if needed.

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

### Step 8 — End-to-End Verification

```bash
# Check all services on Entry Server
sudo systemctl status wg-quick@wg-server exit-relay

# Check WireGuard tunnel
sudo wg show wg-server
# latest handshake: X seconds ago ✓
# transfer: X KiB received, X KiB sent ✓

# Check all services on Exit Node
sudo systemctl status wg-quick@wg0 microsocks

# Full chain test — connect through the inbound and check exit IP
# Using curl with the inbound credentials (if auth is set):
curl --socks5 user:pass@YOUR_VPS_IP:1080 https://api.ipify.org
# ✓ Must return Exit Node IP
```

---

## ♻️ Startup Order

systemd dependencies are wired correctly — everything comes up automatically on reboot:

```
Entry Server
│
├── wg-quick@wg-server      ← starts first
│       │ (Requires)
│       ▼
└── exit-relay              ← starts only after WireGuard is up
        │
        ▼
    Docker / 3x-ui          ← independent, restart: unless-stopped


Exit Node
│
├── wg-quick@wg0            ← starts first, initiates tunnel to Entry Server
│       │ (Requires)
│       ▼
└── microsocks              ← starts only after WireGuard interface is up
```

---

## 🔧 Troubleshooting

<details>
<summary>❌ No handshake in <code>wg show</code></summary>

```bash
# Check Exit Node can reach Entry Server
ping YOUR_VPS_IP

# Check UDP 51820 is open on Entry Server
sudo ufw status | grep 51820

# Check WireGuard config on both sides — keys and endpoint
sudo cat /etc/wireguard/wg-server.conf  # Entry Server
sudo cat /etc/wireguard/wg0.conf        # Exit Node

# Restart WireGuard on both
sudo systemctl restart wg-quick@wg-server  # Entry Server
sudo systemctl restart wg-quick@wg0        # Exit Node
```

</details>

<details>
<summary>❌ <code>curl --socks5 10.99.0.2:10800</code> fails from Entry Server</summary>

```bash
# Check microsocks status on Exit Node
sudo systemctl status microsocks
sudo journalctl -u microsocks -n 30

# Check microsocks is listening on the WireGuard interface
ss -tlnp | grep 10800
# Must show: 10.99.0.2:10800 — NOT 0.0.0.0:10800 or 127.0.0.1:10800

# If listening on wrong address, edit the service:
sudo nano /etc/systemd/system/microsocks.service
# ExecStart=/usr/bin/microsocks -i 10.99.0.2 -p 10800
sudo systemctl daemon-reload && sudo systemctl restart microsocks
```

</details>

<details>
<summary>❌ <code>nc -zv DOCKER_GATEWAY 10800</code> returns "Host is unreachable" from container</summary>

```bash
# UFW is blocking — add the rule from Step 6
# First find the bridge interface
ip link show | grep br-
docker network inspect <network-name> | grep Subnet

# Then allow
sudo ufw allow in on br-XXXXXXXX to DOCKER_GATEWAY port 10800 proto tcp
```

</details>

<details>
<summary>❌ socat relay fails to start</summary>

```bash
sudo systemctl status exit-relay
sudo journalctl -u exit-relay -n 30

# Common cause: WireGuard is not up, no route to 10.99.0.2
ping 10.99.0.2
sudo wg show wg-server

# Fix: start WireGuard first, then relay
sudo systemctl restart wg-quick@wg-server
sudo systemctl restart exit-relay
```

</details>

<details>
<summary>❌ Traffic still exits through Entry Server IP</summary>

```bash
# Check routing rule order in 3x-ui
# Xray Configs → Routing
# The inbound → remote-exit rule must be ABOVE "direct" and "block" rules

# Check Xray config directly
docker exec <container> cat /usr/local/x-ui/bin/config.json | grep -A5 "routing"

# Verify outbound address is DOCKER_GATEWAY, not 10.99.0.2
# (Xray container can't reach 10.99.0.2 directly — that's why socat relay exists)
```

</details>

---

## ❓ FAQ

**Q: Can I use a protocol other than SOCKS5 for the outbound?**  
A: Yes. Once the WireGuard tunnel is up, you can run any proxy on the Exit Node (Xray VLESS, Shadowsocks, HTTP proxy, etc.) and use the matching outbound protocol in 3x-ui. SOCKS5 via `microsocks` is recommended for simplicity — it's a single binary with no config file.

**Q: What if the Exit Node reboots and gets a different IP?**  
A: No problem. The Exit Node connects *outward* to the Entry Server's fixed IP. The WireGuard handshake re-establishes as soon as the Exit Node is back online. `PersistentKeepalive = 25` ensures the tunnel recovers within ~25 seconds.

**Q: Can I add multiple Exit Nodes?**  
A: Yes. Add a new `[Peer]` block in `wg-server.conf` with a different `AllowedIPs` (e.g. `10.99.0.3/32`), run microsocks + socat on each, and create separate outbounds and routing rules in 3x-ui.

**Q: Why socat relay instead of `network_mode: host` in Docker?**  
A: Both work. `socat` relay is the minimal-change approach — no Docker Compose edits required. `network_mode: host` is cleaner architecturally (container sees WireGuard interface directly) but requires modifying the compose file and removing the `ports:` section.

**Q: Does this work without Docker (Xray running as systemd)?**  
A: Yes, and it's simpler. Skip the socat relay and UFW rule entirely — point the SOCKS5 outbound directly to `10.99.0.2:10800` in your Xray config.

---

## 📁 File Summary

```
Entry Server
├── /etc/wireguard/wg-server.conf         WireGuard server config
├── /etc/systemd/system/exit-relay.service socat relay service

Exit Node
├── /etc/wireguard/wg0.conf               WireGuard client config
├── /etc/systemd/system/microsocks.service microsocks service
```

---

<div align="center">

Made with ☕ · Pull requests welcome · [Open an Issue](https://github.com/)

</div>
