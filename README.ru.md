<div align="center">

# 🔀 Обратный туннель для 3x-ui

**Маршрутизация трафика через удалённый узел без белого IP**

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

🌐 **Язык:** [English](README.md) · Русский

</div>

---

## 📖 Описание

**Проблема:** 3x-ui работает в Docker на VPS (Основной сервер). Нужно, чтобы трафик определённого inbound выходил через другую машину (Узел выхода) — у которой **нет статического или белого IP**.

**Решение:** Узел выхода сам инициирует туннель к Основному серверу. Xray маршрутизирует нужные inbound через `socat` relay → туннель → `microsocks` на Узле выхода.

Рассматриваются два варианта туннеля:

| | Решение А · WireGuard | Решение Б · SSH Reverse |
|---|---|---|
| **Подходит для** | Домашний сервер, VPS, свободная сеть | Корпоративная сеть, NAT, жёсткий firewall |
| **Протокол** | UDP | TCP |
| **Порт на Основном сервере** | 51820/UDP | 22/TCP (уже открыт) |
| **Стабильность** | ★★★★★ | ★★★★☆ |
| **Сложность настройки** | Средняя | Низкая |
| **Чувствительность к firewall** | Высокая (UDP может блокироваться) | Низкая (TCP 22 проходит везде) |

> [!TIP]
> **Не знаешь что выбрать?** Начни с Решения А. Если WireGuard handshake не устанавливается (0 пакетов на Узле выхода) — сеть блокирует UDP. Переходи на Решение Б.

---

## 🔀 Архитектура

### Решение А — WireGuard

```
┌──────────────────────────────────────────────────────────┐
│               Основной сервер (VPS)                      │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │           Docker: 3x-ui / Xray                     │  │
│  │  [inbound] ──routing──► [remote-exit outbound]     │  │
│  │                         DOCKER_GATEWAY:10800        │  │
│  └─────────────────┬──────────────────────────────────┘  │
│                    │ Docker bridge                       │
│                    ▼                                     │
│         socat relay (хост)                              │
│         DOCKER_GATEWAY:10800 ──► 10.99.0.2:10800       │
│                                  │                      │
│         WireGuard wg-server      │                      │
│         слушает :51820/udp ◄─────┘ (WG туннель)        │
└──────────────────────────────────────────────────────┬──┘
                                                       │ WireGuard UDP
                                                       │ Узел выхода инициирует
                                                       ▼
                                       ┌───────────────────────┐
                                       │     Узел выхода       │
                                       │  wg0: 10.99.0.2       │
                                       │  microsocks :10800     │
                                       └──────────┬────────────┘
                                                  │
                                                  ▼
                                          Интернет (IP узла выхода)
```

### Решение Б — SSH Reverse Tunnel

```
┌──────────────────────────────────────────────────────────┐
│               Основной сервер (VPS)                      │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │           Docker: 3x-ui / Xray                     │  │
│  │  [inbound] ──routing──► [remote-exit outbound]     │  │
│  │                         DOCKER_GATEWAY:10800        │  │
│  └─────────────────┬──────────────────────────────────┘  │
│                    │ Docker bridge                       │
│                    ▼                                     │
│         socat relay (хост)                              │
│         DOCKER_GATEWAY:10800 ──► 127.0.0.1:10800       │
│                                  │                      │
│         sshd открывает 127.0.0.1:10800 ◄── SSH туннель │
└──────────────────────────────────────────────────────┬──┘
                                                       │ SSH TCP (порт 22)
                                                       │ Узел выхода инициирует
                                                       ▼
                                       ┌───────────────────────┐
                                       │     Узел выхода       │
                                       │  autossh reverse fwd  │
                                       │  microsocks :10800     │
                                       │  (только 127.0.0.1)   │
                                       └──────────┬────────────┘
                                                  │
                                                  ▼
                                          Интернет (IP узла выхода)
```

---

## ✅ Требования

| | Основной сервер | Узел выхода |
|---|---|---|
| **ОС** | Ubuntu 22.04+ | Ubuntu 22.04+ |
| **Сеть** | Публичный IP, Docker + 3x-ui запущен | Только выход в интернет |
| **Открытые порты** | 22/TCP, 443/TCP + **51820/UDP** *(только WG)* | Не нужны |

---

## 📋 Справочник переменных

