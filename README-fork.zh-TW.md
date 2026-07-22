# Docsy Example 分支版本 — 離線 npm 相依套件

本儲存庫是 [Docsy Example](https://github.com/google/docsy-example) 的分支版本，適用於無法從公用套件庫下載 npm 套件的受限網路環境。儲存庫已包含各平台專用的 `node_modules/` 目錄及 Hugo Extended，因此使用者複製儲存庫後不需要執行 `npm install`。

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
- 如果 Hugo 必須下載 Docsy 佈景主題模組，則需要 Go 與 Git。詳情請參閱 [離線支援範圍](#離線支援範圍)。

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

本分支版本已包含 npm 相依套件，但 Docsy 佈景主題仍宣告為遠端 Hugo Module（`github.com/google/docsy/theme`）。因此，在全新的電腦上第一次執行 Hugo 建置時，仍可能嘗試下載該佈景主題及其 Hugo Module 相依套件。

若要在完全隔離的環境中使用，請先在將儲存庫移入該環境之前完成下列其中一項準備：

- 建立必要的 Go Module 快取，並將快取一併移至離線環境；或
- 在可連線的電腦上執行 `npx hugo mod vendor`，並將產生的 `_vendor/` 目錄提交至儲存庫。

Hugo 可直接使用已收錄於儲存庫的模組，不需要重新下載。詳情請參閱官方的 [Hugo Module vendoring 文件](https://gohugo.io/hugo-modules/use-modules/#vendor)。

## 本分支版本的變更

- 從 `.gitignore` 移除 `node_modules/`，並分別在 Linux 與 Windows 分支提交已安裝的 npm 相依套件。
- 將 Hugo Extended 固定為 `0.164.0`。
- 將 `adm-zip` 覆寫為 `0.6.0`，以處理舊版已知的安全漏洞。
- 透過 `package.json` 的 `allowScripts` 設定允許執行 `hugo-extended` 的安裝後指令碼。維護者準備相依套件時使用了 `npm install-scripts approve hugo-extended`；一般使用者不需要執行此命令。

## 更新內附的相依套件

由於內附的 Hugo 執行檔因平台而異，必須分別在兩個分支準備相依套件更新。更新後，請先確認 Hugo 版本並執行正式環境建置，再提交更新後的 `node_modules/` 目錄。
