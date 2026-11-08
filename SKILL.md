---
name: nas-cloudflare-deploy
description: Deploy and expose Synology NAS applications, web games, or internal services to the public internet using Cloudflare Tunnel and Docker (Container Manager) with custom domains. Covers zero-port-forwarding setup, Nginx asset isolation to prevent repo leaks, and Taiwan domain registrar (PChome/TWNIC) NS configuration.
---

# NAS Cloudflare Deploy Skill

本 Skill 專門用於將 Synology NAS 本地服務或靜態網頁/遊戲，透過 **Cloudflare Tunnel (`cloudflared`)** 安全發布至外網自訂網域（免開路由器 Port、免公網固定 IP、自帶 SSL 與防 DDoS）。

## When to Use This Skill

當使用者提出以下需求或遇到相關錯誤時自動觸發：
- 🌐 「將 NAS 上的網頁/遊戲/服務對外公開給玩家或使用者」
- 🛡️ 「用 Cloudflare 連線 Synology NAS」或「設定 Cloudflare Tunnel / Zero Trust」
- 🔒 「避免專案內部原始碼、文件、Token 暴露給外網訪客」
- ⚠️ 部署時遇到 `DNS_PROBE_FINISHED_NXDOMAIN`、`Invalid nameservers` 或 `502 Bad Gateway`
- 🇹🇼 台灣網域商（PChome 買網址、HiNet、遠振）DNS 託管至 Cloudflare 的設定指引

---

## 標準部署架構與工作程序 (SOP)

```
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

## 階段一：網域 NS 權限委派 (Domain Delegation)

若網域尚未在 Cloudflare 作用（出現 `Invalid nameservers` 或 `DNS_PROBE_FINISHED_NXDOMAIN`）：

1. **取得 Cloudflare 專屬 Nameservers**：
   - 進入 Cloudflare Dashboard 點選該網域，取得 2 組 NS（如 `xxx.ns.cloudflare.com` 與 `yyy.ns.cloudflare.com`）。
2. **至原網域註冊商設定（以 PChome 買網址為例）**：
   - 模式選擇 **「自管 DNS」**。
   - 第 1、2 列填入 Cloudflare 的 2 組 NS。
   - **重要**：右側的「IP 位址」與「IPv6 位址」**必須全部保持空白**。
   - 下半部的「轉址設定」**完全不要動**。
3. **觸發驗證**：
   - 回到 Cloudflare 點選 **「Check nameservers now」**，等待 TWNIC 解析生效（約 15~30 分鐘），狀態轉為綠色 `Active`。

---

## 階段二：Cloudflare Zero Trust 建立 Tunnel

1. 進入 [Cloudflare Zero Trust](https://one.dash.cloudflare.com/) → **Networks** → **Tunnels**。
2. 點擊 **Add a tunnel** → 選擇 **Cloudflared** → 命名（如 `nas-app`）→ Save。
3. 環境選擇 **Docker**，複製命令中 `--token` 後面帶有的憑證字串 (`eyJh...`)。

---

## 階段三：NAS 目錄規劃與安全 Docker Compose

在 Synology NAS `/volume1/docker/<專案名稱>/` 目錄下進行配置。

### 1. 推薦標準架構：生產打包模式 (`dist/` + 自訂 `nginx.conf`)

此為最高效、最穩定的正式部署架構。

#### 目錄結構：
```text
/volume1/docker/<專案名稱>/
├── dist/                 # 前端打包產物 (index.html, *.min.js, *.min.css, fonts...)
├── docker-compose.yml    # 容器編排 (掛載 ./dist 與 ./nginx.conf)
└── nginx.conf            # Gzip 壓縮加速 + 入口 no-cache 防卡版
```

#### ① 高效防卡版 `nginx.conf`
* **Gzip 壓縮**：將大型 JS（如 1.5MB+）壓縮至 300~400KB，加載速度提升 4 倍並節省 NAS 上傳頻寬。
* **入口 `index.html` 絕不快取**：杜絕更新版本後玩家卡在舊版問題。
* **靜態資產長效快取**：打包後的 JS/CSS/字型快取 7 天。

```nginx
events { worker_connections 1024; }

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # 開啟 Gzip 壓縮
    gzip on;
    gzip_types text/plain text/css application/javascript application/json image/svg+xml font/woff2;
    gzip_min_length 1000;

    server {
        listen 80;
        server_name localhost;
        root /usr/share/nginx/html;
        index index.html;

        # 入口 index.html 絕不快取，確保改版即刻對外生效
        location = /index.html {
            add_header Cache-Control "no-cache, no-store, must-revalidate";
        }

        # 打包後靜態資源快取 7 天
        location / {
            try_files $uri $uri/ /index.html;
            expires 7d;
            add_header Cache-Control "public";
        }
    }
}
```

#### ② 生產標準 `docker-compose.yml`
```yaml
version: '3.8'

