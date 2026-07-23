# Upgrade notes for Docsy 0.15.0

**Summary:** Notes collected while upgrading one of my Docsy websites from Docsy 0.12.0 to 0.15.0.

## 1. Docsy theme migration to `_vendor`

The Docsy theme was migrated from `themes/docsy/` to `_vendor/github.com/google/docsy/theme/`. This change aligns with Hugo's module system and provides better dependency management.

- Remove the entire `themes/docsy/` directory.
- Add a new `_vendor/` directory containing the Docsy theme.
- Update `hugo.yaml` so that Hugo does not look only in the `themes/` directory. Find and remove the following setting:

  ```yaml
  theme:
    - "docsy"
  ```

Then update `hugo.yaml` as follows:

```yaml
module:
  # Uncomment the line below for temporary local module development
  # replacements: "github.com/google/docsy -> ../../docsy"
  hugoVersion:
    extended: true
    min: 0.160.1
  imports:
    - path: "hugomods/bootstrap"
      disable: false
    - path: "hugomods/icons"
      disable: false
    - path: "hugomods/icons/vendors/font-awesome"  # Do not remove! This import is required for Font Awesome icons referenced in data/cards.yaml.
      disable: false
    - path: github.com/google/docsy/theme
      disable: false
```

## 2. Update Node modules

- Replace the entire `node_modules/` directory with `docsy-example-offline/node_modules/`.
- Update `package.json` to reflect the newer dependency versions. Run:

  ```bash
  npm outdated
  npx npm-check-updates -u
  npm outdated
  ```

## 3. Update layout templates

Update custom template files in the `layouts/` directory that use outdated Hugo or Docsy syntax.

To identify which files need to be updated, use AI agents to generate a diff report, and then update the relevant files accordingly.

## 4. Update the layout and styles

### Fix the site title color

Add the `navbar_theme` parameter to `hugo.yaml`:

```yaml
params:
  ui:
    navbar_theme: "dark" # or "light" depending on theme support
```

### Fix the right-side table of contents hover effect

Add `params.hugo.transpiler: "dartsass"` to `hugo.yaml`.

**Why:** Without this setting, Hugo uses the deprecated LibSass transpiler, which cannot properly handle Bootstrap 5's modern SCSS syntax, including its mixins and CSS variables. As a result, the highlight bar does not appear when you hover over the right-side table of contents because CSS variables such as `--bs-primary`, `--bs-secondary-bg-subtle`, and `--bs-tertiary-bg` are not generated correctly.

## CI/CD pipeline

Because Dart Sass is required, use an **extended** Hugo image.

The following image has been verified: [hugomods/hugo:dart-sass-node-git-0.164.0](https://hub.docker.com/layers/hugomods/hugo/dart-sass-node-git-0.164.0/)

**Note:** Do not use images with names that begin with `std` or `reg`. These are not extended images.

## Troubleshooting

### SCSS transformation fails on Windows

When you build the site on a Windows machine, Hugo may report the following error:

```text
ERROR error building site: render: [en v1.0.0 guest] failed to render pages: render of "/404" failed:
  "D:\work\github\docsy-example-offline\_vendor\github.com\google\docsy\theme\layouts\baseof.html:9:7":
...
  execute of template failed: template: _partials/head-css.html:25:17:
    executing "_partials/head-css.html" at <.RelPermalink>: error calling RelPermalink:
       TOCSS: failed to transform "/scss/main.scss" (text/x-scss).
  Check your Hugo installation; you need the extended version to build SCSS/SASS with transpiler set to 'libsass'.:
  this feature is not available in your current Hugo version, see https://goo.gl/YMrWcn for more information
```

Assume that the following setting already exists in `hugo.yaml`:

```yaml
params:
  hugo:
    transpiler: "dartsass"
```

#### Resolution

Try using the command below to build the website:

```bash
npx hugo server
```

If this works, a global copy of `hugo.exe`—such as `C:\ProgramData\chocolatey\bin\hugo.exe`—is probably present in your `PATH`, and it is the **standard** build rather than the **extended** build.

The standard and extended builds of `hugo.exe` may have the same version number, such as v0.164.0, but they are different distributions. Only the extended build includes Dart Sass.

To resolve the issue, replace the global copy of `hugo.exe` with the extended build.

Alternatively, always use `npx` to build the site:

```bash
npx hugo server
```
