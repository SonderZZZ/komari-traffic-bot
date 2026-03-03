<p align="center">
  <a href="./README.md">简体中文</a> ｜ 
  <a href="./README_EN.md">English</a>
</p>

# komari-traffic-bot (Docker Edition)

A **Dockerized traffic statistics extension** for **Komari Probe**, providing:

- 📊 Daily / Weekly / Monthly traffic reports via Telegram
- 🔥 Top N traffic consumers (supports `/top 6h`, `/top week`, etc.)
- 🤖 Interactive Telegram Bot commands
- 🐳 Docker / docker-compose deployment
- 🕒 Fixed statistics timezone (Asia/Shanghai by default)
- 🧱 Designed for multi-node and long-running environments

> This project does **not replace Komari**.  
> It enhances Komari with **long-term aggregation, arbitrary time window Top lists,
> and Telegram-based querying**.

---

## ✨ Features

- **Scheduled Reports**
  - Daily report at 00:00 (yesterday)
  - Weekly report (last week)
  - Monthly report (last month)
- **Top Traffic Ranking**
  - `/top` – today Top N (up + down)
  - `/top 6h` – last 6 hours
  - `/top week`, `/top month`
- **Telegram Commands**
  - `/today`, `/week`, `/month`
  - `/top [Nh|week|month]`
- **Stability & Reliability**
  - Slow or failed Komari nodes are skipped automatically
  - Telegram network errors are retried
  - Counter reset detection & fallback
- **Data Management**
  - Historical data auto-compression
  - Sampling system for arbitrary Nh queries

---

## 🧩 Requirements

- A running **Komari panel** (API accessible)
- Docker + docker-compose
- Telegram Bot Token
- Telegram Chat ID (user or group)

---

## 🚀 Quick Start (docker-compose)

### 1️⃣ Create data directory and set permissions (required)
This container runs as a non-root user (`uid:gid = 10001:10001`) and needs write access to the `data/` directory.
```
bash
mkdir -p komari-traffic && cd komari-traffic
mkdir -p data
sudo chown -R 10001:10001 data
sudo chmod -R u+rwX,go+rX data
```
> If you encounter `PermissionError: [Errno 13] Permission denied: '/data/...'` in the logs after startup,
> re-execute the above `chown` / `chmod` commands and restart the container.
### 2️⃣ Create .env
```
cp env.example .env
# Then edit .env as needed.

# Or create .env manually:
cat > .env <<'ENV'
# Komari panel base URL (no trailing slash)
KOMARI_BASE_URL=https://your-komari.example

# Komari API timeout (seconds)
KOMARI_TIMEOUT_SECONDS=15

# Komari API auth (optional)
KOMARI_API_TOKEN=
KOMARI_API_TOKEN_HEADER=Authorization
KOMARI_API_TOKEN_PREFIX=Bearer

# Komari fetch concurrency
KOMARI_FETCH_WORKERS=6

# Telegram
TELEGRAM_BOT_TOKEN=123456:YOUR_BOT_TOKEN
TELEGRAM_CHAT_ID=123456789

# Container data directory (do not change)
DATA_DIR=/data

# Statistics timezone (default Asia/Shanghai)
STAT_TZ=Asia/Shanghai

# Top ranking size
TOP_N=3

# Sampling for /top Nh
SAMPLE_INTERVAL_SECONDS=300
SAMPLE_RETENTION_HOURS=720

# History retention
HISTORY_HOT_DAYS=60
HISTORY_RETENTION_DAYS=400

# Logging
LOG_LEVEL=INFO
LOG_FILE=
ENV
```
### 3️⃣ Create crontab
```
cat > crontab <<'CRON'
# Daily report at 00:00
0 0 * * * python /app/komari_traffic_report.py report_daily

# Weekly report (Monday)
5 0 * * 1 python /app/komari_traffic_report.py report_weekly

# Monthly report
10 0 1 * * python /app/komari_traffic_report.py report_monthly
CRON
```
### 4️⃣ docker-compose.yml
```
version: "3.9"

services:
  komari-traffic-bot:
    image: ghcr.io/wirelouis/komari-traffic-bot:latest
    env_file: .env
    environment:
      - TZ=Asia/Shanghai
      - STAT_TZ=Asia/Shanghai
    volumes:
      - ./data:/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "python", "/app/komari_traffic_report.py", "health"]
      interval: 30s
      timeout: 10s
      retries: 3
    command: ["python", "/app/komari_traffic_report.py", "listen"]

  komari-traffic-cron:
    image: ghcr.io/wirelouis/komari-traffic-bot:latest
    env_file: .env
    environment:
      - TZ=Asia/Shanghai
      - STAT_TZ=Asia/Shanghai
    volumes:
      - ./data:/data
      - ./crontab:/app/crontab:ro
    restart: unless-stopped
    command: ["supercronic", "/app/crontab"]
```
Start services:
```
docker compose up -d
```
### 5️⃣ Initialize baseline (run once)
```
docker compose exec komari-traffic-bot \
  python /app/komari_traffic_report.py bootstrap
```
## 🤖 Telegram Command Examples
| Command      | Description                 |
| ------------ | --------------------------- |
| `/today`     | Today traffic (00:00 → now) |
| `/week`      | Current week                |
| `/month`     | Current month               |
| `/top`       | Today Top N                 |
| `/top 6h`    | Top in last 6 hours         |
| `/top week`  | Weekly Top                  |
| `/top month` | Monthly Top                 |
| `/status`    | Instant metrics for all nodes (CPU/MEM/online/upload/download) |
| `/status hk` | Instant metrics filtered by node name keyword |
| `/statusraw hk` | Raw recent payload preview for field debugging |
## 🕒 Timezone

Statistics timezone: STAT_TZ (default Asia/Shanghai)

Scheduler timezone: container TZ

This ensures daily reports are triggered at local midnight.

## 📦 Data Persistence

All runtime data is stored in ./data:

Baselines

Samples (for /top Nh)

History (daily records & compressed archives)

Telegram update offset

Upgrades and restarts will not lose data.


## 🏗 Build your own image and use it in compose
If you prefer using **your own image repository** (Docker Hub / private GHCR), follow these steps.

### 1) Build a local image from this repo
```
docker build -t komari-traffic-bot:local .
```

### 2) Tag with your own repository/name
```
docker tag komari-traffic-bot:local yourname/komari-traffic-bot:v1.0.0
```

### 3) Push to your registry (optional but recommended)
```
docker login
docker push yourname/komari-traffic-bot:v1.0.0
```

### 4) Update `docker-compose.yml` image fields
Use your image for both services:
```
services:
  komari-traffic-bot:
    image: yourname/komari-traffic-bot:v1.0.0
    ...

  komari-traffic-cron:
    image: yourname/komari-traffic-bot:v1.0.0
    ...
```

### 5) Restart services
```
docker compose up -d
docker compose ps
docker compose logs -f komari-traffic-bot
```

> For future upgrades, rebuild/re-tag/re-push, then update the tag in compose (for example `v1.0.1`) and run `docker compose up -d`.

## 🔄 Upgrade
```
docker pull ghcr.io/wirelouis/komari-traffic-bot:latest
docker compose up -d
```


## Notes for `/status`
- If values look wrong/missing, run `/statusraw` first to inspect raw `/api/recent/{uuid}` fields.
- If no direct online field exists, the bot estimates online as `connections.tcp + connections.udp`.