services:
  game:
    image: nginx:alpine
    container_name: app-web
    restart: unless-stopped
    volumes:
      # 整包掛載 dist 資料夾（徹底根治 Inode 脫鉤，日後更新檔案免重啟容器）
      - ./dist:/usr/share/nginx/html:ro
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    ports:
      - "8088:80" # 選用：提供內網 http://<NAS_IP>:8088 優先驗證

  tunnel:
    image: cloudflare/cloudflared:latest
    container_name: app-tunnel
    restart: unless-stopped
    command: tunnel --no-autoupdate run --token <CLOUDFLARE_TUNNEL_TOKEN>
    depends_on:
      - game
```

---

### 2. 備用架構：未打包零建置模式 (Loose Files)

若專案處於極早期原型開發、未經打包（`index.html` 直接引用分散的 `js/`、`css/`、`data/`）：
* **檔案實體隔離**：嚴禁將 `docs/`、`tests/`、`implement_plan/`、`.git/` 複製到 NAS 上。
* **白名單精確掛載**：
  ```yaml
  volumes:
    - ./index.html:/usr/share/nginx/html/index.html:ro
    - ./css:/usr/share/nginx/html/css:ro
    - ./js:/usr/share/nginx/html/js:ro
    - ./data:/usr/share/nginx/html/data:ro
  ```
* ⚠️ **限制**：此模式若在 NAS 覆蓋 `index.html` 會遭遇 Linux Inode 重新指派而脫鉤，**每次更新必須在 Container Manager 手動「重新啟動」專案**。因此正式上線強烈建議採用「生產打包模式」。

---

## 階段四：Cloudflare 路由指向 (Add Route)

在 Tunnel 詳情頁設定公開路徑：

1. 點選 **Routes** 分頁（或點擊 **+ Add route**）。
2. 路由類型選擇：**Published application**（地球圖示）。
3. 填入參數：
   * **Subdomain**：二級前綴（例如 `game` 或 `warrior`）
   * **Domain**：主網域（例如 `yourdomain.tw` 或 `example.com`）
   * **Path**：保持空白
   * **Service URL**：填寫 `http://game:80`
     *(直接使用 Docker Compose 服務名 `game`，無須填寫 NAS 內部 IP)*
4. 儲存後，Cloudflare 會自動新增 CNAME 記錄至 DNS。

---

## 階段五：連線排查決策樹 (Troubleshooting)

| 症狀 / 錯誤碼 | 根本原因 | 排除步驟 |
| :--- | :--- | :--- |
| **`DNS_PROBE_FINISHED_NXDOMAIN`** | DNS 查無此網域名稱，**連線尚未抵達 Cloudflare** | 1. 檢查網域狀態是否為 `Invalid nameservers`。<br>2. 檢查網域商 NS 是否已改為 Cloudflare。<br>3. 執行 `ipconfig /flushdns` 清除快取。 |
| **`502 Bad Gateway`** | Cloudflare Tunnel 連不上 NAS 的 Web 容器 | 1. 檢查 `docker-compose` 中 `game` 容器是否存活。<br>2. 檢查 Service URL 是否寫錯（正確為 `http://game:80`）。 |
| **Container Manager 圖示亮紅點** | 專案建置失敗或容器重啟崩潰 | 1. 查看容器日誌 (Logs)。<br>2. 常見為 Token 填寫錯誤或 `./index.html` 等掛載檔案不存在。 |
| **`403 Forbidden` / `404 Not Found`** | Nginx 找不到檔案或權限錯誤 | 1. 先用區網 `http://<NAS_IP>:8088` 測試。<br>2. 檢查 File Station 中檔案權限與目錄路徑。 |
| **更新檔案後連線仍為舊版** | 1. Docker 單檔掛載因覆蓋產生新 Inode 脫鉤。<br>2. `nginx.conf` 誤對 `index.html` 快取。<br>3. Cloudflare/瀏覽器快取。 | 1. 徹底根治：改採生產打包資料夾掛載（`./dist:/usr/share/nginx/html:ro`），目錄內更新不脫鉤。<br>2. 確認 `nginx.conf` 針對 `location = /index.html` 設為 `no-cache, no-store`。<br>3. 若仍舊，至 Cloudflare 執行 Purge Everything，並於 HTML 引入資源加 `?v=...` 版本號。 |

---

## 替代評估：純靜態託管 (Cloudflare Pages)

若專案為 100% Client-side 靜態遊戲（無後端 Node/Python/DB），且不希望消耗 NAS 家用上傳頻寬：
* 可直接改用 **Cloudflare Pages** 託管。
* 優勢：全球邊緣 CDN 快取、0 NAS 負載、免費 SSL。
* 隱私考量：若不想將專案程式碼儲存於第三方雲端，則堅持採用上述 **NAS + Tunnel** 方案。
