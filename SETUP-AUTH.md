# 啟用真實 Google 登入與後端簽章驗證

本文件說明如何把目前的**示範登入模式**升級為**真實 Google 登入 + 伺服器端 ID token 簽章驗證**（即程式中的 `backend` 模式）。

---

## 0. 目前狀態

線上版本（v10）的 `AUTH_CFG` 兩個必要值都是空的，因此自動落在 **`demo` 模式**：

| 項目 | 目前狀況 |
|---|---|
| 登入頁與閘門 | ✅ 真的在運作（未登入看不到任何算牌／對子功能） |
| 登入狀態保存 | ✅ 真的在運作（`localStorage`，30 天，跨分頁同步） |
| 登出 | ✅ 真的在運作 |
| 是否連線 Google | ❌ 否。不載入 Google 腳本、不送出任何資料 |
| 是否驗證簽章 | ❌ 否。`localStorage` 可被技術人士偽造繞過 |

登入頁上有一張**「啟用真實簽章驗證所需設定」**檢查表，會即時顯示還缺哪幾個值（目前應顯示 `0 / 2 · 尚缺 2 項`）。填好並重新部署後，它會變成綠色 `2 / 2 · 真實簽章驗證就緒`，模式也會自動切換為 `backend`。

---

## 1. 需要你提供的兩個值

只要這兩個值，就能從 `demo` 切到 `backend`：

| # | 設定值 | 填在哪裡 | 從哪裡取得 | 格式範例 |
|---|---|---|---|---|
| **1** | **Google OAuth 2.0 用戶端 ID** | 前端 `AUTH_CFG.googleClientId`<br>後端 `GOOGLE_CLIENT_ID` | Google Cloud Console → APIs & Services → Credentials | `123456789012-abc123def456.apps.googleusercontent.com` |
| **2** | **後端驗簽服務網址** | 前端 `AUTH_CFG.backendUrl` | 你自己部署後拿到的網址（見第 3 節） | `https://auth.example.com` |

**選填（有預設值，不提供也能跑）：**

| 項目 | 預設 | 說明 |
|---|---|---|
| 允許來源清單 `ALLOWED_ORIGINS` | `http://localhost:5173` | 後端 CORS 白名單，正式環境要換成你的網站網址 |
| Session 效期 `SESSION_DAYS` | `7` | 後端 cookie 有效天數（前端登入頁的文字會自動跟著改） |
| Cookie 安全旗標 `COOKIE_SECURE` | `false` | **正式環境（HTTPS）必須設 `true`** |
| 同站策略 `COOKIE_SAME_SITE` | `Lax` | 前後端**不同網域**時必須設 `None` 且需 HTTPS |
| 自訂網域 | 無 | 若你要改用自有網域，需要另外加進 Google 的授權來源 |

---

## 2. Google Cloud Console 設定（取得第 1 個值）

1. 開啟 <https://console.cloud.google.com/>，建立一個專案（或選用既有專案）。
2. 左側選單 → **APIs & Services** → **OAuth consent screen**
   - User Type 選 **External**
   - 填 App name、User support email、Developer contact email
   - **Scopes**：只需要 `.../auth/userinfo.email` 與 `.../auth/userinfo.profile`（即 `email`、`profile`），**不要**加其他範圍
   - 若要對外開放，記得把 Publishing status 從 *Testing* 改成 *In production*，或在 *Test users* 加入測試帳號
3. 左側選單 → **APIs & Services** → **Credentials** → **Create Credentials** → **OAuth client ID**
   - Application type 選 **Web application**
   - **Authorized JavaScript origins** 填入（只填 scheme + host，**不含路徑、不含結尾斜線**）：
     ```
     https://chg983.github.io
     http://localhost:5173        ← 本機測試用，可選
     ```
   - **Authorized redirect URIs**：本專案使用 Google Identity Services 的 **popup 模式**，**不需要填**任何 redirect URI
4. 建立後複製 **Client ID**（結尾是 `.apps.googleusercontent.com`）。
   - ⚠️ 只需要 Client ID。**不要**把 Client Secret 放進前端或 repo。本流程用的是 ID token 驗證，不需要 Secret。

> **注意**：`Authorized JavaScript origins` 只認 host。`https://chg983.github.io/baccarat-counter/` 要填成 `https://chg983.github.io`。

---

## 3. 部署後端（取得第 2 個值）

