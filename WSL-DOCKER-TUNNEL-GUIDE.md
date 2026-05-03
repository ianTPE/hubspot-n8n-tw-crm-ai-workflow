# WSL + Docker + Cloudflare Tunnel：本地測試 n8n Workflow 指南

> 這份指南教你如何在 Windows 的 WSL 裡用 Docker 跑 n8n，再用 Cloudflare Tunnel 打通公網，讓 HubSpot 的 webhook 能打得到你的本機 n8n。**只適合開發測試，不適合正式環境。**

---

## 為什麼需要這個？

HubSpot 的 webhook 是從 HubSpot 伺服器主動打出去的，你的 n8n 必須有一個**公開可達的 HTTPS URL**。localhost 預設做不到，所以需要 Cloudflare Tunnel 幫你開一條臨時通道。

---

## 架構

```
# Quick Tunnel（測試用，每次重啟 URL 可能會變）
HubSpot  ──POST──►  https://xxx.try-cloudflare.com  ──tunnel──►  localhost:5678  ──►  n8n Docker Container

# Named Tunnel（固定 URL，需要 Cloudflare 帳號與自有域名）
HubSpot  ──POST──►  https://n8n.yourdomain.com  ──►  cloudflared daemon  ──►  localhost:5678  ──►  n8n Docker Container
```

---

## 前置條件

- [ ] Windows 10/11（已啟用 WSL2）
- [ ] 網路可連線（下載 Docker image、Cloudflare Tunnel 需要外網）
- [ ] HubSpot Private App Token（見主 README §A）
- [ ] OpenAI API Key

---

## Step 1：確認 WSL2 環境

打開 PowerShell，確認 WSL 版本：

```powershell
wsl --version
```

如果還沒裝 WSL2：

```powershell
wsl --install
```

重開機後，進入 WSL：

```powershell
wsl
```

> 以下所有指令都在 WSL 裡面執行。

---

## Step 2：安裝 Docker

### 方法 A：一鍵安裝 Docker Engine（推薦，較輕量）

這份指南只用於本機開發測試，所以可以使用 Docker 官方 convenience script 省掉手動設定 apt key 和 repository 的步驟。正式伺服器請改用 Docker 官方文件的 repository 安裝法。

```bash
# 下載官方安裝腳本
curl -fsSL https://get.docker.com -o get-docker.sh

# 執行安裝（過程中若提示建議裝 Docker Desktop，可忽略）
sudo sh get-docker.sh
```

設定目前帳號不用 `sudo` 就能跑 Docker：

```bash
sudo usermod -aG docker $USER
```

讓群組設定生效：

```powershell
# 在 PowerShell 執行
wsl --shutdown
```

重新開啟 WSL terminal 後，確認 Docker 可以用：

```bash
docker --version
docker compose version
docker run hello-world
```

看到 `Hello from Docker!` 就代表成功。

### 設定 Docker 自動啟動

WSL 2 的服務啟動方式和一般 Linux server 不完全一樣，建議優先使用 systemd。

#### 自動啟動方案 A：用 systemd（推薦）

先確認 systemd 是否可用：

```bash
systemctl status
```

如果看到 systemd 狀態，直接啟用 Docker：

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

如果出現 `System has not been booted with systemd`，先啟用 systemd：

```bash
sudo nano /etc/wsl.conf
```

加入：

```ini
[boot]
systemd=true
```

接著在 PowerShell 重啟 WSL：

```powershell
wsl --shutdown
```

重新開啟 WSL 後再執行：

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

#### 自動啟動方案 B：手動啟動（沒有 systemd 時）

如果你的 WSL 環境不能用 systemd，每次開 WSL 後先啟動 Docker daemon：

```bash
sudo service docker start
```

### 方法 B：用 Docker Desktop（如果你已經裝了）

如果你已經在 Windows 裝了 Docker Desktop，且開啟了「Use the WSL 2 based engine」，那 WSL 裡直接就能用 `docker` 指令，跳過方法 A。

確認方式：

```bash
docker --version
docker compose version
```

---

