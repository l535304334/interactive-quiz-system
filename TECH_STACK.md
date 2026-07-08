# 技术栈 — 老大专用刷题系统

> 单文件 HTML 应用，无构建工具、无框架、无打包工具
> 主文件：`[题库]交互式刷题页面.html`（约 2.7MB）

---

## 1. 核心技术

| 技术 | 版本 / 规格 | 用途 |
|------|------------|------|
| HTML5 | `<!DOCTYPE html>` | 文档结构 |
| CSS3 | 含 CSS Custom Properties、`backdrop-filter`、`@media` 查询 | 样式与设计 Token |
| JavaScript | ES5+ 兼容语法（`var` / `function` / `Set` / `Array.from`） | 全部业务逻辑 |
| 语义 HTML | `<header>` / `<main>` / `<nav>` / `<section>` | 可访问性 |

### 编码风格

- **变量声明**：`var` 为主（兼容性优先，避免 `let/const` 在老旧环境下的边界问题）
- **作用域隔离**：IIFE `(function(){...})()` 包裹主逻辑
- **事件绑定**：`addEventListener`（不使用 `onclick=` 属性，仅个别历史 inline 调用保留）
- **错误处理**：`try/catch` 包裹所有 `JSON.parse` 与 `localStorage` 操作
- **DOM 操作**：批量更新用 `DocumentFragment` / `requestAnimationFrame`

---

## 2. 外部依赖

| 依赖 | 版本 | 用途 | 加载方式 |
|------|------|------|---------|
| Chart.js | **v4.4.4** | 学习报告 4 张图表（折线 / 雷达 / 柱状 / 饼图） | 内联在 `<script>` 标签中（行 14551–14573） |

### Chart.js 加载策略

- **内联而非 CDN**：避免离线场景下报告功能失效；CDN 在网络受限环境（如考场、内网）会阻塞页面
- **懒加载实例化**：Chart.js 源码随页面加载，但 `new Chart(...)` 实例只在用户首次打开"学习报告"时创建，避免首屏开销
- **暗色适配**：`renderReportCharts(srcStats, palette, isDark)` 根据当前主题切换图表配色

### 无其他依赖

- 无 npm / yarn / pnpm
- 无 CDN 资源（除 Chart.js 已内联）
- 无字体加载（使用系统字体栈）
- 无图片资源（全部 emoji + CSS 绘制）

---

## 3. 数据存储

### 3.1 持久化方案：localStorage

| 项 | 值 |
|----|-----|
| 存储命名空间 | `tiku_v8_`（`STORAGE_PREFIX = 'tiku_v8_'`） |
| 配额警告阈值 | `STORAGE_QUOTA_WARN_BYTES = 10 * 1024 * 1024`（10MB） |
| 配额检测 API | `navigator.storage.estimate()`（fallback 到 catch） |
| 快照上限 | `MAX_SNAPSHOTS = 10` |

### 3.2 封装函数

```js
function lsGet(k, def) {
  try {
    var v = localStorage.getItem(STORAGE_PREFIX + k);
    return v ? JSON.parse(v) : def;
  } catch(e) { return def; }
}

function lsSet(k, v) {
  try {
    localStorage.setItem(STORAGE_PREFIX + k, JSON.stringify(v));
  } catch(e) {
    // QuotaExceededError 处理：尝试淘汰最旧快照后重试
    // ...
  }
}
```

### 3.3 完整 localStorage 键名清单

| 键名（实际存储为 `tiku_v8_<key>`） | 类型 | 用途 |
|------|------|------|
| `questions` | `Array<Question>` | 题库（含自定义题） |
| `wrong` | `Array<qid>` | 错题 ID 列表 |
| `done` | `Array<qid>` | 已答对题 ID 列表 |
| `fav` | `Array<qid>` | 收藏题 ID 列表 |
| `wrongCount` | `Object<qid, count>` | 错次统计 |
| `studySec` | `number` | 累计学习秒数 |
| `todayStudySec` | `number` | 今日学习秒数 |
| `dailyGoal` | `number` | 每日目标题数 |
| `dailyCount` | `number` | 今日已完成题数 |
| `dailyDate` | `string` | 今日日期（用于 0 点重置判断） |
| `nextCustomId` | `number` | 自定义题下一个 ID（默认 1000000） |
| `sources` | `Array<string>` | 类别列表 |
| `srs` | `Object<qid, SRSState>` | SM-2 间隔重复状态 |
| `dailyHistory` | `Object<YYYY-MM-DD, {done, correct, wrong}>` | 每日统计 |
| `snapIndex` | `Array<timestamp>` | 快照索引 |
| `snap_<ts>` | `Object` | 单个快照实体（按时间戳命名） |
| `shuffle` | `boolean` | 洗牌模式开关 |
| `streakDays` | `number` | 连续达标天数 |
| `alarmEnabled` | `boolean` | 闹钟启用 |
| `alarmMinutes` | `number` | 闹钟分钟数 |
| `alarmLoop` | `boolean` | 闹钟循环 |
| `alarmNextAt` | `number` | 下次闹钟触发时间戳 |
| `version` | `string` | 数据格式版本（`v1.0.0`） |

