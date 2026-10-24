---
name: web-release-pipeline
description: 单文件 HTML 平台的改版→静态校验→发布→线上核验完整流水线，含 CDN 缓存坑、Origin 白名单、jsdom 全链路调试法与云服务 key 归属规则。触发词：发布上线、线上核验、CDN 缓存、jsdom 调试、单文件 HTML 平台改版。
agent_created: true
category: capability
version: "1.0.0"
---

# 工程師平台 · 改版與發布流水線

## 0. 專案基本坐标（先讀，別重新摸索）

| 項目 | 值 |
|---|---|
| 權威工作副本 | `<USER_HOME>\WorkBuddy\<WORKSPACE_ID>\hk-engineer-hub\index.html` |
| 線上站（現行，v1.99 起） | `https://<LIVE_DOMAIN>/` |
| 應用 ID | `<APP_ID>`（v1.99 新建應用；舊 `<APP_ID>` 凍結在 v1.98） |
| domainPrefix | `<GITHUB_USER>`（**子域名在應用創建時鎖定，改不了**——要換域名只能新建應用，v1.99 實錄） |
| 部署目錄 | 上式 hk-engineer-hub 資料夾（**只准放 index.html**） |
| 校驗腳本 | `.workbuddy\tools\verify_online_vXXX.js`（輸出 `_out_verifyXXX.txt`） |
| 舊連結 1 | `<LEGACY_DOMAIN>` —— 已隨舊工作區 `<WORKSPACE_ID_LEGACY>` **永久凍結** |
| 舊連結 2 | `<LEGACY_DOMAIN>`（appId `<APP_ID>`）—— v1.98 凍結，仍線上但不再更新；用戶 2026-10-06 指示遷至 <GITHUB_USER> |

> **用戶 2026-10-08 指示：版本編號改 v2.x 遞增（2.1→2.2→2.3…）**；v2.1 已移除私有平臺「公眾號寫作」板塊（CSS/側邊欄/頁面/JS 共 1660 行，摘除前經自包含引用分析），公眾號內容生產走獨立工具 `<TOOL_WORKSPACE>\gzh-assistant\`。
> **v2.2（2026-10-10）**：簡歷工坊新增「中文簡歷一鍵英文化」（作者由另一個 WorkBuddy 窗口完成）。**重要：不同窗口共用同一工作區 `<WORKSPACE_ID>`**——同一台電腦上的多個 WorkBuddy 窗口寫的是同一份 `hk-engineer-hub\index.html`，無需搬運檔案；接手前先跑一輪獨立復核（node --check 各 script 塊 + 線上與本地逐字比對）再動。
> **用戶 2026-10-06 指示：網址改為 <GITHUB_USER>，終極目標 www.<GITHUB_USER>.com** → 唯一對外站 = `https://<LIVE_DOMAIN>/`（v1.99 起）。域名格式固定 `名字.app.workbuddy.host`，`.app` 段去不掉；自定義域名（www.<GITHUB_USER>.com）當前發布工具不支持，待平台開放。所有改版發布到新站，`appId` 用 `<APP_ID>`。

架構：**單一 HTML 檔案同時承載** ①私有平台（側欄、CRM、報告庫、公眾號寫作…）②`#icePub` 對外訪客版（`?pv=1` 進入）③雲服務 SDK 區塊（登入／雲同步／留資）。改對外板塊 = 改同一份檔。

## 1. 改版

- 一次改動動輒涉及 5~8 個錨點（頁籤 `data-tab` ↔ 面板 `id="pane-X"` ↔ `ICE_SUBS` 高亮映射 ↔ `NAMES` 深鏈 ↔ 私有側欄 `openPubPane(...)` ↔ selfCheck ↔ BUILD_TAG）。**改前先用 Grep 把全部錨點位置列出來**，再逐個改。
- `BUILD_TAG`（約 L4947）是**單行超長字符串，歷代版本串接**。新增版本時只改開頭：把 `const BUILD_TAG = 'v1.xx · ...（` 換成 `const BUILD_TAG = 'v1.NEW · 日期（本次說明）。｜ v1.xx · ...（`，**不要**重寫整行。
- selfCheck 功能清單（約 L4950+）同步加一項，用 DOM 查詢寫（例：`!!document.getElementById('x')`），這樣它能真的驗證運行時狀態。
- 【坑】`.d-track/.d-fill` 進度條樣式**只在 `.dist-row` 內定義**，單獨使用不可見 → 獨立使用時改內聯樣式。

