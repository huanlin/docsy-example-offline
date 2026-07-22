# Docsy Example fork — offline npm dependencies

This repository is a fork of [Docsy Example](https://github.com/google/docsy-example) for restricted network environments where npm packages cannot be downloaded from public registries. The platform-specific `node_modules/` directory, including Hugo Extended, is committed to the repository, so users do not need to run `npm install` after cloning it.

## Choose the correct branch

The bundled native executables are platform-specific. Clone the branch that matches the target operating system:

| Operating system | Branch    |
| ---------------- | --------- |
| Linux            | `main`    |
| Windows          | `windows` |

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
- Go and Git are required if Hugo must download the Docsy theme module. See [Offline scope](#offline-scope).

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

This fork bundles the npm dependencies, but the Docsy theme is still declared as a remote Hugo Module (`github.com/google/docsy/theme`). On a clean machine, the first Hugo build may therefore try to download the theme and its Hugo Module dependencies.

For a fully isolated environment, prepare one of the following before moving the repository into that environment:

- Populate and transfer the required Go Module cache; or
- On a connected machine, run `npx hugo mod vendor` and commit the generated `_vendor/` directory.

Hugo uses vendored modules without downloading them. See the official [Hugo Module vendoring documentation](https://gohugo.io/hugo-modules/use-modules/#vendor) for details.

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

After the npm dependencies have been installed, download the Docsy theme and its Hugo Module dependencies into `_vendor/`:

```bash
npx hugo mod vendor
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

### Updating dependencies later

Prepare npm dependency updates independently on Linux and Windows, then commit the refreshed `node_modules/` directory to the corresponding branch. Regenerate `_vendor/` whenever `go.mod`, `go.sum`, or the Docsy version changes, and apply the same vendor commit to both branches. Always repeat the offline verification before publishing the updates.
