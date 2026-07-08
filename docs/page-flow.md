# 页面流程图

> 题库项目（`[题库]交互式刷题页面.html`，v1.0.0）
> 纯前端 localStorage 持久化的通用刷题系统（内置临床医学题库样例）

---

## 1. 启动加载流程

页面打开后的初始化序列。`DOMContentLoaded` 触发后依次加载持久化数据、重建索引、渲染首屏。

```mermaid
flowchart TD
    A[浏览器打开 HTML 文件] --> B[DOMContentLoaded 事件]
    B --> C[loadAll 加载全部持久化数据]
    C --> D{dailyDate 等于今天?}
    D -- 否 --> E[重置 dailyCount=0<br/>todayStudySeconds=0<br/>持久化]
    D -- 是 --> F[跳过每日重置]
    E --> G
    F --> G[rebuildQuestionMap<br/>构建 id→question 映射]
    G --> H[populateSourceFilter<br/>填充科目下拉]
    H --> I[populateExamSource<br/>填充考试选题源]
    I --> J[refreshCustomStats<br/>刷新自定义题统计]
    J --> K[refreshDeleteList<br/>刷新删除列表]
    K --> L[refreshCatList<br/>刷新类别 datalist]
    L --> M[renderCategoryManageList<br/>渲染类别管理 UI]
    M --> N[updateGoalDisplay<br/>更新目标进度]
    N --> O[updateReviewButton<br/>更新复习按钮徽章]
    O --> P[applyFilter<br/>生成 filteredQuestions]
    P --> Q[renderQuestion<br/>渲染首屏题目]
    Q --> R[startTimer<br/>启动学习计时]
    R --> S{存在自动快照<br/>且进度为空?}
    S -- 是 --> T[checkAutoRestore<br/>弹窗提示恢复]
    S -- 否 --> U[初始化完成]
    T --> U
```

---

## 2. 答题主流程

用户答题时的核心交互流。一次答题会同时更新多个状态集合，答对自动跳下一题，答错停留。

```mermaid
flowchart TD
    A[用户点击选项] --> B[handleOptionClick]
    B --> C{container.dataset.answered<br/>已答?}
    C -- 是 --> Z[直接 return]
    C -- 否 --> D[记录 undoSlot<br/>qid + 旧状态快照]
    D --> E[判定 isCorrect<br/>letter === q.answer]
    E --> F[标记 container.dataset.answered]
    F --> G[显示反馈卡<br/>correct/wrong 样式]

    G --> H{isCorrect?}
    H -- 是 --> I[doneSet.add q.id]
    H -- 否 --> J[wrongSet.add q.id]
    J --> K[wrongCountMap q.id++]
    K --> L[doneSet.add q.id]

    I --> M[updateSRS q.id, correct]
    L --> M
    M --> N[incrementDailyCount]
    N --> O[updateStats / updateProgress / updateBadges]
    O --> P[saveWrong / saveDone / saveWrongCount / saveSRS]
    P --> Q[triggerAutoBackup<br/>防抖 2 秒]
    Q --> R[checkEncouragement<br/>每 50 题鼓励]

    R --> S{isCorrect?}
    S -- 是 --> T[延迟 800ms]
    T --> U[goNext 下一题]
    S -- 否 --> V[导航栏切换为<br/>重做 + 下一题]
    V --> W[等待用户操作]
```

---

## 3. 模式切换流程

4 种模式（错题 / 复习 / 收藏 / 浏览）的开关与组合逻辑。多模式同时开启进入复合模式（交集刷题）。