repo 內 `backend/` 提供兩個行為完全一致的實作，挑一個部署即可：

```
backend/
├── node/          Node.js 20 + Express 4（零驗證依賴，用內建 crypto）
├── python/        Python 3.11 + FastAPI + PyJWT
├── shared/        跨語言共用測試向量產生器
└── scripts/       mint-token.py、smoke-http.sh
```

### 3-A. Node.js / Express

```bash
cd backend/node
npm install

export GOOGLE_CLIENT_ID="123456789012-abc123def456.apps.googleusercontent.com"
export ALLOWED_ORIGINS="https://chg983.github.io"
export COOKIE_SECURE="true"          # 有 HTTPS 就要開
export COOKIE_SAME_SITE="None"       # 前後端不同網域時必要
export SESSION_DAYS="7"
export PORT="8787"

npm start
```

### 3-B. Python / FastAPI

```bash
cd backend/python
pip install -r requirements.txt

export GOOGLE_CLIENT_ID="123456789012-abc123def456.apps.googleusercontent.com"
export ALLOWED_ORIGINS="https://chg983.github.io"
export COOKIE_SECURE="true"
export COOKIE_SAME_SITE="None"
export SESSION_DAYS="7"

uvicorn app.main:app --host 0.0.0.0 --port 8787
```

### 後端環境變數一覽

| 變數 | 必填 | 預設 | 說明 |
|---|---|---|---|
| `GOOGLE_CLIENT_ID` | **是** | — | 與前端同一個 OAuth Client ID；也是 `aud` 檢查的比對值 |
| `ALLOWED_ORIGINS` | 建議 | `http://localhost:5173` | 逗號分隔的 CORS 白名單，**不可用 `*`**（帶 credentials 時瀏覽器也會拒絕） |
| `PORT` | 否 | `8787` | 監聽埠 |
| `COOKIE_SECURE` | 正式必開 | `false` | `true` 時 cookie 只走 HTTPS |
| `COOKIE_SAME_SITE` | 跨網域必設 | `Lax` | `Lax` / `None` / `Strict` |
| `COOKIE_DOMAIN` | 否 | 不設定 | 前後端同父網域時可設 `.example.com` |
| `SESSION_DAYS` | 否 | `7` | session 有效天數 |

### 後端端點

| 方法 | 路徑 | 說明 |
|---|---|---|
| `POST` | `/api/auth/google` | 收到 `{ credential }` → 驗章 → `Set-Cookie: sid=…; HttpOnly` |
| `GET` | `/api/me` | 有有效 cookie → `{ ok:true, user:{name,email,picture,sub} }`；否則 `401` |
| `POST` | `/api/auth/logout` | 刪除伺服器 session 並清 cookie |
| `GET` | `/api/health` | 健康檢查 |

### 部署平台（任一皆可）

Render / Railway / Fly.io / Cloud Run / 一台 VPS + Caddy。重點只有三個：

1. **必須是 HTTPS**（GitHub Pages 是 HTTPS，瀏覽器會擋 mixed content）。
2. **`ALLOWED_ORIGINS` 要精確等於 `https://chg983.github.io`**（協定、網域、埠都要完全一致，不要路徑）。
3. **`COOKIE_SECURE=true` 且 `COOKIE_SAME_SITE=None`**——因為前端在 `chg983.github.io`、後端在別的網域，屬於跨站請求。

> **最省事的做法**：把前端與後端放在**同一個網域**（例如 `app.example.com` 前端、`api.example.com` 後端，或反向代理成 `/api/*`），這樣 `SameSite=Lax` 就能用，也避開 Safari ITP 與 Chrome 第三方 cookie 政策的問題。

---

## 4. 前端填入（一行）

打開 `index.html`，找到 `<script>` 最下方的 `AUTH_CFG`：

```js
const AUTH_CFG = {
  googleClientId: "123456789012-abc123def456.apps.googleusercontent.com",  // ← 第 1 個值
  backendUrl:     "https://your-auth-service.onrender.com",                  // ← 第 2 個值
  sessionDays: 30,
  storageKey: "baccarat-auth-v1"
};
```

改完存檔、重新部署（`index.html` 複製到 repo 根目錄）。頁面會自動：

