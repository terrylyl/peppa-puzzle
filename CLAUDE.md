# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目性质

零依赖的静态 PWA 拼图小游戏（中文界面，面向小朋友），部署在 GitHub Pages。**没有构建系统、包管理器、测试框架或 lint 配置**——不要引入 npm/bundler，也不要把代码拆成模块文件，除非用户明确要求。

全部逻辑都在单文件 [index.html](index.html)（约 1820 行）里：

| 区段 | 行号 | 内容 |
| --- | --- | --- |
| `<style>` | 15–810 | 全部 CSS，含 CSS 变量主题、响应式断点（860px / 500px）、横屏矮视口断点（`orientation: landscape and max-height: 560px`）、`prefers-reduced-motion` / `prefers-contrast` / `prefers-reduced-transparency` 适配 |
| `<body>` | 812–894 | 静态 DOM 骨架（棋盘容器、侧栏面板、参考图 `<dialog>`、彩纸层） |
| `<script>` | 896–1816 | 全部游戏逻辑，无模块、无框架、纯全局函数 + 顶层可变状态 |

## 本地运行与验证

必须用 HTTP 服务，不能直接 `file://` 打开（service worker 和 manifest 会失效）：

```bash
python -m http.server 8000
```

**本地验证前先把 service worker 注销掉**，否则它会 cache-first 地把上一版 `index.html` 喂给你，改动看起来"没生效"（实测踩过）：

```js
navigator.serviceWorker.getRegistrations().then(rs => rs.forEach(r => r.unregister()));
caches.keys().then(ks => ks.forEach(k => caches.delete(k)));
```

验证靠手动在浏览器里操作，没有自动化测试。改动后至少覆盖：点击选中→交换、按住拖拽→落格、"帮我放一块"、换关/选关、导入与删除图片、窗口缩放（棋盘尺寸重算）、系统开启"减少动态效果"后的降级路径。

## 核心架构

### 双向索引的棋盘状态

同时维护两个数组，任何交换后都必须让它们保持一致：

- `order[slot] = tileNumber` —— 每个格子里放的是哪块
- `tileSlots[tileNumber] = slot` —— 每块现在在哪个格子

`tileSlots` 由 `order` 派生（`order.forEach((t, slot) => { tileSlots[t] = slot; })`）。"放对"的判定统一是 `tileNumber === slot`。DOM 里的九个 `.tile` 元素**创建后永不重排**，`tileElements[tileNumber]` 索引恒定，位置完全靠 transform 表达。

### 弹簧动画系统

`motions[tileNumber]` 保存 `{x, y, targetX, targetY, vx, vy, springing, dragging}`，`tickSprings()` 是单个 rAF 循环（`stiffness: 360`，临界阻尼 ×0.94），把所有活动块推向目标并写入 `translate3d`。要点：

- 只有一个 `springFrame`，所有块共用；不要为单块另起 rAF。
- 动画结束后的"反馈"（夸奖语、音效、`correct-flash`、通关彩纸）不是立即执行的，而是通过 `pendingFeedback` 挂起，等参与本次移动的所有块都停稳后由 `maybeFinishFeedback()` → `finishMoveFeedback()` 触发。新增交互路径时必须走 `animateTilesToSlots()`，否则反馈会丢失。
- `isReducedMotion()` 为真时所有路径都要有"直接跳到目标 + 立刻触发反馈"的分支，现有函数都成对写了，照抄这个模式。

### 图片渲染方式

不切图。九块共用同一张背景图：`.tile` 设 `background-size: 300% 300%`，用 `background-position: {col*50}% {row*50}%` 取自己那一格。**这意味着图片会被拉伸填满棋盘**——棋盘比例必须等于图片比例，否则画面变形（不是裁剪）。当前关卡图片通过 `--puzzle-image` 注入 `<html>`；`setLevelImage()` 的探测 `Image` 读到真实宽高后同时写入两个变量：`--puzzle-ratio`（`W / H`，给 `aspect-ratio` 用）和 `--puzzle-ratio-num`（数值，给 `calc()` 乘法用，横屏下按可用高度反推棋盘宽度）。改图后必须 `syncBoardSize()` 重新量尺寸并重算所有 tile 的目标坐标。