## Step 3：建立 n8n-hubspot 專案目錄

這個目錄會放 `docker-compose.yml`、`.env`、`.gitignore`，n8n 的資料也會存在裡面的 `n8n-data/` 子目錄。

### 方法 A：從 repo 複製（推薦）

```bash
# 先 clone repo（如果還沒有）
git clone https://github.com/iantpe/hubspot-n8n-tw-crm-ai-workflow.git
# ↑ 如果 repo 在別的地方，換成你的實際 URL

# 複製設定目錄
cp -r hubspot-n8n-tw-crm-ai-workflow/n8n-hubspot ~/n8n-hubspot
cd ~/n8n-hubspot

# 從範本建立 .env（.env 本身不會進 git，只有 .env.example 會）
cp .env.example .env
```

repo 裡已經包含 `docker-compose.yml`、`.env.example`、`.gitignore`，不需要手動建立。

### 方法 B：手動建立

```bash
mkdir -p ~/n8n-hubspot
cd ~/n8n-hubspot
```

建立 `docker-compose.yml`：

```yaml
cat > docker-compose.yml << 'YAML'
services:
  n8n:
    image: n8nio/n8n:latest  # 正式環境建議 pin 版本，例如 n8nio/n8n:1.82.0
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    volumes:
      - ./n8n-data:/home/node/.n8n
    environment:
      - N8N_SECURE_COOKIE=false
      - N8N_PROXY_HOPS=1
      - WEBHOOK_URL=${WEBHOOK_URL}
      - GENERIC_TIMEZONE=Asia/Taipei
      - TZ=Asia/Taipei
YAML
```

建立 `.env.example` 並從範本產生 `.env`：

```bash
cat > .env.example << 'EOF'
# n8n Webhook 公網 URL（由 Cloudflare Tunnel 提供）
# 使用前請先複製此檔案：cp .env.example .env
# 每次啟動 Quick Tunnel 後，把拿到的 URL 貼到 .env，然後執行：
#   docker compose up -d --force-recreate
WEBHOOK_URL=https://your-tunnel-url.try-cloudflare.com
EOF

cp .env.example .env
```

建立 `.gitignore`：

```bash
cat > .gitignore << 'EOF'
.env
n8n-data/
EOF
```

---

### 最終目錄結構

無論用哪種方法，完成後目錄長這樣：

```
~/n8n-hubspot/
├── docker-compose.yml    ← Docker Compose 設定
├── .env.example          ← .env 範本（會進 git）
├── .env                  ← 實際環境變數（從 .env.example 複製，不會進 git）
├── .gitignore            ← 排除 .env 和 n8n-data/
└── n8n-data/             ← n8n 持久化資料（啟動後自動產生）
```

> `docker compose` 會自動讀取同目錄的 `.env`，所以 `WEBHOOK_URL` 不用寫死在 `docker-compose.yml` 裡。之後 tunnel URL 變了，只需要改 `.env` 再重啟就好。
>
> `.env` 不會進 git（`.gitignore` 已排除）。`n8n-data/` 裡有 n8n 加密儲存的 credentials（HubSpot Token、OpenAI Key 等），同樣已被排除。

---

## Step 4：安裝 Cloudflare Tunnel（cloudflared）

```bash
# 下載 cloudflared
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
# ARM 架構（如 Surface Pro X）請改下載 cloudflared-linux-arm64.deb

# 安裝
sudo dpkg -i cloudflared.deb

# 確認版本
cloudflared --version
```

---

## Step 5：啟動 Cloudflare Tunnel，取得公網 URL

先啟動 tunnel，拿到 `https://xxx.try-cloudflare.com`，下一步啟動 n8n 時會直接把它填進 `WEBHOOK_URL`。

> ⚠️ 這時 n8n 還沒啟動，cloudflared 終端機會持續印 `connection refused` 之類的訊息——**這是正常的**，等 Step 6 啟動 n8n 後就會通。先把 URL 複製起來就好。

```bash
cloudflared tunnel --url http://localhost:5678
```

你會看到類似這樣的輸出：

