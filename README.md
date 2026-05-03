# HubSpot + n8n：台灣 SME 的第一個 CRM AI Workflow

> 這不是 HubSpot 教學。這是你導入 HubSpot 之後，怎麼讓它「開始變聰明」的第一步。

---

## 給誰看的？

| 角色 | 你的痛點 | 這份東西幫你做什麼 |
|---|---|---|
| **業務主管** | 新 lead 不知道幾天後才有人跟進 | Slack/LINE 立刻收到通知，AI 已幫你判斷優先順序 |
| **行銷** | 不知道哪些表單填寫者是真的客戶 | AI 自動分類 lead 類型，結果寫回 HubSpot，報表直接可用 |
| **IT / 工程師** | 老闆說「讓 HubSpot 更聰明」但不知道從哪裡下手 | 一個可以 import 進 n8n 的 workflow，改幾個參數就能跑 |

---

## 整個流程一句話

> HubSpot 有新 lead → n8n 自動抓資料 → AI 判斷類型和緊急度 → 通知業務 → 把結果寫回 HubSpot

```
┌──────────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────────┐     ┌──────────────────┐
│  HubSpot     │     │  n8n     │     │   n8n    │     │  OpenAI  │     │  通知          │     │  HubSpot         │
│  Webhook     │ ──► │  Webhook │ ──► │  抓資料   │ ──► │  分類    │ ──► │  Slack / Email │ ──► │  寫回 Note /     │
│  (推送事件)   │     │  (接收)   │     │          │     │          │     │               │     │  Property        │
└──────────────┘     └──────────┘     └──────────┘     └──────────┘     └──────────────┘     └──────────────────┘
```

---

## 名詞快查

| 詞 | 白話 |
|---|---|
| **HubSpot** | 客戶名單系統（像 Excel + 業務系統） |
| **n8n** | 自動化流程工具（像 Zapier，但可以自己架、免費） |
| **LLM / AI** | 幫你判斷內容的模型（像 OpenAI GPT、Claude） |
| **Private App Token** | HubSpot 給你的一把鑰匙，讓外部工具讀寫你的資料 |
| **OAuth2** | 比較複雜的鑰匙，適合要給別人用的情況 |
| **Workflow** | 自動化流程——定義「什麼事發生 → 做什麼反應」 |

> n8n 2.x 命名提醒：在 n8n `2.18.5`，Private App Token 對應的 credential 可能顯示成 **HubSpot Service Key**，欄位名稱是 **Service Key**。這個 credential 內部仍是 `hubspotAppToken`，會用 `Authorization: Bearer <token>` 呼叫 HubSpot，可以貼 HubSpot Private App 的 token。不要選 **HubSpot API**，那是舊的 `hapikey` API Key 路徑。

---

## HubSpot 認證 Decision Tree

這是最容易卡死的地方。HubSpot 有三種方式讓 n8n 連進去讀寫資料，但不是每一種都適合你。

### Decision Tree

```
你的 n8n 是要連「自己公司」的 HubSpot 嗎？
│
├─ 是 ──► 只要自己公司用，不需要給客戶裝
│         │
│         ├─ 是 ──► ✅ 用 Private App Token（往下看 §A）
│         │
│         └─ 否，要做成 SaaS 給別人用 ──► 用 OAuth2（往下看 §B）
│
└─ 否，要連別人的 HubSpot ──► 用 OAuth2（往下看 §B）


那 Service Keys 呢？ ──► ❌ 不建議（往下看 §C）
```

---

### A. Private App Token（✅ 建議第一版用這個）

**適合**：自己公司用、內部自動化、一個 HubSpot 帳號對一個 n8n

**優點**：
- 設定最簡單，5 分鐘搞定
- 不需要 redirect URL、不需要 developer 帳號
- 一個 token 搞定讀寫

