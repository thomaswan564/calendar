# v3.0 部署交接文檔

- 更新時間：2026-10-20
- 交接對象：接手部署與上傳的 Agent
- 目前狀態：**GitHub 倉庫與 Pages 已完成；Cloudflare Worker 尚未部署（`/health` 目前 404）；GitHub Release 尚未建立**
- 工作區：`D:\library\document\vscode\python\multi-tool\github`
- 上傳用資料夾：`D:\library\document\vscode\python\multi-tool\github\deploy`
- GitHub 倉庫：`https://github.com/thomaswan564/calendar`（Public，`main` 分支）
- GitHub Pages：`https://thomaswan564.github.io/calendar/`（Actions 已成功）
- 目前 `deploy` 的 remote：`https://github.com/thomaswan564/calendar.git`

> 這份文件不包含任何 Token、密碼或密鑰。接手 Agent 不應自行猜測或填入假的憑證；缺少的憑證必須由使用者提供或由使用者完成登入。

## 0. 接手後只需要做這幾件事

GitHub 倉庫與 Pages 已完成，**不要重做**。剩下：

1. 建立 GitHub PAT（給 Worker 寫入 `users.json` / `ics/`）
2. 修改 `deploy/worker/wrangler.toml` 的 owner/repo/`WEB_ORIGIN`
3. `wrangler login` → 設定 3 個 secret → `wrangler deploy`
4. 先測試 `*.workers.dev`，再把 `calendar.thomaswan.uk` 切到新 Worker
5. 確認 `/health` 為 `{"ok":true}`，Pages 可以註冊/登錄
6. 用桌面程序測訪客/註冊上傳、二維碼、CalDAV
7. 建立 `v3.0.0` Release
8. 把 `deploy/docs/HANDOVER.md` commit + push

若第 3 步缺少 Cloudflare 憑證，請先向使用者確認，不要嘗試猜測或繞過登入。

---

## 1. 專案是什麼

這是一個「課程表 → ICS → GitHub → 訂閱 / CalDAV」的工具，分成四部分：

| 組件 | 路徑 | 作用 |
| --- | --- | --- |
| 桌面程序 | `deploy/v3.0/code-multi-tool-v3.0.py` | 登入教務系統、下載 PDF、生成 ICS、加密上傳、檔案列表、webcal 二維碼、CalDAV 資訊、GitHub Releases 版本檢測 |
| Cloudflare Worker | `deploy/worker/src/index.js` | 純後端 JSON API、GitHub 讀寫、ICS 分發、完整 CalDAV |
| GitHub Pages 網頁 | `deploy/web/` | 只做註冊、登錄、查看 API |
| GitHub 倉庫 | `https://github.com/thomaswan564/calendar` | 保存程式、`users.json`、`ics/` 日曆、Releases |

### 1.1 GitHub 存儲佈局

```
users.json                                  # 帳戶（用戶名/API key 加密 + 哈希）
ics/
  guest/
    {school}/
      {studentId}.ics                       # 訪客：明文
  {school}/
    {apiKey}.ics                            # 註冊：AES-GCM 密文（主文件）
    {apiKey}/
      {filename}.ics                        # CalDAV 新增事件：密文
```

- 訪客：`ics/guest/{school}/{studentId}.ics`
- 註冊主文件：`ics/{school}/{apiKey}.ics`
- 訂閱 feed：`/{school}/{apiKey}.ics`，會合併主文件與 CalDAV 額外事件
- CalDAV 集合：`/caldav/{school}/{apiKey}/`

### 1.2 加密約定

- AES-256-GCM
- 密鑰：`SHA-256(apiKey)` 的 32 字节
- 文件格式：`CAL1.` + base64url(`IV || ciphertext || tag`)
- 桌面程序加密後上傳；Worker 會解密驗證
- CalDAV `PUT` 收到明文時，Worker 會用同一方案加密後寫入
- **API key 同時是 URL 路徑、CalDAV 用戶名/密碼、解密金鑰**；遺失 API key 就無法解密該日曆

### 1.3 版本檢測