- 隱藏自繪的示範登入按鈕，改用 **Google 官方登入按鈕**
- 把 credential `POST` 到 `{backendUrl}/api/auth/google`
- 改以 `GET {backendUrl}/api/me` 判定登入狀態（**不再信任 `localStorage`**）
- 登入頁的檢查表變成 `2 / 2 · 真實簽章驗證就緒`
- 登入者資訊顯示真實 Google 姓名／頭像，副標顯示 `Google 帳戶 · <email>`

### 三種模式對照

| 模式 | 條件 | 憑證驗證 | 存取控制 |
|---|---|---|---|
| `demo` | 兩個值都空 | 無 | ❌ 可偽造 |
| `google` | 只有 `googleClientId` | 前端解碼，**未驗簽章** | ❌ 可偽造 |
| **`backend`** | 兩個值都填 | **後端驗證 RS256 簽章 + aud + iss + exp** | ✅ 伺服器決定 |

---

## 5. 驗證清單（部署後逐項確認）

| # | 檢查 | 預期結果 |
|---|---|---|
| 1 | 未登入開網站 | 只見登入頁；`#app-shell` 不可見；按 `k`、`9` 不會發牌 |
| 2 | `curl -i {backendUrl}/api/me` | `401` |
| 3 | 點 Google 登入按鈕 | 跳出 Google 官方帳戶選擇器（不是自繪的示範選擇器） |
| 4 | 登入成功後 | 主介面出現，右上顯示真實姓名／頭像；DevTools → Application → Cookies 可見 `sid`，**HttpOnly 打勾** |
| 5 | `localStorage` | **沒有** `baccarat-auth-v1`（backend 模式不寫本機狀態） |
| 6 | 按 F5 重新整理 | 維持登入（透過 `/api/me` 還原） |
| 7 | 手動清掉 cookie 再整理 | 回到登入頁（且偽造 `localStorage` 也無效） |
| 8 | 按登出 | 回到登入頁；`/api/me` 變 `401`；cookie 消失 |
| 9 | 用 `scripts/mint-token.py` 簽一支錯章 token POST 到 `/api/auth/google` | `401` + `bad_signature` |

第 9 項指令：

```bash
python3 backend/scripts/mint-token.py --rogue-key > /tmp/bad.txt
curl -i -X POST "$BACKEND/api/auth/google" \
  -H "Content-Type: application/json" \
  -H "Origin: https://chg983.github.io" \
  --data "{\"credential\":\"$(cat /tmp/bad.txt)\"}"
```

### 自動化測試

```bash
# Node.js：51 項（單元 + 真實 HTTP 端到端）
cd backend/node && node shared/build_cases.js && npm test

# 跨語言共用向量：Node 29 項 / Python 29 項（必須逐筆錯誤代碼相同）
node backend/node/test/shared-cases.test.js
cd backend/python && python3 -m tests.test_shared_cases

# HTTP 煙霧測試（自動起服務、14 項斷言、自動清理）
bash backend/scripts/smoke-http.sh --node
bash backend/scripts/smoke-http.sh --python
```

> `shared/fixtures/` 內的測試金鑰、`cases.json`、`google_jwks.json` **未隨 repo 提供**（避免把私鑰放進公開 repo）。第一次執行測試前先跑一次 `node backend/shared/build_cases.js` 生成即可——它會產生全新的測試金鑰與 29 組測試向量。這些金鑰**只供本機測試**，真實部署會拒絕它們簽的 token。

---

## 6. 後端驗證做了什麼（11 道檢查）

順序是刻意的：**先驗簽章，再談任何 claim**。

| # | 檢查 | 錯誤代碼 | HTTP |
|---|---|---|---|
| 1 | 三段結構 / base64url / JSON | `malformed_token` | 400 |
| 2 | `header.alg` 必須是 `RS256` | `unsupported_alg` | 401 |
| 3 | `header.kid` 在 Google JWKS 中 | `unknown_kid` | 401 |
| 4 | 以 Google 公鑰驗 RS256 簽章 | `bad_signature` | 401 |
| 5 | `exp` 未過期（容許 5 秒時鐘差） | `token_expired` | 401 |
| 6 | `iat` 不在未來 | `token_not_yet_valid` | 401 |
| 7 | `iss` ∈ {`accounts.google.com`, `https://accounts.google.com`} | `wrong_issuer` | 401 |
| 8 | `aud` = 本服務 Client ID | `wrong_audience` | 401 |
| 9 | `sub` 存在 | `missing_sub` | 401 |
| 10 | `email` 存在 | `missing_email` | 401 |
| 11 | `email_verified === true` | `email_unverified` | 403 |