## 2. 靜態校驗（發布前必做）

用 python3 一次性跑（bash 環境缺 coreutils，`mkdir/ls/grep` 常不可用）：

```bash
python3 - <<'EOF'
import re, os
p = r"<USER_HOME>\WorkBuddy\<WORKSPACE_ID>\hk-engineer-hub\index.html"
s = open(p, encoding='utf-8').read()
tabs  = re.findall(r'<div class="tab[^"]*"\s+data-tab="([^"]+)"', s)
panes = re.findall(r'<section class="pane[^"]*" id="pane-([^"]+)">', s)
ids   = re.findall(r'id="([^"]+)"', s)
print("頂欄頁籤:", tabs)
print("頁籤無面板:", set(tabs)-set(panes))          # 須為空
print("重複 id:", {i for i in ids if ids.count(i)>1} or "none")
# 抽出 4 個 inline script 供 node --check
out = r"...\.workbuddy\tmp\jscheck"
for i, sc in enumerate(re.findall(r'<script(?![^>]*src)[^>]*>(.*?)</script>', s, re.S)):
    open(os.path.join(out, f"s{i}.js"),'w',encoding='utf-8').write(sc)
EOF
cd ".../.workbuddy/tmp/jscheck" && for i in 0 1 2 3; do "$NODE" --check s$i.js && echo "s$i OK"; done
```

Node 路徑：`<USER_HOME>/.workbuddy/binaries/node/versions/22.22.2-6/node.exe`。

**驗證陷阱**：用「字串包含」檢查舊頁籤是否移除會**誤報**——selfCheck 的 JS 源碼裡就寫著 `data-tab=\"progress\"`。必須用正則匹配 `<div class="tab"... data-tab="x"`（HTML 標籤形態）才算數。

## 3. 發布

```json
workbuddy_sites_deploy({
  "action": "deploy",
  "directory": "<USER_HOME>\\WorkBuddy\\<WORKSPACE_ID>\\hk-engineer-hub",
  "appId": "<APP_ID>",
  "appName": "工程師平台",
  "domainPrefix": "<GITHUB_USER>",
  "entryHtml": "index.html",
  "userAskedToPublish": true
})
```

- **傳 `appId` 複用既有應用** → 連結不變。不要用 `createNewApp`（會換新連結）。
- **【用戶 2026-10-02 授權：以後自動發布】** 改版 + 本地驗證通過後**直接發布**，不必再問「要不要發布」；發布時 `userAskedToPublish:true`、`updateExistingApp:true`、`appId` 用 `<APP_ID>`。發布後仍**必須**跑線上核驗（見 §4），並在回覆中告知連結與「手機強刷／加 `?cb=時間戳`」。若用戶當輪說「先不要發」或只是隨口問，則不發。
- 發布前確認目錄內**只有 index.html**（校驗腳本、備份都要移走 `.workbuddy\tools\`、`.workbuddy\backups\`），否則入口頁可能判錯。
- 【限制】`workbuddy_sites_deploy` **按會話工作區隔離**：跨工作區無法更新別人的應用（appId 查不到、updateExistingApp 無候選、複製 applications.yaml 也無效——歸屬由服務端按 workspaceKey 校驗）。舊工作區一旦從列表刪除，其應用即失聯。

## 3.5 GitHub Pages（葉工 2026-10-17 要求：用 github.io 網址對外）

**⚠️ 公開倉庫絕不能推整站 `hk-engineer-hub/index.html`**——同一份文件含私有後台（CRM／留資／報告／提示詞），Public repo 等於把後台源碼公開。

正確做法：推**套殼頁** `.workbuddy\tools\gh_pages\index.html`（約 5KB，全屏 iframe 載入主站，網址顯示 github.io，功能全部由主站提供）：

```bash
# 默認即可：本地=套殼頁、倉庫=<GH_REPO>、token=<GITHUB_USER> 主賬號
"<USER_HOME>/.workbuddy/binaries/python/versions/3.14.3/python.exe" \
  .workbuddy/tools/gh_push.py --extra .workbuddy/tools/gh_pages/README.md
