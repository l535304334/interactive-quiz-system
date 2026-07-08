# 数据流图

> 题库项目关键数据流分析
> 主文件：`[题库]交互式刷题页面.html`

---

## 1. 答题数据流

用户点击选项后的完整数据流。一次答题会同时更新多个状态集合，并触发异步快照备份。

```mermaid
sequenceDiagram
    autonumber
    actor U as 用户
    participant UI as 题目卡片
    participant HOC as handleOptionClick
    participant State as 全局状态
    participant Save as save* 函数
    participant LS as localStorage
    participant SRS as updateSRS
    participant Snap as triggerAutoBackup
    participant Render as renderQuestion

    U->>UI: 点击选项
    UI->>HOC: handleOptionClick(q, optIndex, letter, container)
    HOC->>HOC: 检查 container.dataset.answered<br/>已答则直接 return
    HOC->>HOC: 记录 undoSlot（qid + 旧状态快照）
    HOC->>HOC: 判定 isCorrect = (letter === q.answer)

    alt 答对
        HOC->>State: doneSet.add(q.id)
        HOC->>Save: saveDone()
        Save->>LS: lsSet('done', Array.from(doneSet))
        Save->>Snap: triggerAutoBackup()
        HOC->>State: incrementDailyCount()
        HOC->>SRS: updateSRS(q.id, true)
        HOC->>Render: updateStats / updateProgress / updateBadges
        Note over HOC: 答对时不进 wrongSet
    else 答错
        HOC->>State: wrongSet.add(q.id)
        HOC->>Save: saveWrong()
        Save->>LS: lsSet('wrong', Array.from(wrongSet))
        Save->>Snap: triggerAutoBackup()
        HOC->>State: wrongCountMap[q.id]++
        HOC->>Save: saveWrongCount()
        Save->>LS: lsSet('wrongCount', wrongCountMap)
        Save->>Snap: triggerAutoBackup()
        HOC->>State: doneSet.add(q.id)
        HOC->>Save: saveDone()
        HOC->>State: incrementDailyCount()
        HOC->>SRS: updateSRS(q.id, false)
        HOC->>Render: updateStats / updateProgress / updateBadges
    end

    HOC->>SRS: updateSRS(q.id, isCorrect)
    SRS->>State: 更新 srsMap[qid]
    SRS->>Save: saveSRS()
    Save->>LS: lsSet('srs', srsMap)
    SRS->>Render: updateReviewButton()

    HOC->>UI: 显示反馈卡（correct/wrong）
    Note over HOC: 答对时延迟 800ms 自动 goNext<br/>答错时导航栏切换为"重做 + 下一题"

    Snap->>Snap: 防抖 2 秒后执行
    Snap->>LS: saveSnapshot(全状态数据)
    Note over Snap: 写入 snap_<timestamp><br/>更新 snapIndex 与 lastBackup
```

---

## 2. SRS（SM-2 间隔复习）数据流

SM-2 算法的核心数据流，决定每道题的下次复习时间。

```mermaid
sequenceDiagram
    autonumber
    participant HOC as handleOptionClick
    participant USRS as updateSRS
    participant Map as srsMap
    participant Save as saveSRS
    participant LS as localStorage
    participant Btn as updateReviewButton

    HOC->>USRS: updateSRS(qid, correct)
    USRS->>Map: 读取 srsMap[qid]

    alt 题目首次进入 SRS 且答对
        USRS->>USRS: 直接 return（不创建记录）
        Note over USRS: 答对的题不进入复习队列
    else 题目首次进入 SRS 且答错
        USRS->>Map: 创建 {ef:2.5, interval:0, rep:0, next:今天, last:今天}
    else 已有 SRS 记录
        USRS->>USRS: q = correct ? 5 : 2
        USRS->>USRS: ef = ef + (0.1 - (5-q)*(0.08 + (5-q)*0.02))
        USRS->>USRS: if ef < 1.3 → ef = 1.3
        USRS->>USRS: last = today
        alt q < 3（答错）
            USRS->>USRS: rep = 0, interval = 1
        else q >= 3（答对）
            USRS->>USRS: rep === 0 → interval = 1
            USRS->>USRS: rep === 1 → interval = 6
            USRS->>USRS: rep >= 2 → interval = round(interval * ef)
            USRS->>USRS: rep++
        end
        USRS->>USRS: next = addDays(今天, interval)
    end

    USRS->>Map: srsMap[qid] = 更新后的状态
    USRS->>Save: saveSRS()
    Save->>LS: lsSet('srs', srsMap)
    USRS->>Btn: updateReviewButton()
    Btn->>Btn: 计算 getTodayReviewCount()<br/>srsMap[id].next <= today 的数量
    Btn->>Btn: 更新导航栏复习按钮徽章
```

---

## 3. 快照数据流

