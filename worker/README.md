# Calendar Worker v3.0

只保留後端能力：註冊 / 登入 / 查看 API、GitHub 存儲、ICS 分發、CalDAV。**不返回任何 HTML**。

## 目錄

```
worker/
  src/index.js        # 全部後端邏輯
  test/worker.test.mjs
  wrangler.toml
  package.json
```

## 部署

```bash
cd worker
npx wrangler login
npx wrangler secret put GITHUB_TOKEN       # 需要 repo contents: read/write
npx wrangler secret put ENCRYPTION_KEY     # 建議 openssl rand -hex 32
npx wrangler secret put SESSION_SECRET     # 建議 openssl rand -hex 32
npx wrangler deploy
```

`wrangler.toml` 的 `[vars]` 填 GitHub owner/repo、對外域名、允許的網頁來源。

### 環境變量

| 變量 | 必填 | 說明 |
| --- | --- | --- |
| `GITHUB_OWNER` / `GITHUB_REPO` | 是 | 存 `users.json` 與 `ics/` 的倉庫 |
| `GITHUB_TOKEN` | 是 | GitHub PAT（Contents read/write） |
| `GITHUB_BRANCH` | 否 | 默認 `main` |
| `GITHUB_PATH` | 否 | 默認 `ics` |
| `ENCRYPTION_KEY` | 否 | `users.json` 中用戶名/API key 的加密密鑰；務必固定 |
| `SESSION_SECRET` | 否 | 驗證碼簽名密鑰 |
| `WEB_ORIGIN` | 否 | GitHub Pages 來源，逗號分隔；默認 `*` |
| `PUBLIC_BASE_URL` | 否 | 對外 Worker 地址，默認請求 origin |
| `SCHOOLS` | 否 | 允許的學校代碼，默認 `gzhu,hlu` |
| `TURNSTILE_SECRET` | 否 | 配置後註冊使用 Turnstile，否則數學題驗證碼 |

## API 摘要

- `GET /api/captcha`、`POST /api/register`、`POST /api/login`
- `GET /api/me`、`GET /api/files`、`POST /api/upload`、`POST /api/files/delete`
- `POST /api/guest/upload`、`GET /api/guest/files`、`POST /api/guest/delete`
- `GET /{school}/{apiKey}.ics`、`GET /guest/{school}/{studentId}.ics`
- CalDAV：`/caldav/{school}/{apiKey}/`，支援 `PROPFIND / REPORT / GET / PUT / DELETE`
- `GET /{school}/{apiKey}.ics` 會合併主文件與 `ics/{school}/{apiKey}/` 下額外資源的 VEVENT。

完整約定見 [`../docs/ARCHITECTURE.md`](../docs/ARCHITECTURE.md)。

## 測試

```bash
node --test --test-isolation=none test/worker.test.mjs
```

測試用內存 GitHub mock，不會訪問網絡。`--test-isolation=none` 是為了在受限沙箱/CI 中避免 Node 測試 runner 再 spawn 子進程。

## 加密

- 註冊用戶：`CAL1.` + base64url(`IV || AES-256-GCM ciphertext || tag`)，密鑰 = `SHA-256(apiKey)`。
- 桌面程序加密後上傳；Worker 會解密驗證。
- CalDAV `PUT` 收到明文 ICS 時，Worker 會用同一方案加密後寫入。
- 訪客文件明文，路徑 `ics/guest/{school}/{studentId}.ics`。

> `ENCRYPTION_KEY` 只用於 `users.json` 內部的用戶名/API key 加密，不用於 ICS；ICS 密鑰由 API key 派生，因此丟失 API key 就無法解密該日曆。

## 從舊 Worker 遷移

- 保持舊部署的 `ENCRYPTION_KEY` / `SESSION_SECRET` 不變，否則 `users.json` 中舊的 `apiKeyEncrypted` 無法解出，使用者登入時會被分配新的 API key，舊 URL/密文將無法再用新 key 解開。
- 舊文件 `ics/{school}/{username}/{encryptedName}.ics` 只讀兼容：`GET` 若內容是明文 ICS 仍可訂閱；新程序不會再寫這種佈局。
- 舊的 `/admin` 網頁與 `/admin/upload` 已移除；桌面程序改用 `/api/upload`、`/api/files`、`/api/files/delete`。