| Переменная | Описание | Как найти |
|-----------|----------|-----------|
| `YOUR_VPS_IP` | Публичный IP основного сервера | `curl ifconfig.me` |
| `DOCKER_GATEWAY` | Gateway Docker bridge сети | `docker exec <контейнер> ip route \| grep default` → второе поле |
| `BR_IFACE` | Имя Docker bridge интерфейса | `ip link show \| grep br-` |
| `INBOUND_TAG` | Тег inbound в 3x-ui | Виден в JSON конфиге Xray, например `inbound-1080` |

<details>
<summary>💡 Как найти DOCKER_GATEWAY и BR_IFACE</summary>

```bash
# DOCKER_GATEWAY — выполнить на основном сервере
docker exec <имя-контейнера-3x-ui> ip route | grep default
# Пример: default via 172.19.0.1 dev eth0
#                     ^^^^^^^^^^ = DOCKER_GATEWAY

# BR_IFACE — сопоставить по имени сети
docker network ls                                          # найти сеть compose
docker network inspect <имя-сети> | grep Subnet           # подтвердить подсеть
ip link show | grep br-                                    # br-XXXXXXXX = BR_IFACE
```

</details>

---

## 🛡️ Решение А: WireGuard

> Используй когда Узел выхода на домашней сети, VPS или любой сети без блокировки UDP.

### А1 — Основной сервер: WireGuard сервер

```bash
sudo apt update && sudo apt install -y wireguard

# Генерация ключей
sudo bash -c "
  cd /etc/wireguard
  wg genkey | tee server_private.key | wg pubkey > server_public.key
  chmod 600 server_private.key
"

# Публичный ключ — скопируй, нужен для Узла выхода (Шаг А2)
sudo cat /etc/wireguard/server_public.key
```

```bash
# Создать конфиг
SERVER_PRIVKEY=$(sudo cat /etc/wireguard/server_private.key)
sudo bash -c "cat > /etc/wireguard/wg-server.conf << EOF
[Interface]
PrivateKey = ${SERVER_PRIVKEY}
Address = 10.99.0.1/24
ListenPort = 51820

[Peer]
# Заполнить после Шага А2
PublicKey = PLACEHOLDER
AllowedIPs = 10.99.0.2/32
PersistentKeepalive = 25
EOF"

sudo ufw allow 51820/udp
# Пока не запускаем — нужен ключ Узла выхода
```

### А2 — Узел выхода: WireGuard клиент + microsocks

```bash
sudo apt update && sudo apt install -y wireguard microsocks

# Генерация ключей
sudo bash -c "
  cd /etc/wireguard
  wg genkey | tee client_private.key | wg pubkey | tee client_public.key
  chmod 600 client_private.key
"

# Публичный ключ — скопируй на Основной сервер
sudo cat /etc/wireguard/client_public.key
```

```bash
# Создать конфиг WireGuard
CLIENT_PRIVKEY=$(sudo cat /etc/wireguard/client_private.key)
sudo bash -c "cat > /etc/wireguard/wg0.conf << EOF
[Interface]
PrivateKey = ${CLIENT_PRIVKEY}
Address = 10.99.0.2/24

[Peer]
PublicKey = PUBLIC_KEY_ОСНОВНОГО_СЕРВЕРА
Endpoint = YOUR_VPS_IP:51820
AllowedIPs = 10.99.0.1/32
PersistentKeepalive = 25
EOF"

sudo nano /etc/wireguard/wg0.conf  # вставить реальные значения
```

```bash
# microsocks — слушает на WireGuard интерфейсе
sudo tee /etc/systemd/system/microsocks.service << 'EOF'
[Unit]
Description=microsocks SOCKS5 прокси (WireGuard)
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

### А3 — Обмен ключами

```bash
# На Основном сервере — вставить публичный ключ Узла выхода
EXIT_PUBKEY="ВСТАВЬ_КЛЮЧ_УЗЛА_ВЫХОДА"
sudo sed -i "s|PublicKey = PLACEHOLDER|PublicKey = ${EXIT_PUBKEY}|" \
  /etc/wireguard/wg-server.conf
sudo cat /etc/wireguard/wg-server.conf  # проверить
```

### А4 — Запуск и проверка туннеля

```bash
# Основной сервер
sudo systemctl enable --now wg-quick@wg-server