### 关卡与导入图片

`levels = [...baseLevels, ...importedLevels]`，`baseLevels` 是 `peppa1.jpg`–`peppa9.jpg`，`importedLevels` 来自 localStorage（key `peppa-puzzle-imported-images-v1`），导入时经 canvas 压缩到最长边 1400px 的 JPEG dataURL。索引 `>= baseLevels.length` 即导入图（`isImportedLevel()`）。删除导入图会改变后续索引，`completedLevels`（存的是关卡下标）必须同步平移，见 `shiftCompletedAfterDelete()` / `keepOnlyBuiltInCompleted()`。所有 localStorage 访问都包 try/catch，配额不足时降级为"本次会话可用"并在 `importNoteEl` 提示。

### 指针交互

一次只跟踪一个 `pointerSession`。移动距离 ≤10px 视为点击（走 `selectOrSwap()` 的两步选中交换），超过则转为拖拽并按抬手时的中心点落格；松手速度乘 0.14 作为弹簧初速。键盘路径独立：方向键在格子间移动焦点（`tabIndex` roving），Enter/Space 走同一个 `selectOrSwap()`。

## 修改时的固定动作

- **改了 `index.html` 就要提升 [sw.js](sw.js) 里的 `CACHE_NAME`**（当前 `peppa-puzzle-v17`）。service worker 是 cache-first，不换 key 的话回访用户永远拿旧页面。
- 新增静态资源要同时加进 `sw.js` 的 `ASSETS`；`cache.addAll()` 是全有或全无，任何一项 404 会让整个安装失败。
- 改交互时同步维护无障碍属性：`updateTileAccessibility()` 负责 `aria-pressed`、中文 `aria-label`、roving `tabIndex` 和 `selected`/`correct` class。
- 界面文案全部是中文口语化、面向儿童的短句，新增提示照此风格。
- 保持现有代码风格：`const`/`let`、无分号省略、函数声明式、2 空格缩进、事件监听集中在脚本末尾、初始化序列 `createBoard() → updateSoundButton() → startLevel(0) → updateImportMessage()`。

## 部署

- 远端 `origin`：https://github.com/terrylyl/peppa-puzzle.git
- GitHub Pages（legacy 模式）从 **`main` 分支根目录**直接服务，线上地址 https://terrylyl.github.io/peppa-puzzle/ 。**改动只有进了 `main` 才会上线**，推到 `pages-fix` 或其他分支不会触发发布。`.nojekyll` 必须保留（否则 Jekyll 会处理这些文件）。
- 分支现状：`main` 是发布分支；`pages-fix` 是工作分支，两者已分叉——同一处修改在两边存在哈希不同的等价提交（如 hover 修复在 `main` 是 `7b0555e`、在 `pages-fix` 是 `b3df61c`），历史上是逐次 cherry-pick 到 `main` 发布的。合并前先确认改动是否已在 `main` 上有对应提交，避免重复。

## 仓库现状注意事项

- `icons/icon-192.png`、`icon-512.png` 曾长期未提交、线上 404，导致 `cache.addAll()` 整体失败、service worker 从未安装成功。已于 2026-09-04 提交修复，线上实测 SW 现已 `activated` 且缓存 14 个条目。留个教训：`cache.addAll()` 全有或全无，任何一个 `ASSETS` 条目 404 都会让整个 SW 装不上，而 `caches.open()` 仍会把空缓存桶建出来——**"缓存桶存在"不代表 SW 装好了**，要看 `getRegistrations()`。
- 根目录的 `peppa-puzzle-pwa-v*.zip`、`preview-v*.png` 是历史打包/截图产物，未跟踪，不属于运行时资源；`peppa-puzzle-pwa.zip` 仍被 `main` 跟踪但在本地已删除。
- `core.autocrlf=true`：工作区的 `index.html` 是全 CRLF，仓库里的 blob 是 LF，切分支时 git 会重写工作区文件。所以直接 `diff` 本地文件和线上响应会全文件不一致——比对部署状态前先 `tr -d ''`。