- 桌面程序內 `APP_VERSION = "3.0.0"`
- 啟動時或按「檢查更新」：
  `GET https://api.github.com/repos/{owner}/{repo}/releases/latest`
- 比較 `tag_name` 與 `APP_VERSION`
- 遠端較新 → 彈窗，開啟 Release 頁或 `.exe` 資產
- 需要 `cloud_config.json` 的 `github_owner` / `github_repo`，或可從 `web_portal_url` 推斷
- 公開倉庫即可，不需要 Token

---

## 2. 目前狀態

| 項目 | 狀態 |
| --- | --- |
| v3.0 桌面程序 | 已完成，UI 建構測試通過 |
| Worker | 已完成，12 項測試通過 |
| 網頁 | 已完成，JS 語法檢查通過 |
| 版本檢測 | 已完成，18 項 Python 測試通過；GitHub API 實際連線測試通過 |
| `deploy/` 工作檔案 | 66 個（不含 `.git`），約 0.43 MB；整個 repo 含 `.git` 約 0.6 MB |
| GitHub 倉庫 | **已完成**：`https://github.com/thomaswan564/calendar`（Public，`main`） |
| GitHub Pages | **已上線**：`https://thomaswan564.github.io/calendar/`，`Deploy calendar web` 已成功 |
| Cloudflare Worker | **未部署**；`https://calendar.thomaswan.uk/health` 目前回 404（仍是舊 Worker） |
| GitHub Release | **未建立**（`/releases` 為空） |
| `deploy` Git 狀態 | 已 push 到 `origin/main`，最新 commit `22d8278`；`docs/HANDOVER.md` 目前是 untracked，需 commit/push |

### 2.1 已驗證的測試指令

```powershell
cd D:\library\document\vscode\python\multi-tool\github\deploy\worker
node --test --test-isolation=none test/worker.test.mjs
# 預期：tests 12 / pass 12 / fail 0
```

```powershell
cd D:\library\document\vscode\python\multi-tool\github\deploy\v3.0
python -m unittest discover -s tests
# 預期：Ran 18 tests / OK
```

```powershell
python -m py_compile v3.0\code-multi-tool-v3.0.py v3.0\cloud_client.py
```

> Windows 受限沙箱下 `node --test` 可能因 spawn EPERM 失敗，必須加 `--test-isolation=none`。Python unittest 會把進度寫到 stderr，PowerShell 可能顯示 `[exit code: 1]`，只要看到 `OK` 即可。

---

## 3. 開始前必須確認的資訊

接手 Agent 先不要直接部署；以下資訊缺一不可，且**不要自行編造**。

### 3.0 已經確認（截至 2026-10-20）

- GitHub 用戶名：`thomaswan564`
- GitHub 倉庫：`calendar`（Public，`main`）
- 倉庫用途：同一個倉庫同時放程式、`users.json`、`ics/`、Pages、Releases
- Pages 已上線：`https://thomaswan564.github.io/calendar/`
- Worker 域名：`https://calendar.thomaswan.uk`，目前 `/health` 回 404（仍是舊 Worker）
- Release：尚未建立
- `deploy` 已 push 到 `origin/main`，`docs/HANDOVER.md` 尚未 commit

### 3.1 仍需要向使用者確認

1. **Cloudflare 帳號與 Worker 部署方式**
   - 是使用者本人執行 `wrangler login`，還是已有 `CLOUDFLARE_API_TOKEN`？
   - 新 Worker 要用哪個名稱？是否直接覆蓋舊 Worker？
   - `calendar.thomaswan.uk` 是 custom domain 還是 route？是否允許先部署到 `*.workers.dev` 測試？
2. **舊 Worker 的 `ENCRYPTION_KEY` / `SESSION_SECRET` 是否保留**
   - 若替換同一個 Worker，必須沿用舊值，否則舊用戶 `apiKeyEncrypted` 解不開
   - 若無法取得舊值，需告知使用者：舊用戶登入時會拿到新 API key，舊 URL/密文可能無法再用新 key 解密
