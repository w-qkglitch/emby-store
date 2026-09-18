```
# StreamPanel

<p align="center">
  <img src="[https://img.shields.io/badge/](https://img.shields.io/badge/)🎬-StreamPanel-6366f1?style=for-the-badge" height="40"/>
</p>

<h3 align="center">Emby 服务端的高级伴侣：会员商店 · 在线收款 · 自动化运营</h3>

<p align="center">
  <a href="[https://t.me/emby_ying](https://t.me/emby_ying)"><img src="[https://img.shields.io/badge/Telegram-](https://img.shields.io/badge/Telegram-)加入交流群-26A5E4?style=flat-square"/></a>
  <a href="#"><img src="[https://img.shields.io/badge/WIKI-](https://img.shields.io/badge/WIKI-)文档-2c3e50?style=flat-square"/></a>
  <a href="#-功能特性"><img src="[https://img.shields.io/badge/](https://img.shields.io/badge/)亮点速览-Features-2980b9?style=flat-square"/></a>
  <a href="#-快速开始"><img src="[https://img.shields.io/badge/](https://img.shields.io/badge/)快速部署-Deploy-e67e22?style=flat-square"/></a>
  <a href="#-关键环境变量"><img src="[https://img.shields.io/badge/](https://img.shields.io/badge/)配置说明-Config-8e44ad?style=flat-square"/></a>
  <a href="#-安全说明与已知限制"><img src="[https://img.shields.io/badge/](https://img.shields.io/badge/)常见问题-FAQ-c0392b?style=flat-square"/></a>
  <a href="#-license"><img src="[https://img.shields.io/badge/](https://img.shields.io/badge/)许可证-License-34495e?style=flat-square"/></a>
</p>

## 📖 项目简介

StreamPanel 是一套面向 Emby 服务运营者的**现代化第二控制台**：前台展示套餐、在线收款后自动开通 Emby 账号，后台提供订单、用户、数据可视化大盘与全套自动化运维任务。

核心能力：原生兼容 Emby 客户端协议、多渠道扫码收款、付款自动开号 / 到期自动停用、播放数据大盘、数据库自动备份与下载自动化。

---

## ✨ 功能特性

### 前台（面向会员）
- 🛒 **会员商店** — 套餐展示、在线下单、订单状态查询
- 🎟️ **激活码 / 积分兑换** — 兑换会员时长、积分换隐藏库权限
- 📅 **每日签到** — 签到送积分，带频率限制

### 支付与账号
- 💳 **多渠道收款** — 支付宝当面付、微信支付 Native、易支付（Epay），另含 `demo` 演示通道
- 🔄 **Emby 自动同步** — 新注册创建为禁用账号、付款回调后自动启用、到期自动停用（跳过管理员）
- 🔁 **Emby 定时重启** — 按北京时间定时重启指定服务器，10 分钟补跑窗口

### 后台（面向运营者）
- 📊 **运营大盘** — 管理员统计页、每日报表、播放榜定时推送
- 👥 **用户与订单管理** — 会员状态、订单、播放记录与 IP 查询
- 🗂️ **自动化运维** — PostgreSQL 备份（可推送 Telegram）、qBittorrent 下载完成通知、隐藏库权限到期回收
- ⬆️ **在线升级代理** — 后台一键升级，升级前自动备份数据库

### 安全
- 🛡️ 全站 `/api/*` 写操作同源校验（CSRF）
- 🚦 登录失败锁定、注册/兑换/签到接口按 IP/账号限流
- 🔐 Emby / Telegram / 支付密钥经应用层加密后落盘

---

## 🧱 技术栈

- **应用框架**：Next.js（App Router）+ Node.js
- **数据库**：PostgreSQL
- **部署**：Docker Compose（应用 + 数据库一键拉起）
- **任务调度**：应用内进程调度器（单实例），或外部 cron 复用同一套脚本

---

## 🚀 快速开始

### 环境要求
- 已安装 Docker 与 Docker Compose
- 一个可公网 HTTPS 访问的域名（支付回调必须是 HTTPS）

### 1. 拉取并准备配置
```bash
cp .env.example .env
```

至少填写：

- `POSTGRES_PASSWORD` — 数据库强密码
- `APP_URL` — 站点完整地址，如 `[https://panel.example.com](https://panel.example.com)`

> 
> 不设置 `ADMIN_INITIAL_PASSWORD` 时，**系统第一个注册账号自动升级为管理员**。

### 2. docker-compose.yml

```
services:
  postgres:
    image: postgres:16-alpine
    container_name: emby-store-panel-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-emby}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?POSTGRES_PASSWORD must be set in .env}
      POSTGRES_DB: ${POSTGRES_DB:-emby_panel}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-emby} -d ${POSTGRES_DB:-emby_panel}"]
      interval: 5s
      timeout: 5s
      retries: 5

  app:
    image: ${PANEL_IMAGE:-ghcr.io/w-qkglitch/emby-store-panel:latest}
    pull_policy: always
    container_name: emby-store-panel-app
    build:
      context: .
      dockerfile: Dockerfile
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://${POSTGRES_USER:-emby}:${POSTGRES_PASSWORD:?POSTGRES_PASSWORD must be set in .env}@postgres:5432/${POSTGRES_DB:-emby_panel}?schema=public
      ADMIN_INITIAL_EMAIL: ${ADMIN_INITIAL_EMAIL:-admin@example.com}
      ADMIN_INITIAL_PASSWORD: ${ADMIN_INITIAL_PASSWORD}
      EMBY_URL: ${EMBY_URL:-}
      EMBY_API_KEY: ${EMBY_API_KEY:-}
      TELEGRAM_BOT_TOKEN: ${TELEGRAM_BOT_TOKEN:-}
      TELEGRAM_GROUP_ID: ${TELEGRAM_GROUP_ID:-}
      APP_URL: ${APP_URL:-}
      LICENSE_SERVER_URL: ${LICENSE_SERVER_URL:-}
      LICENSE_PURCHASE_URL: ${LICENSE_PURCHASE_URL:-}
      LICENSE_PUBLIC_KEY: ${LICENSE_PUBLIC_KEY:-}
      APP_ENCRYPTION_KEY: ${APP_ENCRYPTION_KEY:-}
      APP_ENCRYPTION_KEY_FILE: /app/data/secrets/app-encryption.key
      UPDATE_AGENT_URL: ${UPDATE_AGENT_URL:-}
      UPDATE_AGENT_TOKEN: ${UPDATE_AGENT_TOKEN:-}
      UPDATE_IMAGE_PREFIX: ${UPDATE_IMAGE_PREFIX:-}
      BACKUP_DIR: ${BACKUP_DIR:-/app/data/backups}
      BACKUP_RETENTION_DAYS: ${BACKUP_RETENTION_DAYS:-30}
    volumes:
      - ./data:/app/data
    # 应用端口只暴露给同一 Docker 网络中的 nginx，避免绕过 HTTPS、限流与代理头校验。
    command: sh -c "npx prisma migrate deploy && npx prisma db seed && npm start"

  # 可选的一键升级代理：只在 Docker 内网监听，不向公网开放 2999 端口。
  # 启用命令见 README；它会在更新前强制创建 PostgreSQL 备份。
  update-agent:
    profiles: ["update-agent"]
    build:
      context: .
      dockerfile: docker/update-agent.Dockerfile
    image: ${UPDATE_AGENT_IMAGE:-emby-store-panel-update-agent:local}
    container_name: emby-store-panel-update-agent
    restart: unless-stopped
    environment:
      UPDATE_AGENT_TOKEN: ${UPDATE_AGENT_TOKEN:-}
      UPDATE_IMAGE_PREFIX: ${UPDATE_IMAGE_PREFIX:-}
      UPDATE_PROJECT_DIR: /opt/emby-store-panel
      UPDATE_AGENT_PORT: "2999"
      BACKUP_RETENTION_DAYS: ${BACKUP_RETENTION_DAYS:-30}
    volumes:
      - .:/opt/emby-store-panel
      - /var/run/docker.sock:/var/run/docker.sock

  # 日报、自动备份、会员同步、到期回收、播放榜、下载通知都由 app 容器内的调度器执行，
  # 不再需要单独的 reporter 容器（原来的 reporter 只在启动时跑一次，等于没在跑）。
  nginx:
    image: nginx:alpine
    container_name: emby-store-panel-nginx
    restart: unless-stopped
    depends_on:
      - app
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certs:/etc/nginx/certs:ro

volumes:
  postgres_data:
```

### 3. 启动服务

```
docker compose up -d --build
docker compose logs -f app
```

访问 `https://你的域名` 即可。

> 
> ⚠️ **app 服务必须挂载 `./data:/app/data`**（仓库 compose 已配置，自定义部署务必保留）。
> 加密密钥与数据库备份都保存在此目录。如果不挂载，重建容器会生成新密钥，已保存配置全部丢失。
> 后台设置保存后刷新变回默认，优先检查该挂载项。

> 
> ⚠️ 生产环境必须配置 HTTPS。支付异步回调地址：
> 
> 
> ```
> /api/payments/alipay/notify
> /api/payments/wechat/notify
> /api/payments/epay/notify
> ```

---

## ⚙️ 关键环境变量

| 变量 | 说明 |
| --- | --- |
| `POSTGRES_PASSWORD` | PostgreSQL 数据库密码 |
| `APP_URL` | 站点完整地址（含协议与端口） |
| `ADMIN_INITIAL_PASSWORD` | 可选。填写则启动自动创建预置管理员；留空则**第一个注册账号自动成为管理员** |
| `ADMIN_INITIAL_EMAIL` | 初始管理员邮箱，默认 `admin@example.com` |
| `APP_ENCRYPTION_KEY` | 加密密钥；留空则首次保存配置时自动生成到 `/app/data/secrets/` |
| `UPDATE_AGENT_TOKEN` | 在线升级代理 token，至少 32 位 |
| `CSRF_PROTECT` | 同源校验开关，反代改写 Host 导致误判时可设 `false` |
| `BACKUP_RETENTION_DAYS` | 本地备份保留天数，默认 30 |

Emby、Telegram、qBittorrent、各支付商户参数**直接在后台"系统设置"页面填写即可**，无需改 `.env`。要清空某项已保存的值，在该项填入 `__CLEAR__` 再保存。

---

## ⏰ 内置定时任务

由应用内调度器执行（单实例，日志带 `[scheduler]` 前缀，按北京时间判断）：

| 任务 | 频率 | 开关 / 配置 |
| --- | --- | --- |
| 自动备份 + 发送 Telegram | 每天，时间可设 | 系统设置 → Telegram 备份 Bot |
| 隐藏库权限到期回收 | 每小时 | 无（积分兑换产生的权限，到期自动关闭并同步 Emby） |
| 会员到期自动同步 | 每天 04:00，时间可设 | 系统设置 → Emby → 会员到期自动同步 |
| 每日报表（写入 DailyReport） | 每小时刷新当天 | 无（在 `/admin/reports` 查看） |
| 每日播放榜推送 | 每天 21:00 | 扩展中心 → 首页榜单（需开启） |
| 下载完成通知（qBittorrent） | 每 5 分钟 | 配置 qBittorrent + Telegram 群 Chat ID |
| Emby 定时重启 | 每分钟检查，到点执行 | 后台"Emby 定时重启"中保存服务器和时间 |

> 
> 任务开关与时间以**后台界面保存的值为准**；`.env` 里的同名变量只作为界面首次保存前的初始默认值。
> 调度器是进程内定时器，**按单实例部署设计**。多副本运行时请改用外部 cron，并把验证码存储改为共享。

裸机或外部 cron 部署时，可复用以下脚本（与应用内逻辑一致，请勿与应用内调度器同时运行）：

```
# 自动备份（含 Telegram 发送、保留策略、当天去重）
* * * * * cd /opt/emby-store-panel && /usr/bin/npm run backup:auto >> /var/log/stream-panel-backup.log 2>&1
# 隐藏库权限到期回收
*/30 * * * * cd /opt/emby-store-panel && /usr/bin/npm run hidden:expire >> /var/log/stream-panel-hidden.log 2>&1
# Emby 定时重启（仅外部 cron 部署使用）
* * * * * cd /opt/emby-store-panel && /usr/bin/npm run emby:restart >> /var/log/stream-panel-emby-restart.log 2>&1
# 每日播放榜
* * * * * cd /opt/emby-store-panel && /usr/bin/npm run ranking:notify >> /var/log/stream-panel-ranking.log 2>&1
# 下载完成通知（需 qBittorrent）
*/5 * * * * cd /opt/emby-store-panel && /usr/bin/npm run downloads:notify >> /var/log/stream-panel-downloads.log 2>&1
# 每日报表
5 * * * * cd /opt/emby-store-panel && /usr/bin/npm run report:daily >> /var/log/stream-panel-report.log 2>&1
# 会员到期同步（跳过管理员）
0 4 * * * cd /opt/emby-store-panel && /usr/bin/npm run membership:sync >> /var/log/stream-panel-membership.log 2>&1
```

---

## 💰 支付接入

`src/lib/payment.ts` 是唯一支付入口。`demo` 通道会立即将订单标为已付款，**仅限本地演示，公网禁止启用**。

接入真实收款时：创建支付单 → 保存第三方交易号 → 验证服务端回调签名 → 调用 `fulfillOrder`。**不要信任浏览器传来的"支付成功"状态。**

- **支付宝当面付 / 微信 Native**：服务端返回 `codeUrl`，前端渲染成二维码；在 `.env` 或后台填 App ID、商户证书、密钥等。
- **易支付（Epay）**：`provider: "EPAY"` 返回跳转收银台 `checkoutUrl`，填 `EPAY_GATEWAY` / `EPAY_PID` / `EPAY_KEY` / `EPAY_TYPE`；采用 MD5 签名，仅当服务商文档一致时启用。

---

## 💾 备份与恢复

管理员登录后访问 `/admin/backups`，可创建、下载、上传、恢复 PostgreSQL 自定义格式备份；恢复会覆盖当前库，并在操作前自动创建保护备份。

- 本地备份默认保留 30 天（`BACKUP_RETENTION_DAYS` 可调）
- 配置 Telegram 后，备份完成自动以文件形式发送到指定 Chat ID
- 升级前在线升级代理会先导出到 `data/backups/pre-update-*.dump`，备份失败则取消升级

> 
> 备份文件应同步到另一台机器或对象存储，避免主机故障时一并丢失。

---

## ⬆️ 在线升级

在 `.env` 设置至少 32 位的 `UPDATE_AGENT_TOKEN`，保留 `UPDATE_AGENT_URL="http://update-agent:2999"`，然后：

```
docker compose --profile update-agent up -d --build update-agent
```

升级代理仅在 Compose 内网监听，不对外开放端口。

---

## ✅ 上线前检查清单

- 公网部署注意：**第一个注册账号自动成为管理员**，上线后尽快自行注册并修改强密码，防止被他人抢先注册拿到管理员权限
- 接入支付商户的**服务端回调验签**，未完成前禁止启用 `demo`
- Emby API Key 设最小管理员权限，存入密钥库，**不要提交 `.env`**
- 配置 HTTPS、订单审计日志、退款流程与隐私政策

---

## 🔒 安全说明与已知限制

- **CSRF**：`src/middleware.ts` 校验所有 `/api/*` 写操作的 `Origin`/`Referer`，支付回调与 Telegram webhook 已豁免；无 `Origin` 的非浏览器客户端（curl/脚本）放行。
- **限流**：登录失败 3 次锁 30 分钟；注册按 IP 每小时 5 个；激活码兑换 20 次/小时、积分兑换 30 次/小时、签到验证码 30 次/小时。
- **签到验证码存进程内存**，进程重启即失效；配合进程内调度器，本面板按**单实例**设计。
- **播放记录查询**走 Playback Reporting 插件的自定义 SQL 接口，该接口不支持参数化，面板已用正则校验用户 ID；若你的 Emby 用户 ID 固定为 32 位 hex，可进一步收紧正则。
- **首任管理员机制**：系统数据库为空无用户时，**第一个注册用户自动提升为管理员**；如果配置 `ADMIN_INITIAL_PASSWORD`，系统启动直接创建管理员，关闭首注册自动升管理员逻辑。

---
