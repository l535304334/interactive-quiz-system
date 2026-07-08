# 题库 · 开发指南

> 题库交互式刷题页面 — 单文件 HTML 应用开发文档
> 版本：v1.0.0 · 适用文件：`[题库]交互式刷题页面.html`

> **本项目处于维护模式**：仅接受 Bug 修复、安全加固、发布阻塞类问题修复。
> 新增功能、重构、样式大改默认拒绝，除非维护者明确批准。

---

## 目录

1. [开发环境准备](#1-开发环境准备)
2. [项目代码结构](#2-项目代码结构)
3. [核心模块与关键函数](#3-核心模块与关键函数)
4. [代码规范](#4-代码规范)
5. [如何添加新功能](#5-如何添加新功能)
6. [如何添加新题目类别](#6-如何添加新题目类别)
7. [如何调试](#7-如何调试)
8. [测试方法](#8-测试方法)
9. [维护模式说明](#9-维护模式说明)

---

## 1. 开发环境准备

本项目**无构建工具、无依赖安装、无打包步骤**。所需环境极简：

### 1.1 必备工具

| 工具 | 用途 | 推荐 |
|------|------|------|
| 文本编辑器 | 编辑 HTML 文件 | VS Code / Sublime / Notepad++ |
| 现代浏览器 | 运行与调试 | Chrome 80+ / Edge 80+ |
| Node.js（可选） | 仅用于 `node --check` 语法校验 | 任意 LTS 版本 |

### 1.2 不需要的东西

- ❌ npm / yarn / pnpm（无 `package.json`）
- ❌ Webpack / Vite / Rollup（无打包）
- ❌ TypeScript 编译器（纯 JS）
- ❌ Babel（不转译，直接用浏览器原生 ES6+）
- ❌ ESLint / Prettier（项目无配置，遵循本文档规范即可）
- ❌ 任何后端服务、数据库、API

### 1.3 启动开发

1. 用编辑器打开 `[题库]交互式刷题页面.html`
2. 直接在浏览器双击打开运行
3. 修改后保存，浏览器刷新即可生效
4. 调试按 `F12` 打开开发者工具

---

## 2. 项目代码结构

整个项目是一个**单文件 HTML 应用**，约 2.7MB。结构如下：

```
[题库]交互式刷题页面.html
├── <head>
│   ├── <meta> viewport / charset / theme-color
│   ├── <title>
│   ├── <style>             ← 全部 CSS（含设计 Token、组件样式、动画、暗色模式）
│   └── <script defer>      ← Chart.js v4.4.4 CDN（仅懒加载）
│
├── <body>
│   ├── <header>            ← 顶部导航（类别筛选、模式按钮、菜单）
│   ├── <main>              ← 主内容区
│   │   ├── 题目卡片区
│   │   ├── 浏览模式区（browseContent）
│   │   ├── 考试模式区
│   │   └── 底部状态条
│   ├── 各类 <div> modal    ← 11 个模态弹窗（infoModal / guideModal / reportModal / goalModal / exportModal / searchModalOverlay / customModal / timerModal / categoryModal / examModal / encModal）
│   └── <script>            ← 全部 JavaScript
│       ├── 全局工具（Toast / Confirm / Prompt / escapeHtml）
│       ├── ALL_QUESTIONS 常量（内置题库数组）
│       ├── 全局变量声明
│       ├── localStorage 封装（lsGet / lsSet）
│       ├── 数据加载与保存（loadAll / saveXxx）
│       ├── SM-2 算法（updateSRS）
│       ├── 快照系统（saveSnapshot / restoreSnapshot）
│       ├── 模式切换（toggleXxxMode / exitAllModes）
│       ├── 答题逻辑（handleOptionClick / renderQuestion）
│       ├── 搜索（buildSearchIndex / doSearch）
│       ├── 报告（buildReport / renderReportCharts）
│       ├── 闹钟与计时（startTimer / saveAlarm）
│       ├── 类别管理（addCategory / deleteCategory / renameCategoryPrompt）
│       ├── 导入导出（exportData / importData）
│       ├── 键盘事件（document.addEventListener('keydown', ...)）
│       └── 启动初始化（loadAll / populateSourceFilter / applyFilter / startTimer / checkAutoRestore）
│
└── <script>                ← Chart.js 库代码内联（已 minified，约 200KB）
```

### 2.1 行数分布（参考）

| 区段 | 大致行号 | 说明 |
|------|---------|------|
| `<head>` + `<style>` | 1 ~ 4950 | CSS 设计 Token、组件样式、动画 |
| `<body>` HTML 结构 | 4950 ~ 5870 | DOM 结构、modal 模板 |
| 全局工具 JS | 5870 ~ 6010 | Toast/Confirm/Prompt/escapeHtml |
| `ALL_QUESTIONS` 常量 | 6008 ~ 11953 | 内置题库数据 |
| 业务逻辑 JS | 11955 ~ 14550 | 核心代码 |
| Chart.js 库 | 14551 ~ EOF | 第三方库内联 |

---

## 3. 核心模块与关键函数

### 3.1 全局工具模块

| 函数 | 行号 | 作用 |
|------|------|------|
| `showToast(type, title, text, duration)` | ~5871 | 全局 Toast 提示（success/error/warning/info） |
| `closeToast(id)` | ~5891 | 关闭指定 Toast |
| `showConfirm(title, message, options)` | ~5899 | 返回 Promise 的确认弹窗（替代 confirm） |
| `showPrompt(title, message, options)` | ~5933 | 返回 Promise 的输入弹窗（替代 prompt），支持 choices 选项模式 |
| `escapeHtml(s)` | ~6003 | HTML 转义（&, <, >, ", '）—— **所有动态 HTML 插入必须调用** |

### 3.2 localStorage 封装

| 函数 | 行号 | 作用 |
|------|------|------|
| `lsGet(k, def)` | ~11962 | 读取键值（自动 JSON.parse，失败返回 def） |
| `lsSet(k, v)` | ~11965 | 写入键值（自动 JSON.stringify，配额满时自动淘汰旧快照） |
| `checkStorageUsage()` | ~11982 | 检查存储用量，超 10MB 警告 |
| `showQuotaToast()` | ~11975 | 存储满时的 Toast 提示 |
| `evictOldestSnapshot()` | ~12245 | 淘汰最旧快照（释放空间） |

**键前缀**：所有键统一加 `tiku_v8_` 前缀（`STORAGE_PREFIX` 常量），避免与其他应用冲突。

### 3.3 数据加载与保存

| 函数 | 行号 | 作用 |
|------|------|------|
| `loadAll()` | ~12061 | 启动时加载所有数据到内存 |
| `saveQuestions()` | ~12059 | 保存题库并触发自动备份与配额检查 |
| `saveWrong()` / `saveDone()` / `saveFav()` | ~12086-88 | 保存错题/已做/收藏集合 |
| `saveWrongCount()` | ~12089 | 保存错题次数映射 |
| `saveSRS()` | ~12095 | 保存 SRS 数据 |
| `saveSources()` | ~12036 | 保存类别数组 |
| `loadDailyHistory()` / `saveDailyHistory()` | ~12228-37 | 加载/保存每日历史统计（保留 90 天） |

### 3.4 SM-2 间隔重复算法

| 函数 | 行号 | 作用 |
|------|------|------|
| `updateSRS(qid, correct)` | ~12096 | 核心算法：根据答题结果更新难度系数与下次复习日期 |
| `todayStr()` | ~12092 | 返回今日 `YYYY-MM-DD` |
| `addDays(dateStr, days)` | ~12093 | 日期加天数 |
| `getTodayReviewCount()` | ~12118 | 统计今日待复习题数 |
| `loadSRS()` / `saveSRS()` | ~12094-95 | SRS 数据加载保存 |

SRS 数据结构（每题一条）：
```js
{
  ef: 2.5,        // 难度系数 Easiness Factor（1.3 ~ 2.5+）
  interval: 0,    // 当前间隔天数
  rep: 0,         // 已成功复习次数
  next: "YYYY-MM-DD",  // 下次复习日期
  last: "YYYY-MM-DD"   // 上次复习日期
}
```

### 3.5 快照系统

| 函数 | 行号 | 作用 |
|------|------|------|
| `saveSnapshot(data)` | ~12259 | 保存快照（带 MAX_SNAPSHOTS=10 上限与自动淘汰） |
| `getSnapshotList()` | ~12283 | 获取快照列表（按时间倒序） |
| `restoreSnapshot(ts)` | ~12300 | 从快照恢复数据 |
| `deleteSnapshot(ts)` | ~12330 | 删除单个快照 |
| `triggerAutoBackup()` | ~13606 | 防抖 2 秒触发自动快照 |
| `checkAutoRestore()` | ~13616 | 启动时检测空数据并提示恢复 |

### 3.6 答题与模式切换

| 函数 | 行号 | 作用 |
|------|------|------|
| `applyFilter()` | （搜索可见） | 根据当前模式/类别过滤题目 |
| `renderQuestion()` | （搜索可见） | 渲染当前题目到卡片 |
| `handleOptionClick(q, idx, letter, container)` | （搜索可见） | 处理选项点击与判定 |
| `nextQuestion()` / `prevQuestion()` | ~12973-74 | 切题 |
| `jumpTo()` | ~12975 | 跳转到指定题号 |
| `toggleWrongMode()` | ~12977 | 切换错题模式 |
| `toggleFavMode()` | ~12447 | 切换收藏模式 |
| `toggleReviewMode()` | ~12213 | 切换复习模式 |
| `toggleBrowseMode()` | ~13066 | 切换浏览模式 |
| `exitAllModes()` | ~12205 | 退出所有模式 |
| `toggleFavorite()` | ~12455 | 收藏/取消当前题 |
| `removeSingleWrong()` | ~13023 | 当前题移出错题本 |

### 3.7 类别管理

| 函数 | 行号 | 作用 |
|------|------|------|
| `getAllSources()` | ~12022 | 获取所有类别（自动补全未登记的） |
| `addCategory()` | ~13978 | 新增类别 |
| `deleteCategory(name)` | ~13995 | 删除空类别 |
| `renameCategoryPrompt(oldName)` | ~14011 | 重命名类别（同步更新所有题目） |
| `addCustomQuestions()` | ~14047 | 添加自定义题 |
| `deleteBySource()` | ~14020 | 按类别删除题目 |
| `deleteAllCustom()` | ~14144 | 清空全部自定义题 |
| `resetProgressPrompt()` | ~14162 | 重置进度 |

### 3.8 导入导出

| 函数 | 行号 | 作用 |
|------|------|------|
| `exportData()` | ~14190 | 导出完整 JSON 备份 |
| `importData(event)` | ~14197 | 导入 JSON（支持合并/覆盖模式） |

导出数据结构：
```js
{
  wrong: [], done: [], fav: [],      // 进度集合
  wrongCount: {},                     // 错题次数
  studySec: 0, todayStudySec: 0,      // 学习时长
  dailyGoal: 0, dailyCount: 0, dailyDate: '',
  nextCustomId: 1000000,
  sources: [], questions: [], srs: {},
  version: 'v1.0.0'
}
```

### 3.9 搜索

| 函数 | 行号 | 作用 |
|------|------|------|
| `buildSearchIndex()` | ~13665 | 分批构建搜索索引（每批 500 题） |
| `openSearchModal()` / `closeSearchModal()` | ~13688-95 | 打开/关闭搜索弹窗 |
| `doSearch()` | ~13700 | 执行搜索（关键词或 #ID） |
| `searchJump(qid)` | ~13774 | 跳转到搜索结果题 |

### 3.10 报告

| 函数 | 行号 | 作用 |
|------|------|------|
| `openReportModal()` / `closeReportModal()` | ~13414-22 | 打开/关闭报告 |
| `computeSrcStats(sources)` | ~13424 | 计算各科完成度 |
| `buildReportHTML(...)` | ~13437 | 构建报告 HTML |
| `updateReportSuggestions(...)` | ~13469 | 生成学习建议 |
| `buildWrongTopHTML()` | ~13488 | 顽固错题 Top 8 |
| `renderReportCharts(...)` | ~13513 | 渲染 4 张 Chart.js 图表 |

### 3.11 计时与闹钟

| 函数 | 行号 | 作用 |
|------|------|------|
| `startTimer()` | ~13294 | 启动计时器（每秒自增） |
| `stopTimer()` | ~13341 | 停止计时器（页面隐藏时） |
| `formatTime(s)` | ~13289 | 格式化秒为 `MM:SS` |
| `updateTimerDisplay()` | ~13290 | 更新计时显示 |
| `saveAlarm()` | ~12540 | 保存闹钟设置 |
| `updateAlarmStatus()` | ~12594 | 更新闹钟状态显示 |

### 3.12 焦点管理

| 函数 | 行号 | 作用 |
|------|------|------|
| `focusModal(m)` | ~14251 | 弹窗打开时自动聚焦（无障碍） |

---

## 4. 代码规范

本项目代码规范统一记录在 `.trae/rules/` 目录下，**所有改动必须遵循**。以下是关键要点摘要：

### 4.1 规范文件索引

| 规范文件 | 适用范围 |
|---------|---------|
| `.trae/rules/00-core.md` | 核心编码原则（思考优先、YAGNI、外科手术式修改） |
| `.trae/rules/01-ai-collaboration.md` | AI 协作规则（确定性逻辑不交模型、Token 预算、暴露冲突） |
| `.trae/rules/02-engineering.md` | 工程实践（先读再写、长任务检查点、惯例优先） |
| `.trae/rules/coding-style.md` | Vanilla JS 编码规范 |
| `.trae/rules/web-coding.md` | Web 编码规范（题库特化） |
| `.trae/rules/animation-performance.md` | 动画与性能规范 |
| `.trae/rules/design-quality.md` | 设计质量标准 |
| `.trae/rules/security.md` | 安全规范（XSS 防护、localStorage 安全） |
| `.trae/rules/environment.md` | 环境维护原则 |
| `.trae/rules/git-safety.md` | Git 安全操作规范 |
| `.trae/rules/privacy.md` | 隐私与密钥安全 |

### 4.2 CSS 设计 Token 体系

项目在 `:root` 中定义了完整的设计 Token，**禁止在组件样式中硬编码色值/间距/圆角/阴影**：

```css
:root {
  /* 颜色 */
  --c-primary: #5b6df5;
  --c-primary-dark: #4453d8;
  --c-bg-surface: #ffffff;
  --c-danger: ...;
  --c-success: ...;
  /* 间距 */
  --sp-1: 4px; --sp-2: 8px; --sp-3: 12px; --sp-4: 16px; --sp-5: 24px; --sp-6: 32px;
  /* 圆角 */
  --r-sm: 8px; --r-md: 12px; --r-lg: 20px; --r-xl: 28px;
  /* 阴影 */
  --sh-3: 0 24px 60px rgba(15,23,42,0.18), 0 12px 24px rgba(15,23,42,0.08);
  /* 字号 */
  --fs-xs: 12px; --fs-sm: 13px; --fs-base: 15px; --fs-md: 17px; --fs-lg: 19px; --fs-xl: 22px; --fs-xxl: 44px;
}
```

新样式优先使用已有 Token；如需新色值，**补入 `:root` 变量区**，不写在组件内。

### 4.3 动画属性白名单

动画**只能**使用 compositor-friendly 属性：

- ✅ 允许：`transform` / `opacity` / `clip-path`（谨慎）
- ❌ 禁止：`width` / `height` / `top` / `left` / `right` / `bottom` / `margin` / `padding`

唯一例外：进度条 `.progress-mini .fill` 必须用 `width`。

`transition` 时长控制在 `0.15s ~ 0.4s`，复杂序列用 `@keyframes`。

### 4.4 函数规模

- **函数保持在 50 行以内**
- 如超标，提取子函数（参考 `renderBrowsePage` → `pageNav` 子函数模式）
- 单 HTML 文件当前体量合理（2.7MB），暂不拆分

### 4.5 `escapeHtml()` 强制使用场景

**所有动态 HTML 插入必须经过 `escapeHtml()`**，包括：

- 用户输入展示（搜索结果、自定义题预览）
- `innerHTML` 赋值前的动态内容
- `insertAdjacentHTML` 拼接的字符串
- Toast / Confirm / Prompt 中的动态文本

例外：完全静态的 HTML 字符串字面量（无变量插值）。

### 4.6 统一弹窗系统

**禁止使用原生 `alert()` / `confirm()` / `prompt()`**，统一使用：

| 场景 | 函数 | 返回 |
|------|------|------|
| 临时提示 | `showToast(type, title, text, duration)` | 无 |
| 确认操作 | `showConfirm(title, message, options)` | `Promise<boolean>` |
| 输入/选择 | `showPrompt(title, message, options)` | `Promise<string \| null>` |

`showPrompt` 支持 `choices` 选项模式（按钮组），用于多选项场景（如「重置进度」选择科目）。

### 4.7 JavaScript 编码规范

- 使用 `const` / `let`，**禁止 `var`**（历史代码中保留的 `var` 不强制改）
- 使用 `addEventListener`，**禁止 `onclick=` 属性**（已有内联 `onclick` 保留，新代码用事件代理）
- JSON 解析必须 `try-catch` 包裹
- localStorage 操作必须 `try-catch` 包裹（已由 `lsGet/lsSet` 封装）
- **禁止** `eval()`、`new Function()`、`innerHTML` 直接渲染用户输入
- **禁止** 同步 XHR

### 4.8 DOM 操作

- 批量 DOM 更新用 `DocumentFragment` 或 `requestAnimationFrame`
- 事件代理优于为每个元素绑定事件

### 4.9 Conventional Commits 提交规范

```
<type>: <description>

<optional body>
```

| Type | 说明 |
|------|------|
| `feat` | 新功能 |
| `fix` | Bug 修复 |
| `refactor` | 重构（不改变功能） |
| `docs` | 文档更新 |
| `test` | 测试相关 |
| `chore` | 构建/工具/依赖 |
| `perf` | 性能优化 |
| `ci` | CI/CD 配置 |

**破坏性操作防护**（禁止自动执行，需明确确认）：

- `git push --force` / `git push --force-with-lease`
- `git reset --hard`
- `git clean -fdx`
- `git rebase --interactive`

**分支保护**：`main` / `master` 分支上的历史改写需二次确认。

### 4.10 失败显性化

- 错误必须抛出、返回或上报，**禁止吞掉或藏于默认值**
- 批处理跳过时，跳过数量和原因要在 Toast 中展示
- 不能 100% 确认成功时，必须明确说明，**禁止默认成功**

参考实现（`addCustomQuestions` 中字段校验）：
```js
if (skipped > 0) {
  var reasonText = Object.keys(skipReasons).map(function(r){
    return r + '(' + skipReasons[r] + ')';
  }).join('、');
  showToast('success', '添加 ' + added + ' 题（跳过 ' + skipped + ' 题）', '跳过原因：' + reasonText);
}
```

---

## 5. 如何添加新功能

### 5.1 流程

1. **先读再写**：阅读相关现有代码，确认是否已有类似功能或工具函数可复用
2. **遵循惯例**：匹配现有命名风格（kebab-case 类名、camelCase 函数名）
3. **使用 Token**：新 UI 必须使用 `--c-*` / `--sp-*` / `--r-*` / `--sh-*` / `--fs-*`
4. **强制 `escapeHtml`**：任何动态 HTML 插入都要转义
5. **统一弹窗**：用 `showToast` / `showConfirm` / `showPrompt`
6. **动画合规**：只用 `transform` / `opacity`
7. **函数 50 行内**：超长则提取子函数
8. **持久化**：如需保存数据，用 `lsSet` / `lsGet`，避免直接操作 `localStorage`
9. **触发自动备份**：进度类数据保存后调用 `triggerAutoBackup()`

### 5.2 添加 UI 组件检查清单

- [ ] 定义在现有设计系统中的角色（模式/工具/信息/反馈）
- [ ] 选择语义色（而非随机色值）
- [ ] 复用已有 Token 间距/圆角/阴影
- [ ] 添加 hover/focus/active 态 + 过渡动效
- [ ] 桌面端 + 移动端响应式（`@media (max-width: 600px)`）
- [ ] ARIA 可访问性（modal 需 `role="dialog"` `aria-modal="true"`）
- [ ] `focusModal()` 焦点管理
- [ ] Esc 键关闭
- [ ] 点击遮罩关闭

### 5.3 添加键盘快捷键

在 `document.addEventListener('keydown', ...)` 中（约 14476 行）添加：

```js
else if (key === 'yourkey') { e.preventDefault(); yourFunction(); }
```

注意：
- 焦点在 input/select/textarea 时仅 `Ctrl+Z` 和 `Esc` 生效
- modal 打开时除 `Esc` 外其他被屏蔽
- 添加后必须在 USER_GUIDE.md 中同步更新快捷键表

---

## 6. 如何添加新题目类别

### 6.1 修改默认类别列表

在 `~11956` 行修改：

```js
var DEFAULT_SOURCES = ['体格检查', '共选', '实验室检查', '问诊诊断', '心电图'];
```

> 注意：此变量仅在首次启动（localStorage 无 `sources` 键）时生效。已有用户的 `sources` 已持久化，需要用户手动通过「类别管理」添加新类别，或清空 localStorage 后重新初始化。

### 6.2 运行时添加类别

用户可通过 UI「类别管理 → 新增类别」添加，无需改代码。代码层面调用：

```js
sources.push('新类别名');
saveSources();
populateSourceFilter();   // 刷新类别下拉框
populateExamSource();     // 刷新考试模式下拉框
```

### 6.3 重命名类别（保持引用完整性）

```js
// renameCategoryPrompt 实现：
for (var j = 0; j < questions.length; j++) {
  if (questions[j].source === oldName) {
    questions[j].source = newName;
  }
}
var idx = sources.indexOf(oldName);
if (idx >= 0) sources[idx] = newName;
saveSources();
saveQuestions();
```

### 6.4 删除类别

仅当该类别下**无题目**时允许删除：

```js
var n = questions.filter(function(q){ return q.source === name; }).length;
if (n > 0) { showToast('warning', '无法删除', '...'); return; }
```

---

## 7. 如何调试

### 7.1 浏览器开发者工具

按 `F12` 打开：

| 面板 | 用途 |
|------|------|
| Elements | 检查 DOM、调试 CSS |
| Console | 输出变量、运行 JS |
| Sources | 断点调试、查看源码 |
| Application → Local Storage | 直接查看/编辑存储数据 |
| Network | 查看 Chart.js CDN 加载情况 |
| Performance | 性能分析 |

### 7.2 Console 常用调试命令

```js
// 查看当前数据
console.log('题库数量:', questions.length);
console.log('错题集合:', wrongSet);
console.log('SRS 数据:', srsMap);

// 查看特定键的 localStorage
localStorage.getItem('tiku_v8_questions');

// 手动触发自动备份
triggerAutoBackup();

// 查看快照列表
getSnapshotList();

// 模拟存储满
showQuotaToast();
```

### 7.3 断点调试

在 Sources 面板找到 `[题库]交互式刷题页面.html`，搜索函数名（如 `updateSRS`），点击行号设置断点，操作触发断点即可单步调试。

### 7.4 模拟暗色模式

在 Elements 面板的 body 元素上手动添加 `class="dark"`，或在系统设置中切换深色模式。

### 7.5 模拟移动端

`F12` → 切换设备工具栏（`Ctrl+Shift+M`）→ 选择设备型号。

### 7.6 调试 localStorage

- **查看**：Application → Local Storage → `file://` 或对应域名
- **清空**：右键 → Clear（仅清空当前源）
- **导出**：在 Console 中执行：
  ```js
  JSON.stringify({
    questions: JSON.parse(localStorage.getItem('tiku_v8_questions')),
    wrong: JSON.parse(localStorage.getItem('tiku_v8_wrong'))
  })
  ```

---

## 8. 测试方法

本项目无自动化测试框架。测试方法如下：

### 8.1 语法校验（Node.js）

由于 JS 内联在 HTML 中，需先提取 `<script>` 内容到临时 `.js` 文件：

```bash
# 提取脚本部分（手动或用工具）
# 然后校验
node --check extracted_script.js
```

如无语法错误，无输出；有错误会指出行号。

### 8.2 浏览器手动测试

#### 8.2.1 核心流程测试清单

- [ ] 双击 HTML 文件能正常打开
- [ ] 内置题库自动初始化
- [ ] 类别筛选切换正确
- [ ] 5 种模式各自能独立开启/关闭
- [ ] 复合模式（任意 2-3 种组合）能取交集
- [ ] A-I 选项点击有判定反馈
- [ ] 答对/答错后状态正确（doneSet/wrongSet 更新）
- [ ] 键盘快捷键全部生效
- [ ] 搜索关键词与 #ID 都能工作
- [ ] 导出 JSON 文件能下载
- [ ] 导入 JSON 合并/覆盖都正确
- [ ] 快照保存/恢复/删除
- [ ] 学习报告 4 张图表正常渲染
- [ ] 闹钟设置/触发/循环
- [ ] 暗色模式跟随系统
- [ ] 移动端滑动切题

#### 8.2.2 边界测试

- [ ] 空题库（清空 localStorage 后启动）
- [ ] 题库为 1 道题
- [ ] 类别下有题时尝试删除（应被拒绝）
- [ ] 导入非法 JSON（应跳过并 Toast 报错）
- [ ] localStorage 满（手动塞满再操作）
- [ ] 跨日启动（修改系统时间）

#### 8.2.3 XSS 测试

- [ ] 自定义题中插入 `<script>alert(1)</script>`，应被转义不执行
- [ ] 搜索框输入 `<img src=x onerror=alert(1)>`，应被转义
- [ ] 导入 JSON 中 question 字段含 HTML，应被转义显示

### 8.3 测试有意义属性

按规范，测试必须验证**正确行为的有意义属性**，不能只测「有返回」或「不报错」。例如：

- ✅ 验证 `updateSRS(qid, true)` 后 `srsMap[qid].next` 是今日 + interval 天
- ✅ 验证 `addCustomQuestions` 跳过题目时 Toast 显示跳过数量
- ❌ 仅验证「函数能调用」

### 8.4 回归测试

修改后必须重新跑一遍核心流程清单，确认无回归。重点关注：

- 修改了 `escapeHtml` 调用 → 验证所有动态 HTML 仍正常显示
- 修改了 `lsGet/lsSet` → 验证所有数据持久化正常
- 修改了模式切换 → 验证复合模式交集正确
- 修改了 SRS 算法 → 验证复习日期计算

---

## 9. 维护模式说明

### 9.1 当前状态

本项目处于**维护模式**（v1.0.0 已发布），代码库已稳定，经过 7 轮 RC 扫描修复。

### 9.2 接受的改动类型

| 类型 | 是否接受 | 说明 |
|------|---------|------|
| Bug 修复 | ✅ 接受 | 影响功能的实际 Bug |
| 安全加固 | ✅ 接受 | XSS、注入、数据泄露类问题 |
| 发布阻塞类问题 | ✅ 优先 | 阻塞发布的严重问题 |
| 性能优化（有数据支撑） | ⚠️ 评估 | 需提供性能数据证明收益 |
| 新功能 | ❌ 默认拒绝 | 除非维护者明确批准 |
| 重构 | ❌ 默认拒绝 | 「不要重构没坏的东西」 |
| 样式大改 | ❌ 默认拒绝 | 维持设计一致性 |

### 9.3 维护原则

1. **外科手术式修改**：只触碰必须改的地方，不改无关代码/注释/格式
2. **匹配现有风格**：命名、缩进、注释风格保持一致
3. **最小修改**：能用一行解决的不写十行
4. **可回滚优先**：改动可回滚，删除文件前先搜索引用确认无依赖
5. **不主动升级依赖**：Chart.js 版本不主动升级，除非有安全问题
6. **环境不动**：不修改 PATH、不安装全局工具、不调整目录结构

### 9.4 提交前检查

- [ ] 代码通过 `node --check` 语法校验
- [ ] 浏览器手动测试核心流程通过
- [ ] 无硬编码凭据（API Key、密码、Token）
- [ ] 无个人隐私文件（实习材料、日志、.env）
- [ ] 无临时文件或构建产物
- [ ] Commit 消息符合 Conventional Commits 格式
- [ ] `git diff --cached | grep -iE "password|secret|api_key|token|apiKey|private_key|\.env"` 无输出

### 9.5 拒绝场景

遇到以下情况应**暂停并征求确认**，不自行决定：

- 代码库存在矛盾模式（如 A 文件用 snake_case，B 文件用 camelCase）
- 多个规则对同一操作有不同要求
- 修改影响范围不明确
- 不确定是否为 Bug 还是设计如此

参考：`.trae/rules/01-ai-collaboration.md` 第 7 条「暴露冲突，不要折中」。

---

## 附录：参考文档

- [USER_GUIDE.md](./USER_GUIDE.md) — 用户指南
- [DEPLOYMENT_GUIDE.md](./DEPLOYMENT_GUIDE.md) — 部署指南
- [DATABASE_DESIGN.md](./DATABASE_DESIGN.md) — 数据结构设计
- `.trae/rules/*.md` — 完整规范文件集
