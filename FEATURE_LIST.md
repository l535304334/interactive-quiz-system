# 功能列表 — 老大专用刷题系统

> 完整功能清单，按模块组织。所有功能均已实现并经过 7 轮 RC 扫描修复。
> 版本：v1.0.0

---

## 1. 答题模式

| 模式 | 快捷键 | 说明 |
|------|--------|------|
| 错题模式 | `W` | 只刷错题本中的题（`wrongSet`），答对后自动从错题本移除；支持错次筛选 |
| 复习模式（SRS） | `V` | 基于 SM-2 算法的"今日复习清单"，仅显示 `nextDate <= today` 的题目 |
| 收藏模式 | `F` | 只刷收藏的题（`favSet`），适合考前重点回顾 |
| 浏览模式 | `B` | 不判对错，纯翻阅题干+答案+解析，用于速记与精读 |
| 考试模式 | `E` | 模拟考试：自定义题数 / 时长 / 类别，一次性作答后统一交卷评分 |

### 复合模式

任意 2 种以上模式可叠加（如"错题 + 复习"），系统自动聚合各模式的题目集到统一的 `filteredQuestions`，并在复合横幅中显示各模式计数与退出按钮组。

### 模式辅助

- **顺序 / 随机洗牌**：`shuffleMode` 开关，洗牌后 `filteredQuestions` 用 Fisher-Yates 重排
- **类别筛选**：动态下拉（`sourceFilterDropdown`），列出题库中所有 source
- **错次筛选**：1+ / 2+ / 3+ / 5+ / 10+ 次错误（`wrongCountFilter`）
- **撤销最近一次作答**：`Ctrl+Z`，恢复上一题的对错状态
- **重做当前题**：`Backspace`，重置当前题作答状态
- **自动跳下一题**：答对后定时自动 advance（`autoAdvanceTimer`）

---

## 2. 题库管理

| 功能 | 入口 | 说明 |
|------|------|------|
| JSON 导入 | 数据管理 → 导入 | 支持完整 JSON（含进度+SRS）或纯题库 JSON；字段校验与补全 |
| JSON 导出 | 数据管理 → 导出 | 一键导出全部数据（题目+进度+SRS+设置）为本地 JSON 文件 |
| 从文件导入题库 | 数据管理 | 仅导入题目，不覆盖进度 |
| 自定义题 | `M` / 自定义题按钮 | `nextCustomId` 自增分配 ID，可设题干/选项/答案/类别 |
| 类别管理 | 数据管理 → 类别 | 动态增 / 删 / 改 source；空类别可删；重命名保证引用完整性 |
| 全局搜索 | `/` | 搜索题干 / 选项 / 答案 / `#ID`（精准定位），结果列表点击跳转 |
| 类别筛选下拉 | 顶部下拉 | 动态读取 `getAllSources()`，含"全部"选项 |
| 来源筛选（考试） | 考试设置 | 考试时可指定单一类别 |

### 数据安全子模块

| 功能 | 说明 |
|------|------|
| 自动快照 | 每次题目 / 错题 / 已答 / 收藏 / 错次变更触发 `triggerAutoBackup()`；最多 `MAX_SNAPSHOTS = 10` 份 |
| 快照列表 | 数据管理中查看所有快照（时间戳），可恢复 / 删除 |
| 启动恢复 | 启动时 `checkAutoRestore()` 检测最新快照，提示用户是否恢复 |
| 配额监控 | `STORAGE_QUOTA_WARN_BYTES = 10MB`，超过时 Toast 警告并展示当前用量 / 配额 |
| 旧版兼容 | 旧版 `autoBackupData` 单一快照格式仍可读取（一次性迁移到新索引系统） |
| 导入校验 | `JSON.parse` try/catch + 字段补全，格式错误时 Toast 提示，不破坏现有数据 |

---

## 3. 学习辅助

### 3.1 SM-2 间隔重复（SRS）

