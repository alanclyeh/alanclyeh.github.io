# alanclyeh.github.io

個人部落格，[acly.notes](https://alanclyeh.github.io/)。Hugo + [PaperMod](https://github.com/adityatelange/hugo-PaperMod)，push 到 `master` 後由 GitHub Actions 自動部署。

## 部署

push 到 `master` 會觸發 `.github/workflows/`，用 `hugo --minify` 建置後發佈到 GitHub Pages。

> **注意**：Settings → Pages 的來源必須維持 **GitHub Actions**。若改回 "Deploy from a branch"，GitHub 內建的 Jekyll 會跟這個 workflow 搶同一個部署，後完成的那個會覆蓋掉另一個，整站可能變成 `README.md` 的內容。