```
+--------------------------------------------------------------------------------------------+
|  Your quick Tunnel has been created! Visit it at (it may take some time to be reachable):  |
|                                                                                            |
|  https://some-random-words.try-cloudflare.com                                              |
+--------------------------------------------------------------------------------------------+
```

**複製這個 `https://xxx.try-cloudflare.com` URL**，這就是你的公網 webhook URL。

> ⚠️ 這個 URL 每次重啟 cloudflared 都會變。測試階段沒關係，但正式環境不能用這個方式。
>
> Cloudflare Quick Tunnel 只適合開發測試：它有 200 concurrent request 限制，且不支援 Server-Sent Events（SSE）。正式環境請改用 Named Tunnel 或 n8n Cloud。

---

## Step 6：啟動 n8n 並設定 WEBHOOK_URL

打開**另一個 WSL 終端機**（cloudflared 佔住原本那個），執行：

```bash
cd ~/n8n-hubspot

# 把 Step 5 拿到的 tunnel URL 寫進 .env
# 把下面的 URL 換成你的 tunnel URL
sed -i 's|WEBHOOK_URL=.*|WEBHOOK_URL=https://你的tunnel網址.try-cloudflare.com|' .env

# 啟動 n8n
docker compose up -d
```

> 為什麼用 `docker compose` 而不是 `docker run`？因為之後 tunnel URL 變了，只需要改 `.env` 再執行 `docker compose up -d --force-recreate`，不用手動 `stop / rm / run` 三步。
>
> 為什麼需要 `WEBHOOK_URL`？n8n 用這個環境變數來產生 webhook 的完整 URL。如果沒設，webhook URL 會是 `http://localhost:5678/webhook/...`，HubSpot 打不到。
>
> `N8N_PROXY_HOPS=1` 是告訴 n8n 前面有一層 proxy/tunnel，讓它正確信任 `X-Forwarded-*` headers。
>
> 如果不加 `N8N_PROXY_HOPS=1`，n8n 日誌可能會出現 `webhook received from untrusted proxy` 相關警告，webhook 仍可能收到，但 URL、protocol 或 proxy 判斷可能不穩定。
>
> `N8N_SECURE_COOKIE=false` 是因為 Cloudflare Tunnel 的 HTTPS 會終止在 Cloudflare 端，n8n 本機跑的是 HTTP，不加這個可能會無法登入。

確認 n8n 跑起來了：

```bash
docker logs n8n --tail 20
```

看到 `Editor is now accessible via: http://localhost:5678/` 就代表成功。

瀏覽器打開 http://localhost:5678 ，先註冊 n8n 的 owner 帳號。

---

## Step 7：Import Workflow

1. 瀏覽器打開 http://localhost:5678
2. 左上選單 → **Import from File** → 選擇 [`workflow.json`](./workflow.json)
3. 設定 Credentials：
   - 點每個 **HubSpot node** → 選擇你的 HubSpot credential（n8n `2.18.5` 顯示為 **HubSpot Service Key**；舊版可能顯示 **HubSpot App Token**）
   - 點 **Get Associated Deals** HTTP Request node：
     - `Authentication` 應該是 **Predefined Credential Type**
     - 如果有 `Credential Type` 欄位，選 **HubSpot Service Key**（舊版可能顯示 **HubSpot App Token**）
     - `Credential` 選剛剛同一個 HubSpot Private App Token credential
     - 如果你看不到 `Credential Type` 欄位，通常代表 `workflow.json` 已經幫你固定成 `hubspotAppToken`；只要在 `Credential` 下拉選同一個 HubSpot credential 即可
   - 點 **OpenAI node** → 選擇你的 OpenAI credential
   - 點 **Slack node** → 選擇 Slack credential（不用 Slack 就刪掉這個 node）
   - 點 **Email node** → 設定 SMTP credential
4. 檢查參數：
   - `Slack` node → 確認 channel 名稱
   - `Send Email` node → **確認 `toEmail` 和 `fromEmail`**（預設是範例值，必須改）
   - `OpenAI` node → 確認 model（預設 `gpt-4o-mini`）
