---
title: "用 Hugo + PaperMod 把 GitHub Pages 從 Jekyll 換掉"
date: 2026-08-30T00:00:00+08:00
draft: false
tags: ["Hugo", "PaperMod", "GitHub Pages", "GitHub Actions"]
categories: ["建站"]
summary: "把一個原本掛著 Jekyll 預設佈景的 GitHub Pages repo，改成 Hugo + PaperMod，並用 GitHub Actions 自動部署的完整過程與踩到的坑。"
ShowToc: true
TocOpen: true
---

這個站原本只是 GitHub Pages 開好之後, 只有 `<h1>Hello!</h1>` 的 `index.html`。 這次把它整個換成 Hugo + [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme，用 Claude Code 當助手，並順手記錄一下流程。

## 為什麼選 Hugo + PaperMod

- **Hugo** 是單一 binary、build 快，寫文章就是丟 Markdown 進 `content/` 就好，簡單快速符合我自己的喜好。
- **PaperMod Theme** 版面乾淨，重點在文章。

併搭配使用 GitHub Action，commit 後自動 build 並發佈。

## 目錄架構

```text
.
├── .github/workflows/hugo.yml   # CI：build + deploy
├── hugo.yaml                    # 站台設定
├── archetypes/default.md        # hugo new 的文章樣板
├── content/
│   ├── posts/                   # 文章放這裡
│   ├── archives.md              # 歸檔頁
│   └── search.md                # 搜尋頁
├── static/                      # 靜態檔（圖片等）
└── themes/PaperMod/             # git submodule
```

## 步驟

### 1. 清掉之前 Jekyll 殘留檔案

`_config.yml` 和 `index.html` 對 Hugo 沒有意義，留著只會混淆，直接刪掉。

### 2. 用 git submodule 裝主題

```bash
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

這裡刻意選 submodule 而不是 Hugo Modules：Hugo Modules 需要 Go toolchain，而 GitHub Pages 官方的 Hugo workflow 樣板只裝 Hugo、沒裝 Go。用 submodule 的話，CI 只要 checkout 時帶上 `submodules: recursive` 就好，剛好樣板裡本來就有。

### 3. 寫 `hugo.yaml`

加入幾個對中文站需要的設定：

```yaml
locale: "zh-tw"
defaultContentLanguage: "zh-tw"
hasCJKLanguage: true
```

- `locale: zh-tw` 這個 key 要注意。很多教學寫的是 `languageCode`，但它在 Hugo 0.158.0 已經 deprecated，build 時會噴 warning，要改用 `locale`。
- `defaultContentLanguage: zh-tw` 會讓 PaperMod 去讀 `i18n/zh-tw.yaml`，「上一頁」「目錄」「複製」這些介面字串就自動變繁中。
- `hasCJKLanguage: true` 很重要。中文詞之間沒有空格，不開這個的話 Hugo 會把一整段中文算成「1 個字」，字數統計和閱讀時間會完全失真，自動摘要也會被切爆。

站內搜尋需要首頁多輸出一份 JSON 當索引：

```yaml
outputs:
  home:
    - HTML
    - RSS
    - JSON
```

再配上 `content/search.md`（`layout: "search"`）跟 `params.fuseOpts`，搜尋頁就會動了。歸檔頁同理，`content/archives.md` 指定 `layout: "archives"`。

程式碼高亮設計的部分：

```yaml
markup:
  highlight:
    noClasses: false
```

PaperMod 自己帶了 chroma 的樣式表，用 class-based 高亮才能跟著深淺色模式一起切。如果設成 `noClasses: true`，顏色會被寫死成 inline style，切到暗色模式就會很難看。

### 4. 對齊 Hugo 版本

GitHub 官方樣板 workflow 裡寫的是 `0.128.0`，但我本機裝的是 `0.165.0` 版本；本機是哪個版本，就把 `HUGO_VERSION` 設成哪個。

```yaml
env:
  HUGO_VERSION: 0.165.0
```



### 5. 本機預覽

```bash
hugo server -D
```

`-D` 會連 `draft: true` 的文章一起顯示，寫到一半的東西也看得到。預設開在 <http://localhost:1313>。

## 之後怎麼寫新文章

```bash
hugo new content posts/my-new-post.md
```

會照 `archetypes/default.md` 產出 front matter，寫完把 `draft: true` 改成 `false`，push 上 master，GitHub Actions 就會自己 build 和部署。

## 小結

整套換下來，真正會卡人的只有兩件事：主題要用 submodule 而不是 Hugo Modules（因為 CI 沒有 Go），以及 CI 的 Hugo 版本要夠新（PaperMod 需要 0.146.0 以上）。其他都是照設定檔填一填的事。
