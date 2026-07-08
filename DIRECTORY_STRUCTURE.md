# 目录结构 — 老大专用刷题系统

> 项目根目录：`c:\Users\lai\Desktop\题库\`

---

## 1. 项目结构树

```
题库/
├── [题库]交互式刷题页面.html      ★ 主文件（约 2.7MB 单文件应用）
├── PROJECT_OVERVIEW.md            ★ 项目概述文档
├── SYSTEM_ARCHITECTURE.md         ★ 系统架构文档
├── FEATURE_LIST.md                ★ 功能列表文档
├── TECH_STACK.md                  ★ 技术栈文档
├── DIRECTORY_STRUCTURE.md         ★ 本文档
├── ai-workspace.yaml              # AI 工作空间配置
├── .ai-workspace-state.json       # AI 工作空间状态（自动生成）
│
├── .trae/                         # Trae IDE 规则与文档
│   ├── rules/                     # 规则文件（强制遵循）
│   │   ├── 00-core.md
│   │   ├── 01-ai-collaboration.md
│   │   ├── 02-engineering.md
│   │   ├── animation-performance.md
│   │   ├── coding-style.md
│   │   ├── design-quality.md
│   │   ├── environment.md
│   │   ├── git-safety.md
│   │   ├── privacy.md
│   │   ├── security.md
│   │   ├── skill-agent-mapping.md
│   │   └── web-coding.md
│   ├── skills/
│   │   └── ponytail/
│   │       └── SKILL.md
│   └── documents/                 # 历史文档与资源
│       ├── 题库系统V9全面优化方案.md
│       ├── _check.js
│       └── chart.min.js
│
├── .claude/
│   └── CLAUDE.md                  # Claude Code 规则（与 .trae/rules 同步）
│
├── .workbuddy/                    # WorkBuddy 配置（备份规则与技能）
│   ├── rules/
│   │   ├── 00-core.md
│   │   ├── 01-ai-collaboration.md
│   │   ├── 02-engineering.md
│   │   ├── coding-style.md
│   │   ├── environment.md
│   │   ├── git-safety.md
│   │   ├── privacy.md
│   │   └── testing.md
│   ├── skills/
│   │   ├── code-tour/
│   │   ├── error-handling/
│   │   ├── frontend-design-direction/
│   │   ├── make-interfaces-feel-better/
│   │   ├── motion-ui/
│   │   ├── ponytail/
│   │   └── production-audit/
│   ├── agents/
│   │   ├── a11y-architect.md
│   │   ├── planner.md
│   │   ├── refactor-cleaner.md
│   │   └── silent-failure-hunter.md
│   └── memory/
│       ├── MEMORY.md
│       └── 2026-07-03.md
│
├── .codebuddy/
│   └── settings.local.json        # CodeBuddy 本地设置
│
└── .ai-backups/                   # AI 工作空间历史快照备份
    ├── 20260706-174534/
    ├── 20260706-174656/
    ├── 20260706-174745/
    └── 20260706-175122/
