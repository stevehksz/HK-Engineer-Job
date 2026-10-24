---
name: web-single-file-fusion
category: capability
version: "1.0.0"
description: 把独立单文件 HTML 工具/模板融合进多页面单文件平台（如工程師平台）的完整方法论。触发词：融合模板、融进工作台/平台、merge template into platform、新板块/新页面融合。涵盖 CSS 作用域前缀化、JS 撞名改名空间、顶层注入 vs DOMContentLoaded 陷阱、String.replace $ 特殊模式、split('}') 注释并入选择器、jsdom 测试要点（scrollIntoView 桩、简繁体断言）。
---

# 单文件 HTML 工具 → 多页面平台融合方法论

适用：源工具是独立单文件 HTML（自带 style/script/body），目标是把它变成平台的一个 `<section class="page" id="page-X">` 新板块。实践案例：工程師平台 v1.30 融合「叶工公众号写作 v10」。

## 一、融合流程（写一次性 Node 脚本执行，防重入）

1. **提取源三段**：`<style>`…`</style>`、`<script>`…`</script>`、body 业务 wrap（从 `<div class="wrap">` 到底部工具条注释**之前**——topbar/bottom-bar 这类固定定位元素不注入，功能按钮改页内操作条如 `.gzh-actions`）。
2. **CSS 前缀化**：逐条规则加 `#page-X ` 前缀；多选择器按逗号分段各自加；丢弃全局规则（`*`、`body`、`html,body`、平台已有的 `#toast`、fixed 定位条）。
3. **JS 撞名改名空间**：先精确列出源工具**列 0 顶层声明**（function/const/let/var，函数内部的 d/e/t/r 等局部变量不算），与平台顶层声明对比；只有真撞名才改（如 `state`→`xState`、删源版 `toast`/`escapeHtml` 定义改用平台版）。改名必须覆盖全部引用形态：点号引用、简写属性（`state,`→`state:xState,`）、`Object.assign`、**动态键 `state[key]`**（点号正则覆盖不到，需 split/join 精确替换）、模板字符串/innerHTML 内嵌。改完逐行校验残留（白名单合法形态）。
4. **JS 注入位置：必须顶层，严禁 DOMContentLoaded 包裹**。平台 script 在 body 末尾，DOM 已就绪；顶层 `function` 声明成为 window 属性（内联 `onclick="fn()"` 可用），顶层 `const` 在全局词法环境（`window.eval` 与内联 handler 均可见）。包进回调 = 全部函数作用域 = 内联 onclick 全失效 + eval 取不到（v1.30 实锤 bug）。
5. **注入锚点**：CSS 注入平台第一个 `</style>` 前；section 注入首个 page section 前；nav 条目/PAGE_META 按平台惯例锚点；JS 注入平台 `document.addEventListener('DOMContentLoaded', init);` 之前（顶层）。
6. **所有 replace 一律函数式**：`p.replace(anchor, () => str)`。字符串式替换时替换串里的 `$'` `$&` `` $` `` `$1` 会被解释为特殊模式破坏文件（模板 JS 常含 `$1`；仅当 pattern 无捕获组时才侥幸字面保留，不能赌）。

## 二、已知坑（v1.30/v1.31 实锤）

- **平台文件可能是 CRLF 行尾（且融合注入块是 LF，全文件混合）**：补丁锚点跨行必须 `\r?\n` 容错，纯 `\n` 锚点会静默失配（误报"不唯一"）；多 `</div>` 等重复结构用 lookahead（`(?=[\s\S]{0,120}?<div ... id="xxx">)`）锁定唯一位置。
- **String.replace 函数式替换的形参=捕获组**：形参个数必须与捕获组数一一对应（其后才是 offset/string）；写了不存在的"第三组"时该参数收到的是匹配偏移量数字，拼进代码变 `397964xxx` 语法错误。
- **非全局 `String.match` 不能当匹配次数用**：返回 `[整体, 捕获组]` 长度恒 2；计数必须加 `g` 标志。
- **长模块注入**：模块代码写独立 `.txt` 文件，补丁脚本 `fs.readFileSync` 读入 + `new Function(code)` 编译级语法校验（不执行），避免 JS-in-JS 转义地狱。
- **对已注入文件打补丁必须防重入**：以新版本特有标识符（存储键/BUILD_TAG）为哨兵，已存在即退出。
- **split('}') 分段会把段首 CSS 注释并入选择器** → 生成 `#page-X /* 注释 */ sel {`（语法有效但丑）；两个注释连写+选择器时，replace 扫原串不重扫插入文本会残留级联 → 用 `do{prev=p;p=p.replace(...)}while(p!==prev)` 迭代到收敛。
- 平台自身就有 `<header class="topbar">`：测试断言"模板 topbar 未注入"要精确匹配 `<div class="topbar">`，不能裸匹配 `class="topbar"`。
- 源模板正文多为**简体**（锚点正则、前缀剥离正则、文案标签都只认简体）：测试输入与断言必须用与源一致的简繁体，否则假失败。
- 源工具保留原有 localStorage 草稿键可实现旧数据无缝延续；800ms 轮询类逻辑原样保留。

## 三、jsdom 测试要点

- `beforeParse` 打桩：`w.confirm`、`w.alert`、**`w.HTMLElement.prototype.scrollIntoView = function(){}`**（jsdom 未实现，genPrompt 等常用）。
- `url:'http://localhost/'` 启用 localStorage；顶层 `let/const` 须经 `W.eval()` 访问且 eval 取值可能抛 ReferenceError → 断言里 try/catch。
- 静态断言（BUILD_TAG/section/nav/锚点/CSS 前缀/无重复注入）+ 动态断言（挂载函数 `typeof W[f]==='function'`、平台 state 未被覆盖、撞名隔离、E2E 流程、页面切换回归）。
- 防重复注入：注入脚本含防重入检查（锚点已存在即退出）；测试前 grep 计数关键标识符应 =1。

## 三b、视觉需求的自检（截图目视）

涉及背景/配色/排版等**视觉需求**时，jsdom 测不出来，必须截图目视。零安装方案（agent-browser npm 安装要数分钟，先用这个）：

```bash
chrome.exe --headless=new --disable-gpu --screenshot=out.png --window-size=1600,900 --hide-scrollbars --virtual-time-budget=4000 "file:///页面路径"
```

- 系统路径：`C:/Program Files/Google/Chrome/Application/chrome.exe`（或 Edge `msedge.exe` 同参数）。
- **`--virtual-time-budget=4000` 必加**：等 JS 渲染完再截，否则截图空白会误判页面坏了。
- 截出的 PNG 用 Read 工具直接查看自查（Read 可渲染 PNG/JPG；读 `.svg` 只返回源码，SVG 造型必须走浏览器截图验证）。
- 自查点：页面是否完整渲染、注入内容形态是否符合预期、顺带留意硬编码残留（v1.32 曾借此发现侧栏品牌徽章停留 v1.12）。

## 四、收尾惯例

测试全绿 → `mv` 桌面文件版本号改名 → 清理测试目录 → 追加日志到 `.workbuddy/memory/YYYY-MM-DD.md` → present_files 交付 + 提醒 Ctrl+Shift+R 强刷。
