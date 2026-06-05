# Live MCN — 網站

從直播電商龍頭背景團隊獨立而出的 MCN 管理公司網站。純手刻靜態 HTML/CSS,無建置工具、無套件依賴,字體由 Google Fonts CDN 載入。

## 結構

```
.
├── index.html / index.css   入口頁
├── pages/                   四頁
│   ├── page1.html + page1.css   給品牌方(對外)
│   ├── page2.html               給直播主・團購主(對外,inline style)
│   ├── page3.html + page3.css   系統架構規劃(內部)
│   └── page4.html               TBD 盤點板(內部,inline style)
├── DESIGN.md                視覺規範(theme / OKLCH 色票 / 字體 / 動效 / bans)
└── docs/
    ├── master-brief.md      商業脈絡與設計 brief(source of truth)
    └── PRODUCT.md           產品/品牌策略
```

> 對外頁(page1 / page2)有嚴格禁字規範,見 `CLAUDE.md` 與 `docs/master-brief.md` 附錄 B。

## 本地預覽

```sh
python3 -m http.server 8000   # 在 repo 根目錄執行,然後開 http://localhost:8000
```

連結皆為相對路徑(`index → pages/pageN.html`、各頁 → `../index.html`),從根目錄開啟即可。