```mermaid
flowchart TD
    A[用户点击模式按钮] --> B{当前模式是否已开启?}
    B -- 已开启 --> C[关闭该模式]
    B -- 未开启 --> D[开启该模式]
    C --> E{剩余激活模式数}
    D --> E
    E -- 0 --> F[exitAllModes<br/>恢复默认筛选]
    E -- 1 --> G[单模式横幅<br/>updateModeBanners]
    E -- 2+ --> H[复合模式横幅<br/>显示组合 + 全部退出]

    F --> I[applyFilter]
    G --> I
    H --> I

    I --> J[根据激活模式筛选 questions]
    J --> K[未做在前 / 已做在后排序]
    K --> L{shuffleMode 开启?}
    L -- 是 --> M[随机打乱顺序]
    L -- 否 --> N[保持顺序]
    M --> O[写入 filteredQuestions]
    N --> O

    O --> P{filteredQuestions 为空?}
    P -- 是 --> Q{单模式 or 复合?}
    Q -- 单模式 --> R[自动关闭该模式<br/>Toast 提示无题目]
    Q -- 复合 --> S[保持开启<br/>渲染空状态提示]
    P -- 否 --> T[renderQuestion 渲染首题]
    R --> I
    S --> U[renderEmptyState]
```

---

## 4. 模拟考试流程

从选题源到交卷判分的完整考试流程。支持错题本组卷、双计时（单题 + 整场）、分页作答。

```mermaid
flowchart TD
    A[用户点击考试按钮 E] --> B[openExamModal]
    B --> C[展示考试设置弹窗<br/>选题源 / 题量 / 时限]
    C --> D{用户确认开始?}
    D -- 取消 --> Z[关闭弹窗]
    D -- 确认 --> E[startExam]

    E --> F[读取选题源<br/>支持错题本组卷]
    F --> G{题目数 > 0?}
    G -- 否 --> H[Toast 提示无题]
    H --> Z
    G -- 是 --> I[随机抽取指定题量]

    I --> J[重置 examAnswers / examStart / examQIdx]
    J --> K[renderExam 渲染首页<br/>20 题/页]
    K --> L[startExamTimer<br/>启动双计时]
    L --> M[用户作答中]

    M --> N{触发条件}
    N -- 单题超时 --> O[标记未答<br/>自动跳下一题]
    N -- 用户翻页 --> P[renderExam 渲染下一页]
    N -- 整场超时 --> Q[强制交卷]
    N -- 用户点击交卷 --> R[submitExam]

    O --> M
    P --> M
    Q --> R
    R --> S[判分 + 统计正确率]
    S --> T[错题自动收录 wrongSet]
    T --> U[显示考试结果弹窗<br/>分数 / 用时 / 错题数]
    U --> V[关闭考试弹窗]
    V --> W[applyFilter 刷新主界面]
```

---

## 5. 全局搜索流程

关键词搜索 + `#ID` 精准定位。搜索索引分批构建（每批 500 题）避免阻塞主线程。

```mermaid
flowchart TD
    A[用户按 / 或点击搜索] --> B[openSearchModal]
    B --> C{searchIndex 已构建?}
    C -- 否 --> D[buildSearchIndex<br/>分批构建索引]
    C -- 是 --> E[直接显示搜索框]
    D --> E

    E --> F[聚焦搜索输入框]
    F --> G[用户输入关键词]
    G --> H{输入以 # 开头?}

    H -- 是 --> I[按题号精准定位<br/>跳过 # 后解析数字]
    I --> J[getQuestionById<br/>查到则跳转]
    J --> K[关闭搜索弹窗]
    K --> L[jumpTo 跳转指定题]

    H -- 否 --> M[doSearch 关键词匹配<br/>题干 / 选项 / 答案]
    M --> N[取前 30 条结果]
    N --> O[高亮匹配片段<br/>escapeHtml 转义]
    O --> P[渲染结果列表]

    P --> Q{用户操作}
    Q -- ↑↓ 选择 --> R[高亮当前项]
    Q -- Enter --> S[跳转选中项]
    Q -- 点击结果 --> S
    Q -- Esc --> K

    R --> Q
    S --> K
    K --> L
```

---

## 6. 学习报告流程

Chart.js 4 图表 + 错题 Top 8 排名 + 智能学习建议。关闭时销毁图表实例释放内存。