> 注：旧版 `autoBackupData` 键仍可读取（一次性迁移到 `snapIndex` 索引系统）。

### 3.4 数据格式（exportData / importData）

```js
{
  wrong: [qid, ...],
  done: [qid, ...],
  fav: [qid, ...],
  wrongCount: { qid: count, ... },
  studySec: number,
  todayStudySec: number,
  dailyGoal: number,
  dailyCount: number,
  dailyDate: "YYYY-MM-DD",
  nextCustomId: number,
  sources: ["类别1", "类别2", ...],
  questions: [
    { id, source, qnum, subnum, question, options, answer }, ...
  ],
  srs: {
    qid: { interval, repetition, easiness, nextDate, lastDate }, ...
  },
  version: "v1.0.0"
}
```

---

## 4. 设计系统

### 4.1 CSS Custom Properties Token 体系

完整的 7 类设计 Token，定义在 `:root` 与 `body.dark` 中：

| Token 前缀 | 类别 | 示例 |
|-----------|------|------|
| `--c-*` | 颜色 | `--c-primary` `--c-success` `--c-danger` `--c-bg-surface` `--c-text-1` |
| `--r-*` | 圆角 | `--r-sm` (8px) `--r-md` (12px) `--r-lg` (20px) `--r-xl` (28px) |
| `--sh-*` | 阴影 | `--sh-1` `--sh-2` `--sh-3` `--sh-glow`（3 层阴影体系） |
| `--fs-*` | 字号 | `--fs-xs` (12px) `--fs-sm` `--fs-base` `--fs-md` `--fs-lg` `--fs-xl` `--fs-xxl` (44px) |
| `--sp-*` | 间距 | `--sp-1` (4px) `--sp-2` (8px) `--sp-3` (12px) `--sp-4` (16px) `--sp-5` (24px) `--sp-6` (32px) |
| `--t-*` | 过渡时长 | `--t-fast` (0.15s) `--t-base` (0.25s) `--t-slow` (0.4s) |
| `--ease-*` | 缓动函数 | `--ease` (cubic-bezier(0.4,0,0.2,1)) `--ease-bounce` (cubic-bezier(0.34,1.56,0.64,1)) |

### 4.2 暗色模式

`body.dark` 类下重定义所有颜色 / 阴影 Token，UI 自动跟随。切换由 `applySystemTheme()` 监听 `prefers-color-scheme` 媒体查询触发。

### 4.3 设计规则（强制）

- 禁止在组件样式中硬编码色值 / 间距 / 圆角 / 阴影
- 新样式优先使用已有 Token 变量
- 如需新色值，补入 `:root` 变量区，不写在组件内

---

## 5. 无构建工具 / 无框架的设计决策

### 5.1 为什么单文件 HTML？

| 决策 | 理由 |
|------|------|
| 单文件（HTML + CSS + JS 内联） | 双击即用，无需部署；可放 U 盘 / 邮件 / 网盘随处携带 |
| 无构建工具（webpack/vite/babel） | 项目体量 2.7MB 单文件，引入构建链是过度工程；源码即运行时 |
| 无前端框架（React/Vue） | 状态管理简单（全局变量 + localStorage），框架引入反而增加体积与学习成本 |
| 无打包工具 | Chart.js 已内联，无第三方资源需要打包 |
| 无 TypeScript | 单文件项目，类型系统收益有限；运行时错误靠 `try/catch` 兜底 |
| 无 CSS 预处理器 | CSS Custom Properties 已足够，预处理器增加构建依赖 |

### 5.2 取舍

