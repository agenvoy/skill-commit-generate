# commit-generate - 技術文件

> 返回 [README](./README.zh.md)

## 前置需求

- 可載入 `SKILL.md` skill 並執行 shell 指令的 agent harness
- Git（需支援 `git diff --cached`、`git status --short`、`git branch --show-current`）
- 目標專案必須是 Git 儲存庫

## 安裝

`<skills-dir>` 為所用 harness 掃描的 skill 目錄。

### 從 GitHub 複製

```bash
git clone https://github.com/agenvoy/skill-commit-generate.git \
    <skills-dir>/commit-generate
```

### 確認安裝

```bash
ls <skills-dir>/commit-generate/SKILL.md
```

安裝完成後，於 harness 中以 `/commit-generate` 呼叫即可。

## 使用方式

### 基本用法

```bash
# 1. 先 stage 要提交的檔案
git add <file1> <file2>

# 2. 於 harness 中呼叫 skill
/commit-generate
```

輸出範例：

```
feat: Add Docker environment auto-detection and database path switching
feat: 新增 Docker 環境自動偵測與資料庫路徑切換機制
```

skill 只輸出 message，不會執行 `git commit`。

### 漏 Stage 提醒

與 staged 檔案同模組、卻仍停在 ` M` 的檔案會在 message 前列出：

```
⚠️ 以下同模組檔案未 stage：internal/auth/token.go

fix: Fix token expiration not handled correctly during user login
fix: 修正使用者登入時 token 過期未正確處理的問題
```

message 仍只描述 staged 內容。

### 跨主題變更

staged diff 橫跨無關模組或觸及 2 個以上主要 Tag 時：

```
⚠️ 偵測到跨主題變更，建議拆分為多次 commit：
- 認證模組重構：internal/auth/*.go
- UI 樣式調整：web/styles/*.css

若仍要合併，以下為概括描述：
refactor: Split auth module and adjust UI styles
refactor: 重構認證模組並調整 UI 樣式
```

### 無 Staged 變更

```
目前沒有 staged 變更，請先執行 `git add` 選擇要提交的檔案。
```

## 設定參考

### 讀取的資料

| 指令 | 用途 | 影響範圍 |
|------|------|----------|
| `git diff --cached` | 本次要描述的變更 | 唯一內容來源 |
| `git status --short` | 找出同模組但未 stage 的檔案 | 僅產生提醒 |
| `git log --oneline -10` | 既有 tag 詞彙與描述顆粒度 | 僅影響用詞 |
| `git branch --show-current` | 分支名中的意圖 | 僅影響用詞 |

四者彼此無依賴，同一輪並行讀取。`git log` 與分支名不影響 Tag 選擇；與本規範衝突時以規範為準。

### Classification Tags

| Tag | 適用情境 |
|-----|----------|
| `feat` | 新功能、新 endpoint、新元件 |
| `fix` | Bug 修正、錯誤處理、nil check |
| `update` | 修改既有行為、參數調整 |
| `add` | 新增檔案／資源（非功能） |
| `remove` | 刪除程式碼或檔案 |
| `refactor` | 重構（行為不變） |
| `perf` | 效能優化 |
| `style` | 格式化、排版 |
| `doc` | 文件、註解 |
| `test` | 測試相關 |
| `chore` | CI/CD、工具、依賴管理 |
| `security` | 安全性修補 |
| `breaking` | 破壞性變更 |

### Tag 優先序

```
BREAKING > FEAT > FIX > SECURITY > UPDATE > REFACTOR > PERF > others
```

### Breaking 訊號（命中 → `breaking`）

| 類別 | 訊號 |
|------|------|
| 符號刪除 | 刪除任何 exported / public 的 function、class、method、type、constant、enum value |
| 簽章變更 | 參數型別、順序、新增必填參數、回傳型別改變 |
| HTTP API | endpoint 刪除、URL 路徑改、response schema 欄位移除或型別變、status code 變更 |
| 設定 | 刪除既有鍵、新增必填鍵、既有鍵型別／格式變更 |
| Migration | `DROP COLUMN`、有資料的表新增必填欄位、型別縮窄 |
| CLI | 旗標刪除、新增必填位置參數、既有旗標語意變更 |
| Import | package／module 結構變更導致既有引用失效 |

### Security 訊號（命中 → `security`，除非同時命中 Breaking）

| 類別 | 訊號 |
|------|------|
| 注入 | SQL / XSS / Command / LDAP / Template injection |
| 驗證授權 | missing auth check、privilege escalation、JWT 驗證缺失 |
| 敏感資料 | 從 log／response／error message 移除密鑰、token、PII |
| 硬編碼 | 移除硬編碼密鑰、token、預設密碼 |
| Web 安全 | CSRF / CORS / CSP / HSTS 修補 |
| CVE | 相依套件 CVE 修補（由 `chore: upgrade` 升級為 `security`） |

### 跨主題判準

任一命中即視為跨主題：

- diff 同時觸及 2 個以上主要 Tag 類別（`feat` / `fix` / `refactor` / `breaking` / `security`）
- 變更檔案橫跨 2 個以上語意無關的模組（以一級目錄或套件邊界判斷）
- 單次 commit 包含 3 個以上彼此無關的主題

格式化、純註解補充、依賴升級可併入主 commit，不視為跨主題。

### 輸出格式規則

| 規則 | 說明 |
|------|------|
| 單一 Tag | 選擇最能代表核心意圖的 Tag |
| 英文 subject | imperative mood，不超過 72 字元 |
| 繁體中文 body | 不超過 50 字，動詞開頭（新增、修正、重構、移除、優化） |
| 合併相關變更 | 多個小改動歸納為單一描述 |
| 單一主題優先 | 跨主題時先警示拆分，再給概括 message |
