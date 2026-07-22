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

## Changes in this fork

- Removed `node_modules/` from `.gitignore` and committed the installed npm dependencies separately for Linux and Windows.
- Pinned Hugo Extended to `0.164.0`.
- Overrode `adm-zip` with version `0.6.0` to address known vulnerabilities in older releases.
- Approved the `hugo-extended` post-install script through the `allowScripts` setting in `package.json`. The maintainer used `npm install-scripts approve hugo-extended` when preparing the dependencies; users do not need to run this command.

## Updating the bundled dependencies

Dependency updates must be prepared separately on both branches because the bundled Hugo executable is platform-specific. After updating, verify the Hugo version and run a production build before committing the refreshed `node_modules/` directory.
