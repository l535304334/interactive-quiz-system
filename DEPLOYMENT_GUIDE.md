# 题库 · 部署指南

> 题库交互式刷题页面 — 单文件 HTML 应用部署文档
> 版本：v1.0.0 · 适用文件：`[题库]交互式刷题页面.html`

---

## 目录

1. [部署方式概览](#1-部署方式概览)
2. [本地使用部署](#2-本地使用部署)
3. [网络部署方式](#3-网络部署方式)
4. [浏览器兼容性要求](#4-浏览器兼容性要求)
5. [注意事项](#5-注意事项)
6. [升级流程](#6-升级流程)
7. [数据迁移说明](#7-数据迁移说明)

---

## 1. 部署方式概览

本项目是**纯静态单文件 HTML 应用**，无后端、无数据库、无构建步骤。部署方式极其灵活：

| 部署方式 | 适用场景 | 复杂度 | 是否需联网 |
|---------|---------|--------|-----------|
| 双击 HTML 文件 | 个人本地使用 | 极简 | 否（除 Chart.js CDN） |
| GitHub Pages | 公开访问、分享 | 简单 | 是（Chart.js CDN） |
| 静态文件服务器（nginx/Apache） | 局域网/内网部署 | 中等 | 否（可本地化 Chart.js） |
| Python http.server | 临时调试/局域网共享 | 极简 | 否（同上） |
| CDN 部署 | 大规模分发 | 中等 | 是 |

### 1.1 部署核心特点

- **无服务器端依赖**：不需要 Node.js/PHP/Java/Python 等运行时
- **无构建步骤**：直接部署 HTML 文件即可
- **无配置文件**：不需要 `nginx.conf` 之外的任何配置
- **跨平台**：Windows/macOS/Linux 任意系统均可

### 1.2 唯一外部依赖

- **Chart.js v4.4.4**：通过 CDN 加载，仅在打开「学习报告」时按需拉取
- 离线场景：首次打开报告会失败，需联网一次缓存；或自行将 CDN 替换为本地内联（项目已内联 Chart.js 库代码，CDN 加载是冗余的备份方案）

---

## 2. 本地使用部署

### 2.1 双击 HTML 文件（推荐）

最简单的部署方式：

1. 将 `[题库]交互式刷题页面.html` 复制到任意目录（如桌面、文档文件夹）
2. **双击文件**，系统用默认浏览器打开
3. 浏览器地址栏显示 `file:///C:/Users/.../%E9%A2%98%E5%BA%93...html`
4. 即可使用

#### 优点

- 零配置、零依赖
- 完全离线可用（首次除外）
- 数据完全本地化，不上传

#### 注意事项

- **同源策略**：`file://` 协议下，浏览器对 localStorage 的处理可能与 `http://` 不同
- **路径稳定**：建议固定一个目录，不要频繁移动文件
- **关联浏览器**：如默认浏览器不理想，可右键 → 打开方式 → 选择浏览器

### 2.2 浏览器固定打开方式

如希望始终用特定浏览器打开：

- **Windows**：右键 HTML 文件 → 属性 → 更改 → 选择浏览器
- **macOS**：右键 → 显示简介 → 打开方式 → 全部更改
- **Linux**：`xdg-mime default google-chrome.desktop text/html`

### 2.3 创建桌面快捷方式

- **Windows**：右键 HTML 文件 → 发送到 → 桌面快捷方式
- **macOS**：拖拽 HTML 到桌面（按住 Option+Cmd 创建别名）

---

## 3. 网络部署方式

### 3.1 GitHub Pages 部署

适合：公开访问、分享给他人、跨设备同步访问（注意数据仍各自独立，因 localStorage 不同源）。

#### 步骤

1. **创建 GitHub 仓库**

   - 访问 https://github.com/new
   - 仓库名建议：`tiku` 或 `question-bank`
   - 可见性：Public（Pages 免费版要求公开仓库）
   - 勾选「Add a README file」

2. **上传 HTML 文件**

   方式 A：网页上传
   - 在仓库页面点 `Add file` → `Upload files`
   - 拖入 `[题库]交互式刷题页面.html`
   - 文件名建议改为英文：`index.html`（这样 Pages 直接根路径访问）
   - 提交 commit

   方式 B：Git 命令
   ```bash
   git clone https://github.com/<your-username>/tiku.git
   cd tiku
   cp "<原始路径>/[题库]交互式刷题页面.html" index.html
   git add index.html
   git commit -m "feat: 初始化题库应用"
   git push origin main
   ```

3. **开启 GitHub Pages**

   - 进入仓库 → Settings → Pages
   - Source：选择 `Deploy from a branch`
   - Branch：`main` / `(root)` / `Save`
   - 等待 1-2 分钟，刷新页面可见访问地址

4. **访问**

   - 地址格式：`https://<your-username>.github.io/tiku/`
   - 首次访问可能有 1-2 分钟构建延迟

#### 优点

- 免费 HTTPS
- 全球 CDN 加速
- 支持 `git` 版本管理

#### 注意事项

- **仓库大小限制**：GitHub 单文件建议 < 100MB，本项目 2.7MB 完全没问题
- **Pages 流量限制**：软限 100GB/月，个人使用绰绰有余
- **HTTPS 强制**：`file://` 的 localStorage 与 `https://` 的不互通，切换部署方式会丢失数据
- **仓库必须公开**（免费账户）：私密仓库 Pages 需 GitHub Pro

### 3.2 静态文件服务器（nginx）

适合：内网部署、企业内部使用、需要稳定 URL 的场景。

#### 步骤

1. **安装 nginx**

   - Ubuntu：`sudo apt install nginx`
   - CentOS：`sudo yum install nginx`
   - macOS：`brew install nginx`
   - Windows：下载 [nginx Windows 版](http://nginx.org/en/download.html)

2. **部署文件**

   ```bash
   # 假设 nginx 默认根目录
   sudo cp "[题库]交互式刷题页面.html" /var/www/html/index.html
   # 或自定义目录
   sudo mkdir -p /var/www/tiku
   sudo cp "[题库]交互式刷题页面.html" /var/www/tiku/index.html
   ```

3. **配置 nginx**

   编辑 `/etc/nginx/sites-available/default` 或 `/etc/nginx/nginx.conf`：

   ```nginx
   server {
       listen 80;
       server_name tiku.example.com;  # 你的域名或 IP

       root /var/www/tiku;
       index index.html;

       location / {
           try_files $uri $uri/ =404;
       }

       # 静态文件缓存优化
       location ~* \.(html|js|css)$ {
           expires 1h;
           add_header Cache-Control "public, no-transform";
       }
   }
   ```

4. **启动 nginx**

   ```bash
   sudo nginx -t        # 测试配置
   sudo systemctl start nginx    # 启动
   sudo systemctl enable nginx   # 开机自启
   ```

5. **访问**

   - 本机：`http://localhost/`
   - 局域网：`http://<服务器IP>/`
   - 配域名后：`http://tiku.example.com/`

#### 优点

- 性能稳定
- 支持自定义域名与 HTTPS（配合 Let's Encrypt）
- 局域网内访问快

### 3.3 静态文件服务器（Apache）

类似 nginx，配置略有不同：

```apache
<VirtualHost *:80>
    ServerName tiku.example.com
    DocumentRoot /var/www/tiku
    DirectoryIndex index.html

    <Directory /var/www/tiku>
        Options -Indexes
        AllowOverride None
        Require all granted
    </Directory>
</VirtualHost>
```

### 3.4 Python http.server（临时调试）

适合：临时调试、局域网快速共享。

#### 步骤

1. 进入 HTML 文件所在目录：

   ```bash
   cd /path/to/题库
   ```

2. 启动服务器：

   ```bash
   # Python 3
   python -m http.server 8000

   # Python 2（不推荐）
   python -m SimpleHTTPServer 8000
   ```

3. 访问：

   - 本机：`http://localhost:8000/`
   - 局域网：`http://<本机IP>:8000/`

#### 优点

- 零配置
- Python 自带，无需安装

#### 注意事项

- 仅适合开发调试，**不适合生产**
- 单线程，并发性能差
- 无 HTTPS

### 3.5 CDN 部署

适合：大规模分发、降低延迟。

#### 方案 A：Cloudflare Pages

1. 注册 Cloudflare 账号
2. Pages → Create a project → Connect to Git
3. 连接 GitHub 仓库（同 3.1）
4. 构建命令：留空
5. 输出目录：`/`
6. 部署后获得 `<project>.pages.dev` 域名

#### 方案 B：Vercel / Netlify

类似流程：

- Vercel：`vercel deploy`（需 CLI）或导入 Git 仓库
- Netlify：拖拽 HTML 文件到控制台即可

#### 优点

- 全球 CDN
- 自动 HTTPS
- 部署极快

---

## 4. 浏览器兼容性要求

### 4.1 最低版本要求

| 浏览器 | 最低版本 | 推荐版本 | 备注 |
|--------|---------|---------|------|
| Chrome | 80+ | 最新 | 推荐 |
| Edge | 80+ | 最新 | 推荐 |
| Firefox | 75+ | 最新 | 完全兼容 |
| Safari | 13+ | 最新 | 完全兼容 |
| Opera | 67+ | 最新 | 完全兼容 |
| Samsung Internet | 12+ | 最新 | 移动端可用 |
| IE 11 | **不支持** | — | 缺失 ES6+、Promise、`navigator.storage.estimate` |

### 4.2 依赖的浏览器特性

| 特性 | 用途 | 兼容性 |
|------|------|--------|
| ES6+ 语法（let/const/箭头函数/模板字符串） | 业务代码 | Chrome 49+ / FF 45+ / Safari 10+ |
| Promise | `showConfirm` / `showPrompt` | Chrome 32+ / FF 29+ / Safari 8+ |
| `localStorage` | 数据持久化 | 全部现代浏览器 |
| `JSON.parse` / `JSON.stringify` | 序列化 | 全部现代浏览器 |
| `Set` / `Map` | 错题/已做/收藏集合 | Chrome 38+ / FF 13+ / Safari 8+ |
| `Array.from` / `Array.isArray` | 集合转换与校验 | Chrome 45+ / FF 32+ / Safari 9+ |
| `navigator.storage.estimate` | 存储配额监控 | Chrome 61+ / FF 57+ / Safari 15.2+ |
| `matchMedia('(prefers-color-scheme: dark)')` | 暗色模式跟随 | Chrome 76+ / FF 67+ / Safari 12.1+ |
| `backdrop-filter` | 玻璃态效果 | Chrome 76+ / FF 103+ / Safari 9+（带 -webkit-） |
| `Blob` + `URL.createObjectURL` | 导出 JSON 文件下载 | 全部现代浏览器 |
| `FileReader` | 导入 JSON 文件读取 | 全部现代浏览器 |
| `requestIdleCallback` | 搜索索引构建（可选） | Chrome 47+ / 不支持 Safari（有 fallback） |
| Touch events | 移动端滑动切题 | 全部移动浏览器 |
| `Chart.js v4.4.4` | 报告图表 | 要求现代浏览器（同上） |

### 4.3 移动端兼容

- iOS Safari 13+
- Android Chrome 80+
- 支持触摸滑动切题
- 响应式布局适配手机/平板

### 4.4 不兼容的场景

- ❌ IE 11 及以下
- ❌ 国产浏览器兼容模式（IE 内核）
- ❌ 老旧 Android WebView（< 60）
- ❌ iOS Safari 12 及以下

---

## 5. 注意事项

### 5.1 localStorage 同源策略

**最重要**的注意事项：localStorage 按「源」隔离，**源 = 协议 + 域名 + 端口**。

| 部署方式 | 源 | 数据互通 |
|---------|-----|---------|
| `file:///C:/.../题库.html` | file 协议空源 | ❌ 与其他不互通 |
| `http://localhost:8000/` | `http://localhost:8000` | ❌ 与 8001 端口不互通 |
| `http://192.168.1.10/` | `http://192.168.1.10` | ❌ 与 localhost 不互通 |
| `http://tiku.example.com/` | `http://tiku.example.com` | ✅ 同域名互通 |
| `https://tiku.example.com/` | `https://tiku.example.com` | ❌ 与 http 不互通 |

**实践建议**：

- 选定一种部署方式后**固定使用**
- 切换部署方式时，先用「导出 JSON」备份数据，再在新环境「导入 JSON」恢复
- 不同源之间无法直接共享进度

### 5.2 文件大小

- 主文件约 **2.7MB**（含数千道题与样式脚本）
- 首次加载需 1-2 秒，之后浏览器缓存后秒开
- CDN 部署建议开启 gzip 压缩，可压缩至约 600KB

#### nginx gzip 配置

```nginx
gzip on;
gzip_types text/html application/json;
gzip_min_length 1024;
gzip_comp_level 6;
```

#### Apache deflate 配置

```apache
<IfModule mod_deflate.c>
    AddOutputFilterByType DEFLATE text/html application/json
</IfModule>
```

### 5.3 Chart.js CDN 依赖

报告图表依赖 Chart.js v4.4.4，通过 CDN 加载：

```html
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.4/dist/chart.umd.js" defer></script>
```

#### 离线部署处理

项目代码内已内联 Chart.js 库（文件末尾约 200KB），CDN 加载是冗余备份方案。完全离线场景下：

1. 移除 `<head>` 中的 CDN `<script>` 标签
2. 或保留 CDN 标签，离线时浏览器加载失败会自动 fallback 到内联版本
3. 首次离线打开报告会失败，联网一次即可缓存

### 5.4 HTTPS 重要性

虽然项目本身不传输任何数据，但建议生产环境用 HTTPS：

- 现代浏览器对 `file://` 与 `http://` 的 localStorage 支持逐渐收紧
- 部分浏览器在非 HTTPS 下限制 `navigator.storage.estimate` 等高级 API
- GitHub Pages / Cloudflare Pages / Vercel 等服务默认提供 HTTPS

### 5.5 移动端部署

移动端浏览器 localStorage 配额可能更小（通常 5MB）：

- 建议定期导出备份
- 题库较大时可能更快触发存储满警告
- Safari iOS 无痕模式下 localStorage 不持久

### 5.6 多用户共享部署

如多个用户访问同一部署（如 nginx 服务器）：

- **每个用户的浏览器各自独立**，数据不互通
- 服务器不存储任何用户数据
- 不同用户看到相同的题库，但进度各自独立

---

## 6. 升级流程

### 6.1 升级原则

本项目设计为**无感升级**：

- 用户数据（localStorage）与代码（HTML 文件）完全分离
- 替换 HTML 文件即可完成升级
- 用户数据自动迁移（如有版本字段）

### 6.2 本地部署升级

1. 备份当前 HTML 文件（重命名为 `[题库]交互式刷题页面.html.bak`）
2. 将新版本 HTML 文件复制到原位置覆盖
3. 双击打开新文件，浏览器刷新
4. localStorage 中的数据自动加载，**无需任何操作**

### 6.3 网络部署升级

#### GitHub Pages

```bash
git clone https://github.com/<your-username>/tiku.git
cd tiku
# 替换 index.html
cp /path/to/new/index.html index.html
git add index.html
git commit -m "chore: 升级到 vX.Y.Z"
git push origin main
```

GitHub Pages 1-2 分钟后自动重新部署。

#### nginx / Apache

```bash
sudo cp /path/to/new/index.html /var/www/tiku/index.html
# 无需重启 nginx，文件替换即生效
# 建议清理浏览器缓存或强制刷新（Ctrl+Shift+R）
```

#### CDN 部署

- Cloudflare Pages：推送代码自动重新构建
- Vercel：`vercel --prod` 或 Git push 自动部署

### 6.4 升级后验证

- [ ] 页面能正常打开
- [ ] 题库数量与升级前一致
- [ ] 错题/已做/收藏数据保留
- [ ] 学习时长保留
- [ ] 快照列表保留
- [ ] 主要功能正常（答题、搜索、报告）

### 6.5 升级回滚

如升级后异常：

1. 用备份的 `.bak` 文件覆盖回原版本
2. 或从 Git 历史回退：
   ```bash
   git log --oneline
   git checkout <旧commit> -- index.html
   git commit -m "revert: 回退升级"
   git push origin main
   ```
3. 用户数据未丢失，仅仅是代码回退

---

## 7. 数据迁移说明

### 7.1 version 字段

导出的 JSON 文件包含 `version` 字段：

```json
{
  "wrong": [],
  "done": [],
  ...
  "version": "v1.0.0"
}
```

- 当前版本：`v1.0.0`
- 用于未来版本升级时识别数据格式
- 导入时如版本不匹配，系统按字段名兼容性处理（多出的字段忽略，缺失的字段用默认值）

### 7.2 STORAGE_PREFIX 与数据隔离

所有 localStorage 键统一加前缀 `tiku_v8_`（`STORAGE_PREFIX` 常量）：

```
tiku_v8_questions
tiku_v8_wrong
tiku_v8_done
tiku_v8_fav
tiku_v8_wrongCount
tiku_v8_srs
tiku_v8_sources
tiku_v8_studySec
tiku_v8_todayStudySec
tiku_v8_goal
tiku_v8_dailyCount
tiku_v8_dailyDate
tiku_v8_nextCustomId
tiku_v8_kMode
tiku_v8_shuffle
tiku_v8_snapIndex
tiku_v8_snap_<timestamp>  (动态，每份快照一个键)
tiku_v8_dailyHistory
tiku_v8_alarmEnabled
tiku_v8_alarmMinutes
tiku_v8_alarmLoop
tiku_v8_alarmNextAt
tiku_v8_streakDays
```

> 还有不带前缀的 `lastBackup`（旧字段兼容）和 `autoBackupData`（已废弃，自动迁移到新快照机制）。

前缀 `tiku_v8_` 用于与其他应用的 localStorage 隔离。**未来若数据格式有大改，会升级为 `tiku_v9_` 等新前缀**，并保留自动迁移逻辑。

### 7.3 数据迁移策略

#### 场景 1：同版本升级（如 v1.0.0 → v1.0.1）

- 无需迁移
- 直接替换 HTML 文件即可
- 数据完全兼容

#### 场景 2：兼容字段新增

- 新版本新增字段（如 `theme` 偏好），旧数据无此字段
- 代码用 `lsGet('theme', 'default')` 兜底默认值
- 无需手动迁移

#### 场景 3：字段格式变更（未来）

如某天 `wrong` 从数组改为对象：

1. 在新版本中检测旧格式（如 `Array.isArray(wrong)`）
2. 转换为新格式
3. 保存到新键（如 `wrong_v2`）
4. 保留旧键一段时间，提示用户导出备份
5. N 个版本后删除旧键

#### 场景 4：完全重构（不推荐）

如数据结构完全重做：

1. 升级 `STORAGE_PREFIX` 为 `tiku_v9_`
2. 启动时检测旧前缀数据
3. 弹出迁移确认框
4. 用户确认后转换并写入新前缀
5. 保留旧前缀数据 30 天，期间可回滚

### 7.4 旧版兼容（已实现）

代码中已有以下兼容处理：

- **legacy autoBackupData**：旧版单文件快照键，新版自动迁移到 `snapIndex` + `snap_<ts>` 多快照机制
  ```js
  // evictOldestSnapshot 中
  var legacy = localStorage.getItem('autoBackupData');
  if (legacy) { localStorage.removeItem('autoBackupData'); return true; }
  ```
- **纯题库数组导入**：导入 JSON 若为 `Array` 而非对象，自动包装为 `{ questions: raw }`
  ```js
  var data = Array.isArray(raw) ? { questions: raw } : raw;
  ```

### 7.5 跨设备数据迁移

#### 流程

1. **设备 A**：「数据 → 题库管理 → 导出 JSON」→ 下载 `题库备份_YYYY-MM-DD.json`
2. **传输文件**：通过 U 盘、网盘、邮件等任何方式
3. **设备 B**：「数据 → 题库管理 → 导入 JSON」→ 选择「合并」或「覆盖」模式
4. 完成

#### 合并模式 vs 覆盖模式

| 模式 | 行为 | 适用场景 |
|------|------|---------|
| 合并 | 题目按 id 去重合并；进度（错题/已做/收藏）累加；错次累加；SRS 覆盖 | 多设备数据汇总 |
| 覆盖 | 完全替换当前数据 | 设备迁移、数据恢复 |

### 7.6 数据迁移到服务器（不建议）

本项目设计为**纯前端 localStorage**，不支持将数据迁移到服务器：

- 无后端 API
- 无数据库
- 无用户系统

如需多设备实时同步，目前只能通过手动导入导出。如必须实现服务器同步，需自行开发后端服务（不在本项目维护范围内）。

---

## 附录：快速部署检查清单

### 本地部署

- [ ] HTML 文件已复制到目标目录
- [ ] 双击能用浏览器打开
- [ ] 内置题库自动加载
- [ ] localStorage 中可见 `tiku_v8_*` 键
- [ ] 报告图表能渲染（首次需联网）

### GitHub Pages 部署

- [ ] GitHub 仓库已创建（公开）
- [ ] HTML 已上传（建议重命名为 `index.html`）
- [ ] Settings → Pages 已开启
- [ ] 访问 `https://<user>.github.io/<repo>/` 正常
- [ ] HTTPS 证书有效

### nginx 部署

- [ ] nginx 已安装并运行
- [ ] HTML 已复制到根目录（建议 `index.html`）
- [ ] nginx 配置已测试（`nginx -t`）
- [ ] 防火墙开放 80/443 端口
- [ ] 通过 IP/域名可访问
- [ ] gzip 压缩已开启（可选）

### 升级验证

- [ ] 新版本 HTML 已替换
- [ ] 浏览器强制刷新（Ctrl+Shift+R）
- [ ] 题库与进度数据保留
- [ ] 主要功能正常
- [ ] 浏览器 Console 无错误

---

## 相关文档

- [USER_GUIDE.md](./USER_GUIDE.md) — 用户指南
- [DEVELOPMENT_GUIDE.md](./DEVELOPMENT_GUIDE.md) — 开发指南
- [DATABASE_DESIGN.md](./DATABASE_DESIGN.md) — 数据结构设计
