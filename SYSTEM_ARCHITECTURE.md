# 系统架构 — 老大专用刷题系统

> 单文件 HTML 应用，无后端、无构建工具、无打包工具
> 主文件：`[题库]交互式刷题页面.html`

---

## 1. 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                  浏览器（用户终端）                          │
│  ┌───────────────────────────────────────────────────────┐  │
│  │        单文件 HTML（2.7MB，IIFE 作用域隔离）           │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌────────┐  │  │
│  │  │  UI 层  │←→│ 业务逻辑│←→│ 数据层  │←→│ 工具层 │  │  │
│  │  │(render*)│  │(toggle* │  │(lsGet/  │  │(escape │  │  │
│  │  │         │  │ start*  │  │ lsSet/  │  │ Html/  │  │  │
│  │  │         │  │ save*)  │  │ save*)  │  │ toast) │  │  │
│  │  └─────────┘  └─────────┘  └────┬────┘  └────────┘  │  │
│  │                                  │                   │  │
│  │            Chart.js v4.4.4（内联）│                   │  │
│  │            └──────懒加载────────┘                   │  │
│  └──────────────────────────────────┬────────────────────┘  │
│                                     │                        │
│                          ┌──────────▼──────────┐             │
│                          │   localStorage      │             │
│                          │   tiku_v8_*         │             │
│                          │   （唯一持久层）    │             │
│                          └─────────────────────┘             │
└─────────────────────────────────────────────────────────────┘
                              │
                  ┌───────────┴───────────┐
                  │  本地 JSON 文件（可选） │
                  │  导出 / 导入           │
                  └─────────────────────────┘
```

**架构特征：**
- **无后端**：所有数据存于浏览器 `localStorage`，不联网、不上传
- **单文件**：HTML + CSS + JS + Chart.js 全部内联在一个 `.html` 文件
- **IIFE 隔离**：主逻辑用 `(function(){...})()` 包裹，避免污染全局
- **唯一外部依赖**：Chart.js v4.4.4（已内联，不依赖 CDN 加载）
- **零构建**：无 webpack / vite / babel，源码即运行时

---

## 2. 模块划分

按职责分为 4 层（注：物理上都在同一文件内，仅逻辑分层）。

### 2.1 数据层（Data Layer）

**职责**：与 `localStorage` 交互的读写封装、状态持久化、快照管理。

| 模块 | 关键符号 | 行号区间（近似） |
|------|---------|------------------|
| 存储封装 | `STORAGE_PREFIX`、`lsGet`、`lsSet` | 11955–11973 |
| 配额监控 | `STORAGE_QUOTA_WARN_BYTES`、`checkStorageUsage`、`showQuotaToast` | 11963–11992 |
| 状态变量 | `wrongSet` / `doneSet` / `favSet` / `wrongCountMap` / `questions` / `srsMap` / `dailyHistory` | 11993–12011 |
| 题库读写 | `saveQuestions`、`loadAll`、`rebuildQuestionMap`、`getQuestionById` | 12013–12081 |
| 进度持久化 | `saveWrong` / `saveDone` / `saveFav` / `saveWrongCount` / `saveAllProgress` | 12086–12090 |
| SRS 持久化 | `loadSRS`、`saveSRS`、`updateSRS` | 12094–12116 |
| 每日统计 | `loadDailyHistory`、`saveDailyHistory`、`recordDailyStat` | 12228–12243 |
| 快照系统 | `saveSnapshot`、`restoreSnapshot`、`getSnapshotList`、`deleteSnapshot`、`evictOldestSnapshot`、`triggerAutoBackup`、`checkAutoRestore` | 12245–12344、13606–13686 |

**存储键命名空间**：所有键以 `tiku_v8_` 前缀存储（`STORAGE_PREFIX = 'tiku_v8_'`）。

### 2.2 业务逻辑层（Business Logic Layer）

**职责**：模式切换、答题判定、计时、闹钟、目标、考试流程、报告生成。

| 模块 | 关键函数 |
|------|---------|
| 模式管理 | `toggleWrongMode` / `toggleReviewMode` / `toggleFavMode` / `toggleBrowseMode` / `exitAllModes` / `updateModeBanners` / `updateAllModeCounts` |
| 答题处理 | `handleOptionClick` / `nextQuestion` / `prevQuestion` / `redoQuestion` / `undoLastAnswer` / `removeSingleWrong` / `toggleFavorite` |
| 洗牌 | `shuffleArray` / `toggleShuffle` / `updateShuffleBtn` |
| 计时 | `startTimer` / `stopTimer` / `updateTimerDisplay` / `openTimerModal` / `closeTimerModal` |
| 闹钟 | `saveAlarm` / 闹钟触发与提示 |
| 目标 | `openGoalModal` / `closeGoalModal` / `saveGoal` |
| 考试 | `startExam` / `renderExam` / `startExamTimer` / `openExamModal` / `closeExamModal` / `returnToNormal` / `submitExam` |
| 报告 | `buildReport` / `renderReportCharts` / `buildReportHTML` / `openReportModal` |
| SRS 调度 | `updateSRS` / `getTodayReviewCount` / `updateReviewButton` |
| 自定义题 | `openCustomModal` / `closeCustomModal` / `saveCustomQuestion` |
| 类别管理 | `renderCategoryManageList` / `addSource` / `renameSource` / `deleteSource` |
| 搜索 | `openSearchModal` / `closeSearchModal` / `searchJump` / `performSearch` |
| 导入导出 | `exportData` / `importData` / `importQuestionsFromFile` |
| 主题 | `applySystemTheme`（暗色模式跟随系统） |
| 来源筛选 | `populateSourceFilter` / `renderSourceFilterDropdown` / `toggleSourceFilter` / `selectSourceFilter` |

### 2.3 UI 层（Presentation Layer）

**职责**：DOM 渲染、视图更新、模态弹窗、Toast 反馈。

| 模块 | 关键函数 |
|------|---------|
| 题目渲染 | `renderQuestion` / `renderEmptyState` / `renderOptions` |
| 浏览页 | `renderBrowsePage` / 浏览分页逻辑 |
| 考试页 | `renderExam` / `renderExamResult` |
| 模态弹窗 | 11 个 modal（见下文） |
| Toast | `showToast` / `closeToast` |
| 自定义弹窗 | `showConfirm` / `showPrompt` / `focusModal` |
| 反馈卡 | 答题后 `.q-feedback` 卡片显示对错与解析 |
| 模式横幅 | `updateModeBanners` 渲染 `.mode-banner` |
| 计时浮层 | `updateTimerDisplay` 渲染 `.bb-timer` |
| 目标进度条 | `updateGoalBar` 渲染 `.goal-bar .fill` |

**11 个模态弹窗**（均带 `role="dialog"` `aria-modal="true"`）：

| ID | 用途 |
|----|------|
| `confirmOverlay` | 通用确认框 |
| `promptOverlay` | 通用输入框 |
| `searchModalOverlay` | 全局搜索 |
| `examModal` | 考试设置 / 作答 |
| `encModal` | 知识点速记卡（Encyclopedia） |
| `infoModal` | 关于信息 |
| `guideModal` | 使用指南 |
| `reportModal` | 学习报告 |
| `goalModal` | 每日目标设置 |
| `timerModal` | 计时与闹钟面板 |
| `exportModal` | 数据导入导出 |

### 2.4 工具层（Utility Layer）

**职责**：纯函数工具，无副作用。

| 函数 | 用途 |
|------|------|
| `escapeHtml(s)` | XSS 防护，所有动态 HTML 插入前调用（58 处） |
| `showToast(type, title, text, duration)` | 全局提示（success/error/warning/info） |
| `showConfirm(title, message, options)` | Promise 化确认框 |
| `showPrompt(title, message, options)` | Promise 化输入框（支持选项 chips） |
| `focusModal(m)` | 弹窗打开后自动聚焦首个输入框 |
| `todayStr()` / `addDays(dateStr, days)` | 日期格式化（`YYYY-MM-DD`） |
| `shuffleArray(arr)` | Fisher-Yates 洗牌 |
| `formatTime(seconds)` | 秒数 → `HH:MM:SS` |

---

## 3. 关键数据流

### 3.1 用户答题数据流（最核心）

```
用户点击选项
    ↓