| 收益 | 代价 |
|------|------|
| 零部署、零依赖、可离线 | 单文件 2.7MB，首屏加载稍慢（本地文件场景可接受） |
| 源码即运行时，调试直观 | 无模块化，函数数量多时需依赖搜索定位 |
| 无构建配置，无版本冲突 | 无 Tree-shaking，Chart.js 全量内联（约 200KB） |

### 5.3 何时考虑拆分？

仅在以下场景才考虑拆分多文件：
- 题库规模超过 10MB，单文件加载不可接受
- 需要多人协作开发（单文件合并冲突严重）
- 引入更多第三方库，内联成本过高

当前 2.7MB 体量 + 单人开发场景下，单文件是最优解。

---

## 6. 浏览器兼容性要求

### 6.1 必需 API

| API | 用途 | 兼容性 |
|-----|------|--------|
| `localStorage` | 数据持久化 | IE8+ / 全平台 |
| `JSON.parse` / `JSON.stringify` | 序列化 | IE8+ / 全平台 |
| `Set` / `Array.from` | 集合操作 | IE11+（不支持 IE10 及以下） |
| `addEventListener` | 事件绑定 | IE9+ / 全平台 |
| `document.querySelector` | DOM 查询 | IE8+ / 全平台 |
| `FileReader` | 文件导入 | IE10+ / 全平台 |
| `matchMedia` | 暗色模式监听 | IE10+ / 全平台 |
| `navigator.storage.estimate` | 配额监控 | Chrome 61+ / Firefox 57+（带 fallback） |
| `requestAnimationFrame` | 批量 DOM 更新 | IE10+ / 全平台 |
| CSS Custom Properties | 设计 Token | Chrome 49+ / Firefox 31+ / Safari 9.1+ |
| `backdrop-filter` | 玻璃态效果 | Chrome 76+ / Firefox 103+ / Safari 9+（带前缀） |

### 6.2 推荐浏览器

- **Chrome / Edge** 90+（最佳体验，全部特性支持）
- **Firefox** 100+
- **Safari** 14+
- **不支持**：IE 全系列、Chrome < 49

### 6.3 移动端

- iOS Safari 14+
- Android Chrome 90+

---

## 7. 性能优化策略

### 7.1 加载性能

| 策略 | 实现 |
|------|------|
| 单文件内联 | HTML + CSS + JS + Chart.js 全部内联，无网络请求（本地文件场景） |
| Chart.js 懒加载实例化 | 源码随页面加载，但 `new Chart()` 只在首次打开报告时调用 |
| 系统字体栈 | `-apple-system, BlinkMacSystemFont, "Microsoft YaHei", "Segoe UI"`，无字体加载 |
| 无图片资源 | 全部 emoji + CSS 绘制，无 HTTP 请求 |

### 7.2 运行时性能

| 策略 | 实现 |
|------|------|
| `DocumentFragment` | 批量 DOM 更新（如浏览页列表渲染）使用 fragment 减少 reflow |
| `requestAnimationFrame` | 高频 UI 更新（计时器每秒刷新）通过 rAF 节流 |
| 查找缓存 | `questionByIdMap` 缓存 id → question 映射，避免线性扫描 |
| `Set` 数据结构 | `wrongSet` / `doneSet` / `favSet` 用 Set，O(1) 查询 |
| 事件代理 | 列表类元素用事件代理，避免为每个子元素绑定监听器 |

### 7.3 动画性能

**Compositor 属性白名单**（不触发 layout / paint）：
- ✅ `transform` / `opacity` / `clip-path` / `filter`（谨慎）
- ❌ `width` / `height` / `top` / `left` / `margin` / `padding`

**例外**：进度条 `.progress-mini .fill` 必须用 `width`（进度语义无法用 transform 表达）。

### 7.4 存储 IO 优化

| 策略 | 实现 |
|------|------|
| 批量写 | `saveAllProgress()` 一次写 5 个键（wrong/done/fav/wrongCount/srs） |
| 高频降频 | 计时器 UI 每秒更新，但 localStorage 仅在 `stopTimer` 时写 |
| 快照节流 | `triggerAutoBackup()` 在每次关键写后触发，但快照写入有去重逻辑 |

### 7.5 可访问性降级

`@media (prefers-reduced-motion: reduce)` 媒体查询下：
- 禁用所有 `@keyframes` 动画
- 简化 `transition` 时长为 `0.01ms`
- 保留功能完整性，仅去除视觉动效
