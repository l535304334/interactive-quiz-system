# 贡献指南

感谢你对"老大专用刷题系统"的关注。在提交贡献前，请务必阅读本指南。

---

## 项目状态：维护模式

本项目当前处于**维护模式**：

- v1.0.0 已发布，核心功能（5 种答题模式、SM-2 间隔复习、Chart.js 学习报告、JSON 导入导出、自动快照、全局搜索、计时与闹钟、键盘模式、暗色模式）已完整可用。
- 作者**不再主动新增大功能**，仅接受以下类型的贡献：
  - Bug 修复（与代码行为不一致的问题）
  - 安全漏洞修复（XSS、数据完整性等）
  - 可访问性改进（ARIA、键盘导航、屏幕阅读器）
  - 性能优化（DOM 操作、内存泄漏、动画合规）
  - 文档完善与笔误修正
  - 已有功能的边缘 case 补充

> 如果你有重大功能想法（例如新增答题模式、云端同步、多用户支持等），请先开 Issue 讨论，**不要直接提交 PR**，以免工作浪费。

---

## 贡献流程

### 1. 提 Issue 确认

- Bug 报告：复现步骤 + 预期行为 + 实际行为 + 浏览器版本
- 功能建议：使用场景 + 现有方案为何不够 + 建议实现思路
- 安全问题：**不要在公开 Issue 描述漏洞细节**，请直接联系作者

### 2. Fork & 分支

```bash
git checkout -b fix/short-description
# 或
git checkout -b docs/short-description
```

分支命名规范：
- `fix/<简短描述>` — Bug 修复
- `docs/<简短描述>` — 文档更新
- `perf/<简短描述>` — 性能优化
- `a11y/<简短描述>` — 可访问性改进
- `refactor/<简短描述>` — 重构（不改变行为）

### 3. 实现与自测

- 修改**最小化**：只触碰必须改的地方，不改无关代码、注释或格式（参考 `.trae/rules/00-core.md` 第 3 条「外科手术式修改」）
- 不重构没坏的东西
- 匹配现有代码风格（参考下方「代码规范」）
- 单文件 HTML 应用：所有改动在 `[题库]交互式刷题页面.html` 内完成

### 4. 自测清单

提交前请逐项确认：

- [ ] 改动后浏览器双击打开，无控制台报错
- [ ] 答题核心流程正常（点击选项判对错、答对自动跳、答错停留）
- [ ] 模式切换正常（错题 / 复习 / 收藏 / 浏览 / 考试 + 复合模式）
- [ ] 数据持久化正常（刷新页面后进度保留）
- [ ] 新增的 `innerHTML` 赋值是否使用了 `escapeHtml()`？（参考 `.trae/rules/security.md`）
- [ ] 新增动画是否仅使用 `transform` / `opacity` / `clip-path`？（参考 `.trae/rules/animation-performance.md`）
- [ ] 新增样式是否使用了设计 Token 变量，而非硬编码色值？（参考 `.trae/rules/web-coding.md`）
- [ ] 模态弹窗是否补 `role="dialog"` `aria-modal="true"`？
- [ ] JSON 解析与 localStorage 操作是否包 `try/catch`？
- [ ] 暗色模式下显示是否正常？
- [ ] 移动端（`@media (max-width: 600px)`）布局是否错位？

### 5. 提交 PR

