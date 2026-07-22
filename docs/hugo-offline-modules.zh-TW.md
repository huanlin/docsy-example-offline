# 本離線儲存庫中的 Hugo Modules

[English version](hugo-offline-modules.md)

本文說明 Hugo 如何找到已收錄的 Docsy 佈景主題、該主題如何與 npm 套件搭配，以及維護者如何確認建置過程不會下載 Hugo Modules。

## 相關檔案

網站在 `hugo.yaml` 中以正式的 module path 宣告 Docsy：

```yaml
module:
  imports:
    - path: github.com/google/docsy/theme
```

`_vendor/modules.txt` 會記錄相同的模組及解析後的版本：

```text
# github.com/google/docsy/theme v0.0.0-20260717130523-6d9367e214fd
```

已收錄的原始碼目錄會對應 module path：

```text
_vendor/
├── modules.txt
└── github.com/
    └── google/
        └── docsy/
            └── theme/
```

收錄模組後仍應保留 `go.mod` 與 `go.sum`。這兩個檔案描述所需的模組關係與 checksum，而 `_vendor/` 則提供建置時實際使用的本地檔案。

## Hugo 如何解析模組

Hugo 讀到 `github.com/google/docsy/theme` 的 import 後，會檢查已收錄的模組資訊，並依 module path 與版本對應至 `_vendor/github.com/google/docsy/theme`。接著 Hugo 會從該本地目錄載入主題設定、模板、assets、翻譯與靜態檔案，而不是透過 Go 下載模組。

Hugo 會自動選用本地目錄，不需要在 `go.mod` 或 `hugo.yaml` 另外加入 replacement。如果找不到符合的已收錄模組，Hugo 才可能回到一般的 Go Module 解析流程，使用本機 Go Module 快取或外部網路。

可以使用以下命令檢查實際採用的模組與 mount 路徑：

```bash
npx --offline hugo config mounts
```

Docsy 項目的 `dir` 應指向本儲存庫中的 `_vendor/github.com/google/docsy/theme/`。

Hugo 官方的 [Use modules: Vendor](https://gohugo.io/hugo-modules/use-modules/#vendor) 與 [Configure modules](https://gohugo.io/configuration/module/) 也有說明這項機制。

## 與 `node_modules` 的關係

`_vendor/` 與 `node_modules/` 分別處理離線建置的不同部分：

| 目錄 | 內容 | 是否因平台而異 |
| --- | --- | --- |
| `_vendor/` | Docsy Hugo Module 的模板、assets、翻譯與靜態檔案 | 否 |
| `node_modules/` | Bootstrap、Font Awesome、PostCSS、npm 工具及 Hugo Extended | 是 |

已收錄的 Docsy 設定會從專案層級的 `node_modules/` mount Bootstrap 與 Font Awesome，因此完整的離線副本需要同時包含這兩個目錄。Linux 與 Windows 可以使用相同的 `_vendor/` commit，但 `node_modules/` 必須分別在兩種作業系統準備。

## 為什麼要透過暫存 source 目錄收錄模組

直接在本儲存庫執行 `hugo mod vendor` 時，Hugo 會將 Docsy 的 `node_modules/bootstrap` 與 Font Awesome mounts 解析為專案中的絕對路徑。在 Windows 上，vendoring 操作可能錯誤地將 `D:\...\node_modules\bootstrap` 之類包含磁碟機代號的路徑接到 Hugo Module 快取路徑後方，因而產生無效路徑。

維護流程會改用包含 `hugo.yaml`、`go.mod` 與 `go.sum`，但沒有 `node_modules/` 的暫存 source 目錄執行 Hugo。如此即可避免 npm mounts 被解析成專案層級的絕對路徑，再將產生的 `_vendor/` 複製回儲存庫。通過驗證的 PowerShell 命令請參閱[〈如何製作這個離線版本〉](../README-fork.zh-TW.md#如何製作這個離線版本)。

## 嚴格的離線驗證

以下命令會禁止 npm 下載缺少的執行檔，也會禁止 Hugo 或 Go 透過網路解析缺少的模組。

PowerShell：

```powershell
$previousGoProxy = $env:GOPROXY
$previousHugoModuleProxy = $env:HUGO_MODULE_PROXY

try {
    $env:GOPROXY = "off"
    $env:HUGO_MODULE_PROXY = "off"
    npx --offline hugo build --renderToMemory
    if ($LASTEXITCODE -ne 0) { throw "Offline Hugo build failed." }
}
finally {
    $env:GOPROXY = $previousGoProxy
    $env:HUGO_MODULE_PROXY = $previousHugoModuleProxy
}
```

Linux：

```bash
GOPROXY=off HUGO_MODULE_PROXY=off npx --offline hugo build --renderToMemory
```

在這些條件下成功建置，代表所需的 Hugo Module 已存在本機。`--renderToMemory` 可避免驗證時將產生的網站寫入 `public/`。

## 可能略過或改變 vendoring 的設定

- `--ignoreVendorPaths` 會要求 Hugo 忽略符合條件的已收錄模組路徑。本儲存庫的一般 scripts 沒有使用此選項。
- 啟用供本地主題開發使用的 Hugo workspace 可能改變模組解析方式，尤其是同時使用 `--ignoreVendorPaths` 時。
- 移除或損壞 `_vendor/modules.txt` 後，Hugo 將無法可靠地把已收錄目錄對應至 import 的模組。
- 修改 `go.mod` 中的 Docsy 需求卻沒有重新產生 `_vendor/`，可能造成模組資訊與原始碼版本不一致。

## 更新已收錄的模組

每當 `go.mod`、`go.sum` 或 Docsy 版本異動時，請在可連線的維護環境重新產生 `_vendor/`。檢查 `_vendor/modules.txt`、執行嚴格的離線驗證，再將完整替換內容提交為一個 commit。請把相同的 vendor commit 套用至 `main` 與 `windows`，不要分別修改 `_vendor/` 內的產生檔案。

## Windows 上的 alias 處理

範例內容包含 `/blog/2018/*` 之類供伺服器端 `_redirects` 輸出使用的 wildcard aliases。Windows 不允許 Hugo 在實體 alias 頁面使用的目錄名稱中包含 `*`。`hugo.yaml` 頂層的 `disableAliases: true` 會停用這些實體頁面，但仍會保留 `.Aliases` 供 `layouts/home.redirects` 使用。
