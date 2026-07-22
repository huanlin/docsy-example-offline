# Docsy Example 分支版本 — 離線相依套件

本儲存庫是 [Docsy Example](https://github.com/google/docsy-example) 的分支版本，適用於受限網路環境。儲存庫已將各平台專用的 npm 相依套件及 Hugo Extended 收錄於 `node_modules/`，並將 Docsy Hugo Module 收錄於 `_vendor/`，因此一般建置在複製儲存庫後不需要下載建置相依套件。

## 選擇正確的分支

內附的原生執行檔因作業系統而異。請複製符合目標作業系統的分支：

| 作業系統 | 分支      |
| -------- | --------- |
| Linux    | `main`    |
| Windows  | `windows` |

Linux：

```bash
git clone --branch main --single-branch https://github.com/huanlin/docsy-example-offline.git
cd docsy-example-offline
```

Windows（PowerShell）：

```powershell
git clone --branch windows --single-branch https://github.com/huanlin/docsy-example-offline.git
Set-Location docsy-example-offline
```

建議使用 `git clone` 取得本專案。GitHub 的原始碼壓縮檔雖然包含已提交的執行檔，但解壓縮時不一定能保留 Linux 的可執行權限及 `node_modules/` 內的符號連結。如果這些檔案中繼資料遺失，npm 命令或內附的 Hugo 執行檔可能無法正常運作。

## 環境需求

- Node.js 22 或更新版本（依 `package.json` 的設定）。
- 不需要另外安裝 Hugo；`node_modules/` 已包含 Hugo Extended。
- 建議使用 Git 複製儲存庫，以保留檔案中繼資料。
- 一般建置不需要 Go 或預先準備 Go Module 快取，因為 `_vendor/` 已包含 Docsy 模組。

## 執行網站

請**不要**執行 `npm install`。

一般使用情況不需要執行此命令，而且它可能會連線至公用套件庫，或取代儲存庫內已備妥的平台專用檔案。

確認內附的 Hugo 執行檔可正常運作：

```bash
npx --offline hugo version
```

啟動開發伺服器：

```bash
npm run serve
```

接著開啟 <http://localhost:1313/>。若要停止伺服器，請按 `Ctrl+C`。

若要在 `public/` 目錄產生正式環境的建置結果，請執行：

```bash
npm run build:production
```

## 離線支援範圍

本分支版本已包含兩類建置相依套件：

- `node_modules/` 包含 npm 套件及平台專用的 Hugo Extended 執行檔。
- `_vendor/` 包含宣告為 `github.com/google/docsy/theme` 的 Docsy Hugo Module。

Hugo 會自動比對 `_vendor/modules.txt` 與 `hugo.yaml` 中的 module import，並在嘗試 Go Module 解析之前載入本地副本。選用的網站功能或自訂內容如果明確要求取得遠端資源，仍需另外將這些資源收錄至儲存庫。

如需了解解析流程、離線驗證命令與維護注意事項，請參閱[〈Hugo 如何使用已收錄的模組〉](docs/hugo-offline-modules.zh-TW.md)。

## 技術文件

其他實作原理、疑難排解與維護資訊收錄於 `docs/` 目錄：

- [Hugo 如何使用已收錄的模組](docs/hugo-offline-modules.zh-TW.md) — 說明模組解析、`_vendor/` 與 `node_modules/` 的關係、嚴格離線驗證、Windows vendoring workaround 及 alias 處理。

## 如何製作這個離線版本

本節說明維護者建立或更新離線儲存庫時所使用的完整流程。一般使用者不需要執行這些步驟。

### 1. 設定 npm 相依套件

- 從 `.gitignore` 移除 `node_modules/`，以便提交已安裝的相依套件。
- 在 `package.json` 中固定所需的 Hugo Extended 版本；本分支目前使用 `0.164.0`。
- 將 `adm-zip` 覆寫為 `0.6.0`，以處理舊版已知的安全漏洞。
- 透過 `package.json` 的 `allowScripts` 設定允許執行 `hugo-extended` 的安裝後指令碼。最初用來建立此設定的命令如下：

```bash
npm install-scripts approve hugo-extended
```

### 2. 安裝各平台專用的 npm 相依套件

在可連線的電腦上執行 `npm install`，然後提交產生的 `node_modules/` 目錄。由於內附的 Hugo 執行檔與 npm 命令啟動檔因平台而異，必須分別在兩種作業系統執行：

- 在 Linux 執行 `npm install`，並將 `node_modules/` 提交至 `main`。
- 在 Windows 執行 `npm install`，並將 `node_modules/` 提交至 `windows`。

### 3. 將 Hugo Modules 收錄至儲存庫

安裝 npm 相依套件後，將 Docsy 佈景主題及其 Hugo Module 相依套件下載至 `_vendor/`。由於 Docsy 會從專案層級的 `node_modules/` mount Bootstrap 與 Font Awesome，直接在儲存庫中執行 `hugo mod vendor` 可能會在 Windows 產生錯誤的組合路徑。請改由只有 Hugo 與 Go 設定檔、但沒有 `node_modules/` 的暫存 source 目錄產生 vendor 目錄：

```powershell
$vendorStage = Join-Path $env:TEMP ("docsy-example-hugo-vendor-" + [guid]::NewGuid().ToString("N"))
New-Item -ItemType Directory -Path $vendorStage | Out-Null
Copy-Item -LiteralPath .\hugo.yaml, .\go.mod, .\go.sum -Destination $vendorStage
npx --offline hugo mod vendor --source $vendorStage
if ($LASTEXITCODE -ne 0) { throw "Hugo Module vendoring failed." }
Remove-Item -LiteralPath .\_vendor -Recurse -Force -ErrorAction SilentlyContinue
Copy-Item -LiteralPath (Join-Path $vendorStage "_vendor") -Destination . -Recurse
Remove-Item -LiteralPath $vendorStage -Recurse -Force
```

提交產生的目錄：

```bash
git add _vendor
git commit -m "Vendor Hugo modules for offline builds"
```

`_vendor/` 的內容與作業系統無關，因此只需產生一次，但兩個分支都必須包含該目錄。如果是在 `main` 建立 commit，可以將同一個 commit 套用到 `windows`：

```bash
git switch windows
git cherry-pick <vendor-commit-hash>
```

### 4. 驗證離線建置

中斷或封鎖網路連線，然後在兩個分支分別確認內附的 Hugo 執行檔，並執行正式環境建置：

```bash
npx --offline hugo version
npm run build:production
```

建置程序應只使用 `node_modules/` 與 `_vendor/` 即可完成。將 `_vendor/` 提交至兩個分支且通過此測試後，一般建置就不再需要另外移轉 Go Module 快取。

如需同時停用 npm 與 Go Module 下載的嚴格驗證方式，請參閱[〈Hugo 如何使用已收錄的模組〉](docs/hugo-offline-modules.zh-TW.md)。

### 日後更新相依套件

npm 相依套件必須分別在 Linux 與 Windows 準備，再將更新後的 `node_modules/` 提交至對應分支。每當 `go.mod`、`go.sum` 或 Docsy 版本異動時，請重新產生 `_vendor/`，並將同一個 vendor commit 套用至兩個分支。發布更新前，務必再次進行離線驗證。