```

| 項目 | 值 |
|---|---|
| 腳本 | `.workbuddy\tools\gh_push.py`（REST API 直推，免 git/gh；SSL 用 PortableGit ca-bundle；內建 `guard_public()` 攔截整站文件） |
| 倉庫 | `<GH_REPO>`（Public，main，**fork** 自 `<GH_REPO_LEGACY>`） |
| Token | `<USER_HOME>\.workbuddy\<GITHUB_TOKEN_FILE>（<GITHUB_USER> 主賬號，fine-grained：Contents + Pages = Read and write）；舊號 token 在 <GITHUB_TOKEN_FILE>，用 `--token-file` 指定 |
| 對外網址 | **`https://<GH_PAGES_URL>/`**（Pages 已開，`status: built`，`https_enforced: true`） |
| 次要網址 | `https://<GH_PAGES_URL_LEGACY>/`（小號，Pages 未開，僅存檔） |

- **實測結論（2026-10-17）**：主站無 `X-Frame-Options`／CSP → 可被 iframe 嵌入；但雲端網關**按 Origin 白名單**：本域／null／無 Origin = 200，`github.io` = **403**（連 OPTIONS 都 403）→ **純靜態鏡像會失去 AI／表單／登入**，故只能套殼。
- 微信內：網關對 `MicroMessenger` UA 一律 403（整域名）→ 套殼頁偵測微信 UA 時不載 iframe，改顯示自製引導頁（在瀏覽器打開＋複製網址）。
- 套殼頁只放行 `pane` / `ref` 參數，`pv` 等後台參數過濾；測試腳本 `.workbuddy\tmp\test_shell.js`（8 項）。
- **【fork 坑】** `<GH_REPO>` 是 fork，上游 `<GITHUB_USER_LEGACY>` 更新**不會自動同步**。日常更新直接推 <GITHUB_USER>（默認行為），不要再往小號推，否則兩邊永久分叉。
- **【README 不要自動推】** `README.md` 由葉工手改維護（818 bytes，sha256 `8a2008d3ad30e926…`，2026-10-17 他自己刪掉「六板塊鏈接表」並修 URL）。自動流程**只推 `index.html`**，加 `--extra README.md` 會覆蓋他的編輯（曾誤覆蓋一次，已還原 commit `9f757296d0a3`）。
- **【幂等】** 腳本默認比對 git blob sha，內容一致即 `⏭ 跳過`，不產生空 commit；需要強制重推用 `--force`。
- **【何時真的要推】** 平台內容（題庫/案例/文案寫入雲數據庫）改動**不需要推送**——套殼頁 iframe 實時讀主站，github.io 立刻是新版。只有套殼頁文件本身（域址、參數白名單、微信引導頁）改動才需要推。
- **自動化**：已建 WorkBuddy automation「工程師平台 · GitHub Pages 自動同步」（每 6 小時：推 index.html + 線上 200 健康檢查，內容一致則跳過）。

## 4. 線上核驗（發布後必做）

```
verify_online_vXXX.js  →  _out_verifyXXX.txt
```

腳本要點：`fetch(LIVE + '?cb=' + Date.now())`、逐項 PASS/FAIL、輸出與本地字符數對比。

- 探針字串**必須取自實際源碼**（曾自擬 `disclaimerBelowSign` 這種不存在的標記 → 假 FAIL）。硬證據是「線上與本地字符數／位元組完全一致」那一項；其餘標記只是輔助。

