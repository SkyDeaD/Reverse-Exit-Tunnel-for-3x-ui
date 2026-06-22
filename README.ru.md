<div align="center">

# 🔀 Обратный туннель для 3x-ui

**Маршрутизация трафика через удалённый узел без белого IP**

*WireGuard · microsocks · socat · Docker · 3x-ui*

<br>

[![WireGuard](https://img.shields.io/badge/WireGuard-88171A?style=flat-square&logo=wireguard&logoColor=white)](https://www.wireguard.com/)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)](https://www.docker.com/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu_22.04+-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com/)
[![3x-ui](https://img.shields.io/badge/3x--ui-latest-blue?style=flat-square)](https://github.com/MHSanaei/3x-ui)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat-square)](https://github.com/)

<br>

🌐 **Язык:** [English](README.md) · Русский

</div>

---

## 📖 Описание задачи

**Проблема:** 3x-ui работает в Docker на VPS. Нужно, чтобы трафик определённого inbound выходил в интернет через другую машину — у которой **нет статического или белого IP**.

**Решение:** Узел выхода сам инициирует WireGuard туннель к основному серверу. Xray маршрутизирует выбранные inbound через `socat` relay → WireGuard → `microsocks` на узле выхода.

```
Клиент → [Основной сервер: 3x-ui inbound] → [socat relay] ──WireGuard──► [Узел выхода: microsocks] → Интернет
```

> [!NOTE]
> Узел выхода никогда не принимает входящие соединения снаружи.  
> Он сам инициирует туннель — NAT, CGNAT, динамический IP: всё работает.

---

## 🏗️ Архитектура

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Основной сервер (VPS)                            │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │               Docker контейнер: 3x-ui / Xray              │     │
│  │                                                            │     │
│  │  [inbound: INBOUND_TAG] ──routing rule──► [remote-exit]  │     │
│  │                             (SOCKS5 outbound               │     │
│  │                              DOCKER_GATEWAY:RELAY_PORT)    │     │
│  └───────────────────────┬────────────────────────────────────┘     │
│                          │  Docker bridge сеть                      │
│                          ▼                                          │
│               socat relay (systemd, на хосте)                      │
│               DOCKER_GATEWAY:RELAY_PORT ──────────────────────►    │
│                                             WireGuard (wg-server)   │
│                                             :51820/udp              │
└─────────────────────────────────────────────────────────────────────┘
                                             │
                                  WireGuard туннель (UDP)
                                  Узел выхода инициирует соединение
                                             │
                                             ▼
                             ┌──────────────────────────────┐
                             │        Узел выхода            │
                             │                              │
                             │  wg0: WG_CLIENT_ADDR         │
                             │  microsocks: PROXY_PORT       │
                             └───────────────┬──────────────┘
                                             │
                                             ▼
                                    Интернет (IP узла выхода)
```

### Путь трафика

```
Клиент
  │  подключается к inbound на основном сервере
  ▼
3x-ui inbound  (тег: INBOUND_TAG)
  │  routing rule → outbound: remote-exit
  ▼
SOCKS5 outbound → DOCKER_GATEWAY:RELAY_PORT
  │  Docker bridge
  ▼
socat relay  (запущен на хосте VPS, не в контейнере)
  │  WireGuard туннель
  ▼
microsocks  (Узел выхода: WG_CLIENT_ADDR:PROXY_PORT)
  │
  ▼
Интернет  ← IP = IP узла выхода ✓
```

---

## ✅ Требования

| Компонент | Где | Что нужно |
|-----------|-----|-----------|
| **Основной сервер** | VPS с публичным IP | Ubuntu 22.04+, Docker, 3x-ui запущен |
| **Узел выхода** | Любая машина | Ubuntu 22.04+, доступ в интернет, **белый IP не нужен** |

> [!IMPORTANT]
> На основном сервере должен быть открыт **UDP порт 51820** (WireGuard).

---

## 📋 Справочник переменных

Замени эти плейсхолдеры на реальные значения по ходу гайда:

| Переменная | Описание | Как найти |
|-----------|----------|-----------|
| `YOUR_VPS_IP` | Публичный IP основного сервера | `curl ifconfig.me` на основном сервере |
| `DOCKER_GATEWAY` | Gateway Docker bridge сети | `docker exec <контейнер> ip route \| grep default` |
| `BR_IFACE` | Имя Docker bridge интерфейса | `ip link show \| grep br-` |
| `WG_SERVER_ADDR` | WireGuard IP основного сервера | Используй `10.99.0.1` (пример) |
| `WG_CLIENT_ADDR` | WireGuard IP узла выхода | Используй `10.99.0.2` (пример) |
| `PROXY_PORT` | Порт microsocks | Используй `10800` (пример) |
| `INBOUND_TAG` | Тег inbound в 3x-ui | Виден в JSON конфиге Xray |
| `OUTBOUND_TAG` | Имя нового outbound | Используй `remote-exit` (пример) |

<details>
<summary>💡 Как найти DOCKER_GATEWAY и BR_IFACE</summary>

```bash
# Найти Docker gateway (выполнить на основном сервере)
docker exec <имя-контейнера-3x-ui> ip route | grep default
# Пример вывода: default via 172.19.0.1 dev eth0
#                                ^^^^^^^^^^^ это и есть DOCKER_GATEWAY

# Найти имя bridge интерфейса
docker network ls
docker network inspect <имя-сети-compose> | grep Subnet
ip link show | grep br-
# Сопоставь ID сети из 'docker network ls' с интерфейсом br-XXXXXXXXX
```

</details>

---

## 🚀 Установка

### Шаг 1 — Основной сервер: WireGuard

```bash
# Установка
sudo apt update && sudo apt install -y wireguard

# Генерация ключей
sudo bash -c "
  cd /etc/wireguard
  wg genkey | tee server_private.key | wg pubkey > server_public.key
  chmod 600 server_private.key
"

# Публичный ключ основного сервера — скопируй, нужен для узла выхода
sudo cat /etc/wireguard/server_public.key
```

Создать `/etc/wireguard/wg-server.conf`:

```bash
SERVER_PRIVKEY=$(sudo cat /etc/wireguard/server_private.key)

sudo bash -c "cat > /etc/wireguard/wg-server.conf << EOF
[Interface]
PrivateKey = ${SERVER_PRIVKEY}
Address = 10.99.0.1/24
ListenPort = 51820

[Peer]
# Узел выхода — PublicKey заполним после Шага 2
PublicKey = PLACEHOLDER
AllowedIPs = 10.99.0.2/32
PersistentKeepalive = 25
EOF"

# Проверить конфиг
sudo cat /etc/wireguard/wg-server.conf
```

```bash
# Открыть порт
sudo ufw allow 51820/udp

# WireGuard пока НЕ запускаем — нужен публичный ключ узла выхода
```

---

### Шаг 2 — Узел выхода: WireGuard + microsocks

```bash
# Установка
sudo apt update && sudo apt install -y wireguard microsocks

# Генерация ключей
sudo bash -c "
  cd /etc/wireguard
  wg genkey | tee client_private.key | wg pubkey | tee client_public.key
  chmod 600 client_private.key
"

# Публичный ключ узла выхода — скопируй на основной сервер
sudo cat /etc/wireguard/client_public.key
```

Создать `/etc/wireguard/wg0.conf`:

```bash
CLIENT_PRIVKEY=$(sudo cat /etc/wireguard/client_private.key)

sudo bash -c "cat > /etc/wireguard/wg0.conf << EOF
[Interface]
PrivateKey = ${CLIENT_PRIVKEY}
Address = 10.99.0.2/24

[Peer]
PublicKey = ВСТАВИТЬ_PUBLIC_KEY_ОСНОВНОГО_СЕРВЕРА
Endpoint = YOUR_VPS_IP:51820
AllowedIPs = 10.99.0.1/32
PersistentKeepalive = 25
EOF"

# Вставить реальные значения
sudo nano /etc/wireguard/wg0.conf
# Заменить: ВСТАВИТЬ_PUBLIC_KEY_ОСНОВНОГО_СЕРВЕРА и YOUR_VPS_IP
```

Создать systemd сервис для microsocks:

```bash
sudo tee /etc/systemd/system/microsocks.service << 'EOF'
[Unit]
Description=microsocks SOCKS5 прокси для обратного туннеля
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
# Пока не запускаем — сначала обмен ключами
```

---

### Шаг 3 — Обмен ключами

**На основном сервере** — вставить публичный ключ узла выхода:

```bash
EXIT_PUBKEY="ВСТАВЬ_СЮДА_PUBLIC_KEY_УЗЛА_ВЫХОДА"

sudo sed -i \
  "s|PublicKey = PLACEHOLDER|PublicKey = ${EXIT_PUBKEY}|" \
  /etc/wireguard/wg-server.conf

# Проверить результат
sudo cat /etc/wireguard/wg-server.conf
```

Ожидаемый итоговый конфиг основного сервера:

```ini
[Interface]
PrivateKey = <приватный ключ основного сервера>
Address = 10.99.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <публичный ключ узла выхода>
AllowedIPs = 10.99.0.2/32
PersistentKeepalive = 25
```

---

### Шаг 4 — Запуск туннеля

**На основном сервере:**

```bash
sudo systemctl enable --now wg-quick@wg-server
sudo wg show wg-server
```

**На узле выхода:**

```bash
sudo systemctl start wg-quick@wg0
sudo systemctl start microsocks

# Проверить WireGuard интерфейс
ip addr show wg0
# Ожидаемо: inet 10.99.0.2/24

# Проверить microsocks
ss -tlnp | grep 10800
# Ожидаемо: 10.99.0.2:10800  LISTEN
```

**Проверка туннеля (с основного сервера):**

```bash
# 1. Появился handshake?
sudo wg show wg-server
# Ищи: latest handshake: X seconds ago ✓

# 2. Пинг до узла выхода
ping -c3 10.99.0.2

# 3. Главная проверка — трафик выходит через IP узла выхода?
curl --socks5 10.99.0.2:10800 https://api.ipify.org
# ✓ Должен вернуть IP узла выхода, а не IP основного сервера
```

> [!WARNING]
> Если проверка №3 не прошла — туннель сломан. Не продолжай.  
> Смотри раздел [Устранение неполадок](#-устранение-неполадок).

---

### Шаг 5 — Основной сервер: socat relay

Мост между Docker сетью (`DOCKER_GATEWAY`) и WireGuard (`WG_CLIENT_ADDR`).

```
Docker контейнер → DOCKER_GATEWAY:10800 ──socat──► 10.99.0.2:10800 → microsocks
```

```bash
sudo apt install -y socat
```

```bash
# Замени 172.19.0.1 на твой реальный DOCKER_GATEWAY
sudo tee /etc/systemd/system/exit-relay.service << 'EOF'
[Unit]
Description=socat relay: Docker bridge → WireGuard → microsocks узла выхода
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

# Проверить что relay слушает
ss -tlnp | grep 10800
# Ожидаемо: 172.19.0.1:10800  LISTEN  socat
```

---

### Шаг 6 — UFW: разрешить Docker → relay

По умолчанию `deny (incoming)` в ufw блокирует трафик из Docker контейнера на `DOCKER_GATEWAY:10800`.

```bash
# Найти bridge интерфейс Docker сети
docker network ls
docker network inspect <имя-сети-compose> | grep Subnet
ip link show | grep br-
# Сопоставь ID сети с интерфейсом br-XXXXXXXX
```

```bash
# Разрешить трафик из Docker сети на relay порт
# Замени br-XXXXXXXX на реальное имя интерфейса
sudo ufw allow in on br-XXXXXXXX to 172.19.0.1 port 10800 proto tcp

# Проверить
sudo ufw status verbose | grep 10800
```

```bash
# Финальная проверка из контейнера
docker exec <имя-контейнера-3x-ui> nc -zv 172.19.0.1 10800
# ✓ Ожидаемо: 172.19.0.1 (172.19.0.1:10800) open
```

---

### Шаг 7 — 3x-ui: Outbound + Routing Rule

#### 7.1 Добавить SOCKS5 outbound

**Исходящие → + Исходящие**

| Поле | Значение |
|------|----------|
| Протокол | `socks` |
| Тег | `remote-exit` |
| Адрес | `172.19.0.1` ← твой `DOCKER_GATEWAY` |
| Порт | `10800` |
| Имя пользователя | *(оставить пустым)* |
| Пароль | *(оставить пустым)* |

Нажать **Создать** → **Сохранить**.

<details>
<summary>JSON эквивалент</summary>

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

#### 7.2 Добавить routing rule

**Конфигурации Xray → Маршрутизация → + Маршрутизация**

| Поле | Значение |
|------|----------|
| Теги входящих | выбери нужный inbound из дропдауна |
| Тег исходящего | выбери `remote-exit` |
| Остальное | *(оставить пустым)* |

Нажать **Создать** → **Сохранить** → Xray перезапустится автоматически.

> [!IMPORTANT]
> Новое правило должно стоять **выше** правил `direct` и `blocked` в списке.  
> Перетащи его вверх если нужно.

<details>
<summary>JSON эквивалент</summary>

```json
{
  "type": "field",
  "inboundTag": ["твой-inbound-тег"],
  "outboundTag": "remote-exit"
}
```

</details>

---

### Шаг 8 — Финальная проверка всей цепочки

```bash
# Статус всех сервисов на основном сервере
sudo systemctl status wg-quick@wg-server exit-relay

# Состояние WireGuard туннеля
sudo wg show wg-server
# latest handshake: X seconds ago ✓
# transfer: X KiB received, X KiB sent ✓

# Статус на узле выхода
sudo systemctl status wg-quick@wg0 microsocks

# Сквозной тест — подключиться через inbound и проверить IP выхода
curl --socks5 user:pass@YOUR_VPS_IP:PORT https://api.ipify.org
# ✓ Должен вернуть IP узла выхода
```

---

## ♻️ Порядок запуска при ребуте

Зависимости systemd выстроены правильно — всё поднимается автоматически:

```
Основной сервер
│
├── wg-quick@wg-server      ← стартует первым
│       │ (Requires)
│       ▼
└── exit-relay              ← стартует только после поднятия WireGuard
        │
        ▼
    Docker / 3x-ui          ← независимо, restart: unless-stopped


Узел выхода
│
├── wg-quick@wg0            ← стартует первым, инициирует туннель
│       │ (Requires)
│       ▼
└── microsocks              ← стартует только после поднятия WG интерфейса
```

---

## 🔧 Устранение неполадок

<details>
<summary>❌ Нет handshake в выводе <code>wg show</code></summary>

```bash
# Проверить что узел выхода видит основной сервер
ping YOUR_VPS_IP

# Проверить что UDP 51820 открыт на основном сервере
sudo ufw status | grep 51820

# Проверить конфиги на обоих машинах — ключи и endpoint
sudo cat /etc/wireguard/wg-server.conf  # основной сервер
sudo cat /etc/wireguard/wg0.conf        # узел выхода

# Перезапустить WireGuard на обоих
sudo systemctl restart wg-quick@wg-server  # основной сервер
sudo systemctl restart wg-quick@wg0        # узел выхода
```

</details>

<details>
<summary>❌ <code>curl --socks5 10.99.0.2:10800</code> не работает с основного сервера</summary>

```bash
# Проверить статус microsocks на узле выхода
sudo systemctl status microsocks
sudo journalctl -u microsocks -n 30

# Проверить что microsocks слушает на WG интерфейсе
ss -tlnp | grep 10800
# Должно быть: 10.99.0.2:10800 — НЕ 0.0.0.0 и НЕ 127.0.0.1

# Если слушает на неправильном адресе — исправить сервис:
sudo nano /etc/systemd/system/microsocks.service
# ExecStart=/usr/bin/microsocks -i 10.99.0.2 -p 10800
sudo systemctl daemon-reload && sudo systemctl restart microsocks
```

</details>

<details>
<summary>❌ <code>nc -zv DOCKER_GATEWAY 10800</code> возвращает "Host is unreachable" из контейнера</summary>

```bash
# UFW блокирует — добавить правило из Шага 6
# Сначала найти bridge интерфейс
ip link show | grep br-
docker network inspect <имя-сети> | grep Subnet

# Добавить разрешение
sudo ufw allow in on br-XXXXXXXX to DOCKER_GATEWAY port 10800 proto tcp
```

</details>

<details>
<summary>❌ socat relay не запускается</summary>

```bash
sudo systemctl status exit-relay
sudo journalctl -u exit-relay -n 30

# Частая причина: WireGuard не поднят, нет маршрута к 10.99.0.2
ping 10.99.0.2
sudo wg show wg-server

# Сначала поднять WireGuard, потом relay
sudo systemctl restart wg-quick@wg-server
sudo systemctl restart exit-relay
```

</details>

<details>
<summary>❌ Трафик всё равно выходит через IP основного сервера</summary>

```bash
# Проверить порядок правил в 3x-ui
# Конфигурации Xray → Маршрутизация
# Правило inbound → remote-exit должно стоять ВЫШЕ direct и blocked

# Проверить что outbound указывает на DOCKER_GATEWAY, а не на 10.99.0.2
# (Xray контейнер не может напрямую достучаться до 10.99.0.2 — для этого и нужен socat relay)
```

</details>

---

## ❓ Частые вопросы

**Q: Можно ли использовать другой протокол вместо SOCKS5 для outbound?**  
A: Да. Как только WireGuard туннель поднят, на узле выхода можно запустить любой прокси (Xray VLESS, Shadowsocks, HTTP proxy и т.д.) и использовать соответствующий протокол outbound в 3x-ui. SOCKS5 через `microsocks` рекомендуется за простоту — это один бинарник без конфигурационных файлов.

**Q: Что если узел выхода перезагрузится и получит другой IP?**  
A: Всё работает. Узел выхода сам подключается к фиксированному IP основного сервера. WireGuard восстанавливает handshake как только узел выходит в онлайн. `PersistentKeepalive = 25` гарантирует восстановление туннеля в течение ~25 секунд.

**Q: Можно ли добавить несколько узлов выхода?**  
A: Да. Добавь новый блок `[Peer]` в `wg-server.conf` с другим `AllowedIPs` (например `10.99.0.3/32`), запусти microsocks + socat на каждом узле, создай отдельные outbound и routing rules в 3x-ui.

**Q: Почему socat relay, а не `network_mode: host` в Docker?**  
A: Оба варианта работают. `socat` relay — минимальное изменение, не требует правки docker-compose. `network_mode: host` архитектурно чище (контейнер напрямую видит WireGuard интерфейс), но требует изменения compose файла и удаления секции `ports:`.

**Q: Работает ли это без Docker (Xray запущен как systemd)?**  
A: Да, и даже проще. Пропусти шаги с socat relay и UFW правилом — укажи SOCKS5 outbound напрямую на `10.99.0.2:10800` в конфиге Xray.

---

## 📁 Итоговая карта файлов

```
Основной сервер
├── /etc/wireguard/wg-server.conf              конфиг WireGuard сервера
├── /etc/systemd/system/exit-relay.service     socat relay сервис

Узел выхода
├── /etc/wireguard/wg0.conf                    конфиг WireGuard клиента
├── /etc/systemd/system/microsocks.service     microsocks сервис
```

---

<div align="center">

Сделано с ☕ · Pull requests приветствуются · [Открыть Issue](https://github.com/)

</div>