3. 是否已有要發佈的 **exe**，以及 Release tag（建議 `v3.0.0`）
4. 是否需要 Cloudflare Turnstile（可選；不配置就用數學題驗證碼）
5. 是否授權切換 `calendar.thomaswan.uk` 的 custom domain（這是會影響線上舊 Worker 的操作，需明確同意）

### 3.2 憑證從哪裡來

- `GITHUB_TOKEN`：由使用者建立 GitHub PAT
  - Classic：Settings → Developer settings → Personal access tokens → Tokens (classic) → `repo`
  - Fine-grained：選定 `thomaswan564/calendar` → `Contents: Read and write`、`Metadata: Read`
  - **不要**把 Token 寫進倉庫、`wrangler.toml`、文件或聊天記錄
- Cloudflare 登入：
  - `npx wrangler login` 會開瀏覽器，通常需要使用者本人完成 OAuth
  - 若使用 `CLOUDFLARE_API_TOKEN`，只放在執行環境，不要寫入檔案
- `ENCRYPTION_KEY` / `SESSION_SECRET`：
  - 用 `python -c "import secrets; print(secrets.token_hex(32))"` 生成
  - 遷移舊 Worker 時沿用舊值

---

## 4. 檔案地圖

### 4.1 要上傳的 `deploy/`

`deploy/` 是整理好的上傳快照，**它的內容要成為 GitHub 倉庫根目錄**：

```
deploy/
  README.md
  .gitignore
  .github/workflows/pages.yml
  docs/ARCHITECTURE.md
  v3.0/
    code-multi-tool-v3.0.py
    cloud_client.py
    requirements.txt
    cloud_config.example.json
    _vendor/qrcode/...
    tests/test_cloud_client.py
  worker/
    src/index.js
    wrangler.toml
    package.json
    test/worker.test.mjs
    README.md
  web/
    index.html
    register.html
    account.html
    config.js
    app.js
    style.css
    README.md
```

### 4.2 關鍵檔案

| 檔案 | 用途 |
| --- | --- |
| `deploy/worker/src/index.js` | Worker 全部後端邏輯 |
| `deploy/worker/wrangler.toml` | Worker 名稱、GitHub 倉庫、域名、CORS |
| `deploy/web/config.js` | 網頁呼叫的 Worker 地址 |
| `deploy/v3.0/cloud_config.example.json` | 桌面程序配置模板 |
| `deploy/v3.0/code-multi-tool-v3.0.py` | 桌面程序 |
| `deploy/v3.0/cloud_client.py` | 加密 / API / 版本檢測 |
| `deploy/.github/workflows/pages.yml` | GitHub Pages 部署 workflow |
| `deploy/docs/ARCHITECTURE.md` | 架構與接口細節 |

### 4.3 注意：`deploy/` 是快照

根目錄的 `v3.0/`、`worker/`、`web/`、`docs/` 是原始版本；`deploy/` 是複製出來的部署快照。

- 建議：部署與修正直接以 `deploy/` 為準，完成後再同步回根目錄
- 或者：改根目錄後重新整理 `deploy/`，避免兩邊不一致
- **不要把 `deploy/` 整個資料夾上傳成倉庫裡的一層**，否則 Pages 會找不到 `web/`

---

## 5. 部署步驟

### 5.1 建立 GitHub 倉庫並上傳（已完成）

目前倉庫已存在：`https://github.com/thomaswan564/calendar`，Public，`main` 分支，`deploy` 已 push。

- 最新 commit：`22d8278`（`v3.0 initial`）
- 倉庫根目錄已包含 `v3.0/`、`worker/`、`web/`、`docs/`、`.github/`
- 唯一待辦：`deploy/docs/HANDOVER.md` 目前是 untracked，請 commit + push：

```powershell
cd D:\library\document\vscode\python\multi-tool\github\deploy
git add docs/HANDOVER.md
git commit -m "Add deployment handover"
git push
```

以下為原始流程，僅供必要時重建倉庫參考：

1. 安裝 GitHub Desktop，登入 GitHub。
2. `File` → `Add local repository...` → 選 `...\github\deploy`
3. 若提示不是 Git 倉庫，點 `create a repository`
   - Name：`calendar`
   - Local path：`...\github\deploy`
   - 不要勾 Initialize README（已有）