```mermaid
flowchart TD
    A[用户点击报告按钮 R] --> B[openReportModal]
    B --> C[buildReport 编排函数]

    C --> D[computeSrcStats<br/>计算各科统计]
    D --> E[buildReportHTML<br/>生成报告 HTML]
    E --> F[updateReportSuggestions<br/>生成学习建议]
    F --> G[buildWrongTopHTML<br/>顽固错题 Top 8]
    G --> H[renderReportCharts<br/>渲染 4 张 Chart.js 图表]

    H --> I[30 天答题量折线]
    H --> J[正确率趋势折线]
    H --> K[各科完成度 + 正确率雷达]
    H --> L[错题分布 doughnut]

    I --> M[显示报告弹窗]
    J --> M
    K --> M
    L --> M

    M --> N{用户操作}
    N -- 查看报告 --> O[浏览图表与建议]
    N -- 关闭弹窗 --> P[关闭报告弹窗]
    P --> Q[chart.destroy ×4<br/>销毁 Chart.js 实例]
    Q --> R[清空 canvas 容器]
    R --> S[释放内存]
```

---

## 7. 数据管理流程

数据管理弹窗聚合了快照、导入导出、类别管理、自定义题等功能入口。

```mermaid
flowchart TD
    A[用户点击数据按钮] --> B[openExportModal]
    B --> C[显示数据管理弹窗]
    C --> D[refreshDmMetaBar<br/>刷新快照时间线]

    D --> E{用户操作}

    E -- 手动快照 --> F[autoBackupNow]
    F --> G[saveSnapshot<br/>写入 snap_timestamp]
    G --> H[Toast 提示成功]

    E -- 恢复快照 --> I[选择快照时间点]
    I --> J[restoreSnapshot]
    J --> K[showConfirm 二次确认]
    K --> L{用户确认?}
    L -- 否 --> C
    L -- 是 --> M[JSON.parse 快照数据]
    M --> N[恢复全部状态<br/>wrongSet / doneSet / favSet / srsMap 等]
    N --> O[save* 持久化]
    O --> P[刷新所有 UI]
    P --> Q[showEncouragement 恢复成功]

    E -- 删除快照 --> R[deleteSnapshot]
    R --> S[showConfirm 二次确认]
    S --> T{用户确认?}
    T -- 否 --> C
    T -- 是 --> U[移除 snap_timestamp]
    U --> V[更新 snapIndex]
    V --> W[Toast 提示成功]

    E -- 导出 --> X[exportData]
    X --> Y[收集全部状态]
    Y --> Z[生成 JSON Blob]
    Z --> AA[触发下载]

    E -- 导入 --> AB[选择 JSON 文件]
    AB --> AC[importQuestionsFromFile<br/>FileReader 读取]
    AC --> AD{解析成功?}
    AD -- 否 --> AE[Toast 错误提示]
    AD -- 是 --> AF{合并 or 覆盖?}
    AF -- 合并 --> AG[去重追加<br/>wrongSet / doneSet / favSet / questions]
    AF -- 覆盖 --> AH[替换全部状态]
    AG --> AI[save* 持久化]
    AH --> AI
    AI --> AJ[刷新所有 UI]
    AJ --> AK[Toast 成功提示]

    E -- 类别管理 --> AL[renderCategoryManageList]
    E -- 自定义题 --> AM[openCustomModal]
    E -- 重置进度 --> AN[resetProgressPrompt]
```

---

## 8. 模态弹窗导航关系

10 个模态弹窗的打开路径与关系。所有弹窗支持 Esc 关闭 + 点击遮罩关闭 + `focusModal` 自动聚焦。