# Узел выхода
sudo systemctl start wg-quick@wg0
sudo systemctl start microsocks
```

```bash
# Проверка с Основного сервера — все три должны пройти:
sudo wg show wg-server                                   # latest handshake: X seconds ago ✓
curl --socks5 10.99.0.2:10800 https://api.ipify.org     # должен вернуть IP узла выхода ✓
```

> [!WARNING]
> Если handshake есть но `curl` зависает, а `tcpdump -i wg0` на Узле выхода показывает 0 пакетов — сеть блокирует входящий UDP. Переходи на **Решение Б**.

### А5 — socat relay (Основной сервер)

```bash
sudo apt install -y socat

# Замени 172.19.0.1 на свой DOCKER_GATEWAY
sudo tee /etc/systemd/system/exit-relay.service << 'EOF'
[Unit]
Description=socat: Docker bridge → WireGuard → microsocks Узла выхода
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

ss -tlnp | grep 10800  # должно быть 172.19.0.1:10800 LISTEN
```

---

## 🔑 Решение Б: SSH Reverse Tunnel

> Используй когда Узел выхода за корпоративным firewall, строгим NAT или любой сетью, блокирующей UDP.

### Б1 — Узел выхода: SSH ключ

```bash
# Установка пакетов
sudo apt update && sudo apt install -y autossh microsocks

# Генерация выделенного SSH ключа для туннеля
ssh-keygen -t ed25519 -f ~/.ssh/entry_tunnel -N ""

# Публичный ключ — скопируй на Основной сервер (Шаг Б2)
cat ~/.ssh/entry_tunnel.pub
```

### Б2 — Основной сервер: авторизация ключа

```bash
# Добавить публичный ключ Узла выхода
echo "ВСТАВЬ_ПУБЛИЧНЫЙ_КЛЮЧ_УЗЛА_ВЫХОДА" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Проверить SSH соединение вручную перед следующим шагом:
# ssh -i ~/.ssh/entry_tunnel ubuntu@YOUR_VPS_IP
```

> [!NOTE]
> `GatewayPorts` в sshd_config **не нужен** — socat подключается к `127.0.0.1:10800` на том же хосте где работает sshd.

### Б3 — Узел выхода: microsocks + autossh

```bash
# microsocks только на localhost (не выставляется наружу)
sudo tee /etc/systemd/system/microsocks.service << 'EOF'
[Unit]
Description=microsocks SOCKS5 прокси (SSH туннель)
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
# autossh — обратный туннель
# Замени YOUR_VPS_IP и ubuntu на реальные значения
sudo tee /etc/systemd/system/ssh-tunnel.service << 'EOF'
[Unit]
Description=SSH обратный туннель к Основному серверу
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

### Б4 — Основной сервер: socat relay

```bash
sudo apt install -y socat

# Замени 172.19.0.1 на свой DOCKER_GATEWAY
# Цель — 127.0.0.1 (SSH туннель), а не 10.99.0.2
sudo tee /etc/systemd/system/exit-relay.service << 'EOF'
[Unit]
Description=socat: Docker bridge → SSH туннель → microsocks Узла выхода
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

ss -tlnp | grep 10800  # должно быть 172.19.0.1:10800 LISTEN
```

### Б5 — Проверка SSH туннеля

```bash
# Основной сервер — порт SSH туннеля должен появиться в течение нескольких секунд
ss -tlnp | grep 10800
# Ожидаемо: 127.0.0.1:10800  LISTEN  (открыт sshd)

# Полная проверка цепочки
curl --socks5 127.0.0.1:10800 https://api.ipify.org
# ✓ Должен вернуть IP Узла выхода
```

---

## ⚙️ Общее: UFW правило (оба решения)

Docker контейнер не может достучаться до `DOCKER_GATEWAY:10800` без явного правила UFW.

```bash
# Найти bridge интерфейс Docker сети
docker network ls
docker network inspect <имя-сети-compose> | grep Subnet
ip link show | grep br-
```

```bash
# Разрешить трафик из Docker сети на socat relay
# Замени br-XXXXXXXX на реальное имя интерфейса
sudo ufw allow in on br-XXXXXXXX to 172.19.0.1 port 10800 proto tcp

# Проверить из контейнера
docker exec <контейнер-3x-ui> nc -zv 172.19.0.1 10800
# ✓ Ожидаемо: open
```

---

## ⚙️ Общее: Настройка 3x-ui (оба решения)

Конфигурация 3x-ui **одинакова** для обоих решений — адрес `DOCKER_GATEWAY:10800` не меняется.

### Добавить SOCKS5 outbound

**Исходящие → + Исходящие**