自动备份与恢复的数据流。`triggerAutoBackup` 使用 2 秒防抖避免频繁写盘。

```mermaid
sequenceDiagram
    autonumber
    participant Trig as triggerAutoBackup
    participant Timer as autoBackupTimer
    participant Save as saveSnapshot
    participant LS as localStorage
    participant Idx as snapIndex
    participant Evict as evictOldestSnapshot
    participant Last as lastBackup

    Note over Trig: 触发源：saveWrong / saveDone / saveFav /<br/>saveWrongCount / saveQuestions / saveSources / saveSRS

    Trig->>Timer: if (autoBackupTimer) clearTimeout
    Trig->>Timer: setTimeout(执行, 2000)

    Note over Timer: 2 秒内若无新触发则执行

    Timer->>Save: saveSnapshot(全状态数据)
    Save->>Save: ts = Date.now()
    Save->>Idx: 读取 lsGet('snapIndex', [])
    Save->>Idx: idx.push(ts)

    alt idx.length > MAX_SNAPSHOTS(10)
        loop while idx.length > 10
            Save->>Idx: old = idx.shift()
            Save->>LS: localStorage.removeItem('snap_' + old)
        end
    end

    Save->>LS: localStorage.setItem('snap_' + ts, JSON.stringify(data))

    alt 写入抛出异常（配额满）
        Save->>Evict: evictOldestSnapshot()
        alt 淘汰成功
            Evict->>Idx: idx.shift() + removeItem
            Save->>LS: 重试 setItem('snap_' + ts, data)
            alt 重试仍失败
                Save->>Idx: 从 idx 移除 ts
                Save->>Save: showQuotaToast() 提示用户
            end
        else 无可淘汰
            Save->>Idx: 从 idx 移除 ts
            Save->>Save: showQuotaToast() 提示用户
        end
    end

    Save->>Idx: lsSet('snapIndex', idx)
    Save->>Last: localStorage.setItem('lastBackup', String(ts))
```

### 恢复快照流程

```mermaid
sequenceDiagram
    autonumber
    participant U as 用户
    participant RS as restoreSnapshot
    participant C as showConfirm
    participant LS as localStorage
    participant State as 全局状态
    participant Save as save* 函数
    participant UI as 渲染层

    U->>RS: 点击"恢复"按钮
    RS->>C: showConfirm('恢复快照', 警告当前数据被覆盖)
    C-->>RS: Promise<ok>

    alt 用户取消
        RS->>RS: return
    else 用户确认
        RS->>LS: getItem('snap_' + ts)
        LS-->>RS: raw JSON
        RS->>RS: JSON.parse(raw)

        RS->>State: wrongSet = new Set(data.wrong)
        RS->>Save: saveWrong()
        RS->>State: doneSet = new Set(data.done)
        RS->>Save: saveDone()
        RS->>State: favSet = new Set(data.fav)
        RS->>Save: saveFav()
        RS->>State: wrongCountMap = data.wrongCount
        RS->>Save: saveWrongCount()
        RS->>State: questions = data.questions
        RS->>Save: saveQuestions()
        RS->>State: sources = data.sources
        RS->>Save: saveSources()
        RS->>State: srsMap = data.srs
        RS->>Save: saveSRS()
        RS->>State: dailyGoal / dailyCount / dailyDate / nextCid 等
        RS->>Save: lsSet(...) 各键

        RS->>UI: populateSourceFilter / populateExamSource
        RS->>UI: refreshCustomStats / refreshDeleteList / refreshCatList
        RS->>UI: renderCategoryManageList
        RS->>UI: applyFilter()
        RS->>UI: showEncouragement('恢复成功')
    end
```

---

## 4. 导入数据流

支持两种导入模式：覆盖（默认）与合并。合并模式用于跨设备同步进度。

