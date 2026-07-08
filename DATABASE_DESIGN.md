# 题库 · 数据库设计文档

> 题库交互式刷题页面 — localStorage 数据结构设计
> 版本：v1.0.0 · 适用文件：`[题库]交互式刷题页面.html`

> **重要说明**：本项目为**纯前端 localStorage 持久化**应用，**无后端数据库**。
> 本文档描述的是浏览器 `localStorage` 中的数据结构设计。

---

## 目录

1. [数据存储概览](#1-数据存储概览)
2. [数据键值表](#2-数据键值表)
3. [questions 数据结构](#3-questions-数据结构)
4. [srs 数据结构](#4-srs-数据结构)
5. [wrongCount 数据结构](#5-wrongcount-数据结构)
6. [快照数据结构](#6-快照数据结构)
7. [数据关系图](#7-数据关系图)
8. [数据完整性策略](#8-数据完整性策略)
9. [数据迁移策略](#9-数据迁移策略)
10. [存储配额管理](#10-存储配额管理)

---

## 1. 数据存储概览

### 1.1 存储介质

- **介质**：浏览器 `localStorage`
- **位置**：用户本地浏览器，不上传任何服务器
- **容量限制**：通常 5-10MB（浏览器与源不同有差异）
- **生命周期**：除非用户主动清除浏览器数据，否则永久保留
- **隔离机制**：按「源」（协议+域名+端口）隔离

### 1.2 键前缀策略

所有业务键统一加前缀 `tiku_v8_`（常量 `STORAGE_PREFIX`）：

```js
var STORAGE_PREFIX = 'tiku_v8_';
```

**目的**：

- 与同源下其他应用的 localStorage 隔离
- 版本前缀（`v8`）用于未来数据格式大改时区分

> 历史遗留：还存在不带前缀的 `lastBackup`（旧字段）和 `autoBackupData`（已废弃），代码中自动迁移到新机制。

### 1.3 序列化方式

- 所有键值经 `JSON.stringify` 写入
- 读取时经 `JSON.parse` 解析
- 由 `lsGet(k, def)` / `lsSet(k, v)` 统一封装

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
    if (evictOldestSnapshot()) {
      try { localStorage.setItem(STORAGE_PREFIX + k, JSON.stringify(v)); return; }
      catch(e2) {}
    }
    showQuotaToast();
  }
}
```

**容错设计**：

- 读取出错返回默认值（不抛异常）
- 写入失败时自动淘汰最旧快照重试
- 重试仍失败则提示用户导出备份

### 1.4 数据流向概览

```
内存（运行时变量）
   ↑↓ loadAll() / saveXxx()
localStorage（持久化）
   ↑↓ triggerAutoBackup() / restoreSnapshot()
快照（snap_<ts>，最多 10 份）
   ↑↓ exportData() / importData()
JSON 文件（用户备份）
```

---

## 2. 数据键值表

### 2.1 业务数据键

所有键前加 `tiku_v8_` 前缀。`def` 列为 `lsGet` 调用时的默认值。

| 键名 | 类型 | 用途 | 默认值 | 持久化函数 |
|------|------|------|--------|-----------|
| `questions` | Array<Question> | 题库主表 | `null`（首次从 `ALL_QUESTIONS` 初始化） | `saveQuestions()` |
| `sources` | Array<string> | 类别名列表 | `['体格检查','共选','实验室检查','问诊诊断','心电图']` | `saveSources()` |
| `wrong` | Array<number> | 错题 ID 数组 | `[]` | `saveWrong()` |
| `done` | Array<number> | 已做题 ID 数组 | `[]` | `saveDone()` |
| `fav` | Array<number> | 收藏题 ID 数组 | `[]` | `saveFav()` |
| `wrongCount` | Object<number, number> | 错题次数映射 `{qid: count}` | `{}` | `saveWrongCount()` |
| `srs` | Object<number, SrsItem> | SM-2 复习数据 `{qid: {...}}` | `{}` | `saveSRS()` |
| `nextCustomId` | number | 下一自定义题 ID | `1000000` | `nextCustomId()` |
| `studySec` | number | 累计学习秒数 | `0` | `stopTimer()` |
| `todayStudySec` | number | 今日学习秒数 | `0` | `stopTimer()` |
| `goal` | number | 每日题数目标 | `0` | `saveGoal()` |
| `dailyCount` | number | 今日已做题数 | `0` | `incrementDailyCount()` |
| `dailyDate` | string | 今日日期（用于跨日重置） | `''` | 启动时检测 |
| `kMode` | boolean | 键盘模式开关 | `false` | `applyKeyboardMode()` |
| `shuffle` | boolean | 乱序模式开关 | `false` | `toggleShuffle()` |
| `dailyHistory` | Object<string, DailyStat> | 每日统计 `{YYYY-MM-DD: {...}}`（保留 90 天） | `{}` | `saveDailyHistory()` |
| `snapIndex` | Array<number> | 快照时间戳索引 | `[]` | `saveSnapshot()` |
| `alarmEnabled` | boolean | 闹钟是否启用 | `false` | `saveAlarm()` |
| `alarmMinutes` | number | 闹钟间隔（分钟） | `45` | `saveAlarm()` |
| `alarmLoop` | boolean | 闹钟循环模式 | `false` | `saveAlarm()` |
| `alarmNextAt` | number | 下次提醒时间戳（ms） | `0` | `startTimer()` / `saveAlarm()` |
| `streakDays` | number | 连续学习天数 | `1` | （在计时面板读取，由打卡逻辑写入） |

### 2.2 动态键（每份快照一个键）

| 键名 | 类型 | 用途 |
|------|------|------|
| `tiku_v8_snap_<timestamp>` | string（JSON） | 一份完整快照，`<timestamp>` 为毫秒时间戳 |

由 `saveSnapshot(data)` 写入，`restoreSnapshot(ts)` 读取，`deleteSnapshot(ts)` 删除。

### 2.3 不带前缀的兼容键

| 键名 | 类型 | 状态 | 处理 |
|------|------|------|------|
| `lastBackup` | string | 活跃 | 存储最近一次备份时间戳（无前缀） |
| `autoBackupData` | string（JSON） | 已废弃 | 旧版单文件快照，新版自动迁移到 `snap_<ts>` 机制后删除 |

---

## 3. questions 数据结构

### 3.1 单题字段

```typescript
interface Question {
  id: number;          // 题目 ID（内置题 0+，自定义题 1000000+）
  source: string;      // 类别名（如 "体格检查"）
  qnum: string;        // 原题号（如 "462"）
  subnum: string;      // 子题号（如 "1"）
  question: string;    // 题干文本
  options: string[];   // 选项数组（如 ["A.xxx", "B.xxx", ...]）
  answer: string;      // 正确答案字母（A-I）
  answerText?: string; // 答案文本（可选，导入时若缺失则用 options[letter] 兜底）
}
```

### 3.2 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | number | ✅ | 唯一标识。内置题从 0 递增；自定义题从 `1000000` 起递增 |
| `source` | string | ✅ | 类别名。必须存在于 `sources` 数组中（导入时自动补入） |
| `qnum` | string | ❌ | 原题号（教材编号），可为空字符串 |
| `subnum` | string | ❌ | 子题号，可为空字符串 |
| `question` | string | ✅ | 题干。导入时校验非空字符串 |
| `options` | string[] | ✅ | 选项数组。导入时校验非空数组 |
| `answer` | string | ✅ | 正确答案。必须是 A-I 范围内的字母（导入时校验 `letterIdx.hasOwnProperty(answer.toUpperCase())`） |
| `answerText` | string | ❌ | 答案完整文本。导入时若缺失则用 `options[letterIdx[answer]]` 兜底 |

### 3.3 ID 分配规则

| 范围 | 用途 | 分配方式 |
|------|------|---------|
| `0` ~ `ALL_QUESTIONS.length-1` | 内置题 | 写死在 `ALL_QUESTIONS` 常量中 |
| `1000000`+ | 自定义题 | 由 `nextCustomId()` 递增分配，`nextCustomId` 持久化到 localStorage |

### 3.4 查找优化

为避免 O(n) 查找，维护了内存中的索引：

```js
var questionByIdMap = null;  // {id: question}

function rebuildQuestionMap() {
  questionByIdMap = {};
  for (var i = 0; i < questions.length; i++) {
    questionByIdMap[questions[i].id] = questions[i];
  }
}

function getQuestionById(id) {
  if (!questionByIdMap) rebuildQuestionMap();
  return questionByIdMap[id];
}
```

- `saveQuestions()` 后自动 `rebuildQuestionMap()`
- `loadAll()` 后调用 `rebuildQuestionMap()`
- 后续 `getQuestionById(id)` 为 O(1)

### 3.5 题目示例

```json
{
  "id": 0,
  "source": "体格检查",
  "qnum": "462",
  "subnum": "1",
  "question": "麻疹( )",
  "options": [
    "A.米糠样脱屑",
    "B.片状脱屑",
    "C.银白色鳞状脱屑",
    "D.全身片状剥削性脱屑",
    "E.大水疱样脱屑"
  ],
  "answer": "A",
  "answerText": "A.米糠样脱屑"
}
```

---

## 4. srs 数据结构

### 4.1 算法

基于 **SM-2（SuperMemo 2）间隔重复算法**，根据答题结果动态调整每道题的复习间隔。

### 4.2 数据结构

```typescript
interface SrsItem {
  ef: number;        // 难度系数 Easiness Factor（1.3 ~ 2.5+）
  interval: number;  // 当前间隔天数
  rep: number;       // 已成功复习次数
  next: string;      // 下次复习日期 YYYY-MM-DD
  last: string;      // 上次复习日期 YYYY-MM-DD
}

type SrsMap = { [qid: number]: SrsItem };
```

### 4.3 字段说明

| 字段 | 类型 | 范围 | 说明 |
|------|------|------|------|
| `ef` | number | 1.3 ~ 2.5+（初始 2.5） | 难度系数。答错降低，答对升高。低于 1.3 强制设为 1.3 |
| `interval` | number | 1 ~ N（天） | 当前复习间隔。1 天起步，逐次拉长 |
| `rep` | number | 0 ~ N | 已成功复习次数。答错重置为 0 |
| `next` | string | YYYY-MM-DD | 下次复习日期。每次答题后更新为 `today + interval` |
| `last` | string | YYYY-MM-DD | 上次复习日期。每次答题后更新为 today |

### 4.4 算法逻辑（updateSRS）

```js
function updateSRS(qid, correct) {
  var s = srsMap[qid];
  if (!s) {
    if (correct) return;  // 答对且无记录 → 不入复习队列
    s = { ef: 2.5, interval: 0, rep: 0, next: todayStr(), last: todayStr() };
  }
  var q = correct ? 5 : 2;  // 答对 q=5，答错 q=2

  // 难度系数调整
  s.ef = s.ef + (0.1 - (5 - q) * (0.08 + (5 - q) * 0.02));
  if (s.ef < 1.3) s.ef = 1.3;

  s.last = todayStr();

  if (q < 3) {  // 答错：重置
    s.rep = 0;
    s.interval = 1;
  } else {  // 答对：拉长间隔
    if (s.rep === 0) s.interval = 1;        // 第 1 次：1 天后
    else if (s.rep === 1) s.interval = 6;   // 第 2 次：6 天后
    else s.interval = Math.round(s.interval * s.ef);  // 之后：interval × ef
    s.rep = s.rep + 1;
  }

  s.next = addDays(todayStr(), s.interval);
  srsMap[qid] = s;
  saveSRS();
  updateReviewButton();
}
```

### 4.5 复习触发

```js
function getTodayReviewCount() {
  var today = todayStr(), n = 0;
  for (var id in srsMap) {
    if (srsMap[id].next <= today) n++;
  }
  return n;
}
```

- 进入复习模式 → 只显示 `srsMap[qid].next <= 今日` 的题
- 顶部「复习」按钮有角标显示待复习数

### 4.6 数据示例

```json
{
  "0": {
    "ef": 2.36,
    "interval": 6,
    "rep": 2,
    "next": "2026-07-14",
    "last": "2026-07-08"
  },
  "5": {
    "ef": 1.5,
    "interval": 1,
    "rep": 0,
    "next": "2026-07-09",
    "last": "2026-07-08"
  }
}
```

---

## 5. wrongCount 数据结构

### 5.1 数据结构

```typescript
type WrongCountMap = { [qid: number]: number };
```

键为题目 ID，值为该题的累计错误次数。

### 5.2 用途

- 「顽固错题 Top 8」排行榜（按错误次数排序）
- 错题次数筛选（1次+/2次+/3次+/5次+/10次+）
- 学习报告中的错题分布统计

### 5.3 维护逻辑

- 答错时：`wrongCountMap[qid] = (wrongCountMap[qid] || 0) + 1`
- 答对不影响 wrongCount（保留历史记录）
- 「清空错题」时同时清空：`wrongCountMap = {}`
- 「删除题目」时清理无效 ID：
  ```js
  Object.keys(wrongCountMap).forEach(function(id){
    if (!validIds[id]) delete wrongCountMap[id];
  });
  ```
- 「合并导入」时累加：
  ```js
  Object.keys(data.wrongCount).forEach(function(id){
    wrongCountMap[id] = (wrongCountMap[id] || 0) + data.wrongCount[id];
  });
  ```

### 5.4 数据示例

```json
{
  "5": 1,
  "12": 3,
  "47": 1,
  "1024": 8
}
```

---

## 6. 快照数据结构

### 6.1 完整快照格式

每份快照是一个完整的「数据备份单元」，包含所有业务数据：

```typescript
interface Snapshot {
  // 进度集合
  wrong: number[];          // 错题 ID 数组
  done: number[];           // 已做题 ID 数组
  fav: number[];            // 收藏题 ID 数组

  // 错题统计
  wrongCount: { [qid: number]: number };

  // 学习时长
  studySec: number;         // 累计学习秒数
  todayStudySec: number;    // 今日学习秒数

  // 每日目标
  dailyGoal: number;
  dailyCount: number;
  dailyDate: string;

  // 自定义题 ID 计数器
  nextCustomId: number;

  // 类别与题库
  sources: string[];
  srs: { [qid: number]: SrsItem };
  // 注意：快照不保存 questions，因题库较大且变化少
  // 但导出 JSON 会保存 questions

  // 版本标识
  version: string;          // 'v1.0.0'
}
```

> **注意**：自动快照（`triggerAutoBackup` / `saveSnapshot`）**不包含 `questions`**，因为题库本身变化少且体积大。但「导出 JSON」（`exportData`）会包含完整 questions。

### 6.2 快照存储机制

#### 键格式

```
tiku_v8_snap_<timestamp>
```

其中 `<timestamp>` 为 `Date.now()` 毫秒时间戳，例如 `tiku_v8_snap_1720425600000`。

#### 索引键

`tiku_v8_snapIndex` 存储所有快照时间戳数组：

```json
[1720425600000, 1720512000000, 1720598400000]
```

#### 上限与淘汰

```js
var MAX_SNAPSHOTS = 10;  // 快照上限

// saveSnapshot 中
while (idx.length > MAX_SNAPSHOTS) {
  var old = idx.shift();
  localStorage.removeItem(STORAGE_PREFIX + 'snap_' + old);
}
```

超过 10 份时，**自动移除最旧的快照**（FIFO 队列）。

#### 配额满时的处理

```js
try {
  localStorage.setItem(STORAGE_PREFIX + 'snap_' + ts, JSON.stringify(data));
} catch(e) {
  if (evictOldestSnapshot()) {  // 淘汰最旧一份重试
    try { localStorage.setItem(...); } catch(e2) {
      // 重试仍失败：从索引中移除时间戳，提示用户
      showQuotaToast();
    }
  } else {
    showQuotaToast();
  }
}
```

### 6.3 快照触发时机

#### 自动触发（防抖 2 秒）

`triggerAutoBackup()` 在以下操作后触发：

- `saveQuestions()` — 题库变更（添加/删除自定义题、导入、重命名类别）
- `saveWrong()` / `saveDone()` / `saveFav()` — 进度变更
- `saveWrongCount()` — 错次变更

```js
function triggerAutoBackup() {
  if (autoBackupTimer) clearTimeout(autoBackupTimer);
  autoBackupTimer = setTimeout(function() {
    var data = { wrong, done, fav, wrongCount, studySec, todayStudySec,
                 dailyGoal, dailyCount, dailyDate, nextCustomId, sources, srs,
                 version: 'v1.0.0' };
    saveSnapshot(data);
  }, 2000);
}
```

**防抖设计**：连续操作 2 秒内只触发一次快照，避免频繁写入。

#### 手动触发

UI 上「立即快照」按钮直接调用 `saveSnapshot(data)`。

### 6.4 快照恢复

```js
function restoreSnapshot(ts) {
  var raw = localStorage.getItem(STORAGE_PREFIX + 'snap_' + ts);
  var data = JSON.parse(raw);

  if (Array.isArray(data.wrong)) { wrongSet = new Set(data.wrong); saveWrong(); }
  if (Array.isArray(data.done)) { doneSet = new Set(data.done); saveDone(); }
  if (Array.isArray(data.fav)) { favSet = new Set(data.fav); saveFav(); }
  if (data.wrongCount && typeof data.wrongCount === 'object') { wrongCountMap = data.wrongCount; saveWrongCount(); }
  if (Array.isArray(data.questions)) { questions = data.questions; saveQuestions(); }
  if (Array.isArray(data.sources)) { sources = data.sources.slice(); saveSources(); }
  if (data.srs && typeof data.srs === 'object') { srsMap = data.srs; saveSRS(); }
  // ... 其他字段
}
```

**字段类型校验**：恢复时每个字段都做类型检查（`Array.isArray` / `typeof`），不合规字段跳过，不影响其他字段恢复。

### 6.5 启动自动恢复

```js
function checkAutoRestore() {
  // 若进度数据为空，但存在快照
  if (wrongSet.size === 0 && doneSet.size === 0 && favSet.size === 0
      && !studySeconds && !dailyGoal) {
    var snapIdx = lsGet('snapIndex', []);
    var backup = null;
    if (snapIdx && snapIdx.length > 0) {
      var latestTs = snapIdx[snapIdx.length - 1];
      backup = localStorage.getItem(STORAGE_PREFIX + 'snap_' + latestTs);
    }
    // 回退到 legacy autoBackupData
    if (!backup) backup = localStorage.getItem('autoBackupData');
    if (backup) {
      // 弹出恢复提示框，显示备份数据规模，询问是否恢复
    }
  }
}
```

启动后 500ms 自动执行，发现空数据 + 有快照时弹出恢复对话框。

---

## 7. 数据关系图

### 7.1 文字关系描述

```
                        ┌─────────────────────┐
                        │   questions (主表)    │
                        │   Array<Question>    │
                        │                     │
                        │   id (PK)            │
                        │   source             │
                        │   question           │
                        │   options            │
                        │   answer             │
                        └──────────┬──────────┘
                                   │
                                   │ id 关联
                                   │
            ┌──────────────┬───────┼───────┬──────────────┐
            │              │       │       │              │
            ▼              ▼       ▼       ▼              ▼
       ┌─────────┐  ┌─────────┐ ┌─────┐ ┌──────────┐ ┌─────────┐
       │  wrong  │  │  done   │ │ fav │ │wrongCount│ │   srs   │
       │Array<id>│  │Array<id>│ │Array│ │{id:count}│ │{id:item}│
       └─────────┘  └─────────┘ └─────┘ └──────────┘ └─────────┘
            │              │       │       │              │
            └──────┬───────┴───────┴───────┴──────────────┘
                   │
                   │ 通过 qid 查询 questions
                   ▼
            ┌────────────┐
            │ questionByIdMap │
            │ {id: question}   │  ← 内存索引（非 localStorage）
            └────────────┘

  ┌─────────────┐
  │   sources   │  ← 类别列表（独立存储，但 source 字段需在此列表中）
  │ Array<str> │
  └─────────────┘

  ┌──────────────┐         ┌─────────────────┐
  │  snapIndex   │ ──────► │  snap_<ts>      │ (最多 10 份)
  │  Array<ts>   │         │  完整快照数据    │
  └──────────────┘         └─────────────────┘
```

### 7.2 关系说明

#### 7.2.1 questions 是主表

- 所有其他数据通过 `qid`（题目 ID）关联到 `questions`
- 题目 ID 在 `questions` 中是主键（PK），全局唯一
- 内置题 ID：`0` ~ `ALL_QUESTIONS.length - 1`
- 自定义题 ID：`1000000`+（由 `nextCustomId` 分配）

#### 7.2.2 进度集合（wrong / done / fav）

- 三个集合均为 `Array<number>`，元素为题目 ID
- **关系**：`qid ∈ questions.id`
- 运行时用 `Set<number>` 维护（`wrongSet` / `doneSet` / `favSet`）
- 持久化时转为数组：`Array.from(wrongSet)`
- 加载时转回 Set：`new Set(lsGet('wrong', []))`

#### 7.2.3 wrongCount 映射

- `Object<number, number>`，键为题目 ID，值为错次
- **关系**：`qid ∈ questions.id`
- 与 `wrong` 是「超集」关系：`wrongCount` 中的 qid 通常都在 `wrong` 中（除非被「移出错题」但保留错次记录）

#### 7.2.4 srs 映射

- `Object<number, SrsItem>`，键为题目 ID
- **关系**：`qid ∈ questions.id`
- 与 `wrong` 关系密切：错题答错时进入 srs（`updateSRS(qid, false)`），答对时更新 srs（`updateSRS(qid, true)`）
- 答对但无 srs 记录时，不入 srs 队列（`if (correct) return`）

#### 7.2.5 sources 与 questions.source

- `sources` 是类别名数组，独立存储
- `questions[i].source` 必须在 `sources` 中（或会被自动补入）
- 重命名类别时同步更新所有 `questions[i].source`（保持引用完整性）
- 删除类别时仅允许删除空类别（无题目引用）

#### 7.2.6 questionByIdMap（内存索引）

- 不是 localStorage 键，是运行时内存对象
- 由 `rebuildQuestionMap()` 在启动或题库变更时重建
- 提供 O(1) 查找：`getQuestionById(id)`

#### 7.2.7 snapIndex 与 snap_<ts>

- `snapIndex` 是时间戳数组，索引所有快照
- `snap_<ts>` 是每个时间戳对应的快照数据
- 一对多关系：`snapIndex` 中的一个时间戳对应一个 `snap_<ts>` 键
- 上限 10 份，FIFO 淘汰

### 7.3 关系完整性维护

#### 删除题目时清理关联

```js
function deleteBySource() {
  // ...
  var validIds = {};
  questions.forEach(function(q){ validIds[q.id] = true; });
  wrongSet = new Set(Array.from(wrongSet).filter(function(id){ return validIds[id]; }));
  saveWrong();
  doneSet = new Set(Array.from(doneSet).filter(function(id){ return validIds[id]; }));
  saveDone();
  favSet = new Set(Array.from(favSet).filter(function(id){ return validIds[id]; }));
  saveFav();
  Object.keys(wrongCountMap).forEach(function(id){
    if (!validIds[id]) delete wrongCountMap[id];
  });
  saveWrongCount();
  // srs 不会自动清理（保留历史），但 srs 中的无效 id 在统计时会被忽略
}
```

#### 重命名类别时同步更新

```js
function renameCategoryPrompt(oldName) {
  // ...
  for (var j = 0; j < questions.length; j++) {
    if (questions[j].source === oldName) {
      questions[j].source = newName;
    }
  }
  var idx = sources.indexOf(oldName);
  if (idx >= 0) sources[idx] = newName;
  saveSources();
  saveQuestions();
}
```

---

## 8. 数据完整性策略

### 8.1 类型校验

#### 8.1.1 数组校验

```js
if (Array.isArray(data.wrong)) { wrongSet = new Set(data.wrong); saveWrong(); }
if (Array.isArray(data.done)) { doneSet = new Set(data.done); saveDone(); }
if (Array.isArray(data.fav)) { favSet = new Set(data.fav); saveFav(); }
if (Array.isArray(data.questions)) { questions = data.questions; saveQuestions(); }
if (Array.isArray(data.sources)) { sources = data.sources.slice(); saveSources(); }
```

不合规字段（非数组）会被跳过，不影响其他字段恢复。

#### 8.1.2 对象校验

```js
if (data.wrongCount && typeof data.wrongCount === 'object') { wrongCountMap = data.wrongCount; saveWrongCount(); }
if (data.srs && typeof data.srs === 'object') { srsMap = data.srs; saveSRS(); }
```

#### 8.1.3 数值校验

```js
if (typeof data.todayStudySec === 'number') {
  todayStudySeconds = data.todayStudySec;
  lsSet('todayStudySec', todayStudySeconds);
}
```

### 8.2 JSON 解析容错

所有 `JSON.parse` 都用 `try-catch` 包裹：

```js
function lsGet(k, def) {
  try {
    var v = localStorage.getItem(STORAGE_PREFIX + k);
    return v ? JSON.parse(v) : def;
  } catch(e) {
    return def;  // 解析失败返回默认值
  }
}
```

```js
// importData 中
try {
  var raw = JSON.parse(e.target.result);
} catch(err) {
  showToast('error', '导入失败', '文件格式不正确 - ' + err.message);
}
```

### 8.3 字段完整性校验（导入时）

自定义题添加时对每条记录做完整校验：

```js
var letterIdx = {'A':0,'B':1,'C':2,'D':3,'E':4,'F':5,'G':6,'H':7,'I':8};

for (var i = 0; i < arr.length; i++) {
  var item = arr[i];
  var reason = null;
  if (!item || typeof item !== 'object') reason = '非对象';
  else if (!item.question || typeof item.question !== 'string' || !item.question.trim())
    reason = 'question 缺失/空';
  else if (!Array.isArray(item.options) || item.options.length === 0)
    reason = 'options 缺失/空数组';
  else if (!item.answer || typeof item.answer !== 'string' || !item.answer.trim())
    reason = 'answer 缺失/空';
  else if (!letterIdx.hasOwnProperty(item.answer.toUpperCase()))
    reason = 'answer 不在 A-I 范围';

  if (reason) {
    skipped++;
    skipReasons[reason] = (skipReasons[reason] || 0) + 1;
    continue;
  }
  // ... 添加题目
}
```

**失败显性化**：跳过数量和原因都在 Toast 中展示，不藏于日志：

```js
if (skipped > 0) {
  var reasonText = Object.keys(skipReasons).map(function(r){
    return r + '(' + skipReasons[r] + ')';
  }).join('、');
  showToast('success', '添加 ' + added + ' 题（跳过 ' + skipped + ' 题）', '跳过原因：' + reasonText);
}
```

### 8.4 引用完整性

#### 8.4.1 类别引用完整性

- `questions[i].source` 必须在 `sources` 数组中
- `getAllSources()` 会自动补全未登记的类别：
  ```js
  for (i = 0; i < questions.length; i++) {
    var s = questions[i].source;
    if (s && !seen[s]) { seen[s] = true; result.push(s); sources.push(s); dirty = true; }
  }
  if (dirty) saveSources();
  ```
- 删除类别时禁止删除有题目的类别（`canDelete = (n === 0)`）

#### 8.4.2 题目 ID 引用完整性

- 删除题目时同步清理 `wrong` / `done` / `fav` / `wrongCount` 中的无效 ID
- `srs` 中的无效 ID 不主动清理（保留历史记录），但在统计时通过 `getQuestionById` 判空跳过

### 8.5 跨日重置逻辑

```js
var today = new Date().toDateString();
if (dailyDate !== today) {
  dailyCount = 0;
  dailyDate = today;
  todayStudySeconds = 0;
  lsSet('dailyCount', 0);
  lsSet('dailyDate', today);
  lsSet('todayStudySec', 0);
}
```

启动时检测日期变更，重置今日数据。

### 8.6 dailyHistory 保留期

```js
function loadDailyHistory() {
  dailyHistory = lsGet('dailyHistory', {});
  var keys = Object.keys(dailyHistory);
  if (keys.length > 90) {
    keys.sort();
    while (keys.length > 90) { delete dailyHistory[keys.shift()]; }
    lsSet('dailyHistory', dailyHistory);
  }
}
```

每日历史统计保留最近 90 天，超出自动删除最旧的。

---

## 9. 数据迁移策略

### 9.1 version 字段

导出的 JSON 与快照数据包含 `version` 字段：

```js
{
  // ... 业务数据
  version: 'v1.0.0'
}
```

- 当前版本：`v1.0.0`
- 用于未来版本升级时识别数据格式
- 导入时如版本不匹配，系统按字段名兼容性处理

### 9.2 同版本升级

无需迁移：

- 直接替换 HTML 文件
- localStorage 数据完全兼容
- 用户无感知

### 9.3 兼容字段新增

新版本新增字段（如 `theme` 偏好），旧数据无此字段：

- 代码用 `lsGet('theme', 'default')` 兜底默认值
- 用户无感知，无需手动迁移

### 9.4 字段格式变更（未来场景）

如某天 `wrong` 从数组改为对象：

1. 检测旧格式：`Array.isArray(wrong)`
2. 转换为新格式
3. 保存到新键（如 `wrong_v2`）
4. 保留旧键一段时间，提示用户导出备份
5. N 个版本后删除旧键

### 9.5 完全重构（不推荐）

如数据结构完全重做：

1. 升级 `STORAGE_PREFIX` 为 `tiku_v9_`
2. 启动时检测旧前缀数据
3. 弹出迁移确认框
4. 用户确认后转换并写入新前缀
5. 保留旧前缀数据 30 天，期间可回滚

### 9.6 已实现的兼容处理

#### 9.6.1 旧版 autoBackupData 兼容

```js
function evictOldestSnapshot() {
  var idx = lsGet('snapIndex', []);
  if (!idx || idx.length === 0) {
    // 旧版单文件快照兼容
    var legacy = localStorage.getItem('autoBackupData');
    if (legacy) {
      localStorage.removeItem('autoBackupData');
      return true;
    }
    return false;
  }
  // ...
}

function checkAutoRestore() {
  // ...
  // 回退到 legacy autoBackupData
  if (!backup) {
    try { backup = localStorage.getItem('autoBackupData'); } catch(e) {}
  }
}
```

旧版的 `autoBackupData` 单文件快照机制在新版中自动迁移到 `snapIndex` + `snap_<ts>` 多快照机制，并删除旧键。

#### 9.6.2 纯题库数组导入兼容

```js
function importData(event) {
  // ...
  var raw = JSON.parse(e.target.result);
  // 兼容纯题库数组
  var data = Array.isArray(raw) ? { questions: raw } : raw;
  // ...
}
```

导入 JSON 若为 `Array` 而非对象，自动包装为 `{ questions: raw }`，兼容纯题库数组格式。

### 9.7 迁移触发时机

| 时机 | 处理 |
|------|------|
| 应用启动 `loadAll()` | 加载所有数据到内存，跨日重置 |
| 应用启动后 500ms `checkAutoRestore()` | 检测空数据并提示恢复快照 |
| `evictOldestSnapshot()` | 检测并清理 legacy `autoBackupData` |
| `importData()` | 检测导入格式（数组 vs 对象），按字段名兼容恢复 |

---

## 10. 存储配额管理

### 10.1 浏览器配额限制

| 浏览器 | localStorage 配额 | 备注 |
|--------|------------------|------|
| Chrome | ~5-10MB（按源） | 实际可用约 5MB |
| Firefox | ~5MB | 严格限制 |
| Safari | ~5MB | iOS Safari 无痕模式不持久 |
| Edge | ~5-10MB（按源） | 同 Chrome |

### 10.2 配额监控

```js
var STORAGE_QUOTA_WARN_BYTES = 10 * 1024 * 1024;  // 10MB 警告阈值

function checkStorageUsage() {
  if (!navigator.storage || !navigator.storage.estimate) return;
  navigator.storage.estimate().then(function(est) {
    if (est.usage > STORAGE_QUOTA_WARN_BYTES) {
      var usageMB = (est.usage / 1024 / 1024).toFixed(1);
      var quotaMB = est.quota ? (est.quota / 1024 / 1024).toFixed(0) : '?';
      showToast('warning', '存储占用较高',
        '已用 ' + usageMB + 'MB / ' + quotaMB + 'MB，建议导出备份');
    }
  }).catch(function(){});
}
```

- 在 `saveQuestions()` 后自动调用
- 超 10MB 触发警告 Toast
- 提示用户导出备份

> **注意**：`navigator.storage.estimate()` 在 Safari 15.2 之前不支持，会静默跳过。

### 10.3 自动快照滚动

```js
var MAX_SNAPSHOTS = 10;

function saveSnapshot(data) {
  var ts = Date.now();
  var idx = lsGet('snapIndex', []);
  idx.push(ts);
  while (idx.length > MAX_SNAPSHOTS) {
    var old = idx.shift();
    try { localStorage.removeItem(STORAGE_PREFIX + 'snap_' + old); } catch(e) {}
  }
  // ... 写入新快照
}
```

- 最多保留 10 份快照
- 超出时自动移除最旧的（FIFO）
- 写入失败时调用 `evictOldestSnapshot()` 淘汰一份重试

### 10.4 配额满时的处理

```js
function lsSet(k, v) {
  try {
    localStorage.setItem(STORAGE_PREFIX + k, JSON.stringify(v));
  } catch(e) {
    if (evictOldestSnapshot()) {  // 淘汰最旧一份快照
      try {
        localStorage.setItem(STORAGE_PREFIX + k, JSON.stringify(v));
        return;
      } catch(e2) {}
    }
    showQuotaToast();  // 仍失败 → 提示用户导出
  }
}
```

**降级策略**：

1. 第一次写入失败 → 淘汰最旧快照
2. 淘汰后重试 → 成功则继续，失败则提示
3. 提示用户：`showQuotaToast()` → 「请导出备份后清理，或删除部分快照释放空间」

### 10.5 用户侧配额管理

通过 UI「数据 → 题库管理 → 快照列表」：

- 查看所有快照（按时间倒序）
- 每条显示时间戳、错题数、已做题数、收藏数
- 手动「删除」单个快照释放空间
- 手动「恢复」从快照还原数据

### 10.6 容量优化建议

#### 10.6.1 题库优化

- 题库是占空间最大的数据（数千题约 1-2MB）
- 删除不用的自定义题可释放空间
- 避免重复导入相同题库

#### 10.6.2 快照管理

- 定期手动清理旧快照
- 导出 JSON 后可清空快照（保留最近 1-2 份即可）

#### 10.6.3 dailyHistory 限制

- 系统已限制保留 90 天
- 无需用户干预

### 10.7 容量估算参考

| 数据类型 | 单条大小 | 1000 条 | 10000 条 |
|---------|---------|---------|----------|
| Question | ~500B | 500KB | 5MB |
| SrsItem | ~100B | 100KB | 1MB |
| Snapshot（不含 questions） | ~50KB | - | - |
| Snapshot（含 questions） | ~1-2MB | - | - |

---

## 附录：数据流时序图

### A.1 启动时数据流

```
浏览器加载 HTML
    ↓
执行 loadAll()
    ↓
┌─────────────────────────────────────────┐
│ 读取 wrong / done / fav / wrongCount  │
│ 读取 srs / dailyHistory                │
│ 读取 sources（无则用 DEFAULT_SOURCES） │
│ 读取 questions（无则用 ALL_QUESTIONS） │
│ 读取 studySec / todayStudySec / goal   │
│ 读取 dailyCount / dailyDate            │
└─────────────────────────────────────────┘
    ↓
rebuildQuestionMap()  ← 建立 id 索引
    ↓
检测 dailyDate 跨日 → 重置 todayStudySec / dailyCount
    ↓
applyKeyboardMode() / updateShuffleBtn()
    ↓
populateSourceFilter() / applyFilter()
    ↓
startTimer()  ← 启动每秒计时
    ↓
500ms 后 checkAutoRestore()  ← 检测空数据
    ↓
requestIdleCallback(buildSearchIndex)  ← 空闲时构建搜索索引
```

### A.2 答题时数据流

```
用户点击选项
    ↓
handleOptionClick(q, idx, letter, container)
    ↓
┌────────────────────────────────────────┐
│ 判定对错                                │
│ 对：doneSet.add(q.id) → saveDone()    │
│ 错：wrongSet.add(q.id) → saveWrong()  │
│      wrongCountMap[q.id]++ → saveWrongCount() │
└────────────────────────────────────────┘
    ↓
updateSRS(q.id, correct) → saveSRS()
    ↓
recordDailyStat(correct ? 'correct' : 'wrong')
    ↓
incrementDailyCount()  ← 今日计数 +1
    ↓
各 saveXxx() 触发 triggerAutoBackup()  ← 防抖 2 秒
    ↓
2 秒后 saveSnapshot(data)  ← 写入 snap_<ts>
    ↓
checkStorageUsage()  ← 检查配额
```

### A.3 导出导入数据流

```
导出：
exportData()
    ↓
组装 {wrong, done, fav, wrongCount, studySec, todayStudySec,
      dailyGoal, dailyCount, dailyDate, nextCustomId, sources,
      questions, srs, version: 'v1.0.0'}
    ↓
JSON.stringify(data, null, 2)
    ↓
new Blob([json], {type: 'application/json'})
    ↓
URL.createObjectURL(blob)
    ↓
a.download = '题库备份_YYYY-MM-DD.json'
    ↓
a.click()  ← 触发下载

导入：
input[type=file] change 事件
    ↓
FileReader.readAsText(file)
    ↓
JSON.parse(result)
    ↓
兼容处理：Array → {questions: raw}
    ↓
合并 or 覆盖模式分支
    ↓
逐字段类型校验 → 写入对应 Set/Map/Array
    ↓
各 saveXxx() 持久化
    ↓
triggerAutoBackup()  ← 触发快照
```

---

## 相关文档

- [USER_GUIDE.md](./USER_GUIDE.md) — 用户指南
- [DEVELOPMENT_GUIDE.md](./DEVELOPMENT_GUIDE.md) — 开发指南
- [DEPLOYMENT_GUIDE.md](./DEPLOYMENT_GUIDE.md) — 部署指南