- **算法**：SuperMemo 2 简化版
- **入口**：`updateSRS(qid, correct)` 在每次答题后调用
- **状态字段**：`{ interval, repetition, easiness, nextDate, lastDate }`
- **调度规则**：
  - 答对：`easiness` 调整、`interval` 按重复次数递增、`nextDate = today + interval`
  - 答错：`repetition` 重置为 0、`interval` 重置为 1（明天再复习）
- **今日复习清单**：`getTodayReviewCount()` 计算 `nextDate <= today` 的题目数
- **复习按钮角标**：`updateReviewButton()` 在按钮上显示今日待复习数

### 3.2 计时系统

| 功能 | 说明 |
|------|------|
| 今日时长 | `todayStudySeconds`，每日 0 点重置（`dailyDate` 检测） |
| 累计时长 | `studySeconds`，跨日累计 |
| 计时浮层 | 底部 `.bb-timer` 实时显示今日时长，点击展开 `timerModal` |
| 计时启停 | 答题时自动计时，离开页面或切后台时停止 |
| 写盘策略 | UI 每秒更新，但 localStorage 仅在 `stopTimer` 时写（避免高频 IO） |

### 3.3 闹钟

| 功能 | 说明 |
|------|------|
| 自定义闹钟 | 可设分钟数（`alarmMinutes`，默认 45） |
| 循环模式 | `alarmLoop` 开关，到点后自动重置下一次 |
| 到点提示 | Toast + 视觉提示，可一键延后 |
| 持久化 | `alarmEnabled` / `alarmMinutes` / `alarmLoop` / `alarmNextAt` 存 localStorage |

### 3.4 每日目标

| 功能 | 说明 |
|------|------|
| 目标设置 | `openGoalModal()` 设置每日题数 `dailyGoal` |
| 进度环 | `timerModal` 内 SVG 进度环显示 `dailyCount / dailyGoal` |
| 进度条 | 顶部 `.goal-bar .fill` 显示百分比 |
| 距目标剩余 | `timerModal` 显示"距目标还剩 N 题" |
| 连续打卡 | `streakDays` 连续达标天数 |

### 3.5 学习报告

入口：`R` / 数据管理 → 学习报告

**Chart.js 4 张图表**（`renderReportCharts`）：

| 图表 | 类型 | 数据来源 |
|------|------|---------|
| 30 天答题量 | 折线图 | `dailyHistory`（每日 done 数） |
| 正确率趋势 | 折线图 | `dailyHistory`（correct / done） |
| 各科完成度 | 雷达图 | 按 source 聚合 done / total |
| 错题分布 | 柱状图 / 饼图 | 按 source 聚合 wrongSet |

**附加分析**：
- 学科正确率排行（最强 / 最弱学科）
- 错题 Top 榜（按 `wrongCountMap` 排序，前 3 名带排名色 `--c-rank-1/2/3`）
- 总答题数 / 总题库数 / 完成率

---

## 4. 数据安全

| 功能 | 说明 |
|------|------|
| XSS 防护 | `escapeHtml()` 全覆盖（58 处），所有动态 HTML 插入前转义 |
| JSON 校验 | 导入时 `try/catch` 包裹 `JSON.parse`，字段缺失时用默认值补全 |
| localStorage 异常 | `lsSet` 内 `try/catch`，配额超限时 Toast 警告 |
| 数据版本号 | 导出 JSON 含 `version: 'v1.0.0'`，便于未来迁移 |
| 快照索引 | `snapIndex` 数组保存所有快照时间戳，实体键 `snap_<ts>` |
| 快照淘汰 | 超过 `MAX_SNAPSHOTS = 10` 时自动 `evictOldestSnapshot` |
| 导出备份 | 完整导出含全部进度，可作为离线备份 |
| 旧版兼容 | `autoBackupData` 旧格式仍可读取，自动迁移到新索引 |

---

## 5. 交互体验

### 5.1 键盘模式

按 `K` 切换键盘模式（开启后按键直接选选项，无需点击）：