5. 點右上角 **Active** 開關啟用 workflow

> 如果 HubSpot credential 清單看到 **HubSpot Service Key**，請選它，並把 HubSpot Private App Token 貼到 **Service Key** 欄位。n8n `2.18.5` 把原本的 App Token credential 顯示成這個名稱；不要選 **HubSpot API**，那才是舊的/deprecated API Key credential。
>
> 如果 OpenAI node 顯示 `resource=chat` 不支援，請手動換成新版 **OpenAI Chat Model** node，再重新設定模型與 credential。

---

## Step 8：取得 Webhook URL 並設定 HubSpot

1. 在 n8n 裡點開 **Webhook** node
2. 你會看到 **Production URL**，格式像：`https://你的tunnel網址.try-cloudflare.com/webhook/hubspot-new-contact`
3. 複製這個 URL

然後到 HubSpot 設定 webhook（詳細步驟見主 README「設定 HubSpot Webhook」章節）：

1. HubSpot → Automation → Workflows → 建新 Workflow → Contact-based
2. 觸發條件：`Contact created`
3. 加動作：`Send a webhook` → POST → 貼上 Production URL
4. Request body → Custom → 填入：

```json
{
  "contactId": "{{ contact.objectId }}",
  "email": "{{ contact.email }}",
  "firstname": "{{ contact.firstname }}",
  "lastname": "{{ contact.lastname }}",
  "company": "{{ contact.company }}",
  "jobtitle": "{{ contact.jobtitle }}"
}
```

5. 儲存並啟用 Workflow

---

## Step 9：測試

### 先測 n8n 端

1. 在 n8n 裡點 **Webhook** node → 點 **Test** 按鈕（進入測試模式）
2. 複製畫面上顯示的 **Test URL**（通常會是 `/webhook-test/...`，不是 `/webhook/...`）
3. 開另一個終端機，手動發一個測試請求：

> Quick Tunnel 啟動後通常需要 10–30 秒才會傳播到 Cloudflare 邊緣節點，如果第一次 curl 拿到 502，等 30 秒再試。

```bash
curl -X POST https://你的tunnel網址.try-cloudflare.com/webhook-test/hubspot-new-contact \
  -H "Content-Type: application/json" \
  -d '{
    "contactId": "12345",
    "email": "test@example.com",
    "firstname": "Test",
    "lastname": "User",
    "company": "Test Corp",
    "jobtitle": "CEO"
  }'
```

4. 確認 n8n 有收到事件，且後續 node 有正確執行

### 再測 HubSpot 端

1. 確認 n8n 的 Webhook node 已切回 **Production 模式**（關掉 Test，開啟 Active）
2. 在 HubSpot 建一個測試 contact（公司名填「Test Corp」方便辨識）
3. 確認 n8n 有收到 webhook 事件
4. 確認整條流程跑通：high 有 Slack、medium 有 Email，三種 urgency 都會在 HubSpot contact 底下新增 Note

---

## 每次重新開機的啟動順序

WSL 關掉或電腦重開後，需要重新啟動：

```bash
# 1. 啟動 Docker daemon
# 如果你已經用 systemd enable docker，這步通常不需要手動做。
sudo service docker start

# 2. 啟動 Cloudflare Tunnel（會得到一個新的 URL）
cloudflared tunnel --url http://localhost:5678
# 複製新的 tunnel URL
```

開另一個終端機：

```bash
cd ~/n8n-hubspot

# 3. 更新 .env 裡的 WEBHOOK_URL
# 把下面的 URL 換成新的 tunnel URL
sed -i 's|WEBHOOK_URL=.*|WEBHOOK_URL=https://新的tunnel網址.try-cloudflare.com|' .env

# 4. 用新的 WEBHOOK_URL 重啟 n8n
docker compose up -d --force-recreate

# 5. 更新 HubSpot 的 webhook URL（因為 quick tunnel URL 每次都會變）
```