```mermaid
flowchart LR
    MAIN([主界面]) --> EXAM[考试设置弹窗<br/>openExamModal]
    MAIN --> GOAL[每日目标弹窗<br/>openGoalModal]
    MAIN --> TIMER[计时与闹钟弹窗<br/>openTimerModal]
    MAIN --> SEARCH[全局搜索弹窗<br/>openSearchModal]
    MAIN --> REPORT[学习报告弹窗<br/>openReportModal]
    MAIN --> EXPORT[数据管理弹窗<br/>openExportModal]
    MAIN --> INFO[介绍弹窗<br/>openInfoModal]
    MAIN --> GUIDE[使用指南弹窗<br/>openGuideModal]
    MAIN --> ENC[鼓励弹窗<br/>showEncouragement]

    EXPORT --> CUSTOM[自定义题弹窗<br/>openCustomModal]
    EXPORT --> CATEGORY[类别管理面板<br/>内嵌于数据弹窗]

    EXAM --> RESULT[考试结果弹窗<br/>内嵌于考试流]
    TIMER --> ALARM[闹钟响铃弹窗<br/>内嵌于计时流]

    CUSTOM --> EXPORT2[返回数据弹窗]

    subgraph 关闭方式
        ESC[Esc 键]
        MASK[点击遮罩]
        BTN[关闭按钮]
    end

    ESC -.-> EXAM
    ESC -.-> GOAL
    ESC -.-> TIMER
    ESC -.-> SEARCH
    ESC -.-> REPORT
    ESC -.-> EXPORT
    ESC -.-> INFO
    ESC -.-> GUIDE
    ESC -.-> ENC
    ESC -.-> CUSTOM
```

---

## 9. 键盘快捷键流程

全局键盘事件处理。input/select/textarea 聚焦时屏蔽（保留 Ctrl+Z 与 Esc），弹窗打开时屏蔽模式切换键。

```mermaid
flowchart TD
    A[用户按键] --> B[keydown 事件]
    B --> C{焦点在 input/select/textarea?}
    C -- 是 --> D{是 Ctrl+Z 或 Esc?}
    D -- 是 --> E[放行处理]
    D -- 否 --> Z[屏蔽]
    C -- 否 --> F{有弹窗打开?}

    F -- 是 --> G{是 Esc?}
    G -- 是 --> H[关闭当前弹窗]
    G -- 否 --> I{是 Ctrl+Z?}
    I -- 是 --> J[undoLastAnswer]
    I -- 否 --> Z
    F -- 否 --> K[进入主快捷键分发]

    K --> L{按键分类}
    L -- A-I / 1-9 --> M{键盘模式开启?}
    M -- 是 --> N[选择对应选项]
    M -- 否 --> Z
    L -- 方向键 --> O[prevQuestion / nextQuestion]
    L -- Space --> P[下一题]
    L -- Backspace --> Q[redoQuestion 重做]
    L -- Ctrl+Z --> R[undoLastAnswer 撤销]
    L -- W --> S[toggleWrongMode]
    L -- F --> T[切换收藏]
    L -- X --> U[移出当前错题]
    L -- B --> V[toggleBrowseMode]
    L -- V --> W[toggleReviewMode]
    L -- E --> X[openExamModal]
    L -- R --> Y[openReportModal]
    L -- K --> AA[切换键盘模式]
    L -- G --> AB[openGoalModal]
    L -- T --> AC[openTimerModal]
    L -- H --> AD[openGuideModal]
    L -- M --> AE[openCustomModal]
    L -- 斜杠 --> AF[openSearchModal]
```

---

## 10. 计时与闹钟流程

学习计时 + 自定义闹钟 + 系统休息提醒三者协同。计时器每秒自增，闹钟先到者响。

```mermaid
flowchart TD
    A[页面加载] --> B[startTimer]
    B --> C[setInterval 每秒触发]
    C --> D[studySeconds++]
    D --> E[todayStudySeconds++]
    E --> F[updateTimerDisplay<br/>更新大圆环 + 文字]
    F --> G[updateGoalDisplay<br/>更新目标进度]

    G --> H{闹钟已启用?}
    H -- 否 --> I[检查系统休息提醒]
    H -- 是 --> J{当前时间 >= 闹钟时间?}

    J -- 否 --> I
    J -- 是 --> K[触发闹钟弹窗]
    K --> L{循环模式?}
    L -- 是 --> M[重置下次响铃时间<br/>可选 +24h]
    L -- 否 --> N[关闭闹钟]

    I --> O{已学习 >= 1 小时<br/>且距上次提醒 >= 30 分钟?}
    O -- 是 --> P[showEncouragement<br/>休息提醒]
    O -- 否 --> Q[继续计时]

    P --> Q
    Q --> R{页面可见?}
    R -- 是 --> C
    R -- 否 --> S[暂停 setInterval<br/>visibilitychange 恢复]
```