第 2 步的演算法白名單是擋 `alg:none` 與 `alg:HS256`（拿公鑰當 HMAC secret）混淆攻擊的關鍵。

JWKS 來源：`https://www.googleapis.com/oauth2/v3/certs`，快取 TTL 依回應的 `Cache-Control: max-age`；遇到未知 `kid` 會強制刷新一次（處理金鑰輪替），並有 30 秒冷卻避免被洪水打爆。

---

## 7. 上線前安全檢查清單

- [ ] `COOKIE_SECURE=true`，全站 HTTPS
- [ ] `ALLOWED_ORIGINS` 是精確白名單（不可 `*`）
- [ ] Session 存放區改 **Redis 或資料庫**——範例用記憶體，重啟即失效、無法多實例
- [ ] 若用 `SameSite=None`，加上 CSRF token 或嚴格的 `Origin` 檢查（範例已檢查 Origin，但仍建議補 token）
- [ ] 移除 `/api/jwks-status` 之類的除錯端點
- [ ] 速率限制改用正式方案（範例是記憶體計數，多實例無效）
- [ ] 日誌記 `error.code` 與 `sub` 供稽核；**絕不可**記錄完整 token 或 session id
- [ ] 想清楚「授權」：登入只回答「你是誰」，「能做什麼」要另外設計
- [ ] Client Secret **不要**放進前端或 repo（本流程不需要它）

---

## 8. 限制與已知問題

1. **跨網域 cookie 是最大風險點。** 前端在 `chg983.github.io`、後端在別的網域時，必須 `SameSite=None; Secure`。Safari 的 ITP 與 Chrome 的第三方 cookie 政策可能導致登入狀態無法保持。**最穩定的做法是前後端同網域。**
2. **範例用記憶體存 session**：程序重啟後所有人被登出，也無法水平擴展。
3. **純前端模式（`google` / `demo`）不具備真正的存取控制**，只適合展示。
4. **登入只控制介面可見性**，不影響任何機率或期望值計算。
5. **靜態網站無法真正藏住內容**：GitHub Pages 上的 `index.html` 是公開檔案，任何人可直接下載原始碼。若內容本身是機密的，需要把資料放到後端 API、由後端驗 session 後才回傳。
6. **測試金鑰不可用於生產**：repo 不提供私鑰，需自行執行 `build_cases.js` 生成；即使外流也無法通過真實 Google 驗證。

---

## 9. 常見錯誤對照

| 症狀 | 原因 | 解法 |
|---|---|---|
| 登入按鈕沒出現，或 Google 彈窗報 `origin_mismatch` | `Authorized JavaScript origins` 沒填或含路徑 | 只填 `https://chg983.github.io` |
| `wrong_audience` | 前端 Client ID 與後端 `GOOGLE_CLIENT_ID` 不同 | 兩邊用同一個值 |
| `bad_signature`（但在 Google 登入成功後） | 前端把憑證送到錯的後端，或後端用了別人的公鑰 | 確認 `backendUrl` 正確、後端 JWKS 指回 Google |
| `/api/me` 一直 `401`，但登入當下成功 | 跨網域 cookie 被瀏覽器丟掉 | 設 `SameSite=None; Secure`，或改成同網域 |
| 瀏覽器 console 出現 CORS 錯誤 | `ALLOWED_ORIGINS` 沒含前端網址 | 加入 `https://chg983.github.io`（精確比對） |
| `email_unverified` | 該 Google 帳號的 email 未驗證 | 換帳號，或先到 Google 帳戶完成驗證 |
| 頁面卡在「載入中」 | Google 腳本被網路或地區封鎖 | 檢查是否能載入 `accounts.google.com/gsi/client` |
| 登入頁仍顯示「示範登入」 | `AUTH_CFG` 沒填或沒重新部署 | 檢查檢查表是否 `2 / 2` |

---

## 10. 一句話總結

> 把**同一個** Google OAuth Client ID 填進前端的 `AUTH_CFG.googleClientId` 與後端的 `GOOGLE_CLIENT_ID`，再把部署好的後端網址填進 `AUTH_CFG.backendUrl`，重新部署即可。三個值對齊、HTTPS、CORS 白名單精確，就完成了。