| Поле | Значение |
|------|----------|
| Протокол | `socks` |
| Тег | `remote-exit` |
| Адрес | `172.19.0.1` ← твой `DOCKER_GATEWAY` |
| Порт | `10800` |
| Имя пользователя / Пароль | *(оставить пустым)* |

Нажать **Создать** → **Сохранить**.

<details>
<summary>JSON эквивалент</summary>

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

### Добавить routing rule

**Конфигурации Xray → Маршрутизация → + Маршрутизация**

| Поле | Значение |
|------|----------|
| Теги входящих | выбери нужный inbound из дропдауна |
| Тег исходящего | выбери `remote-exit` |
| Остальное | *(оставить пустым)* |

Нажать **Создать** → **Сохранить** → Xray перезапустится.

> [!IMPORTANT]
> Новое правило должно стоять **выше** `direct` и `block` в списке правил.

<details>
<summary>JSON эквивалент</summary>

```json
{
  "type": "field",
  "inboundTag": ["inbound-1080"],
  "outboundTag": "remote-exit"
}
```

</details>

---

## ✅ Финальная проверка

```bash
# Основной сервер — все сервисы работают
sudo systemctl status exit-relay

# Только Решение А
sudo wg show wg-server          # latest handshake: недавно ✓

# Только Решение Б
ss -tlnp | grep 10800           # 127.0.0.1:10800 LISTEN ✓

# Контейнер достигает relay
docker exec <контейнер-3x-ui> nc -zv 172.19.0.1 10800   # open ✓

# Сквозной тест
curl --socks5 user:pass@YOUR_VPS_IP:INBOUND_PORT https://api.ipify.org
# ✓ Возвращает IP Узла выхода, а не Основного сервера
```

---

## ♻️ Порядок запуска при ребуте

### Решение А — WireGuard

```
Узел выхода                        Основной сервер
─────────────────────              ──────────────────────────────────
wg-quick@wg0                       wg-quick@wg-server
  │ (Requires)                       │ (Requires)
  ▼                                  ▼
microsocks                         exit-relay
(10.99.0.2:10800)                  (172.19.0.1:10800 → 10.99.0.2:10800)
                                     │
                                     ▼
                                   Docker 3x-ui  (restart: unless-stopped)
```

### Решение Б — SSH Reverse Tunnel

```
Узел выхода                        Основной сервер
─────────────────────              ──────────────────────────────────
microsocks (127.0.0.1:10800)       sshd (всегда запущен)
  │ (Wants)                          │ ← SSH подключение от Узла выхода
  ▼                                  │   открывает 127.0.0.1:10800
ssh-tunnel (autossh) ─────────────►│
                                     ▼
                                   exit-relay
                                   (172.19.0.1:10800 → 127.0.0.1:10800)
                                     │
                                     ▼
                                   Docker 3x-ui  (restart: unless-stopped)
```

---

## 🛡️ Watchdog (рекомендуется для Решения А)

WireGuard под NAT может терять сессию. Скрипт восстанавливает туннель автоматически:

```bash
# Узел выхода
sudo tee /usr/local/bin/wg-watchdog.sh << 'EOF'
#!/bin/bash
LAST=$(sudo wg show wg0 latest-handshakes | awk '{print $2}')
DIFF=$(( $(date +%s) - LAST ))
if [ "$DIFF" -gt 180 ]; then
    logger "wg-watchdog: handshake устарел (${DIFF}с), перезапуск"
    systemctl restart wg-quick@wg0
fi
EOF

sudo chmod +x /usr/local/bin/wg-watchdog.sh
(sudo crontab -l 2>/dev/null; echo "*/2 * * * * /usr/local/bin/wg-watchdog.sh") \
  | sudo crontab -
```

> Решение Б (SSH) не требует watchdog — autossh сам обрабатывает переподключение.

---

## 🔄 Миграция: WireGuard → SSH

Если UDP заблокирован в сети Узла выхода — сносим WireGuard, поднимаем SSH:

### Очистка на Основном сервере

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

### Очистка на Узле выхода

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

После очистки следуй шагам **Б1–Б5**. UFW правило и конфигурация 3x-ui остаются без изменений.

---

## 🔧 Устранение неполадок

<details>
<summary>❌ [WG] Нет handshake после нескольких минут</summary>

