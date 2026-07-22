# Hugo modules in this offline repository

[繁體中文版](hugo-offline-modules.zh-TW.md)

This document explains how Hugo finds the vendored Docsy theme, how the vendored theme interacts with npm packages, and how maintainers can verify that a build does not download Hugo Modules.

## The files involved

The site declares Docsy by its canonical module path in `hugo.yaml`:

```yaml
module:
  imports:
    - path: github.com/google/docsy/theme
```

The same module and its resolved version are recorded in `_vendor/modules.txt`:

```text
# github.com/google/docsy/theme v0.0.0-20260717130523-6d9367e214fd
```

The vendored source directory mirrors the module path:

```text
_vendor/
├── modules.txt
└── github.com/
    └── google/
        └── docsy/
            └── theme/
```

`go.mod` and `go.sum` remain useful even after vendoring. They describe the requested module graph and checksums, while `_vendor/` supplies the local files used by the build.

## How Hugo resolves the module

At build time, Hugo's resolution of `github.com/google/docsy/theme` can be summarized as follows:

```text
module import in hugo.yaml
            |
            v
_vendor/modules.txt at the repository root
            |
            v
_vendor/github.com/google/docsy/theme
            |
            v
Use local templates, SCSS, and static files
```

When Hugo reads the import for `github.com/google/docsy/theme`, it checks the vendored module metadata and matches the module path and version to `_vendor/github.com/google/docsy/theme`. Hugo then loads the theme configuration, templates, assets, translations, and static files from that local directory instead of downloading the module through Go.

The local directory is selected automatically; no replacement entry is required in `go.mod` or `hugo.yaml`. If no matching vendored module is available, Hugo can fall back to normal Go Module resolution, which may use the local Go Module cache or the network.

You can inspect the effective module and mount paths with:

```bash
npx --offline hugo config mounts
```

For the Docsy entry, `dir` should point into this repository's `_vendor/github.com/google/docsy/theme/` directory.

Hugo's official documentation describes the same behavior in [Use modules: Vendor](https://gohugo.io/hugo-modules/use-modules/#vendor) and [Configure modules](https://gohugo.io/configuration/module/).

## Relationship with `node_modules`

`_vendor/` and `node_modules/` solve different parts of the offline build:

| Directory | Contents | Platform-specific |
| --- | --- | --- |
| `_vendor/` | Docsy Hugo Module templates, assets, translations, and static files | No |
| `node_modules/` | Bootstrap, Font Awesome, PostCSS, npm tools, and Hugo Extended | Yes |

The vendored Docsy configuration mounts Bootstrap and Font Awesome from the project-level `node_modules/`. A complete offline checkout therefore needs both directories. The same `_vendor/` commit can be used on Linux and Windows, while `node_modules/` must be prepared separately on each operating system.

## Why vendoring uses a temporary source directory

When `hugo mod vendor` runs directly in this repository, Hugo resolves Docsy's `node_modules/bootstrap` and Font Awesome mounts to absolute paths in the project. On Windows, the vendoring operation can then incorrectly append a drive-qualified path such as `D:\...\node_modules\bootstrap` to the Hugo Module cache path, producing an invalid path.

The maintainer workflow avoids this by invoking Hugo with a temporary source directory containing `hugo.yaml`, `go.mod`, and `go.sum`, but no `node_modules/`. Hugo can then vendor the Docsy module without resolving the npm mounts to project-level absolute paths. The generated `_vendor/` is copied back into the repository. See [How this offline fork is prepared](../README-fork.md#how-this-offline-fork-is-prepared) for the tested PowerShell commands.

## Strict offline verification

The following commands prevent npm from downloading a missing executable and prevent Hugo or Go from resolving a missing module over the network.

PowerShell:

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

Linux:

```bash
GOPROXY=off HUGO_MODULE_PROXY=off npx --offline hugo build --renderToMemory
```

A successful build under these conditions demonstrates that the required Hugo Module is available locally. `--renderToMemory` avoids writing the generated site to `public/` during verification.

## Settings that can bypass or change vendoring

- `--ignoreVendorPaths` tells Hugo to ignore matching vendored module paths. The normal scripts in this repository do not use this option.
- Activating a Hugo workspace for local theme development can change module resolution, especially when combined with `--ignoreVendorPaths`.
- Removing or corrupting `_vendor/modules.txt` prevents Hugo from reliably matching the vendored directory to the imported module.
- Changing the Docsy requirement in `go.mod` without regenerating `_vendor/` can leave the module metadata and source out of sync.

## Updating the vendored module

Regenerate `_vendor/` on a connected maintenance machine whenever `go.mod`, `go.sum`, or the Docsy version changes. Review `_vendor/modules.txt`, run the strict offline verification, and commit the complete replacement as one change. Apply the same vendor commit to both `main` and `windows`; do not independently edit generated files inside `_vendor/`.

## Alias handling on Windows

The example content contains wildcard aliases such as `/blog/2018/*` for the server-side `_redirects` output. Windows cannot use `*` in the physical directory names that Hugo normally creates for client-side alias pages. The top-level `disableAliases: true` setting in `hugo.yaml` disables those physical pages while keeping `.Aliases` available to `layouts/home.redirects`.