handleOptionClick(q, idx, letter, container)
    ↓
判断对错 → 更新 wrongSet/doneSet/favSet/wrongCountMap
    ↓
updateSRS(qid, correct)  ← SM-2 算法更新复习间隔
    ↓
recordDailyStat('correct' | 'wrong')  ← 更新 dailyHistory
    ↓
saveAllProgress()  ← 批量持久化
    ├── lsSet('wrong', [...])
    ├── lsSet('done', [...])
    ├── lsSet('fav', [...])
    ├── lsSet('wrongCount', {...})
    └── lsSet('srs', {...})
    ↓
triggerAutoBackup()  ← 触发自动快照
    ↓
renderQuestion()  ← 渲染下一题 + 反馈卡
    ↓
（可选）autoAdvanceTimer → 自动跳下一题
```

### 3.2 模式切换数据流

```
用户点击模式按钮 / 按快捷键
    ↓
toggleWrongMode() / toggleReviewMode() / ...
    ↓
更新 wrongMode / reviewMode / favMode / browseMode 状态变量
    ↓
updateModeBanners()  ← 渲染模式横幅（含复合模式聚合）
updateAllModeCounts()  ← 更新按钮角标
    ↓
applyFilter()  ← 重新计算 filteredQuestions
    ↓
currentIndex = 0
renderQuestion()  ← 渲染新题目流的第 1 题
```

### 3.3 启动数据流

```
页面加载 (DOMContentLoaded)
    ↓
loadAll()  ← 读取所有 localStorage 数据到运行时变量
    ├── sources = loadSources() || DEFAULT_SOURCES
    ├── questions = lsGet('questions') || ALL_QUESTIONS
    ├── wrongSet = new Set(lsGet('wrong'))
    ├── doneSet / favSet / wrongCountMap / srsMap / dailyHistory
    ├── studySeconds / todayStudySeconds / dailyGoal / dailyCount
    └── shuffleMode / alarmEnabled / alarmMinutes / alarmLoop
    ↓