```bash
# Проверить что UDP 51820 открыт на Основном сервере
sudo ufw status | grep 51820

# Проверить что Узел выхода видит Основной сервер
ping -c3 YOUR_VPS_IP

# Проверить конфиги — ключи должны совпадать крест-накрест
sudo cat /etc/wireguard/wg-server.conf   # Основной сервер
sudo cat /etc/wireguard/wg0.conf         # Узел выхода

# Перезапустить WireGuard на обоих
sudo systemctl restart wg-quick@wg-server   # Основной сервер
sudo systemctl restart wg-quick@wg0         # Узел выхода
```

</details>

<details>
<summary>❌ [WG] Handshake есть, но curl зависает (tcpdump показывает 0 пакетов)</summary>

Сеть блокирует **входящий UDP**. Узел выхода может отправлять пакеты, но не получать ответы.  
→ **Мигрируй на Решение Б (SSH).**

</details>

<details>
<summary>❌ [SSH] 127.0.0.1:10800 не слушает на Основном сервере</summary>

```bash
# Проверить статус ssh-tunnel на Узле выхода
sudo systemctl status ssh-tunnel
sudo journalctl -u ssh-tunnel -n 30

# Проверить SSH вручную с Узла выхода
ssh -i ~/.ssh/entry_tunnel -N -R 10800:127.0.0.1:10800 ubuntu@YOUR_VPS_IP
# Если ошибка — проверить authorized_keys на Основном сервере

cat ~/.ssh/authorized_keys | grep -c "ssh-"   # должно быть >= 1
```

</details>

<details>
<summary>❌ nc -zv DOCKER_GATEWAY 10800 возвращает "Host is unreachable" из контейнера</summary>

```bash
# Не хватает UFW правила — добавить
sudo ufw allow in on br-XXXXXXXX to DOCKER_GATEWAY port 10800 proto tcp

# Проверить
docker exec <контейнер> nc -zv DOCKER_GATEWAY 10800
```

</details>

<details>
<summary>❌ curl возвращает IP Основного сервера вместо Узла выхода</summary>

```bash
# Проверить порядок правил в 3x-ui
# Конфигурации Xray → Маршрутизация
# Правило с нужным inbound → remote-exit должно стоять ВЫШЕ direct и block

# Адрес outbound должен быть DOCKER_GATEWAY, а не 10.99.0.2 или 127.0.0.1
# Контейнер Xray не имеет к ним прямого доступа — для этого и нужен socat relay
```

</details>

---

## ❓ Частые вопросы

**Q: Можно добавить несколько Узлов выхода с разными inbound?**  
A: Да. Каждый Узел выхода получает свой туннель (WG peer или SSH на другом порту), свой socat relay (другой порт биндинга), свой outbound и routing rule в 3x-ui.

**Q: Можно использовать другой протокол вместо SOCKS5?**  
A: Да. Как только туннель поднят, на Узле выхода можно запустить любой прокси (Xray VLESS, Shadowsocks, HTTP CONNECT и т.д.) и использовать соответствующий outbound в 3x-ui. microsocks рекомендуется за простоту — один бинарник без конфигурационных файлов.

**Q: Работает ли это без Docker (Xray как systemd)?**  
A: Да, и проще. Пропусти socat relay и UFW шаг — укажи outbound напрямую на `10.99.0.2:10800` (WG) или `127.0.0.1:10800` (SSH) в конфиге Xray.

**Q: Узел выхода перезагрузился и получил другой IP — туннель сломается?**  
A: Нет. Узел выхода всегда сам подключается к фиксированному IP Основного сервера. Туннель восстанавливается автоматически в течение нескольких секунд.

---

## 📁 Карта файлов

```
Основной сервер
├── /etc/systemd/system/exit-relay.service    socat relay (оба решения)
├── /etc/wireguard/wg-server.conf             конфиг WireGuard сервера (Решение А)
└── ~/.ssh/authorized_keys                    ключ Узла выхода (Решение Б)

Узел выхода
├── /etc/systemd/system/microsocks.service    microsocks SOCKS5 прокси
├── /etc/wireguard/wg0.conf                   конфиг WireGuard клиента (Решение А)
├── /usr/local/bin/wg-watchdog.sh             watchdog скрипт (Решение А)
├── /etc/systemd/system/ssh-tunnel.service    autossh туннель (Решение Б)
└── ~/.ssh/entry_tunnel                        SSH приватный ключ (Решение Б)
```

---

<div align="center">

Сделано с ☕ · Pull requests приветствуются · [Открыть Issue](https://github.com/)

</div>