**缺點**：
- Token 屬於建立者個人——如果那人離職，token 會失效
- 無法自動 refresh，過期要手動換
- 不適合多租戶（multi-tenant）場景

**設定步驟**：

1. 登入 HubSpot → 左上 ⚙️ **Settings**
2. 左側選單 → **整合工具** → **舊版應用程式**
3. 右上角點 **建立舊版應用程式** → 選 **Private**
4. 填 App 名稱（例如 `n8n-workflow`）和描述
5. 切到 Scopes tab → 點 **Add new scope**，加入以下權限：

| 類別 | 權限 | Scope 名稱 |
|---|---|---|
| CRM | Contacts 讀寫 | `crm.objects.contacts.read` / `crm.objects.contacts.write` |
| CRM | Deals 讀寫 | `crm.objects.deals.read` / `crm.objects.deals.write` |
| CRM | Companies 讀寫 | `crm.objects.companies.read` / `crm.objects.companies.write` |
| CRM | Owners 讀 | `crm.objects.owners.read` |
| CRM | Lists 寫 | `crm.lists.write` |

6. 點 **Update** 儲存 scopes → 點右上 **Create app** → 點 **Continue creating**
7. 進入 App 詳情頁 → 切到 **Auth** tab → 點 **Show token** → 複製 token
8. 進 n8n → Credentials → Add credential → 選 **HubSpot Service Key**（或舊版顯示為 **HubSpot App Token**）→ 把 Private App Token 貼到 **Service Key / APP Token** 欄位

> ⚠️ 如果 n8n 畫面只看到 **HubSpot API**、**HubSpot Developer API**、**HubSpot OAuth2 API**、**HubSpot Service Key**，請選 **HubSpot Service Key**。在 n8n `2.18.5`，這個選項就是原本的 App Token credential。不要選 **HubSpot API**，那會打開舊的/deprecated API Key 欄位。
>
> n8n 官方文件目前列出的 HubSpot 支援認證方式是 **App token**（給 HubSpot node）、**Developer API key**（給 HubSpot Trigger node）和 **OAuth2**；一般 API Key 已被 HubSpot deprecated，但舊選項可能還會留在 n8n UI 裡。