| 按键 | 功能 |
|------|------|
| `A`–`I` | 选中第 1–9 个选项 |
| `1`–`9` | 备选：数字键选选项 |
| `←` / `↑` | 上一题 |
| `→` / `↓` / `Space` | 下一题 |
| `Backspace` | 重做当前题 |
| `Ctrl+Z` | 撤销最近一次作答 |

### 5.2 全局快捷键（无需开启键盘模式）

| 按键 | 功能 |
|------|------|
| `W` | 切换错题模式 |
| `V` | 切换复习模式（SRS） |
| `F` | 切换收藏 / 收藏当前题 |
| `B` | 切换浏览模式 |
| `E` | 打开 / 关闭考试设置 |
| `R` | 打开学习报告 |
| `K` | 切换键盘模式 |
| `G` | 打开每日目标设置 |
| `T` | 打开计时与闹钟面板 |
| `H` | 打开使用指南 |
| `M` | 打开自定义题编辑器 |
| `X` | 移除当前题的错题标记 |
| `/` | 打开全局搜索 |
| `Esc` | 关闭任意打开的模态弹窗 |

### 5.3 暗色模式

- 跟随系统：`prefers-color-scheme` 媒体查询
- 切换机制：`applySystemTheme()` 监听 `darkMQ`，添加 / 移除 `body.dark` 类
- 完整 Token 覆盖：`body.dark` 下重定义 `--c-*` / `--sh-*` 等所有 Token

### 5.4 可访问性

| 特性 | 说明 |
|------|------|
| ARIA 弹窗 | 11 个 modal 全部带 `role="dialog"` `aria-modal="true"` |
| Esc 关闭 | 所有弹窗支持 Esc 关闭，无键盘陷阱 |
| 点击遮罩关闭 | 点击弹窗外区域关闭弹窗 |
| 焦点管理 | `focusModal()` 弹窗打开后自动聚焦首个输入框 |
| `prefers-reduced-motion` | 媒体查询降级动画（行 4832） |
| 语义 HTML | `<header>` / `<main>` / `<nav>` / `<section>` |

### 5.5 动效

| 元素 | 动画 | 属性 |
|------|------|------|
| `.toast` | toastIn | `transform` + `opacity` |
| `.q-feedback` | fadeInScale | `transform: scale` + `opacity` |
| `.modal` | scaleIn | `transform: scale` + `opacity` |
| `.mode-banner` | slideInRight | `transform: translateX` + `opacity` |
| `.info-hero-emoji` | float | `transform: translateY` |
| `.browse-banner` | shimmer | `background-position` |
| `.nav-mode-btn:hover` | icon | `transform: scale + rotate` |
| `.progress-mini .fill` | 进度条 | `width`（唯一例外：进度条必须） |

所有动画时长控制在 `0.15s–0.4s`，cubic-bezier 缓动（`--ease` / `--ease-bounce`）。

### 5.6 移动端

- 响应式：`clamp()` + `@media (max-width: 600px)` 媒体查询
- 触摸滑动切题：`touchStartX/Y` 记录起点，滑动方向触发上一题 / 下一题
- 视口适配：`<meta name="viewport" content="width=device-width, initial-scale=1.0">`

---

## 6. 视觉设计

| 模块 | 设计 |
|------|------|
| 头部导航 | 半透明玻璃态 + `backdrop-filter: blur(20px)` |
| 主色调 | 蓝紫渐变（`--c-primary` #5b6df5 → `--c-purple` #a78bfa） |
| 卡片 | 白色表面 + 3 层阴影（`--sh-1/2/3`） + 左侧色条装饰 |
| 模式横幅 | 4 色渐变（红错 / 蓝复习 / 金收藏 / 紫浏览） |
| 胶囊导航 | 4 组 `nav-group` 布局 |
| 设计 Token | 完整 CSS Custom Properties 体系（`--c-*` / `--r-*` / `--sh-*` / `--fs-*` / `--sp-*` / `--t-*` / `--ease-*`） |
| 语义色 | 红错 / 绿对 / 金收藏 / 蓝复习 / 紫浏览 |