```

> 注：`.ai-backups/` 下每个时间戳子目录是当时 `.trae/` + `.workbuddy/` + `.claude/` 的完整快照，用于回滚。本项目文档（PROJECT_OVERVIEW.md 等 5 个文件）由本次任务新增。

---

## 2. 主文件内部结构

**主文件**：`[题库]交互式刷题页面.html`（约 2.7MB）

按行号区间划分（近似值，可能因编辑略有偏移）：

```
行号区间          内容
─────────────────────────────────────────────────────────
1–6              HTML 头部（DOCTYPE / meta / title）
7                <style> 开始
8–95             CSS Custom Properties Token 定义（:root + body.dark）
96–4831          组件 CSS 样式
4832–4839        @media (prefers-reduced-motion: reduce) 降级
4840             </style> 结束
4842             <body> 开始
4842–4860        通用遮罩弹窗（confirmOverlay / promptOverlay）
4935–5185        头部导航 + 主体结构 + Hero 信息区
5185–5867        各模态弹窗 HTML（11 个 modal）
5868             <script> 开始（主应用脚本）
5868–11954       工具函数 + ALL_QUESTIONS 题库数据
11955–11973      存储封装（STORAGE_PREFIX / lsGet / lsSet）
11974–12011      配额监控 + 运行时状态变量声明
12013–12090       题库读写 + 进度持久化
12092–12116       SRS（SM-2）算法
12118–12227       模式管理 + 横幅渲染
12228–12344       每日统计 + 快照系统
12346–12446       暗色主题 + 来源筛选
12447–12737       模式切换 + 收藏 + 目标 + 计时模态
12738–13289       题目渲染 + 浏览页渲染
13290–13436       计时器 + 闹钟
13437–13615       学习报告（buildReport / renderReportCharts）
13616–13687       启动恢复（checkAutoRestore）
13688–13816       全局搜索
13817–13953       自定义题编辑器
13954–14189       类别管理
14190–14250       导入导出
14251–14256       focusModal 工具
14257–14475       考试模式（startExam / renderExam / startExamTimer）
14476–14549       键盘事件处理 + 启动初始化
14550            </script> 结束（主应用脚本）
14551            <script> 开始（Chart.js 内联）
14554–14564       Chart.js v4.4.4 源码注释
14564–14572       Chart.js v4.4.4 压缩源码（单行）
14573            </script> 结束（Chart.js）
14574            </body> 结束
```

### 2.1 CSS 区（行 7–4840）

| 子区 | 行号 | 内容 |
|------|------|------|
| Token 定义 | 8–95 | `:root` 与 `body.dark` 的 CSS Custom Properties |
| 全局重置 | 96–110 | `* { margin:0; padding:0; box-sizing:border-box }` + body 基础样式 |
| 头部导航 | 111–300 | `.header` 玻璃态 + `nav-group` 胶囊布局 |
| 题目卡片 | 301–700 | `.question-card` + `.q-feedback` 反馈卡 |
| 模态弹窗 | 701–1500 | `.modal-overlay` + `.info-modal` 11 个弹窗样式 |
| 模式横幅 | 1501–1800 | `.mode-banner` 4 色渐变 |
| 浏览页 | 1801–2200 | `.browse-card` + shimmer 动画 |
| 报告 | 2201–2700 | Chart.js 容器 + 学科排行 |
| Toast | 2701–2900 | `.toast` 滑入动画 |
| 移动端响应 | 2901–3500 | `@media (max-width: 600px)` |
| 暗色模式覆盖 | 3501–4831 | `body.dark` 下的组件样式补充 |
| 动效降级 | 4832–4839 | `@media (prefers-reduced-motion: reduce)` |

### 2.2 HTML 区（行 4842–5867）

包含 11 个模态弹窗 + 头部导航 + 主体结构 + Hero 信息区。

### 2.3 JS 区（行 5868–14550）

主应用脚本，IIFE 包裹。包含全部业务逻辑、题库数据（ALL_QUESTIONS）、存储封装、模式管理、答题处理、计时、报告、考试、搜索、类别管理、导入导出、键盘事件等。

### 2.4 Chart.js 区（行 14551–14573）

内联 Chart.js v4.4.4 压缩源码（UMD 版本），含版权注释。源码原路径：`/npm/chart.js@4.4.4/dist/chart.umd.js`。

---

## 3. 配置文件说明

### 3.1 `ai-workspace.yaml`

AI 工作空间配置文件，定义项目元数据：

```yaml
name: med-exam
description: 医学考试刷题系统 — 纯 HTML/CSS/JS 单页应用
profiles: [personal, vanilla-js]
platforms: [claude-code, trae, workbuddy]
project_docs:
  - "[题库]交互式刷题页面.html"
