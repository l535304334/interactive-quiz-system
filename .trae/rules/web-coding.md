# Web 编码规范 — 题库项目

> 来源: C:\Users\lai\.claude\rules\ecc\web\coding-style.md + common\coding-style.md
> 适用范围: 题库 HTML 单文件应用

## CSS 设计 Token

项目已建立完整的设计 Token 体系（`--c-*` / `--r-*` / `--sh-*` / `--fs-*` / `--sp-*`），必须坚持：

- 禁止在组件样式中硬编码色值、间距、圆角、阴影
- 新样式优先使用已有的 Token 变量
- 如需新色值，补入 `:root` 变量区，不写在组件内

## 动画属性白名单

优先使用 compositor-friendly 属性：
- ✅ `transform` / `opacity` / `clip-path` / `filter`（谨慎）
- ❌ `width` / `height` / `top` / `left` / `margin` / `padding`

项目中 `.q-feedback`、`.toast`、`.modal` 的入场动画已使用 `transform` + `opacity`，保持此模式。

## 语义 HTML

已使用的语义元素：
- `<header>` — 顶部导航栏
- `<main>` — 主内容区
- `<nav>` — 题目导航栏
- `<section>` — 数据管理分区

禁止事项：
- 不要用纯 `<div>` 栈代替已有语义元素
- 模态弹窗需添加 `role="dialog"` `aria-modal="true"`

## CSS 命名规范

- 组件类名：kebab-case（已统一，如 `.q-feedback` / `.browse-card` / `.nav-mode-btn`）
- 状态类：`btn-on` / `btn-star-active` / `.show` / `.active`
- 新增类名必须遵循此模式

## 函数与文件规模

- 函数保持在 50 行以内（当前项目多数函数已符合）
- 如超标，提取子函数（如 `renderBrowsePage` → `pageNav` 子函数 已做）
- 整个 HTML 文件当前体量合理（2.6MB），暂不拆分

## 禁止模式

- `innerHTML` 插入未转义用户数据时，必须使用 `escapeHtml()`（项目中已定义）
- 不引入新依赖时不加 `<script src="...">`
- `console.log` 仅用于调试，不保留在生产代码中
