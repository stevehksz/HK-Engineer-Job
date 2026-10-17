# HK-Engineer-Job

香港工程師 · 考牌｜求職｜留學 —— 對外網站（GitHub Pages 入口）。

- 🌐 **本站（GitHub）**：https://steveyehksz.github.io/HK-Engineer-Job/
- 🌐 主站（原始伺服器）：https://stevehksz.app.workbuddy.host/

## 這是什麼

`index.html` 是一個**極輕量套殼頁**（約 4 KB）：把完整網站以全屏 iframe 載入，令網址保持 `github.io`，而所有互動功能（AI 問答、CV 表單、案例牆、測驗）仍然由主站提供服務。

- 支援深鏈：`?pane=ice` / `?pane=edu` / `?pane=jobs` / `?pane=cv` / `?pane=author` / `?pane=shop`
- 只放行 `pane` 與 `ref` 參數（`pv` 等後台參數一律過濾）
- 微信內：平台網關按 UA 攔截整站，故顯示自製引導頁（提示「在瀏覽器打開」＋複製網址）

## 為什麼不直接放整站

本專案為單文件應用（私有後台與對外版同一份 HTML）。公開倉庫**不會**收錄私有後台代碼，因此採用套殼方式；網站內容與更新以主站為準。

> 更新方式：主站改版後**即時生效**（套殼頁只是載入主站，本身無需改動）；只有主站網址變更時才需要改 `index.html` 內的 `BASE` 常量。
