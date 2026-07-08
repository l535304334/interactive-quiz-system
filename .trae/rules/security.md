# 安全规范 — 题库项目

> 来源: C:\Users\lai\.claude\rules\ecc\common\security.md + web\security.md
> 适用范围: 题库 HTML 单文件应用

## 数据安全特性

本项目为纯前端 localStorage 应用，不上传/不同步任何数据到服务器。安全关注点：

### XSS 防护（核心）

- ✅ 已定义 `escapeHtml()` 函数，所有动态 HTML 插入前转义
- ✅ 用户输入均经过 `escapeHtml()` 处理（如搜索结果展示、题目编辑）
- ⚠️ 检查点：`innerHTML` 赋值前是否都走了 `escapeHtml()`

### localStorage 安全

- 所有数据存在本地浏览器，不上传服务器
- 导出/导入功能走的是本地 JSON 文件，不经过网络
- 无第三方数据收集或埋点

### 弹窗安全

- 点击弹窗外关闭（防止用户被困在模态窗中）
- Esc 键可关闭所有弹窗
- 无键盘陷阱

## 安全检查清单（每次改动后）

- [ ] 新增的 `innerHTML` 赋值是否使用了 `escapeHtml()`？
- [ ] 用户可输入的字段（搜索框/导入 JSON/自定义题）是否正确转义？
- [ ] 有没有硬编码任何密钥/token？（本项目无此需求，但保持警觉）
- [ ] JSON 导入是否做了格式校验？（✅ `try/catch` + `parse` 已就位）
- [ ] localStorage 数据是否在异常时有备份？（✅ 5 份快照 + 导出功能）

## 已知风险（接受）

- localStorage 可被浏览器清空（用户已被告知定期导出备份）
- 单文件 HTML 无 CSP/HTTPS 保护（因为是本地文件，非服务器部署）
- 没有 CSRF 防护需求（无后端）
