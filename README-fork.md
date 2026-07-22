# Docsy Example fork - offline version for Linux

This is an offline version of [Docsy Example](https://github.com/google/docsy-example), built for restricted network environments that cannot run the `npm install` command to pull dependencies from public repositories.

本儲存庫專為 **Linux 離線環境** 儲存 Hugo 相關的編譯依賴套件。

## What's changed

- Removed `node_modules/` from `.gitignore`, thus the `node_modules` folder is stored in the repo. 
- Use the newer version of adm-zip: v0.6.0. Reason: resolve known vulnerabilities.