- **【大坑】CDN 緩存**：發布後**立刻**直接 GET 首頁會拿到**上一版全文**（本次先看到 v1.75）。必須帶 `?cb=<timestamp>` 或 `Cache-Control: no-cache`。若仍見舊版，等 1 分鐘重跑。**沒有 cache-buster 的核驗結論一律不可信**。
- **【反向坑】cache-buster 只能加在 HTML 頁面**：`/.cloud/database/rest/*` 是 PostgREST，**未知查詢參數會被拒**（回 HTML 而非 JSON，報 `Unexpected token '<'`）。驗 REST 用 `Cache-Control: no-cache` 標頭，別加 `&cb=`。
- 判定通過的黃金標準：`live === local`（字串完全相等）+ 全部標記性檢查 PASS。

## 4.5 前端全鏈路調試（最高效的一招）

使用者報「看不到／點了沒反應」時，**先分清「代碼 bug」還是「操作時序問題」**，別急著改代碼：

**A. 雲端數據到底有沒有？**（匿名即可讀，PostgREST 風格）

```bash
python3 - <<'EOF'
import urllib.request
KEY='<PUBLISHABLE_KEY>'  # 見 §6（v1.99 起的新 key）
H={'x-wb-webapp-access-key':KEY,'Accept':'application/json','User-Agent':'Mozilla/5.0'}
u="https://<LIVE_DOMAIN>/.cloud/database/rest/site_config?select=id,data,updated_at&id=eq.cases&limit=1"
print(urllib.request.urlopen(urllib.request.Request(u,headers=H),timeout=25).read()[:400])
EOF
```
- 表 `site_config`：`id='author'`（作者配置）、`id='cases'`（成功案例展示牆）等。
- 連結 `/.cloud/*` 是**按 key 路由**，所以任一域名下查都通。

**B. 頁面 JS 真的能跑通嗎？** jsdom + 注入 node 的 fetch：

```python
# NODE_PATH=<node workspace>/node_modules 執行
new JSDOM(html, { runScripts:'dangerously', url:'https://<LEGACY_DOMAIN>/',
  beforeParse(w){ w.fetch=fetch; w.Headers=Headers; w.AbortController=AbortController; } })
# 之後可 w.ICEPublic.go('jobs')、button.dispatchEvent(new w.MouseEvent('click',{bubbles:true}))
# 再檢查 document.getElementById('pubCaseWall').innerHTML.length
```
- jsdom 自帶 `window` 但**沒有 fetch**，不補丁的話雲端 SDK 初始化失敗（`cloud===null`），會誤判成「拉不到數據」。
- jsdom 不跑佈局/CSS，只能驗證「元素是否存在、內容是否渲染、事件是否接上」。

**C. 真瀏覽器**：`agent-browser` 在 `<node workspace>/node_modules/agent-browser/bin/agent-browser.js`（v0.27，`set device "iPhone 12"` / `screenshot` / `eval`）。但**本機沙箱可能直接 SIGTERM 掉瀏覽器進程**，跑不通就別硬撐，回到 A+B。

## 4.6 本專案高頻坑（前端）