---

## 11. SRS 间隔复习调度流程

SM-2 算法决定每道题的下次复习时间。答对拉长间隔，答错次日复习。

```mermaid
flowchart TD
    A[updateSRS qid, correct] --> B{srsMap 中存在 qid?}

    B -- 否 --> C{correct?}
    C -- 是 --> D[直接 return<br/>答对的题不进复习队列]
    C -- 否 --> E[创建新记录<br/>ef=2.5, interval=0, rep=0<br/>next=今天, last=今天]

    B -- 是 --> F[读取现有记录]
    F --> G[q = correct ? 5 : 2]
    G --> H[ef = ef + 0.1 - 5-q * 0.08 + 5-q * 0.02]
    H --> I{ef < 1.3?}
    I -- 是 --> J[ef = 1.3]
    I -- 否 --> K[保持 ef]
    J --> L[last = 今天]
    K --> L

    L --> M{q < 3 答错?}
    M -- 是 --> N[rep = 0<br/>interval = 1<br/>next = 明天]
    M -- 否 --> O{rep 值}
    O -- 0 --> P[interval = 1<br/>rep = 1]
    O -- 1 --> Q[interval = 6<br/>rep = 2]
    O -- 2+ --> R[interval = round<br/>interval * ef<br/>rep++]

    P --> S[next = 今天 + interval 天]
    Q --> S
    R --> S
    N --> S

    S --> T[srsMap qid = 更新后状态]
    T --> U[saveSRS 持久化]
    U --> V[updateReviewButton<br/>刷新今日待复习数]
    V --> W[计算 getTodayReviewCount<br/>srsMap id.next <= today]
    W --> X[更新导航栏复习按钮徽章]
```

---

## 12. 页面状态机总览

主界面的核心状态机。用户在不同模式间切换时，`applyFilter` 重新生成 `filteredQuestions` 并触发渲染。

```mermaid
stateDiagram-v2
    [*] --> 默认模式: 页面加载 / loadAll

    默认模式 --> 错题模式: toggleWrongMode / W 键
    默认模式 --> 复习模式: toggleReviewMode / V 键
    默认模式 --> 收藏模式: toggleFavMode / F 键
    默认模式 --> 浏览模式: toggleBrowseMode / B 键

    错题模式 --> 复习模式: 叠加开启
    错题模式 --> 收藏模式: 叠加开启
    错题模式 --> 浏览模式: 叠加开启
    复习模式 --> 收藏模式: 叠加开启
    复习模式 --> 浏览模式: 叠加开启
    收藏模式 --> 浏览模式: 叠加开启

    错题模式 --> 默认模式: 再次点击 / 无题自动关闭
    复习模式 --> 默认模式: 再次点击 / 无题自动关闭
    收藏模式 --> 默认模式: 再次点击 / 无题自动关闭
    浏览模式 --> 默认模式: 再次点击

    复合模式 --> 默认模式: exitAllModes / 全部退出
    复合模式 --> 单模式: 关闭其中一个

    note right of 默认模式
        filteredQuestions = 全部题
        按未做在前排序
    end note

    note right of 复合模式
        filteredQuestions = 多模式交集
        横幅显示组合
        空时保持开启 + 空状态提示
    end note

    默认模式 --> 考试模式: openExamModal / E 键
    考试模式 --> 默认模式: 交卷 / 关闭弹窗

    默认模式 --> 搜索态: openSearchModal / 键
    搜索态 --> 默认模式: 跳转结果 / Esc

    默认模式 --> 报告态: openReportModal / R 键
    报告态 --> 默认模式: 关闭弹窗 / Esc

    默认模式 --> 数据管理态: openExportModal
    数据管理态 --> 默认模式: 关闭弹窗 / Esc
```