- PR 标题遵循 [Conventional Commits](https://www.conventionalcommits.org/)（见下方）
- PR 描述说明：改了什么 / 为什么改 / 如何自测 / 是否有破坏性变更
- 一个 PR 只解决一个问题，不混合多个无关改动

---

## 代码规范

本项目在 `.trae/rules/` 目录下维护了完整的编码与设计规范，**所有贡献必须遵循**。重点摘录如下：

### 核心原则（`.trae/rules/00-core.md`）

1. **编码前先思考** — 明确假设；不确定就问，不靠猜
2. **简洁优先（YAGNI）** — 只写解决问题的最少代码，不写投机性功能
3. **外科手术式修改** — 只触碰必须改的地方，不改无关代码
4. **目标驱动** — 定义"成功是什么样"并迭代验证

### JavaScript（`.trae/rules/coding-style.md` + `web-coding.md`）

- 使用 `const` / `let`，**不用 `var`**
- 使用 `addEventListener`，**不用 `onclick=` 属性**（HTML 中已有 `onclick` 是历史遗留，新代码不要继续使用）
- JSON 解析必须 `try/catch` 包裹
- localStorage 操作必须 `try/catch` 包裹
- 批量 DOM 更新用 `DocumentFragment` 或 `requestAnimationFrame`
- 事件代理优于为每个元素绑定事件
- **禁止** `eval()`、`new Function()`
- **禁止** `innerHTML` 直接渲染用户输入（必须经 `escapeHtml()`）
- **禁止** 同步 XHR
- 函数保持在 50 行以内，超标则提取子函数

### CSS（`.trae/rules/web-coding.md` + `animation-performance.md`）

- 使用 CSS Custom Properties（`--var`）做主题
- **禁止硬编码色值** — 新样式必须使用 `:root` 中已有的 Token（`--c-*` / `--r-*` / `--sh-*` / `--fs-*` / `--sp-*`）；如需新色值，补入 `:root` 变量区，不写在组件内
- 动画**只用** `transform` / `opacity` / `clip-path`（compositor-friendly）
- **禁止动画** `width` / `height` / `top` / `left` / `right` / `bottom` / `margin` / `padding`（进度条 `width` 是唯一例外）
- 响应式：`clamp()` + `@media` 查询
- 类名 kebab-case（如 `.q-feedback` / `.browse-card` / `.nav-mode-btn`）
- 状态类：`btn-on` / `btn-star-active` / `.show` / `.active`

### HTML 语义

- 已使用的语义元素：`<header>` / `<main>` / `<nav>` / `<section>`
- 模态弹窗必须 `role="dialog"` `aria-modal="true"`
- 不用纯 `<div>` 栈代替已有语义元素

### 安全（`.trae/rules/security.md`）

- 所有动态 HTML 插入前必须经 `escapeHtml()` 转义
- 用户可输入字段（搜索框 / 导入 JSON / 自定义题）必须正确转义
- 不硬编码任何密钥 / token（本项目无此需求）
- JSON 导入必须做格式校验（`try/catch` + `parse`）

### 性能（`.trae/rules/animation-performance.md`）

- 避免在 scroll 事件中做复杂计算
- `will-change` 仅用于高频动画元素，用后移除
- 简单过渡用 CSS `transition`，复杂序列用 `@keyframes`
- 不引入第三方动画库（GSAP / Framer Motion 对单文件 HTML 过度）
- `transition` 时长控制在 `0.15s` – `0.4s`

### 设计（`.trae/rules/design-quality.md`）

本项目采用「玻璃态 + 蓝紫渐变」风格。新 UI 组件须满足以下至少 4 项：

1. 清晰的层级对比
2. 有意的间距节奏（统一 Token `--sp-*` + 非均匀 padding）
3. 深度 / 层次感（阴影 `--sh-*` + 左侧色条 + backdrop-filter）
4. 排版有字符感
5. 颜色语义化（红错 / 绿对 / 金收藏 / 蓝复习 / 紫浏览）
6. hover / focus / active 态
7. grid-breaking 布局
8. 动效澄清流程
9. 数据可视化一体

**禁止模式**：默认灰色卡片网格 / 统一间距"安全灰白"设计 / 模板化 Hero 区域 / 单一装饰色。

### Git 安全（`.trae/rules/git-safety.md`）

- **禁止自动执行**破坏性操作：`git push --force` / `git reset --hard` / `git clean -fdx` / `git rebase -i`
- 在 `main` / `master` 分支上任何历史改写操作均需二次确认
- 提交前检查：无硬编码凭据、无个人隐私文件、无临时文件

---

## 提交规范

遵循 [Conventional Commits](https://www.conventionalcommits.org/)：

```
<type>: <description>

<optional body>
```

### Type 列表

| Type | 说明 | 示例 |
|------|------|------|
| `feat` | 新功能 | `feat: 新增错题导出为 Anki 卡片` |
| `fix` | Bug 修复 | `fix: 复合模式下错次筛选未生效` |
| `refactor` | 重构（不改变功能） | `refactor: 拆分 renderBrowsePage 子函数` |
| `docs` | 文档更新 | `docs: 补充键盘快捷键说明` |
| `test` | 测试相关 | （本项目当前无自动化测试） |
| `chore` | 构建 / 工具 / 依赖 | `chore: 升级 Chart.js 至 4.5.0` |
| `perf` | 性能优化 | `perf: 搜索索引构建改用 requestIdleCallback` |
| `ci` | CI/CD 配置 | （本项目当前无 CI） |
| `style` | 代码格式（不影响功能） | `style: 统一缩进为 4 空格` |

### 范围（可选）

可在 type 后加 `(scope)` 限定改动范围：

```
fix(srs): 答对时 ef 因子未正确抬升
feat(search): 支持 #ID 范围查询（如 #100-200）
perf(report): 雷达图改用 dataset 复用
```

建议 scope：`srs` / `search` / `report` / `exam` / `browse` / `wrong` / `fav` / `review` / `timer` / `alarm` / `backup` / `import` / `export` / `category` / `keyboard` / `theme` / `a11y`

### Body（可选）

说明「为什么」而非「做了什么」（diff 已经说明做了什么）：

```
fix(srs): 答对时 ef 因子未正确抬升

updateSRS 中 q=5 时 (5-q)=0，导致 ef 增量计算为 0.1-0=0.1，
但实际应为 0.1-(0.08+0)=0.02。修正公式后 ef 增量符合 SM-2 原始算法。
```

### 破坏性变更

破坏性变更（不兼容的 API / 数据 schema 变更）必须在 footer 标注 `BREAKING CHANGE:`：

```
feat(backup): 快照 schema 升级至 v2

BREAKING CHANGE: 快照字段从扁平结构改为嵌套结构，旧版快照需通过
迁移脚本转换。运行 migrateSnapshot() 自动迁移。
```

---

## 开发环境

本项目无需任何开发环境配置：

1. **代码编辑器** — 任意（VS Code / Sublime / Vim 均可）
2. **运行环境** — 现代浏览器（Chrome / Edge / Firefox / Safari）
3. **依赖管理** — 无 npm 依赖；Chart.js 通过 CDN 加载
4. **构建工具** — 无；直接编辑 HTML 文件即可
5. **测试** — 当前无自动化测试，依赖手动自测清单

### 推荐编辑器配置

- 文件编码：UTF-8（无 BOM）
- 换行符：LF（或 CRLF，但请保持一致）
- 缩进：4 空格（与现有代码一致）
- 保存时自动格式化：关闭（避免无关格式变更污染 diff）

---

## 联系方式

- 提交 Issue：通过仓库的 Issues 入口
- 安全问题：**不要公开 Issue**，请通过仓库提供的私密联系方式

---

再次感谢你的贡献。每一条符合规范的 PR 都会被认真审阅。