4. Summary 填 `v3.0 initial` → `Commit to main`
5. `Publish repository`（Public）
6. 確認倉庫根目錄直接是 `v3.0/`、`worker/`、`web/`、`docs/`、`.github/`，不是 `deploy/...`

若用網頁上傳：

1. 建立 Public 倉庫，Add file → Upload files
2. 把 `deploy` 裡的**內容**（不是 `deploy` 資料夾本身）拖上去
3. `.github` 是隱藏資料夾，網頁可能漏掉；需 Add file → Create new file，建立 `.github/workflows/pages.yml` 並貼上內容
4. Commit

### 5.2 建立 GitHub Token

1. GitHub 頭像 → Settings → Developer settings
2. Personal access tokens → Tokens (classic) → Generate new token (classic)
3. Note：`calendar-worker`
4. 勾選 `repo`
5. Generate → 複製 `ghp_...`

Token 只會在下一步貼到 Cloudflare secret。

### 5.3 配置 Worker

編輯 `deploy/worker/wrangler.toml`：

```toml
name = "calendar-worker-v3"
main = "src/index.js"
compatibility_date = "2026-01-01"
workers_dev = true

[vars]
GITHUB_OWNER = "thomaswan564"
GITHUB_REPO = "calendar"
GITHUB_BRANCH = "main"
GITHUB_PATH = "ics"
PUBLIC_BASE_URL = "https://calendar.thomaswan.uk"
WEB_ORIGIN = "https://thomaswan564.github.io"
SCHOOLS = "gzhu,hlu"
```

> 目前 `deploy/worker/wrangler.toml` 仍是 `YOUR_GITHUB_USER` / `YOUR_GITHUB_REPO` / `YOURNAME.github.io` 佔位符，部署前必須替換成上面的實際值。

說明：

- `PUBLIC_BASE_URL`：正式的 Worker 對外地址。若先部署到 `*.workers.dev`，先填該地址，切換域名後再改
- `WEB_ORIGIN`：Pages 的 **origin**，例如 `https://alice.github.io`
  - 不要填 `https://alice.github.io/calendar/`
  - 多個來源用逗號分隔
  - 未確定前可暫用 `*`，Pages 上線後一定要收緊
- `GITHUB_PATH` 默認 `ics`，不用改
- `name` 可改成現有 Worker 名稱，讓 `wrangler deploy` 直接覆蓋；也可用新名稱，之後再把 custom domain 切過來

### 5.4 設置 Secret 並部署

```powershell
cd D:\library\document\vscode\python\multi-tool\github\deploy\worker
npx wrangler login

npx wrangler secret put GITHUB_TOKEN
# 貼上 ghp_...

npx wrangler secret put ENCRYPTION_KEY
# 貼上 64 位 hex 隨機值；若替換舊 Worker，必須沿用舊值

npx wrangler secret put SESSION_SECRET
# 貼上另一組 64 位 hex；若替換舊 Worker，沿用舊值

npx wrangler deploy
```

生成隨機值：

```powershell
python -c "import secrets; print(secrets.token_hex(32))"
```

### 5.5 綁定 `calendar.thomaswan.uk`

Cloudflare Dashboard：

1. Workers & Pages → 選新 Worker
2. Settings → Domains & Routes → Add → Custom Domain
3. 填 `calendar.thomaswan.uk`
4. 若舊 Worker 佔用此域名：
   - 先從舊 Worker 移除
   - 再加到新 Worker
5. 等狀態變成 Active

也可以使用 `wrangler.toml` 的 route/custom domain，但 Dashboard 對新手較直觀。**切換域名前先確認新 Worker 在 `workers.dev` 可用。**

### 5.6 驗證 Worker

```powershell
curl.exe https://calendar.thomaswan.uk/health
```

預期：

```json
{"ok":true}
```

若 500「Worker 配置不完整」→ 檢查 `GITHUB_OWNER/REPO/TOKEN`
若 404 → 域名仍指向舊 Worker，或 custom domain 未生效