```mermaid
sequenceDiagram
    autonumber
    actor U as 用户
    participant Input as 文件输入
    participant FR as FileReader
    participant Parse as JSON.parse
    participant State as 全局状态
    participant Save as save* 函数
    participant LS as localStorage
    participant UI as 渲染层
    participant Toast as showToast

    U->>Input: 选择 JSON 文件
    Input->>FR: reader.readAsText(file)
    FR->>Parse: e.target.result

    alt 解析失败
        Parse->>Toast: showToast('error', '导入失败', '文件格式不正确 - ' + err.message)
    else 解析成功
        Parse->>Parse: 兼容纯数组 → {questions: raw}
        Parse->>Parse: 读取 mergeImport 复选框

        alt 合并模式（merge）
            loop data.wrong 中每个 id
                State->>State: wrongSet.add(id)
            end
            Save->>LS: saveWrong()
            loop data.done 中每个 id
                State->>State: doneSet.add(id)
            end
            Save->>LS: saveDone()
            loop data.fav 中每个 id
                State->>State: favSet.add(id)
            end
            Save->>LS: saveFav()
            State->>State: wrongCountMap[id] += data.wrongCount[id]
            Save->>LS: saveWrongCount()
            loop data.questions 中每题
                alt id 不存在
                    State->>State: questions.push(q)
                end
            end
            Save->>LS: saveQuestions()
            loop data.sources 中每个类别
                alt sources 中不存在
                    State->>State: sources.push(s)
                end
            end
            Save->>LS: saveSources()
            loop data.srs 中每个 id
                State->>State: srsMap[id] = data.srs[id]
            end
            Save->>LS: saveSRS()
        else 覆盖模式
            State->>State: wrongSet = new Set(data.wrong)
            Save->>LS: saveWrong()
            State->>State: doneSet = new Set(data.done)
            Save->>LS: saveDone()
            State->>State: favSet = new Set(data.fav)
            Save->>LS: saveFav()
            State->>State: wrongCountMap = data.wrongCount
            Save->>LS: saveWrongCount()
            State->>State: questions = data.questions
            Save->>LS: saveQuestions()
            State->>State: sources = data.sources.slice()
            Save->>LS: saveSources()
            State->>State: srsMap = data.srs
            Save->>LS: saveSRS()
        end

        Parse->>State: nextCustomId 取较大值
        Save->>LS: lsSet('nextCustomId', nextCid)
        Parse->>State: studySec / todayStudySec / dailyGoal / dailyCount / dailyDate
        Save->>LS: lsSet(...) 各键

        Parse->>UI: populateSourceFilter / populateExamSource
        Parse->>UI: refreshCustomStats / refreshDeleteList / refreshCatList
        Parse->>UI: renderCategoryManageList
        Parse->>UI: applyFilter()
        Parse->>Toast: showToast('success', '导入成功', '共 N 道题')
    end
```

---

## 5. 数据一致性保障

### 5.1 写穿透策略

每次状态变更立即调用对应 `save*` 函数落盘，不存在"内存已改但未持久化"的窗口。代价是连续答题会触发多次 `lsSet`，但 `triggerAutoBackup` 通过 2 秒防抖避免了快照频繁写入。

### 5.2 配额保护链

`lsSet` 失败 → `evictOldestSnapshot` 淘汰最旧快照 → 重试一次 → 仍失败 → `showQuotaToast` 显式提示用户导出。任何失败都不静默吞掉。

### 5.3 快照索引与数据原子性

`snapIndex`（快照时间戳数组）与 `snap_<ts>`（快照数据）分开存储。写入顺序：
1. 先写 `snap_<ts>` 数据
2. 再写 `snapIndex` 索引

若第一步失败，索引未更新，不会出现"索引指向空数据"的脏状态。删除时同样先 `removeItem` 数据，再更新索引。

### 5.4 恢复操作的二次确认

`restoreSnapshot` / `deleteSnapshot` 必须通过 `showConfirm` 二次确认才能执行，避免误操作覆盖当前进度。恢复成功后通过 `showEncouragement` 显式反馈。

### 5.5 每日数据重置

`loadAll` 中检测 `dailyDate !== today`，自动重置 `dailyCount = 0` 与 `todayStudySeconds = 0` 并持久化。避免跨日累计错误。

---

## 6. 错误处理策略

| 错误类型 | 处理方式 | 用户感知 |
|---------|---------|---------|
| `lsGet` JSON 解析失败 | try-catch 返回默认值 `def` | 静默降级，使用默认空状态 |
| `lsSet` 配额超限 | `evictOldestSnapshot` 重试 → `showQuotaToast` | Toast 提示"存储已满，请导出备份后清理" |
| `JSON.parse` 导入失败 | try-catch → `showToast('error', '导入失败', err.message)` | 错误 Toast 显示具体原因 |
| `FileReader.onerror` | `showToast('error', '读取失败', ...)` | 错误 Toast |
| `saveSnapshot` 写入失败 | `evictOldestSnapshot` 重试 → 失败则回滚索引 + `showQuotaToast` | Toast 提示 |
| `restoreSnapshot` 解析失败 | `showToast('error', '恢复失败', e.message)` | 错误 Toast |
| `checkStorageUsage` 超 10MB | `showToast('warning', '存储占用较高', '已用 X MB')` | 警告 Toast |
| `getAllSources` 发现未登记类别 | 自动补入 `sources` 并 `saveSources` | 静默修复 |

### 6.1 错误显性化原则

- 所有 catch 块要么通过 Toast 显式反馈，要么通过返回默认值降级（且降级路径不影响功能正确性）。
- 批处理跳过时（如 `getSnapshotList` 中损坏的快照）静默跳过，但通过 `refreshDmMetaBar` 在 UI 上展示"上次快照"时间，间接提示备份状态。
- 不存在"默认成功"的写法：所有写操作要么显式成功，要么显式失败。
