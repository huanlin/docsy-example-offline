# Docsy Example fork — offline dependencies

This repository is a fork of [Docsy Example](https://github.com/google/docsy-example) for restricted network environments. It includes the platform-specific npm dependencies and Hugo Extended in `node_modules/`, together with the Docsy Hugo Module in `_vendor/`, so a normal build does not need to download build dependencies after cloning the repository.

## Choose the correct branch

The bundled native executables are platform-specific. Clone the branch that matches the target operating system:

| Operating system | Branch    |
| ---------------- | --------- |
| Linux            | `main`    |
| Windows          | `windows` |

**Key concept:** Only `node_modules/` must be prepared separately for each operating system. `_vendor/`, documentation, and platform-independent configuration should be identical on both branches.

> **Warning:** Do not merge `main` and `windows` directly. Each branch tracks a different platform-specific `node_modules/` tree, so merging the branches can create large conflicts or mix incompatible executables and npm command shims. Put shared changes such as documentation, `_vendor/`, and platform-independent configuration in dedicated commits, then cherry-pick those commits to the other branch. To synchronize only selected files, use `git restore --source=<branch> -- <files>` and commit the result.

Linux:

```bash
git clone --branch main --single-branch https://github.com/huanlin/docsy-example-offline.git
cd docsy-example-offline
```

Windows (PowerShell):

```powershell
git clone --branch windows --single-branch https://github.com/huanlin/docsy-example-offline.git
Set-Location docsy-example-offline
```

Using `git clone` is recommended. A GitHub source archive contains the committed executables, but extracting it may not preserve Linux executable permissions or the symbolic links in `node_modules/`. If that metadata is lost, npm commands or the bundled Hugo executable may not work correctly.

## Requirements

- Node.js 22 or later, as specified by `package.json`.
- No separate Hugo installation is required; Hugo Extended is included in `node_modules/`.
- Git is recommended for cloning the repository so file metadata is preserved.
- Go and a populated Go Module cache are not required for normal builds because the Docsy module is included in `_vendor/`.

## Run the site

Do **not** run `npm install`.

It is unnecessary for normal use and may attempt to access public package registries or replace the bundled platform-specific files.

Verify that the bundled Hugo executable works:

```bash
npx --offline hugo version
```

Start the development server:

```bash
npm run serve
```

Then open <http://localhost:1313/>. Press `Ctrl+C` to stop the server.

To create a production build in the `public/` directory, run:

```bash
npm run build:production
```

## Offline scope

This fork includes both categories of build dependency:

- `node_modules/` contains the npm packages and the platform-specific Hugo Extended executable.
- `_vendor/` contains the Docsy Hugo Module declared as `github.com/google/docsy/theme`.

Hugo automatically matches the entry in `_vendor/modules.txt` with the module import in `hugo.yaml` and loads the local copy before attempting Go Module resolution. Optional site features or custom content that explicitly fetch remote resources may still require those resources to be bundled separately.

For the resolution process, offline verification commands, and maintenance details, see [How Hugo uses vendored modules](docs/hugo-offline-modules.md).

## Technical documentation

Additional implementation details, troubleshooting notes, and maintenance guidance are kept in the `docs/` directory:

- [How Hugo uses vendored modules](docs/hugo-offline-modules.md) — explains module resolution, the relationship between `_vendor/` and `node_modules/`, strict offline verification, the Windows vendoring workaround, and alias handling.

## How this offline fork is prepared

This section describes the maintainer workflow used to create or update the offline repository. End users do not need to perform these steps.

### 1. Configure the npm dependencies

- Remove `node_modules/` from `.gitignore` so the installed dependencies can be committed.
- Pin Hugo Extended to the required version in `package.json`; this fork currently uses `0.164.0`.
- Override `adm-zip` with version `0.6.0` to address known vulnerabilities in older releases.
- Approve the `hugo-extended` post-install script through the `allowScripts` setting in `package.json`. The command originally used to create this setting was:

```bash
npm install-scripts approve hugo-extended
```

### 2. Install the platform-specific npm dependencies

Run `npm install` on a connected machine, then commit the resulting `node_modules/` directory. This must be done separately for each operating system because the bundled Hugo executable and npm command shims are platform-specific:

- Run `npm install` on Linux and commit `node_modules/` to `main`.
- Run `npm install` on Windows and commit `node_modules/` to `windows`.

### 3. Vendor the Hugo Modules

After the npm dependencies have been installed, download the Docsy theme and its Hugo Module dependencies into `_vendor/`. Because Docsy mounts Bootstrap and Font Awesome from the project-level `node_modules/`, running `hugo mod vendor` directly from the repository can produce an invalid combined path on Windows. Generate the vendor directory from a temporary source directory that contains the Hugo and Go configuration files but no `node_modules/`:

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

Commit the generated directory:

```bash
git add _vendor
git commit -m "Vendor Hugo modules for offline builds"
```

The contents of `_vendor/` are platform-independent, so they only need to be generated once. Both branches must contain the directory, however. If it was committed on `main`, apply the same commit to `windows`:

```bash
git switch windows
git cherry-pick <vendor-commit-hash>
```

### 4. Verify the offline build

Disconnect or block network access, then verify the bundled Hugo executable and run a production build on both branches:

```bash
npx --offline hugo version
npm run build:production
```

The build should complete using only `node_modules/` and `_vendor/`. After `_vendor/` has been committed to both branches and this test passes, the repository no longer needs a transferred Go Module cache for normal builds.

See [How Hugo uses vendored modules](docs/hugo-offline-modules.md) for a stricter verification that disables both npm and Go Module downloads.

### Updating dependencies later

Prepare npm dependency updates independently on Linux and Windows, then commit the refreshed `node_modules/` directory to the corresponding branch. Regenerate `_vendor/` whenever `go.mod`, `go.sum`, or the Docsy version changes, and apply the same vendor commit to both branches. Always repeat the offline verification before publishing the updates.