### 5.7 啟用 GitHub Pages（已完成）

目前已上線：`https://thomaswan564.github.io/calendar/`

- `Deploy calendar web` workflow 已成功
- `deploy/web/config.js` 已指向 `https://calendar.thomaswan.uk`
- 剩下唯一動作：Worker 部署後，把 `deploy/worker/wrangler.toml` 的 `WEB_ORIGIN` 設為 `https://thomaswan564.github.io`，再 `npx wrangler deploy`

以下為原始流程，供重建時參考：

1. 編輯 `deploy/web/config.js`：

```js
window.CALENDAR_CONFIG = {
  apiBase: "https://calendar.thomaswan.uk",
  siteName: "课程表日历"
};
```

2. Commit + Push
3. 倉庫 Settings → Pages → Source 選 **GitHub Actions**
4. Actions → `Deploy calendar web`：
   - 若沒有自動執行，按 **Run workflow**
5. 等 workflow 綠色勾勾
6. Pages 網址：

```
https://thomaswan564.github.io/calendar/
```

7. Worker 部署後，把 `deploy/worker/wrangler.toml` 的 `WEB_ORIGIN` 改成：

```toml
WEB_ORIGIN = "https://thomaswan564.github.io"
```

8. 重新部署：

```powershell
npx wrangler deploy
```

### 5.8 配置桌面程序

```powershell
cd D:\library\document\vscode\python\multi-tool\github\deploy\v3.0
pip install -r requirements.txt
python -m playwright install

Copy-Item cloud_config.example.json cloud_config.json
```

編輯 `cloud_config.json`：

```json
{
  "cloud_base_url": "https://calendar.thomaswan.uk",
  "web_portal_url": "https://thomaswan564.github.io/calendar/",
  "github_owner": "thomaswan564",
  "github_repo": "calendar",
  "update_check": true
}
```

- `github_owner` / `github_repo` 是 **放 Release 的程式碼庫**
- 若與資料庫同一個，填同一個
- 留空時程式會嘗試從 `web_portal_url` 推斷

啟動：

```powershell
python code-multi-tool-v3.0.py
```

### 5.9 建立 Release（版本檢測）

1. 確認 `deploy/v3.0/code-multi-tool-v3.0.py` 的 `APP_VERSION` 是 `3.0.0`
2. 倉庫 → Releases → Draft a new release
3. Tag：`v3.0.0` → Create new tag on publish
4. Title：`v3.0.0`
5. 若有 exe，附加到 Assets
6. Publish release

之後每次更新：

- 修改 `APP_VERSION`（例如 `3.0.1`）
- 建立 tag `v3.0.1` 的 Release
- 程式啟動時會提示更新

打包 exe（可選）：

```powershell
cd D:\library\document\vscode\python\multi-tool\github\deploy\v3.0
pip install pyinstaller
pyinstaller --noconfirm --windowed --name Calendar-v3.0.0 --add-data "_vendor;_vendor" code-multi-tool-v3.0.py
```

Windows 的 `--add-data` 分隔符是 `;`；macOS/Linux 是 `:`。

---

## 6. 驗證清單（Definition of Done）

### 6.1 GitHub / Worker

- [x] 倉庫根目錄有 `v3.0/`、`worker/`、`web/`、`docs/`、`.github/`
- [ ] `https://calendar.thomaswan.uk/health` 返回 `{"ok":true}`（目前 404）
- [ ] 註冊後 GitHub 倉庫出現 `users.json`
- [ ] 上傳後出現 `ics/guest/...` 或 `ics/{school}/...`
- [ ] 舊 Worker 的 `ENCRYPTION_KEY` / `SESSION_SECRET` 已按情況保留

### 6.2 Pages

- [x] `https://thomaswan564.github.io/calendar/` 可打開
- [ ] 可註冊（數學題或 Turnstile，需 Worker 先上線）
- [ ] 可登錄
- [ ] 可看到/複製 API key、webcal、CalDAV
- [ ] 瀏覽器 Console 沒有 CORS 錯誤

### 6.3 桌面程序

