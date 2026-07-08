# 技能 & Agent 参考映射 — 题库项目

> 来源: C:\Users\lai\.claude\skills\ecc\ + C:\Users\lai\.claude\agents\
> 作用: 文档化可用技能与 Agent，供后续开发时查阅

---

## 可用技能参考（Knowledge Base，非自动触发）

以下技能的原文件位于 `C:\Users\lai\.claude\skills\ecc\`，开发时可参考其内容但不自动触发：

### 前端相关

| 技能 | 路径 | 适用场景 |
|------|------|---------|
| `frontend-design-direction` | `skills/ecc/frontend-design-direction/` | 新页面/组件设计前确定方向 |
| `frontend-patterns` | `skills/ecc/frontend-patterns/` | 状态管理、表单、键盘导航模式（React 为主，JS 可参考） |
| `motion-ui` | `skills/ecc/motion-ui/` | 动效系统设计（Framer Motion 为主，CSS 动画可参考设计原则） |
| `error-handling` | `skills/ecc/error-handling/` | 错误处理模式（TS/Python/Go，JS 部分可参考） |

### 代码质量

| 技能 | 路径 | 适用场景 |
|------|------|---------|
| `coding-standards` | `skills/ecc/coding-standards/` | 命名规范、KISS/DRY/YAGNI 审查 |
| `plankton-code-quality` | `skills/ecc/plankton-code-quality/` | 代码质量检测（ESLint/Biome hook 思路，当前项目无自动 lint） |

### 项目管理

| 技能 | 路径 | 适用场景 |
|------|------|---------|
| `strategic-compact` | `skills/ecc/strategic-compact/` | 对话上下文过长时压缩 |
| `verification-loop` | `skills/ecc/verification-loop/` | 改动后验证是否正确生效 |

---

## 可用 Agent 参考

以下 Agent 定义位于 `C:\Users\lai\.claude\agents\`：

| Agent | 用途 | 本项目适用性 |
|-------|------|-------------|
| `code-reviewer.md` | 代码质量/模式/最佳实践审查 | ⭐⭐⭐ 每次大改动后建议使用 |
| `code-simplifier.md` | 检测并简化复杂代码 | ⭐⭐ 重构时可参考 |
| `security-reviewer.md` | 安全漏洞检测（XSS/注入） | ⭐⭐⭐ XSS 相关改动必检 |
| `performance-optimizer.md` | 性能瓶颈分析 | ⭐⭐ 大量 DOM 操作优化时 |
| `code-explorer.md` | 陌生代码库探索 | ⭐ 已熟悉本项目，不常用 |
| `architect.md` | 系统架构设计 | ⭐ 仅架构级重构时 |

---

## 可用命令参考

以下命令位于 `C:\Users\lai\.claude\commands\`，仅作功能参考（Trae 不支持直接调用）：

| 命令 | 等价操作 |
|------|---------|
| `/code-review` | 请在 Trae 中手动说"审查代码"触发 `code-review-and-quality` skill |
| `/plan` | 手动说"制定计划"触发 `planning-and-task-breakdown` skill |
| `/security-scan` | 手动说"安全扫描"触发 `TRAE-security-review` skill |
| `/refactor-clean` | 手动说"清理代码"触发 `code-simplification` skill |

---

## Trae ↔ ECC Skill 映射

| ECC Skill | Trae 等价 Skill |
|-----------|----------------|
| `frontend-design-direction` | `brainstorming`（设计前先发散） |
| `coding-standards` | `code-review-and-quality` |
| `frontend-patterns` | `web-dev`（含前端最佳实践） |
| `error-handling` | 手动遵循 `.trae/rules/` 中的规范 |
| `motion-ui` | 手动遵循 `.trae/rules/animation-performance.md` |
| `code-reviewer` agent | `code-review-and-quality` skill |
| `security-reviewer` agent | `TRAE-security-review` skill |
| `code-simplifier` agent | `code-simplification` skill |

---

## 快速参考卡片

```
改 UI → 先读 .trae/rules/design-quality.md
改样式 → 先读 .trae/rules/web-coding.md
触动效 → 先读 .trae/rules/animation-performance.md
改数据处理 → 先读 .trae/rules/security.md
改完代码 → 用 code-review-and-quality skill 自查
```