> ⚠️ **Token 只會顯示一次**，請立刻存到密碼管理器。如果忘了，只能重新產生。
>
> 💡 建立 Private App 時會看到 **Webhook** tab——**不用填**。這個 tab 是給 Webhook API 訂閱用的（需要 Developer 帳號）。我們的 workflow 用的是 HubSpot Workflow 裡的「Send a webhook」動作來推送事件，不需要在這裡註冊 webhook。設定方式見下方「設定 HubSpot Webhook」章節。
>
> 💡 HubSpot 已將 Private Apps 改名為 **舊版應用程式**（Legacy Apps），入口從原本的 Settings → Integrations → Private Apps 移到 Settings → 整合工具 → 舊版應用程式。舊的「私人應用程式」入口會顯示搬遷提示並導向新位置。參考：[HubSpot Legacy Private Apps 文件](https://developers.hubspot.com/docs/apps/legacy-apps/private-apps/overview)

---

### B. OAuth2（進階用，第一版通常不需要）

**適合**：做 SaaS 產品、給客戶裝、需要多個 HubSpot 帳號授權

**優點**：
- 用戶自己授權，不需要把 token 給你
- 自動 refresh，不會過期斷線
- 適合多租戶架構

**缺點**：
- 需要 HubSpot Developer 帳號
- 需要設定 Redirect URL
- 設定步驟多，除錯成本高
- n8n Cloud 可以一鍵連接，但 self-hosted 需要手動設

**設定步驟（self-hosted n8n）**：

1. 到 [HubSpot Developer](https://developers.hubspot.com/) 註冊 developer 帳號
2. Apps → Create app → 填 App Info
3. Auth tab → 複製 Client ID、Client Secret、App ID
4. Scopes → 加入和 Private App Token 相同的權限
5. Redirect URL → 填 n8n 的 OAuth callback URL（格式：`https://你的n8n網址/rest/oauth2-credential/callback`）
6. 進 n8n → Credentials → Add credential → HubSpot API → 選 OAuth2 → 填入 Client ID、Client Secret
7. 點 Connect my account → 瀏覽器會跳轉到 HubSpot 授權頁面 → 授權完成

> ⚠️ 如果你的 n8n 跑在 localhost 或內網，OAuth2 的 redirect 會需要額外處理。第一版建議先用 Private App Token。

---

### C. HubSpot 原生 Service Keys（❌ 不建議）

> 注意：這裡說的是 HubSpot 平台原生的 Server-to-Server OAuth **Service Keys**。n8n `2.18.5` credential 清單裡的 **HubSpot Service Key** 是另一回事：它內部仍是 `hubspotAppToken`，可用來貼 Private App Token。

| 項目 | 說明 |
|---|---|
| **是什麼** | HubSpot 的 Server-to-Server OAuth，設計給後端服務用的 |
| **為什麼不建議** | 很多功能不支援（例如 webhook 註冊、部分 CRM write 操作） |
| **適用場景** | 只有在做 batch 資料同步、不需要即時事件觸發時才考慮 |
| **n8n 支援** | n8n 的 HubSpot node 不原生支援 Service Keys，需要用 HTTP Request node 自己呼叫 API |

> 結論：如果你不確定，就用 Private App Token。Service Keys 留給需要 batch 同步的進階場景。

---

## n8n Credentials 設定對照

| 認證方式 | n8n Credential 類型 | 需要填什麼 | 適用 Node |
|---|---|---|---|
| Private App Token | HubSpot Service Key（舊版可能顯示 HubSpot App Token） | 一個 Access Token / Private App Token | HubSpot node（讀寫聯絡人、Deal、Note） |
| OAuth2 | HubSpot API → OAuth2 | Client ID + Client Secret | HubSpot node（多租戶場景） |
| — | Webhook node | 不需要 credential | 接收 HubSpot 推送事件 |

> 💡 **第一版最簡路徑**：Webhook node 接收事件 + Private App Token 讀寫 HubSpot 資料。全程不需要 Developer 帳號。

---

## Workflow 說明

這個 workflow 有七個主要步驟：

| 步驟 | Node | 做什麼 |
|---|---|---|
| 1 | `Webhook` | 接收 HubSpot 推送的「新 Contact」事件 |
| 2 | `HubSpot` (Get Contact) | 用 contact ID 去抓完整資料（姓名、公司、職稱、email） |
| 3 | `HTTP Request` (Get Associated Deals) | 用 HubSpot Associations API 抓這個 contact 關聯的 deal |
| 4 | `OpenAI` | 分類 urgency，推測缺失資料，標注 confirmed / inferred / missing |
| 5 | `Code` (Parse AI Classification) | 把 AI 回傳的 JSON 字串 parse 成後續節點可讀的簡報欄位 |
| 6 | `Switch` | 根據 AI 分類結果，走不同通知路徑 |
| 6a | `Slack` | 高緊急 → Slack 發送業務跟進簡報 |
| 6b | `Send Email` | 中緊急 → Email 發送業務跟進簡報 |
| 6c | `HubSpot` (Create Note) | 低緊急 → 不通知，只寫回 HubSpot Note |
| 7 | `HubSpot` (Create Note) | 三種緊急度都會把跟進簡報寫回原 contact 的 HubSpot Note |

### AI 分類與資料補完邏輯

AI 會根據以下資訊判斷：

- **公司名稱 & 規模** → 是不是大客戶？
- **職稱** → 是決策者還是詢價者？
- **來源** → 是主動填表還是被動匯入？
- **Deal 關聯** → 這個 contact 是否已經有關聯 deal？

AI 輸出會分成四塊，讓業務知道哪些可以信、哪些要問：

| 區塊 | 用途 | 範例 |
|---|---|---|
| `confirmed_data` | HubSpot 已有、客戶實填或系統已確認的資料 | Email、姓名、公司名稱 |
| `inferred_data` | AI 根據現有資料推測，必須標注信心程度 | 公司規模：中型、產業：製造業 |
| `missing_data` | 業務需要補問的欄位 | 預算、導入時間表、決策流程 |
| `follow_up_questions` | 業務可直接使用的問句 | 「您目前團隊大概幾位同仁會使用？」 |

分類結果：

| 類型 | 定義 | 通知方式 |
|---|---|---|
| 🔴 **High** | 大客戶 / 高意願 / 決策者 | Slack 即時通知 |
| 🟡 **Medium** | 有興趣但需要培育 | Email 通知 |
| 🟢 **Low** | 初步詢價 / 資訊收集 | 只寫回 HubSpot，不打擾 |

### 業務跟進簡報格式

通知業務時，訊息會長這樣：

```text
✅ 確認資料
Email: amy@example.com
姓名: Amy Chen

🔍 AI 推測，待業務確認
公司規模: 中型（信心：medium；依據：公司名稱與職稱）
產業: 製造業（信心：low；依據：公司 domain / 公司名稱線索）

❓ 待確認
預算: HubSpot 尚未提供，業務需確認
時間表: 尚未知道是否本季導入

📋 建議問法
1. 您目前團隊大概幾位同仁會使用這套流程？
2. 如果評估順利，預計希望什麼時間開始導入？
```

這個設計的重點是：AI 可以先幫業務做功課，但所有推測值都會標成 `ai_inferred_pending_confirmation`，等業務跟進後再用真實資料覆蓋。

### HubSpot Property 建議

如果你想把「AI 推測 vs 業務確認」做成 KPI，建議在 HubSpot contact 建這些自訂欄位：

| Property | 類型 | 用途 |
|---|---|---|
| `ai_inferred_company_size` | Single-line text | AI 推測公司規模 |
| `ai_inferred_industry` | Single-line text | AI 推測產業 |
| `ai_inference_confidence` | Dropdown | `low` / `medium` / `high` |
| `data_completion_status` | Dropdown | `ai_inferred` / `sales_confirmed` / `incomplete` |
| `sales_confirmed_at` | Date picker | 業務完成確認的日期 |
| `missing_fields_to_confirm` | Multi-line text | 待確認欄位清單 |

第一版 workflow 先用 HubSpot Note 保存完整簡報，避免 import 時因為自訂欄位不存在而失敗。等欄位建好後，可以再加一個 HubSpot Update Contact node，把 AI 推測值寫到上述 properties。

### KPI 追蹤方式

| KPI | 定義 |
|---|---|
| 資料補完時效 | 收到 AI 跟進簡報後，業務在 7 天內把 `data_completion_status` 改成 `sales_confirmed` |
| 補完比例 | 一週內完成確認的 leads / 所有新 leads |
| AI 推測可用率 | 業務確認後未被修改的 AI 推測欄位 / AI 推測欄位總數 |

---

## 已修正的 Import 注意事項

這份 `workflow.json` 已處理幾個常見踩雷點：

| 問題 | 修正方式 |
|---|---|
| AI 回傳 JSON 字串，Switch 讀不到 `urgency` | 在 `AI Lead Classifier` 後加入 `Parse AI Classification` Code node |
| HubSpot note 沒有掛到 contact | `Write Back to HubSpot` 已加入 `contactId`，會寫回原 contact |
| Deal 節點抓到全部 deals | `Get Associated Deals` 改用 HubSpot Associations API，只抓目前 contact 關聯的 deals；若要用金額評分，可再加一個 batch read deals 節點 |
| Low urgency 沒後續處理 | `low` 分支直接連到 `Write Back to HubSpot`，保留紀錄但不打擾業務 |
| Switch node 三個 output 在某些 n8n 版本沒自動展開 | Import 後打開 `Route by Urgency`，確認 high / medium / low 三個 output 都已連到對應節點，缺的話手動拖線 |

> 若你的 n8n 版本匯入 OpenAI node 後顯示 `resource=chat` 不支援，請在該節點手動改用新版 OpenAI Chat node，或改成舊版 `text / complete` 參數。其他節點邏輯不需要變。

---

## Import 指南

### 前置條件

- [ ] n8n 已安裝並運行，且可以從外部訪問（HubSpot 需要能打到你們的 webhook URL）

  **n8n 必須跑在公開可達的環境**，有三個選項：

  | 選項 | 月費估算 | 說明 | 適合誰 |
  |---|---|---|---|
  | **1. n8n Cloud Starter** | ~NT$830（月繳）/ ~NT$690（年繳省 17%）| 開箱即用，自帶 HTTPS URL，2,500 次執行/月 | 技術資源少的團隊、想最快跑起來 |
  | **2. VPS 自架 Community 版** | ~NT$150–400 | 自架在 Hetzner / Vultr 等，需設定 domain + HTTPS + `WEBHOOK_URL`，執行次數無限制 | 有一點 DevOps 基礎、想省月費 |
  | **3. 本機 + tunnel 工具** | 免費 | 用 ngrok / Cloudflare Tunnel 暫時打通 localhost（[詳細步驟](./WSL-DOCKER-TUNNEL-GUIDE.md)） | 只適合開發測試 |

  <details>
  <summary>📊 n8n Cloud 定價明細</summary>

  | 方案 | 月繳（€/月） | 年繳（€/月，省 17%） | 月繳台幣 | 執行次數/月 | 適合對象 |
  |---|---|---|---|---|---|
  | **Starter** | €24 | €20 | ~NT$830 | 2,500 次 | 個人、小團隊初試 |
  | **Pro** | €60 | €50 | ~NT$2,070 | 10,000 次 | 成長中的小團隊 |
  | **Business** | €800 | €667 | ~NT$27,600 | 40,000 次 | 中大型企業 |
  | **Enterprise** | 客製報價 | — | 洽詢 | 無限制 | 大企業 |

  > 數字取自 [n8n 官網定價頁](https://n8n.io/pricing/)。匯率以 1 EUR ≈ 34.5 TWD 估算。所有方案皆**不限 active workflow 數與使用者數**，計費僅看執行次數。**第一版測試建議先選月繳**，跑穩再轉年繳省 17%。實際以官網最新報價為準。
  </details>

  <details>
  <summary>💡 Starter 夠用嗎？算給你看</summary>

  n8n 的 execution 是以「workflow 執行次數」計算（一次完整 run = 1 execution，不是 node 數）。假設每天平均 8 個新 lead，一個月約 240 次 webhook 觸發 = 240 executions——Starter 的 2,500 次有將近 10 倍 buffer，完全夠用。如果同時跑多條 workflow 或 lead 量更大，再升 Pro。
  </details>

  > **建議**：第一版從 **n8n Cloud Starter 月繳（~NT$830/月）** 開始——zero DevOps、webhook URL 立刻可用、穩定不斷線、隨時可停。等流程跑穩、確定要長期用，再轉年繳省 17%（~NT$690/月）或遷到 VPS 自架降低長期成本。
- [ ] HubSpot 帳號，且已有聯絡人資料
- [ ] 已建立 HubSpot Private App Token（見上方 §A）
- [ ] OpenAI API Key
- [ ] （選用）Slack workspace + Bot Token
- [ ] （選用）SMTP 設定（用於 Email 通知）

### Import 步驟

1. 下載 [`workflow.json`](./workflow.json)
2. 打開 n8n → 左上選單 → Import from File → 選擇 `workflow.json`
3. 設定 Credentials：
   - 點每個 HubSpot node → 選擇你的 HubSpot credential（n8n `2.18.5` 顯示為 **HubSpot Service Key**；舊版可能顯示 **HubSpot App Token**）
   - 點 `Get Associated Deals` HTTP Request node → credential type 選 **HubSpot Service Key / HubSpot App Token**，credential 選同一個 Private App Token
   - 點 OpenAI node → 選擇你的 OpenAI credential
   - 點 Slack node → 選擇你的 Slack credential（如果不用 Slack，可以刪掉這個 node）
   - 點 Email node → 設定 SMTP credential
4. 檢查每個 node 的參數：
   - `Slack` node → 確認 channel 名稱
   - `Send Email` node → **確認收件人 `toEmail` 和寄件人 `fromEmail`**（預設是範例值，必須改成你的實際 email）
   - `OpenAI` node → 確認 model（預設 `gpt-4o-mini`，可換成其他模型）
5. 點右上角 **Active** 開關啟動 workflow
6. 複製 Webhook node 顯示的 **Test URL** 和 **Production URL**

> 如果建立 HubSpot credential 時只看到 **API Key** 欄位，請不要在那裡硬填。那是 HubSpot 已淘汰的一般 API Key 路徑。n8n `2.18.5` 請改選 **HubSpot Service Key**，把 HubSpot Private App Token 貼到 **Service Key** 欄位；舊版 n8n 則可能顯示為 **HubSpot App Token / APP Token**。

### 設定 HubSpot Webhook

這個 workflow 用 Webhook 接收 HubSpot 事件，而不是用 n8n 的 HubSpot Trigger node。好處是：只需要 Private App Token，不需要 Developer 帳號。

**方法一：用 HubSpot Workflow（推薦，最簡單）**

1. 登入 HubSpot → Automation → Workflows
2. 建立新 Workflow → 選「From scratch」→ 類型選「Contact-based」
3. 設定觸發條件：`Contact created` 或你想要的篩選條件
4. 加動作：`Send a webhook`
   - Webhook URL → 貼上 n8n 的 **Production URL**
   - Method → `POST`
   - Request body → 選 `Custom`，填入：
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

**方法二：用 HubSpot API（進階）**

如果你需要更精確的事件控制（例如只監聽特定屬性變更），可以用 HubSpot 的 Webhooks API 註冊 webhook。這需要建立一個 Public App，流程比較複雜。第一版建議用方法一。

### 測試建議

- 先在 n8n 點 Webhook node 的 **Test** 按鈕，進入測試模式
- 在 HubSpot 建一個測試 contact（例如公司名填「Test Corp」）
- 確認 n8n 有收到 webhook 事件
- 確認整條流程跑通：high 有 Slack、medium 有 Email，三種 urgency 都會在原 HubSpot contact 底下新增 Note
- 測試成功後，記得切回 Production 模式（關掉 Test，開啟 Active）

---

## 自訂建議

### 換 AI 模型

OpenAI node 可以換成其他 LLM：

| 想用 | n8n Node | 備註 |
|---|---|---|
| Claude | Anthropic node | 需要 Anthropic API Key |
| Gemini | Google Gemini node | 需要 Google AI API Key |
| 自架模型 | HTTP Request node | 呼叫自己的 Ollama / vLLM endpoint |

### 加 LINE 通知

把 Slack node 複製一份，類型換成 `HTTP Request`，呼叫 LINE Messaging API：

```
POST https://api.line.me/v2/bot/message/push
Header: Authorization: Bearer {LINE_CHANNEL_ACCESS_TOKEN}
Body: { "to": "{USER_ID}", "messages": [{ "type": "text", "text": "..." }] }
```

### 改分類邏輯

在 OpenAI node 的 prompt 裡調整分類條件。例如：

- 加「是否來自競品」的判斷
- 加「是否在特定產業」的判斷
- 把三級分類改成五級

### 加 Deal 自動建立

在 HubSpot Create Note 之前加一個 HubSpot node：

- 類型：`Create` → `Deal`
- 條件：AI 分類為 High 時，自動建一個 deal
- Pipeline 和 Stage 可以根據你的 HubSpot 設定調整

---

## 常見問題

### HubSpot Webhook 沒有觸發？

1. 確認 n8n workflow 是 Active 狀態
2. 確認 HubSpot Workflow 有啟用，且 webhook URL 是 n8n 的 **Production URL**（不是 Test URL）
3. 確認你的 n8n URL 可以從 HubSpot 訪問到（self-hosted 不能用 `localhost`，需要公開域名或 tunnel）
4. 如果用 tunnel 工具（如 ngrok），確認 tunnel 還活著
5. 在 n8n 的 Webhook node 點 Test 進測試模式，手動在 HubSpot 建一個 contact，看 n8n 有沒有收到請求

### OpenAI 分類不準？

1. 調整 prompt，加入你的產業特定判斷條件
2. 換用更強的模型（`gpt-4o` 或 `claude-sonnet`）
3. 在 prompt 裡加 few-shot 範例（例如「如果職稱是 CTO，分類為 High」）

### n8n 跑在 localhost，HubSpot 打不到怎麼辦？

HubSpot 的 webhook 是從 HubSpot 伺服器主動打出去的，你的 n8n 必須有一個**公開可達的 HTTPS URL**，localhost 預設無法接收。

有三種解法，適合不同場景：

| 方法 | 指令 | 適合場景 | 注意事項 |
|---|---|---|---|
| **ngrok** | `ngrok http 5678` | 臨時測試 | 免費版每次重啟 URL 會變，HubSpot 那邊的 webhook URL 也要跟著改 |
| **Cloudflare Tunnel** | `cloudflared tunnel --url http://localhost:5678` | 開發期間 | 免費、比 ngrok 穩定，但本機關機就斷 |
| **n8n Cloud Starter** | 直接用 | 正式環境 | ~NT$830/月（月繳），最簡單，開箱即用；長期用轉年繳可降到 ~NT$690 |
| **VPS 自架** | Docker + domain + HTTPS | 正式環境 | ~NT$150–400/月（Hetzner / Vultr），Community 版免費、執行次數無限制，但進階功能（全域變數、使用者管理等）需付 Business 費用 |

> ⚠️ 本機架設不適合正式環境：筆電關機 → webhook 斷線，ngrok 免費版 URL 每次變動 → HubSpot webhook 也要跟著改，IT 不在場時沒人處理。

### Token 過期怎麼辦？

Private App Token 預設不會過期，但如果建立者被移除 HubSpot 帳號，token 會失效。建議：
- 用一個 service account 建立 Private App
- 或者在團隊共用的密碼管理器裡記錄 token
- 長期考慮遷移到 OAuth2

---

## 檔案清單

| 檔案 | 說明 |
|---|---|
| [`README.md`](./README.md) | 你正在看的這份文件 |
| [`workflow.json`](./workflow.json) | 可直接 import 到 n8n 的 workflow JSON |
| [`WSL-DOCKER-TUNNEL-GUIDE.md`](./WSL-DOCKER-TUNNEL-GUIDE.md) | WSL + Docker + Cloudflare Tunnel 本地測試指南 |
| [`n8n-hubspot/docker-compose.yml`](./n8n-hubspot/docker-compose.yml) | Docker Compose 設定（搭配 `.env` 使用） |
| [`n8n-hubspot/.env.example`](./n8n-hubspot/.env.example) | 環境變數範本（使用前 cp .env.example .env） |
| [`n8n-hubspot/.gitignore`](./n8n-hubspot/.gitignore) | 排除 `.env` 和 `n8n-data/` |

---

## 授權

MIT License — 自由使用、修改、散佈。不需要標註來源，但如果這份東西幫到你，歡迎分享給其他台灣 SME。
