# Docsy Example fork - offline version

This is an offline version of [Docsy Example](https://github.com/google/docsy-example), built for restricted network environments that cannot run the `npm install` command to pull dependencies from public repositories.

**Important:** The `main` branch is for Linux. For Windows, use the `windows` branch.

## What's changed

- Removed `node_modules/` from `.gitignore`, thus the `node_modules` folder is stored in the repo.
- package.json
  - Use the newer version of adm-zip: v0.6.0. Reason: resolve known vulnerabilities.
  - Allow hugo-extended to run postinstall.js. Command executed: `npm install-scripts approve hugo-extended`.

## Notes

To verify the local version of Hugo, run:

```bash
npx hugo version
```