> ⚠️ 這就是為什麼本機 + Quick Tunnel 只適合測試——每次重啟 tunnel URL 都可能會變，n8n 的 `WEBHOOK_URL` 和 HubSpot webhook URL 也要跟著改。如果你會頻繁重啟，強烈建議直接跳到下方「[想用固定 URL，不要每次重啟都變](#想用固定-url不要每次重啟都變)」改用 Named Tunnel。

---

## 常見問題

### cloudflared 警告：`ICMP proxy feature is disabled`

這是 WSL 的權限限制，不影響 HTTP tunnel 功能。可以安全忽略。

### cloudflared 警告：`failed to sufficiently increase receive buffer size`

QUIC 連線的 UDP buffer 不夠大，可能略微影響傳輸效率但不會斷線。如果想消除這個警告：

```bash
# 暫時加大 UDP buffer（重開 WSL 後會還原）
sudo sysctl -w net.core.rmem_max=7500000
sudo sysctl -w net.core.wmem_max=7500000

# 如果要永久生效
sudo bash -c 'echo "net.core.rmem_max=7500000" >> /etc/sysctl.d/99-cloudflared.conf'
sudo bash -c 'echo "net.core.wmem_max=7500000" >> /etc/sysctl.d/99-cloudflared.conf'
```

### Docker 啟動失敗：`Cannot connect to the Docker daemon`

```bash
sudo service docker start
```

### n8n 頁面打不開

```bash
# 確認 container 在跑
docker ps

# 看 log
docker logs n8n --tail 50
```

### Cloudflare Tunnel 連不上 n8n

1. 確認 n8n 有在跑：`curl http://localhost:5678`
2. 確認 cloudflared 指向正確的 port：`--url http://localhost:5678`
3. 確認 WSL 的 port forwarding 沒被擋

### HubSpot webhook 打不到 n8n

1. 確認 cloudflared 還活著（那個終端機不能關）
2. 確認 HubSpot 裡的 webhook URL 是 **tunnel 的公網 URL**，不是 localhost
3. 確認 n8n 的 `WEBHOOK_URL` 環境變數跟 tunnel URL 一致
4. 用 curl 從外部測試 tunnel URL 是否通

### n8n 登入頁面一直跳轉 / 無法登入

確認啟動時有加 `-e N8N_SECURE_COOKIE=false`。沒加的話，n8n 會要求 HTTPS cookie，但 tunnel 的 HTTPS 終止在 Cloudflare，n8n 本機跑的是 HTTP，會衝突。

### 想用固定 URL，不要每次重啟都變

需要設定 Cloudflare **Named Tunnel**，這需要：
1. 一個 Cloudflare 帳號
2. 一個你管理的域名（已託管在 Cloudflare）

設定步驟：

```bash
# 登入 Cloudflare
cloudflared tunnel login

# 建立命名 tunnel
cloudflared tunnel create n8n-dev

# 設定 DNS 指向 tunnel
cloudflared tunnel route dns n8n-dev n8n.你的域名.com

# 用設定檔啟動（URL 永遠固定）
cat > ~/.cloudflared/config.yml << EOF
tunnel: n8n-dev
credentials-file: ${HOME}/.cloudflared/_tunnel-credentials-xxx.json

ingress:
  - hostname: n8n.你的域名.com
    service: http://localhost:5678
  - service: http_status:404
EOF

cloudflared tunnel run n8n-dev
```

這樣 `https://n8n.你的域名.com` 就是固定 URL，重啟也不會變。

---

## 清理

測試完畢，要移除所有東西：

```bash
cd ~/n8n-hubspot

# 停掉並移除 container
docker compose down

# 移除整個 n8n-hubspot 目錄（⚠️ 會刪掉所有 credentials 和 workflow 歷史）
rm -rf ~/n8n-hubspot

# 移除 cloudflared
sudo dpkg -r cloudflared
rm cloudflared.deb
```

---

## 下一步

測試跑通之後，如果要上正式環境，參考主 README 的建議：

- **n8n Cloud Starter**（月繳 ~NT$830，年繳省 17% 約 ~NT$690/月）—— 最省心，webhook URL 永遠穩定
- **VPS 自架**（~NT$150–400/月）—— 最省錢，需要一點 DevOps
