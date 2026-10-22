> **已停止維護（只讀）**：本倉庫不再接受功能更新。課表程式、ICS/CalDAV 網站、Cloudflare Worker 原始碼與自行部署指引已整合至 [thomaswan564/multi-tool](https://github.com/thomaswan564/multi-tool)。請到新倉庫查看最新功能與文件。此倉庫僅保留歷史代碼；現有雲端日曆服務端點目前仍由 Cloudflare Worker 提供。

# 课程表日历 v3.0

v3.0 重構：桌面程序承擔主要功能，網頁極簡化，Worker 只做後端，GitHub 保存日曆。

## 三個組件

| 目錄 | 內容 |
| --- | --- |
| [`v3.0/`](v3.0/README.md) | Python 桌面程序：下載課表、生成 ICS、加密上傳、雲端文件列表、webcal 二維碼、CalDAV 資訊、GitHub Releases 版本檢測 |
| [`worker/`](worker/README.md) | Cloudflare Worker：註冊/登錄/查看 API、GitHub 讀寫、ICS 分發、完整 CalDAV |
| [`web/`](web/README.md) | GitHub Pages 靜態頁：僅註冊、登錄、查看 API |

架構與接口約定見 [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)。

## 存儲佈局

```
users.json
ics/
  guest/{school}/{studentId}.ics      # 訪客：明文
  {school}/{apiKey}.ics               # 註冊：AES-256-GCM 密文（主文件）
  {school}/{apiKey}/*.ics             # CalDAV 新增事件：密文
```

- 訪客：學校 + 學號，無 API key。
- 註冊：API key 同時作為路徑、CalDAV 用戶名/密碼和加密密鑰派生來源。
- 加密格式：`CAL1.` + base64url(`IV || ciphertext || tag`)，密鑰 = `SHA-256(apiKey)`。

## 快速開始

1. **部署 Worker**：見 [`worker/README.md`](worker/README.md)。設置 `GITHUB_OWNER/REPO/TOKEN`、`ENCRYPTION_KEY`、`SESSION_SECRET`、`WEB_ORIGIN`。
2. **發佈網頁**：編輯 [`web/config.js`](web/config.js) 的 `apiBase`，按 [`web/README.md`](web/README.md) 用 GitHub Actions 發佈。
3. **配置程序**：把 [`v3.0/cloud_config.example.json`](v3.0/cloud_config.example.json) 複製為 `v3.0/cloud_config.json`，填入 Worker 與 Pages 地址。
4. **運行程序**：

```bash
cd v3.0
pip install -r requirements.txt
python -m playwright install
python code-multi-tool-v3.0.py
```

## 測試

```bash
# Worker（內存 GitHub mock）
cd worker
node --test --test-isolation=none test/worker.test.mjs

# Python 客戶端
cd v3.0
python -m unittest discover -s tests -v
```

## 遷移與兼容

- 舊 `users.json` 中的明文 `username/password/apiKey` 會自動遷移為哈希 + 密文。
- 舊 `ics/{school}/{username}/...ics` 只讀兼容：`GET` 到明文 ICS 仍可訂閱；新程序不再寫入該佈局。
- 舊 Worker 的 `/admin` 網頁已移除；請使用桌面程序管理文件。

## 安全提示

- API key 是日曆的解密密鑰，丟失後無法恢復舊文件；請讓用戶保存在密碼管理器。
- 訪客文件是公開明文，URL 可被猜到；敏感課表請註冊後上傳。
- 不要把 `GITHUB_TOKEN`、`ENCRYPTION_KEY` 寫進前端或倉庫。