| 坑 | 症狀 | 對策 |
|---|---|---|
| **函數同名覆蓋** | 新功能上線後「某舊板塊空白」——例：v1.78 的案例牆渲染器與模擬試題的 `renderCases`（渲染 `caseBox`）**同名**，後聲明者覆蓋前者 | 新函數命名**先 Grep 全檔確認不重名**（`function <name>(`）；取名 `renderCaseWall` 這類帶業務後綴的名字 |
| **ICEPublic 導出漏項** | 函數定義好了、但 boot 裡的 `typeof window.ICEPublic.xxx === 'function'` 判斷直接跳過 → 靜默不渲染 | 新增公開渲染函數時，**同時**加進 `window.ICEPublic = { ... }` 導出清單 |
| **單次拉取 + 頁面早於數據打開** | 使用者「看不到新內容」，其實數據已存在 | ①點對應頁籤/子頁籤時 `refreshCases()` 重拉；②`visibilitychange` 回到前台重拉（手機常見）；③面板放「🔄 重新整理」按鈕；④失敗時給明確文字，別只留空狀態 |
| **手機舊瀏覽器不支援 `aspect-ratio`** | 圖片區塌陷 | 加 `min-height`；窄屏 `@media(max-width:560px)` 改單列 |
| **離線/被限制時** | 顯示「整理中」讓人以為沒發佈成功 | catch 分支把提示改成「載入失敗，請點🔄重試」 |
| **靜默空牆（最陰險）** | 手機/訪客只看見預設文案「🏆 案例整理中」，有按鈕但按了也沒反應，**沒有任何錯誤提示** | 預設文案只在 `catch` 才被替換 → 訊息一直不變 = 函式**靜默 early-return**。全檔搜 `return;` 找靜默返回點：`if (!cloud) return;` 是最常犯的（見 §4.7） |
| **SDK 掛了 → 雲端全空** | 同一個頁面，桌面正常、手機全部雲端資料（案例牆／作者資訊／留資）都拿不到 | 讀取走**同源 REST**，SDK 只做兜底；`<script src>` 加鏡像 fallback（見 §4.7） |
| **發佈者「看得見」造成誤判** | 管理員剛按完「發佈」本機看得到牆，就以為對外 OK | `saveCasesCfg` 會直接寫 `window.ICE_CASES_CFG`（本機記憶）→ 掩蓋匿名訪客讀不到雲端的事實。驗證一定要用 **jsdom 模擬「無 SDK」或無痕視窗** |
| **IIFE 包裹的 script：頂層函數不掛 window** | `onclick="pubDocReset()"` 點了**靜默沒反應**；`node --check` 全過、也無撞名——因為公開版主 script（block#2）整體包在 `(function(){ "use strict"; … })()` IIFE 裡，頂層 `function X(){}` 是 IIFE 私有，**不會變成 window 屬性**（jsdom 補 `window.pubDocReset` 測試才暴露） | 動態 onclick 要調的函數：①統一掛 `ICEPublic.*`（已有導出機制）；②或聲明後顯式 `window.X = X;`。**靜態檢查抓不到這類 bug，必須 jsdom 點擊級冒煙** |
| **雲端網關按 Referer/Origin 白名單校驗（v1.86 實錄）** | 手機案例牆「⚠️ 案例載入失敗（REST HTTP 403 / SDK [object Object]）」——帶 `Referer`/`Origin` 的請求 403 `access_denied`（"the request origin is not allowed for this client"），不帶則 200。內嵌 publishableKey 是舊 appId 時代開通的，白名單沒有現域名。**瀏覽器 fetch 自動帶 Referer 必 403；node 直打無 Referer → 桌面核驗一直 200 假象** | ①`<head>` 加 `<meta name="referrer" content="no-referrer">`（全站請求不帶 Referer，SDK/寫入亦受益；同源 GET 瀏覽器本就不帶 Origin）→ 實測 200；②REST fetch 顯式 `referrerPolicy:'no-referrer'` 雙保險；③核驗腳本要**模擬瀏覽器形態**（手機UA + Sec-Fetch-*）而非裸 node 請求。診斷工具：`tools/probe403.js`（九組請求頭矩陣）、`probe403b.js`（修復前後對照） |
| **SDK 錯誤顯示 [object Object]** | 失敗提示無法定位原因 | 統一 `fmtErr(e)`：`Error.message` / 物件 `JSON.stringify` 截 160 字元 / 兜底 `String(e)` |


## 4.7 ⚠️ 雲端 SDK 依賴是手機端最大殺手（v1.82 實錄）

**症狀**：`https://<LEGACY_DOMAIN>/` 在手機上「🏆 成功案例」只有按鈕 + 「案例整理中」，桌面卻正常。