- [ ] 訪客模式：學校 + 學號上傳成功
- [ ] 訪客文件列表可刷新/刪除
- [ ] 註冊模式：貼 API key → 驗證成功
- [ ] 加密上傳成功
- [ ] 訂閱地址可讀：

```powershell
curl.exe -i https://calendar.thomaswan.uk/gzhu/<APIKEY>.ics
```

- [ ] 二維碼顯示 `webcal://...`
- [ ] CalDAV：

```powershell
curl.exe -i -X PROPFIND -u "<APIKEY>:<APIKEY>" -H "Depth: 1" https://calendar.thomaswan.uk/caldav/gzhu/<APIKEY>/
```

  預期 `207`，XML 含 `<cal:calendar` 與 `<d:getetag>`

### 6.4 版本檢測

- [ ] `cloud_config.json` 已填 `github_owner` / `github_repo`
- [ ] 已建立 `v3.0.0` Release
- [ ] 程式日誌顯示「當前已是最新版本」或正確提示
- [ ] 建立 `v3.0.1` 測試 Release 後，舊程式能提示下載（測試後可刪除該 Release）

---

## 7. 舊 Worker 遷移注意

若 `calendar.thomaswan.uk` 目前是舊 Worker：

1. **先備份**舊 Worker 的 `users.json` 與 `ics/`（GitHub 倉庫本身已有）
2. 保留舊 `ENCRYPTION_KEY` / `SESSION_SECRET`
   - 若無法取得，告知使用者：舊用戶登入時會拿到新 API key，舊文件仍存在但新 key 無法解密舊密文
3. 新 Worker 部署到 `workers.dev` 先測試
4. 再把 custom domain 從舊 Worker 切到新 Worker
5. 舊文件 `ics/{school}/{username}/{encryptedName}.ics` 仍可透過兼容路徑只讀訂閱
6. 新程式上傳後會生成 `ics/{school}/{apiKey}.ics`；舊用戶需要重新上傳一次
7. **不要刪除**舊 `users.json`、舊 `ics/`，直到確認新舊都正常

---

## 8. 安全與限制

- **不要**把 `GITHUB_TOKEN`、`ENCRYPTION_KEY`、`SESSION_SECRET` 寫入倉庫或文件
- **不要**把 `cloud_config.json` 提交到 GitHub（`.gitignore` 已忽略）
- API key 是日曆解密密鑰；遺失無法恢復
- 訪客文件是公開明文；敏感課表請用註冊模式
- Public 倉庫的 `users.json` 雖然用戶名/API key 已加密，但仍不建議放真實敏感資料
- `WEB_ORIGIN` 正式環境不要長期用 `*`
- Worker 的 GitHub Token 只需 `Contents: Read and write`
- 版本檢測使用未認證 GitHub API，公開倉庫每小時有速率限制；啟動一次檢查足夠

---

## 9. 常見問題

| 症狀 | 原因 / 處理 |
| --- | --- |
| `/health` 404 | 域名還在舊 Worker；檢查 Custom Domain 綁定 |
| `/health` 500「配置不完整」 | `GITHUB_OWNER` / `GITHUB_REPO` / `GITHUB_TOKEN` 缺失 |
| Worker 回 `GitHub 写入失败：401` | Token 過期、權限不足、或填錯倉庫 |
| 註冊後沒有 `users.json` | Token 沒有 Contents write；或倉庫分支不是 `main` |
| Pages 404 | 上傳時多了 `deploy/` 一層；workflow 沒跑；Pages 未設 GitHub Actions |
| 網頁登錄 `Failed to fetch` / CORS | `WEB_ORIGIN` 與 Pages origin 不一致；改完要重新 `wrangler deploy` |
| 程式顯示「未配置 GitHub 倉庫」 | `cloud_config.json` 的 `github_owner` / `github_repo` 為空，或 `web_portal_url` 不是 `*.github.io/<repo>/` |
| 程式顯示「沒有 Release」 | 只建了 tag，沒有 Publish Release；或倉庫是 Private |
| CalDAV 401 | 用戶名/密碼都要填 API key；URL 要帶 `{school}/{apiKey}` |
| CalDAV 404 | 還沒有上傳主文件；先用程式上傳一次 ICS |
| 訂閱 404 | 訪客文件尚未上傳；或註冊文件路徑不是 `/{school}/{apiKey}.ics` |
| Python 測試顯示 exit code 1 | unittest 進度寫在 stderr；看到 `OK` 即通過 |