```

| 字段 | 值 | 说明 |
|------|-----|------|
| `name` | `med-exam` | 项目标识 |
| `description` | 医学考试刷题系统 | 项目描述 |
| `profiles` | `[personal, vanilla-js]` | 配置 Profile |
| `platforms` | `[claude-code, trae, workbuddy]` | 适配的 AI 平台 |
| `project_docs` | 主文件路径 | AI 工具识别的文档入口 |

### 3.2 `.ai-workspace-state.json`

AI 工作空间状态（自动生成，不应手动编辑）。记录上次同步时间、规则版本等。

### 3.3 `.codebuddy/settings.local.json`

CodeBuddy（代码助手工具）的本地设置。

---

## 4. 规则文件说明（`.trae/rules/`）

> 这些规则文件是项目编码与协作的强制规范，开发时必须遵循。
> `.workbuddy/rules/` 与 `.claude/CLAUDE.md` 内容与之同步。

| 文件 | 用途 |
|------|------|
| `00-core.md` | 核心编码规则（思考优先 / 简洁优先 / 外科手术式修改 / 目标驱动） |
| `01-ai-collaboration.md` | AI 代理协作规则（确定性逻辑禁止交给模型 / Token 预算 / 暴露冲突） |
| `02-engineering.md` | 工程实践（先读再写 / 测试规范 / 长任务检查点 / 惯例优先 / 失败显性化） |
| `animation-performance.md` | 动画与性能规范（compositor 属性白名单 + 项目动画审计表） |
| `coding-style.md` | Vanilla JS 编码规范（单文件组织 / const-let / addEventListener / try-catch） |
| `design-quality.md` | 设计质量标准（玻璃态 + 蓝紫渐变风格 + 设计质量 Checklist） |
| `environment.md` | 环境维护与开发原则（最小修改 / 可回滚优先 / 不追整洁） |
| `git-safety.md` | Git 安全操作规范（Commit 消息格式 / 破坏性操作防护 / 分支保护） |
| `privacy.md` | 个人隐私与密钥安全（禁止推送的内容 / 凭据管理 / 提交前检查） |
| `security.md` | 安全规范（XSS 防护 / localStorage 安全 / 弹窗安全 / 检查清单） |
| `skill-agent-mapping.md` | 技能 & Agent 参考映射（可用技能与 Agent 索引） |
| `web-coding.md` | Web 编码规范（CSS Token / 动画白名单 / 语义 HTML / 命名规范） |

### 4.1 规则来源

- `00-core.md` / `01-ai-collaboration.md` / `02-engineering.md` / `environment.md` / `git-safety.md` / `privacy.md` / `coding-style.md`：来自 `personal/` Profile（通用规则）
- `animation-performance.md` / `design-quality.md` / `security.md` / `skill-agent-mapping.md` / `web-coding.md`：项目特化规则（题库项目专用）

### 4.2 规则同步机制

`<!-- AUTO-GENERATED by sync-ai v1.0.0 -->` 标记的文件由 `sync-ai` 工具自动同步，**不应手动编辑**。修改规则应在 canonical source 处修改后重新同步。

---

## 5. 历史文档与资源（`.trae/documents/`）

| 文件 | 用途 |
|------|------|
| `题库系统V9全面优化方案.md` | V9 版本全面优化方案设计文档 |
| `_check.js` | 检查脚本（开发期使用） |
| `chart.min.js` | Chart.js 离线副本（与主文件内联版本一致） |

---

## 6. AI 备份目录（`.ai-backups/`）

每次 AI 工作空间同步时生成的时间戳快照，命名格式 `YYYYMMDD-HHMMSS`。每个子目录是当时 `.trae/` + `.workbuddy/` + `.claude/` 的完整副本，用于：

- 规则回滚：新规则引入问题时恢复到上一版本
- 历史追溯：查看规则演进历史

**不应手动删除**（除非用户明确同意，参见 `environment.md` 的"不追整洁"原则）。

当前存在的备份：

- `20260706-174534`
- `20260706-174656`
- `20260706-174745`
- `20260706-175122`

---

## 7. 文件清单速查

### 7.1 必须存在的文件

| 文件 | 作用 | 删除影响 |
|------|------|---------|
| `[题库]交互式刷题页面.html` | 主应用 | 应用无法运行 |
| `ai-workspace.yaml` | AI 工具配置 | AI 工具无法识别项目 |
| `.trae/rules/*.md` | 编码规则 | AI 协作无规范约束 |
| `.claude/CLAUDE.md` | Claude Code 规则 | Claude Code 无法加载规则 |

### 7.2 可选 / 可重建的文件

| 文件 | 作用 | 处理 |
|------|------|------|
| `.ai-workspace-state.json` | 同步状态 | 可由 sync-ai 重建 |
| `.ai-backups/*` | 历史快照 | 可删除（但建议保留至少最新 1 份） |
| `.trae/documents/*` | 历史文档 | 可删除（不影响运行） |
| `.codebuddy/settings.local.json` | CodeBuddy 设置 | 不使用 CodeBuddy 可删 |

### 7.3 本次新增文档

| 文件 | 用途 |
|------|------|
| `PROJECT_OVERVIEW.md` | 项目概述 |
| `SYSTEM_ARCHITECTURE.md` | 系统架构 |
| `FEATURE_LIST.md` | 功能列表 |
| `TECH_STACK.md` | 技术栈 |
| `DIRECTORY_STRUCTURE.md` | 目录结构（本文档） |
