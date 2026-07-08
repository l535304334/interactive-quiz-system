# 动画与性能规范 — 题库项目

> 来源: C:\Users\lai\.claude\rules\ecc\web\performance.md + skills\ecc\motion-ui\SKILL.md
> 适用范围: 题库 HTML 单文件应用

## 动画性能规则

### Compositor 属性白名单
项目中所有动画必须优先使用以下属性：
- `transform` — 位移/缩放/旋转
- `opacity` — 透明度
- `clip-path` — 裁剪路径（谨慎）

### 禁止动画的属性
- `width` / `height`
- `top` / `left` / `right` / `bottom`
- `margin` / `padding`

### 当前项目动画审计

| 动画元素 | 使用属性 | 合规 |
|---------|---------|------|
| `.toast` toastIn | `transform` + `opacity` | ✅ |
| `.q-feedback` fadeInScale | `transform: scale` + `opacity` | ✅ |
| `.modal` scaleIn | `transform: scale` + `opacity` | ✅ |
| `.mode-banner` slideInRight | `transform: translateX` + `opacity` | ✅ |
| `.info-hero-emoji` float | `transform: translateY` | ✅ |
| `.browse-banner` shimmer | `background-position` | ✅ |
| `.nav-mode-btn:hover` icon | `transform: scale + rotate` | ✅ |
| `.progress-mini .fill` | `width`（唯一例外：进度条必须） | ⚠️ |

### 动效设计原则

1. **动效是交互设计**，不是装饰
2. 每个动画都要回答"它传达了什么状态？"
3. 如果动效不改善 UX → 删除
4. `transition` 时长控制在 `0.15s-0.4s`

### 全局性能约束

- 避免在 scroll 事件中做复杂计算（当前项目无滚动动画，安全）
- `will-change` 仅用于高频动画元素，用后移除
- 简单过渡用 CSS `transition`，复杂序列用 `@keyframes`
- 不引入第三方动画库（GSAP/Framer Motion 对单文件 HTML 过度）

## 加载策略（本项目简化版）

由于本项目是本地 HTML 文件（非网络加载）：
- Chart.js 通过 CDN 加载（已在 `<head>` 中，带 `defer`）
- 其他所有资源内联
- 无需图片优化（无图片资源）
- 无需字体加载策略（使用系统字体栈）
