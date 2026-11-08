# NAS Cloudflare Deploy Skill

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![AI Agent Ready](https://img.shields.io/badge/AI%20Agent-Antigravity%20%7C%20Claude%20Code%20%7C%20Cursor-success)](SKILL.md)

Deploy and safely expose Synology NAS applications, web games, or local services to the public internet using **Cloudflare Tunnel (`cloudflared`)** and **Docker (Container Manager)** with custom domains.

專門用於將 Synology NAS 本地服務或前端網頁/遊戲，透過 **Cloudflare Tunnel** 安全發布至外網自訂網域（**免開路由器 Port、免公網固定 IP、自帶 SSL 與防 DDoS**）。

---

## ⚡ 快速安裝 (Installation)

進入你的專案目錄，依據你的 AI Agent 工具執行以下指令：

### 1. Antigravity / Claude Code（專案內使用）
```bash
# 建立目錄
mkdir -p .agent/skills/nas-cloudflare-deploy

# 下載 Skill 定義檔
curl -sL https://raw.githubusercontent.com/Qhc2981/skill-nas-cloudflare-deploy/main/SKILL.md -o .agent/skills/nas-cloudflare-deploy/SKILL.md
```

### 2. Antigravity 全域安裝（所有專案通用）
```bash
mkdir -p ~/.gemini/antigravity/skills/nas-cloudflare-deploy
curl -sL https://raw.githubusercontent.com/Qhc2981/skill-nas-cloudflare-deploy/main/SKILL.md -o ~/.gemini/antigravity/skills/nas-cloudflare-deploy/SKILL.md
```

---

## 📋 支援場景 (When to Use)

- 🌐 **將 NAS 上的網頁 / 遊戲 / 服務對外公開** 給玩家或訪客。
- 🛡️ **Zero Trust 安全穿透**：不想開路由器 Port、沒有申請家用公網固定 IP。
- 🔒 **防範原始碼洩漏**：Nginx 容器精確掛載打包產物，徹底隔離 `.git`、`docs` 與原始碼。
- 🇹🇼 **台灣網域商 NS 委派**：PChome 買網址、HiNet、遠振 DNS 託管至 Cloudflare 避坑指南。
- ⚠️ **部署錯誤排查決策樹**：徹底解決 `DNS_PROBE_FINISHED_NXDOMAIN`、`502 Bad Gateway`、Linux Inode 脫鉤更新失效等問題。

---

## 📂 專案結構 (Directory Structure)

```text
skill-nas-cloudflare-deploy/
├── SKILL.md                  # Agent 核心技能描述與決策工作流
├── README.md                 # 說明文件與安裝指南
├── LICENSE                   # MIT 授權條款
└── templates/                # 推薦生產設定檔
    ├── docker-compose.yml    # Nginx + cloudflared 容器編排
    └── nginx.conf            # Gzip 壓縮 + 入口 no-cache 防卡版設定
```

---

## 🛠️ 架構簡圖 (Architecture)

```text
[外部玩家 / 訪客]
       │ (HTTPS: subdomain.domain.com)
       ▼
[Cloudflare Edge (免費 SSL / CDN 快取 / WAF 防護)]
       │ (Outbound 加密 Tunnel 通道，零開 Port)
       ▼
[Synology NAS (Container Manager / Docker)]
       ├── cloudflared (Tunnel Agent 容器)
       └── nginx (靜態網頁容器，精準掛載防外洩)
```

---

## 📄 License

本專案遵循 [MIT License](LICENSE)。歡迎自由分叉、修改或整合至你自己的 Agent 流程中。