---

## 10. 回滾

- Worker：
  - Cloudflare Dashboard → Worker → Deployments → 選上一個版本回滾
  - 或 `npx wrangler deployments list` / `npx wrangler rollback <id>`
- Pages：
  - revert `web/config.js` 或 workflow 變更，重新 push
  - 或暫時關閉 Pages
- 資料：
  - 不要刪除 `users.json`、`ics/`
  - 舊文件仍可只讀訂閱
- 密鑰：
  - 若曾更換 `ENCRYPTION_KEY`，恢復舊值可讓舊 `apiKeyEncrypted` 重新可解；若已生成新 key，需逐一處理

---

## 11. 建議的 Agent 執行順序

1. 讀 `deploy/README.md`、`deploy/docs/ARCHITECTURE.md`、本文件
2. 向使用者確認第 3.1 節的 Cloudflare / 舊密鑰 / exe / Turnstile 資訊
3. 把 `deploy/docs/HANDOVER.md` commit + push（GitHub 倉庫與 Pages 已完成）
4. 建立 GitHub PAT，編輯 `deploy/worker/wrangler.toml` 的實際 owner/repo/WEB_ORIGIN
5. `wrangler login` → `secret put` × 3 → 先部署到 `workers.dev` 測試
6. 綁定 `calendar.thomaswan.uk`，`/health` 驗證
7. 確認 `WEB_ORIGIN` 是 `https://thomaswan564.github.io`，重新 `wrangler deploy`
8. 建立 `cloud_config.json`，跑桌面程序，測訪客與註冊上傳
9. 測 CalDAV 與二維碼
10. 建立 `v3.0.0` Release，測版本檢測
11. 更新本文件與 README，填入實際 URL（不要填密鑰）
12. 回報完成清單與未解問題

---

## 12. 需要回報給使用者的結果

完成後回報：

- GitHub 倉庫 URL
- Pages URL
- Worker URL / custom domain
- `/health` 結果
- 測試帳號是否建立（不要回報密碼）
- 訪客/註冊上傳測試結果
- CalDAV `PROPFIND` 狀態碼
- 版本檢測結果
- 任何未完成項目或風險

---

## 附錄 A：值填寫模板

```text
GITHUB_OWNER = thomaswan564
GITHUB_REPO = calendar
GITHUB_BRANCH = main
GITHUB_PATH = ics

PUBLIC_BASE_URL = https://calendar.thomaswan.uk
WEB_ORIGIN = https://thomaswan564.github.io
PAGES_URL = https://thomaswan564.github.io/calendar/

CLOUDFLARE_WORKER_NAME = ____________________
CLOUDFLARE_CUSTOM_DOMAIN = calendar.thomaswan.uk
CLOUDFLARE_DEPLOY_METHOD = wrangler / dashboard

RELEASE_TAG = v3.0.0
APP_VERSION = 3.0.0
```

> `GITHUB_TOKEN`、`ENCRYPTION_KEY`、`SESSION_SECRET` 不要寫在這份文件；只存在 Cloudflare secret 或使用者的密碼管理器。

## 附錄 B：關鍵程式位置

- Worker 入口：`deploy/worker/src/index.js`
- Worker 測試：`deploy/worker/test/worker.test.mjs`
- 網頁：`deploy/web/index.html` / `register.html` / `account.html`
- Pages workflow：`deploy/.github/workflows/pages.yml`
- 桌面程序：`deploy/v3.0/code-multi-tool-v3.0.py`
- 加密/API/版本檢測：`deploy/v3.0/cloud_client.py`
- 桌面程序測試：`deploy/v3.0/tests/test_cloud_client.py`
- 架構文件：`deploy/docs/ARCHITECTURE.md`