**推理鏈**（可複用）：
1. 「案例整理中」是 **HTML 預設文案**，只有 `catch` 分支才會改寫成「載入失敗」。
2. 使用者按了「🔄 重新整理」文案**仍不變** → 不是網路錯誤，是函式**根本沒執行到 try**。
3. 全檔搜靜默返回點 → `if (!cloud) return;`：`cloud` 為 null = `window.WorkBuddyCloud` 未載入。
4. SDK 來自 `https://cdn.jsdelivr.net/npm/@tencent-ai/workbuddy-cloud-sdk@dev/lib/index.global.js` —— **手機網路／App 內置瀏覽器常攔掉它**（本機測四個鏡像全 200，但不可假設手機亦可達）。
5. `boot()` 裡的 `if (!cloud) { setView('public'); …; return; }` 正好解釋「頁面其餘部分完全正常、只有雲端資料全空」。

**根本事實**：SDK 的讀寫就是 `fetch('${endpoint}/.cloud/database/rest/<table>')` + 標頭 `x-wb-webapp-access-key`。**公開讀取完全不需要 SDK**，直接用同源 `fetch` 即可（還免 CORS、免 CDN）。

**修復三件套（v1.82 落地版）**：
1. `readSiteConfig(id)`：**同源 REST 優先 → SDK 兜底**，回傳 `{ok,data,via,err}`：
   ```js
   function restConfigUrl(id){                       // 同源時用相對路徑，避免 CORS
     const q = '/.cloud/database/rest/site_config?id=eq.' + encodeURIComponent(id) + '&select=id,data';
     let same=false; try{ same=(location.origin===CFG.endpoint)||(location.host&&CFG.endpoint.indexOf(location.host)>=0); }catch(e){}
     return same ? q : CFG.endpoint + q;
   }
   async function restGetConfig(id){
     const res = await fetch(restConfigUrl(id), { headers:{'x-wb-webapp-access-key':CFG.publishableKey,'Accept':'application/json'},
       cache:'no-store', credentials:'omit' });
     if(!res.ok) throw new Error('HTTP '+res.status);
     const arr = await res.json();
     return (Array.isArray(arr)&&arr[0]&&arr[0].data!==undefined) ? arr[0].data : null;   // null = 無此列
   }
   ```
   `ok:false` 才是真失敗（可提示重試）；`ok:true,data:null` = 尚未發佈（顯示空狀態）。**兩者必須區分**，否則又回到靜默空牆。
2. SDK `<script src>` **後面**接一段內聯鏡像兜底：`gcore.jsdelivr.net → fastly.jsdelivr.net → unpkg`（`s.onerror=next` 串聯），並把初始化抽成可重入的 `initCloud()` + `waitSDK(maxMs)` 輪詢。
3. `boot()` 重排：`if(!isLocal()){ fetchAuthorCfg(); fetchCasesCfg(); }` **先跑**（不依賴 SDK）→ `initCloud()` → 若 `!cloud` 則先 `setView('public')` 讓頁面可用，再 `await waitSDK(9000)`，補到就繼續登入分流、補不到就安全返回。**順序錯了會讓 SDK 掛掉的手機白屏 9 秒。**

**附帶加固**：讀取失敗自動重試 1 次（1.5s）；空牆 `scheduleCaseRetry()` 最多補拉 2 次（2.5s，需計數器防無限輪詢）；失敗訊息帶診斷原因（`REST HTTP xxx / SDK 未載入`）——**永遠不要讓失敗表現成預設空狀態**。

**回歸測試（`_jsdom_v182.js`，三場景必跑）**：
| 場景 | 注入 | 期望 |
|---|---|---|
| A 無 SDK + REST 正常 | 不注入 `WorkBuddyCloud`，`fetch` 走真 REST | `#pubCaseWall` 有卡片、真實標題、base64 圖、`#pubCaseEmpty` display:none、errs 空 |
| B 無 SDK + REST 掛但 SDK 可用 | fetch 對 REST 拋錯 + mock `WorkBuddyCloud` | 兜底渲染成功 |
| C 兩者皆掛 | 兩者都失敗 | 顯示「⚠️ 案例載入失敗（REST … / SDK 未載入）」，**不是**「案例整理中」 |

jsdom 的重點：`beforeParse(w){ w.fetch=…; }` 要把**相對 URL 用 `new URL(u, w.location.href)` 補成絕對位址**，否則 node fetch 會拋 `Invalid URL`；並對非 REST 的請求回 404 避免真連網。

