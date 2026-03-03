<p align="center">
  <a href="./README.md">简体中文</a> ｜ 
  <a href="./README_EN.md">English</a>
</p>

# komari-traffic-bot（Docker 版）

基于 **Komari 探针** 的流量统计增强工具，提供：

- 📊 Telegram **流量日报 / 周报 / 月报**
- 🔥 **Top N 流量消耗榜**（支持 `/top 6h`、`/top week` 等任意时间窗口）
- 🤖 Telegram Bot **交互式查询**
- 🐳 **Docker / docker-compose 部署**
- 🕒 统计口径固定为 **北京时间（Asia/Shanghai）**
- 🧱 适合多节点、长期运行场景

> 本项目不替代 Komari 官方功能，而是在其基础上补充  
> **长期统计 / 任意时间窗口 Top 榜 / Telegram 查询能力**。

---

## ✨ 功能特性

- **日报 / 周报 / 月报**
  - 每天 00:00 自动推送昨日日报
  - 每周一推送上周周报
  - 每月 1 号推送上月月报
- **Top 流量消耗榜**
  - `/top`：今日 Top N（上下行合计）
  - `/top 6h`：最近 6 小时 Top
  - `/top week`、`/top month`
- **交互命令**
  - `/today` `/week` `/month`
  - `/top [Nh|week|month]`
- **稳定性**
  - Komari 节点超时自动跳过，不影响整体报表
  - Telegram 网络异常自动重试
  - 探针/节点重启自动兜底计数器
- **数据管理**
  - 历史数据自动压缩归档
  - 采样数据用于支持任意 Nh 查询

---

## 🧩 依赖说明

- 已部署并可访问的 **Komari 面板**
- Docker + docker-compose
- 一个 Telegram Bot Token
- Telegram Chat ID（个人 / 群组）

---

## 🚀 快速部署（docker-compose）

### 1️⃣ 创建目录并赋予权限
容器默认以**非 root 用户（uid:gid = 10001:10001）**运行，需要对 `data/` 目录具有写权限。
```
bash
mkdir -p komari-traffic && cd komari-traffic
mkdir -p data
sudo chown -R 10001:10001 data
sudo chmod -R u+rwX,go+rX data
```
> 如果启动后日志中出现 `PermissionError: [Errno 13] Permission denied: '/data/...'`，
> 请重新执行上述 `chown` / `chmod` 命令后重启容器。
### 2️⃣ 创建 .env 配置文件
```
cat > .env <<'ENV'
# Komari 面板地址（不要以 / 结尾）
KOMARI_BASE_URL=https://your-komari.example

# Komari API 超时（秒）
KOMARI_TIMEOUT_SECONDS=15

# Komari API 鉴权（可选）
KOMARI_API_TOKEN=
KOMARI_API_TOKEN_HEADER=Authorization
KOMARI_API_TOKEN_PREFIX=Bearer

# Komari 节点并发请求数
KOMARI_FETCH_WORKERS=6

# Telegram
TELEGRAM_BOT_TOKEN=123456:YOUR_BOT_TOKEN
TELEGRAM_CHAT_ID=123456789

# 容器内数据目录（固定）
DATA_DIR=/data

# 统计时区（默认 Asia/Shanghai）
STAT_TZ=Asia/Shanghai

# Top 榜数量
TOP_N=3

# /top Nh 采样：每 5 分钟采样一次，保留 30 天
SAMPLE_INTERVAL_SECONDS=300
SAMPLE_RETENTION_HOURS=720

# 历史数据策略
HISTORY_HOT_DAYS=60
HISTORY_RETENTION_DAYS=400

# 日志
LOG_LEVEL=INFO
LOG_FILE=
ENV
```
### 3️⃣ 准备 crontab
```
cat > crontab <<'CRON'
# 每天 00:00：昨日日报
0 0 * * * python /app/komari_traffic_report.py report_daily

# 每周一 00:05：上周周报
5 0 * * 1 python /app/komari_traffic_report.py report_weekly

# 每月 1 号 00:10：上月月报
10 0 1 * * python /app/komari_traffic_report.py report_monthly
CRON
```
### 4️⃣ 使用 docker-compose 启动
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
启动：
```
docker compose up -d
docker compose ps
```
### 5️⃣ 初始化（只需一次
```）
docker compose exec komari-traffic-bot \
  python /app/komari_traffic_report.py bootstrap
```
## 🤖 Telegram 命令示例
| 命令           | 说明               |
| ------------ | ---------------- |
| `/today`     | 今日流量（00:00 → 当前） |
| `/week`      | 本周流量             |
| `/month`     | 本月流量             |
| `/top`       | 今日 Top N         |
| `/top 6h`    | 最近 6 小时 Top      |
| `/top week`  | 本周 Top           |
| `/top month` | 本月 Top           |
| `/status`    | 查看全部节点瞬时状态（CPU/内存/在线/延迟） |
| `/status hk` | 按节点名关键词筛选瞬时状态 |

## 🕒 关于时区
统计口径时区：STAT_TZ（默认 Asia/Shanghai）

定时触发时区：容器 TZ（默认 Asia/Shanghai）

因此：

“每天 0 点” = 北京时间 0 点

与宿主机系统时区无关

## 📦 数据说明
所有数据均保存在 ./data 目录中：

baseline（起点快照）

samples（用于 /top Nh）

history（日报历史 & 压缩归档）

Telegram offset

升级 / 重启容器不会丢数据。

## 🔄 升级方式
```
docker pull ghcr.io/wirelouis/komari-traffic-bot:latest
docker compose up -d
```

## 🛠 升级到瞬时状态命令（/status）
如果你是通过 **docker compose** 部署的，
**无需修改 `docker-compose.yml` 结构**。

这个功能是代码能力，和你是否使用 `ghcr.io/wirelouis/komari-traffic-bot:latest` 没有强绑定：
- 若你使用官方镜像：拉取你正在使用的标签并重启即可；
- 若你使用自建镜像：重新构建并 `docker compose up -d` 即可。

参考操作：
```
# 官方镜像（示例）
docker pull <你的镜像:你的标签>
docker compose up -d

# 自建镜像（示例）
docker compose build --no-cache
docker compose up -d
```

可选检查：
```
docker compose logs -f komari-traffic-bot
```
看到 bot 正常启动后，直接在 Telegram 发送 `/status` 或 `/status 节点关键词` 即可。

## ⚠️ 常见问题
/top 6h 没数据？
需要采样积累时间（默认每 5 分钟一次）

Komari 某节点超时？
会被自动跳过，不影响整体报表

Telegram 偶发断连？
已内置自动重试