rebuildQuestionMap()  ← 建立 id→question 查找缓存
    ↓
checkAutoRestore()  ← 检测最新快照，提示是否恢复
    ↓
applySystemTheme()  ← 应用暗色模式
    ↓
populateSourceFilter() / renderSourceFilterDropdown()
    ↓
applyFilter() → renderQuestion()  ← 首次渲染
    ↓
startTimer()  ← 启动计时器
```

### 3.4 数据导入流

```
用户选择 JSON 文件
    ↓
importData(event) 或 importQuestionsFromFile(e)
    ↓
FileReader 读取 → JSON.parse（try/catch）
    ↓
字段校验与补全 → 合并到运行时变量
    ↓
saveAllProgress() / saveQuestions() / saveSources()
    ↓
rebuildQuestionMap()
    ↓
applyFilter() → renderQuestion()
    ↓
showToast('success', '导入成功', ...)
```

---

## 4. 状态管理方式

**无框架、无状态库**。状态由两类载体管理：

### 4.1 运行时全局变量（IIFE 内）

所有运行时状态以模块级 `var` 声明，IIFE 包裹避免污染全局：

```js
var questions = [];           // 题库
var wrongSet = new Set();     // 错题集合
var doneSet = new Set();      // 已答集合
var favSet = new Set();       // 收藏集合
var wrongCountMap = {};       // 错次统计
var srsMap = {};              // SM-2 状态
var filteredQuestions = [];   // 当前题目流
var currentIndex = 0;         // 当前题号
var wrongMode = false, favMode = false, keyboardMode = false, shuffleMode = false, reviewMode = false, browseMode = false;
var studySeconds = 0, todayStudySeconds = 0;
var dailyGoal = 0, dailyCount = 0, dailyDate = '';
var isExamMode = false, examQuestions = [], examAnswers = {};
// ... 等
```

### 4.2 localStorage 持久化

写操作显式调用 `lsSet(key, value)`，不自动双向绑定：

```js
function lsSet(k, v) {
  try {
    localStorage.setItem(STORAGE_PREFIX + k, JSON.stringify(v));
  } catch(e) { /* QuotaExceededError 处理 */ }
}
```

**持久化策略**：
- 关键状态变更（错题/已答/收藏/错次/题目）立即写盘 + 触发快照
- 计时器每秒更新 UI，但只在 `stopTimer` 时写盘（避免高频 IO）
- 每日统计 `dailyHistory` 在每次答题后写盘

### 4.3 状态一致性

- 写入 localStorage 后，对应内存变量同步更新（单向数据流）
- `rebuildQuestionMap()` 在题库变更后重建查找缓存，保证 `getQuestionById` 高效
- `applyFilter()` 在模式 / 类别 / 错次筛选变更后重新计算 `filteredQuestions`

---

## 5. 依赖关系

### 5.1 运行时依赖

| 依赖 | 版本 | 用途 | 加载方式 |
|------|------|------|---------|
| Chart.js | v4.4.4 | 学习报告 4 图表 | 内联（行 14551–14573），首次打开报告时懒加载实例化 |
| 浏览器原生 API | — | localStorage、FileReader、matchMedia、requestAnimationFrame、Set | 全平台支持 |

**无其他外部依赖**。无 npm、无 CDN（Chart.js 已内联）、无字体加载。

### 5.2 模块内依赖关系

```
UI 层 ──调用──→ 业务逻辑层 ──调用──→ 数据层
  │                  │                 │
  │                  └──调用──→ 工具层 ←┘
  │
  └──直接调用──→ 工具层（escapeHtml、showToast）
```

业务逻辑层不反向调用 UI 层（无回调地狱），UI 层是最终消费者。

---

## 6. 扩展点

### 6.1 数据格式扩展（向前兼容）

`exportData` 输出的 JSON 包含 `version: 'v1.0.0'` 字段。新增字段时：
- 新代码读取旧数据：用 `lsGet(key, defaultValue)` 提供回退值
- 旧代码读取新数据：忽略未知字段（JSON.parse 不报错）

### 6.2 类别（source）扩展

`sources` 数组运行时可变，`getAllSources()` 会扫描题库中所有 `q.source` 自动补全。新增学科只需：
- 通过"类别管理"UI 添加
- 或导入含新 source 的题目，系统自动识别

### 6.3 自定义题扩展

`nextCustomId`（默认 1000000）自增分配，与内置题库 ID 空间隔离（内置题 ID < 1000000）。

### 6.4 模式扩展

新增答题模式需修改：
1. 新增状态变量（如 `xxxMode`）
2. 新增 `toggleXxxMode()` 函数
3. 在 `updateModeBanners` / `updateAllModeCounts` 注册
4. 在 `applyFilter` 加入过滤逻辑
5. 在键盘事件处理中绑定快捷键

### 6.5 不建议的扩展

- **拆分多文件**：违反单文件设计哲学，破坏"双击即用"
- **引入框架**：React/Vue 对 2.7MB 单文件过度，且无构建工具链
- **引入后端**：破坏"零部署、零依赖"的核心价值主张