## 5. 交付與收尾

1. 桌面留一份版本快照：`工程師平台_v1.XX_雲端版.html`（僅新增，不刪舊檔）。
2. 追加工作日志 `.workbuddy\memory\YYYY-MM-DD.md`。
3. `present_files` 交付（HTML 會自動開預覽）。
4. 回覆裡附線上連結，**緊跟一句加粗的**：前往「设置—数据管理—应用」中管理。

## 6. 雲服務端點（改域名時的重要知識）

- SDK：`@tencent-ai/workbuddy-cloud-sdk`；請求頭 `x-wb-webapp-access-key: <publishableKey>`。
- 接口基址：`${endpoint}/.cloud/{auth|database/rest|storage|llm}`。
- `publishableKey`（`wbpk_<appId>_<secret>`）**必須屬於當前應用的雲環境**。v1.99 起為 `<PUBLISHABLE_KEY>…`（新應用 `<APP_ID>` 的環境，Origin 白名單自動含 <GITHUB_USER> 域名，POST 登入恢復正常）。舊 key `<PUBLISHABLE_KEY>…` 屬舊應用環境，Origin 白名單不含任何現用域名——GET 可靠 no-referrer 繞過，**POST（登入/寫入）繞不開**，這就是「管理員登入報 the request origin is not allowed」的根因。
- 【雲環境遷移實錄 v1.99】新環境表不會自動建：先 `exec_sql migrate` 建 `site_config(id TEXT PK, data JSONB, updated_at TIMESTAMPTZ)`、`site_stats(id TEXT PK, count INT)` + GRANT anon/authenticated + RLS 全放行 policy，再從舊環境 REST 讀（無 Referer 即 200）逐行 upsert。**auth 用戶不隨遷移**，管理員要在新站重新註冊。
- ✅ 實測：新舊域名對同一 key 返回**完全相同**的響應（`401 UNAUTHENTICATED / authorization not found`）→ `/.cloud/*` 是**按 key 路由**、與域名無關。所以 `CFG.endpoint` 可以安全換成新域名，且與新站同源還能避開跨域。
- 判斷 key 是否被接受：用錯頭名會得到 `invalid_client`；頭名正確但未登入得到 `UNAUTHENTICATED / authorization not found`（=key 已被接受）。

## 7. 環境注意

- bash 的 PATH 缺 coreutils：`mkdir/ls/grep/cat` 常不可用 → 一律用 python3 腳本，或 PowerShell（但 PowerShell 工具輸出常為空，讀檔用 Read）。
- `agent-browser` CLI 位於 `<USER_HOME>\.workbuddy\binaries\node\workspace\node_modules\agent-browser\bin\agent-browser.js`（用 node 直接跑，`--help` 可列指令）。**但本機沙箱下 `open` 會被 SIGTERM 殺掉**——真瀏覽器驗證通常是走不通的，改用 §4.5 的 jsdom 全鏈路測試（實測有效）。
- 【坑】Edit 工具偶發「報成功但未落盤」：關鍵編輯後**必須用 python/Grep 複核關鍵字符串**，不符就重做（同一坑已踩多次）。
- 【坑 · v1.84 新增鐵律】**同一檔案、同一輪不要平行發多個 Edit**：一次發兩筆時只落盤一筆（報成功但內容未變），要等 Grep 才發現。改法＝同一輪只發一筆 Edit，或每筆後立刻 Grep 複核；批次改同一檔案請改為逐筆執行。
- 【檢查腳本升級建議】`check_vXXX.py` 除了原有的語法／符號／重複 id／撞名，再加兩項（v1.84 起）：
  1. **舊硬編碼字符串計數歸零**（例如重構後 `RG_HK_CV_SYSTEM + '【本次任務】'` 應為 0），用來抓「Edit 未落盤」。
  2. **隱形字符掃描**（ZWSP U+200B / ZWNJ / BOM / NBSP 應為 0；ZWJ 可能來自 emoji 屬正常）。
