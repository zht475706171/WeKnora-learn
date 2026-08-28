# 12 · Tier 1 → Tier 2 → Tier 3 全过程串联

> 把前 11 篇笔记串起来,一篇看完整条链路:从文档进来,到切出 chunk,每层干啥、怎么衔接、失败怎么兜底。

---

## 一、整体架构:3 层策略链

```
文档(text)
  ↓
ProfileDocument(扫一遍,统计结构信号)
  ↓
SelectStrategy(根据 profile 选链)
  ↓
┌─────────────────────────────────────────────────────────┐
│  chain = [Tier1, Tier2, Tier3]  (按文档特征动态选)      │
│                                                          │
│  for tier in chain:                                      │
│      out = runTier(tier, text, cfg, profile)             │
│      if ValidateChunks(out) 通过: return out             │
│      else: 记录失败原因,继续下个 tier                   │
│                                                          │
│  都失败 → 返回 Tier3 的最后结果(不空)                  │
└─────────────────────────────────────────────────────────┘
  ↓
[]Chunk(每个带 Content + ContextHeader + Start/End + Seq)
```

**3 层不是平行选择,是瀑布降级链**:第一层失败就落第二层,第二层失败落第三层,第三层是兜底,失败也返回结果。

---

## 二、入口:ProfileDocument(体检)

`profiler.go::ProfileDocument` 扫一遍文档,统计:

### 统计的结构信号

| 信号 | 记到哪 |
|------|--------|
| `#` 标题(1-6 级) | `MdHeadingCounts[1..6]` + `MdHeadingTotal` |
| `1.1 Foo` 编号标题 | `NumberedSectionCount` |
| `第一章/Chapter 1/Kapitel 1` | `ChineseChapterCount / EnglishChapterCount / GermanChapterCount` |
| `INTRODUCTION` 全大写 | `AllCapsShortLineCount` |
| `---` 视觉分隔 | `VisualSepCount` |
| `\f` 分页符 | `FormFeedCount` |
| `Page 2 of 10` 页脚 | `RepeatedFooterCount` |
| `\n\n\n` 多空行 | `BlankParagraphBreaks` |
| `\|...\|` 表格行 | `HasTables` |
| ` ``` ` 代码块 | `HasCode` + `CodeRatio` |
| 语言 | `DetectedLangs`(采 4096 字符样本) |

**关键设计**:
- 跳过 ` ``` ` 代码块内的行(`inFence` 状态机),防代码注释里的 `#` 被误识别
- 语言检测只采样 4096 字符(防大文档 O(N) 扫描)

### 算 DominantHeadingLevel(主导标题层级)

```go
for level := 1; level <= 6; level++ {
    if p.MdHeadingCounts[level] >= 3 {
        return level
    }
}
for level := 6; level >= 1; level-- {
    if p.MdHeadingCounts[level] > 0 {
        return level
    }
}
return 0
```

**2 段逻辑**:
1. **首选**:从 H1 往 H6 扫,第一个出现 **≥3 次** 的层级(防止文档只有 1 个 H1 切了没意义)
2. **退化**:实在没 ≥3 次的,从最深的往回找,有任何出现的就用
3. 都没有 → 返回 0(没 Markdown 标题)

---

## 三、选链:SelectStrategy

`profiler.go::SelectStrategy` 根据 profile 算 chain:

```go
func SelectStrategy(p *DocProfile) []StrategyTier {
    var chain []StrategyTier

    // Tier 1 候选:有 Markdown 标题
    if p.MdHeadingTotal >= 3 && p.HeadingDensity() > 0.005 && p.DominantHeadingLevel() > 0 {
        chain = append(chain, TierHeading)
    }

    // Tier 2 候选:有结构信号
    if p.HeuristicMarkerTotal() >= 5 || p.FormFeedCount > 0 ||
        p.GermanChapterCount+p.EnglishChapterCount+p.ChineseChapterCount > 0 {
        chain = append(chain, TierHeuristic)
    }

    // Tier 3 永远加(兜底)
    chain = append(chain, TierLegacy)
    return chain
}
```

### 3 种典型 chain

| 文档特征 | chain |
|---------|------|
| 有 `#` 标题 + 有 `第一章` | `[Tier1, Tier2, Tier3]` |
| 无 `#` 标题,但有 `第一章/1.1/\f` | `[Tier2, Tier3]` |
| 无任何结构信号 | `[Tier3]` |

**Tier 3 永远在链尾**,保证一定有结果返回。

### 用户可以强制指定 strategy

```go
case StrategyHeading:   return []StrategyTier{TierHeading, TierLegacy}
case StrategyHeuristic: return []StrategyTier{TierHeuristic, TierLegacy}
case StrategyRecursive: return []StrategyTier{TierLegacy}
case StrategyAuto:      fallthrough  // 默认,走 ProfileDocument + SelectStrategy
```

---

## 四、链式调度:Split(strategy.go)

```go
func Split(text string, cfg SplitterConfig) []Chunk {
    cfg = ensureDefaults(cfg)  // 填默认值(ChunkSize=512, Overlap=80)
    chain, profile := resolveChainWithProfile(text, cfg)
    totalChars := len([]rune(text))

    var lastOut []Chunk
    for i, tier := range chain {
        out := runTier(tier, text, cfg, profile)
        if v := ValidateChunks(out, totalChars, cfg.ChunkSize); v.OK {
            return out  // ★ 第一层通过的就用,直接返回
        }
        // 失败:记录原因,继续下个 tier
        if tier == TierLegacy && i == len(chain)-1 {
            lastOut = out  // Tier 3 是最后,即使失败也保留
        }
    }
    if lastOut != nil { return lastOut }
    return SplitText(text, cfg)  // 终极兜底
}
```

**核心**:**第一层通过的就用,直接返回**。不是 3 层都跑取最好的,是瀑布式 —— 第一个通过校验的就停。

### runTier 分发

```go
func runTier(tier, text, cfg, profile) []Chunk {
    switch tier {
    case TierHeading:   return splitByHeadings(text, cfg, profile)
    case TierHeuristic: return splitByHeuristics(text, cfg, profile)
    case TierLegacy:    return SplitText(text, cfg)
    }
}
```

---

## 五、ValidateChunks(校验器,决定降级)

`validator.go::ValidateChunks` 4 个判据:

```go
// 判据 1:碎 chunk 数 > 总数/4 且 > 2
if tinyCount > len(chunks)/4 && tinyCount > 2 {
    return ValidationResult{Reason: "too many tiny chunks"}
}

// 判据 2:最大 chunk 都 < ChunkSize/4 且文档 > ChunkSize
if maxLen < chunkSize/4 && totalChars > chunkSize {
    return ValidationResult{Reason: "all chunks far below target size"}
}

// 判据 3:任何 chunk > 2*ChunkSize
if maxLen > 2*chunkSize {
    return ValidationResult{Reason: "chunk exceeds 2x target size"}
}

// 判据 4:1 chunk 且文档 > 2*ChunkSize
if len(chunks) == 1 && totalChars > 2*chunkSize {
    return ValidationResult{Reason: "single chunk for large document"}
}
```

**碎定义**:`< 50 字`(除最后一个,尾部残留小是正常的)

**校验失败 → 整层作废,降级到下个 tier**。

---

## 六、Tier 1:heading(标题切)

### 适用
有清晰 `# ## ###` Markdown 标题的文档。

### 核心流程(7 步)

**Step 1:体检**(profile 已在链头做)

**Step 2:选主导层级**(`DominantHeadingLevel`)
- H1 5 次 ≥3 → primaryLevel=1,按 H1 切
- H1 才 1 次,H2 5 次 → primaryLevel=2,按 H2 切
- 都没 → primaryLevel=0,降级

**Step 3:找切点**(`findHeadingBoundaries`)
- 再扫一遍文本,跳过 ` ``` ` 代码块
- 匹配 `MarkdownHeadingPattern = (?m)^(#{1,6})\s+(.+?)\s*#*\s*$`
- 命中且 `level ≤ primaryLevel` 的记为切点

**Step 4:初始化标题栈**(`NewHeadingHierarchy`)
```go
stack = ["", "", "", "", "", ""]  // 6 个空,对应 H1-H6
```

**Step 5:逐个 section 切**

每两个相邻切点之间是一段 section。对每个 section:
- (a) section 起始是标题行 → `hierarchy.Observe(标题行)` 更新栈
- (b) 取栈快照当面包屑:`breadcrumb := hierarchy.BreadcrumbWithHashes()` → `# XX\n## YY`
- (c) 同时扫 section 内更深层标题(`observeSubHeadings`)让栈同步
- (d) 判断大小:
  - `面包屑字符数 + section 字符数 ≤ ChunkSize` → 一个 chunk,Content=section 内容,ContextHeader=面包屑
  - 否则 → 调 `SplitText`(Tier 3)二次切,子 chunk 面包屑按 Start 位置反查 `sectionBreadcrumbs`

**Step 6:合并碎 chunk**(`coalesceTinyChunks`)
- 4 个条件全满足才合并:
  - `sharedHeader != ""`(有共享前缀,安全阀)
  - `cur.End == next.Start`(相邻,保位置不变量)
  - `curLen < target`(target=ChunkSize/2,最小 200)
  - `curLen+nextLen ≤ chunkSize`(硬上限)
- 合并后面包屑用 `commonHeadingPrefix`(两个面包屑按行逐行对比到第一个不同行停)
- **副作用**:跨章节合并面包屑降级(深层 → 浅层),接受的取舍

**Step 7:重新编号 Seq**(下游 knowledge.go 依赖 Seq 连续)

### Tier 1 独有
- **6 级标题栈** → 每个 chunk 带 ContextHeader(面包屑)
- **coalesceTinyChunks** → 合并 FAQ 类碎 chunk
- **sectionBreadcrumbs** → 超大 section 二次切后子 chunk 带正确子标题面包屑

### Tier 1 降级条件
- 无标题(primaryLevel=0)
- 切点 ≤ 1 个
- 校验失败(ValidateChunks 不通过)

---

## 七、Tier 2:heuristic(启发式切)

### 适用
没 Markdown `#` 标题,但有"第一章""1.1""Page 2 of 10""\f""---""INTRODUCTION"结构信号的文档。

### 核心流程(8 步)

**Step 1:短文本直接走 Tier 3**
```go
if totalRunes <= cfg.ChunkSize {
    return SplitText(text, cfg)
}
```

**Step 2:找所有候选切点**(`findHeuristicBoundaries`)

8 个正则,按优先级:

```
PrioFormFeed       = 100   \f              分页符
PrioNumberedHead   = 90    1.1 Foo         编号标题
PrioChapterMarker  = 85    第一章/Chapter 1 中英德章节
PrioAllCapsHeading = 70    INTRODUCTION    全大写标题
PrioVisualSep      = 60    ---             视觉分隔
PrioPageFooter     = 50    Page 2 of 10    页脚
PrioBlankBlock     = 40    \n\n\n          多空行
```

**关键**:
- 逐行扫(用 `inFence` 跳过代码块)
- 每行最多命中一个(用 `added` 标志)
- FormFeed 和 ExcessiveBlanks 是字符级/多行级,全文扫

**Step 3:排除保护区间内的切点**(`dropBoundsInsideSpans`)
- 保护区间 = 表格/代码/LaTeX/图片/链接/行内代码
- 严格内部的切点丢弃,边缘的保留

**Step 4:无切点 → 降级 Tier 3**

**Step 5:加哨兵**
- 末尾加 `boundary{runeStart: totalRunes}` 让 bin-packer 能 flush
- 开头没 0 位置切点就补一个

**Step 6:贪心装箱**(greedy bin-packing)

```go
minChunkSize := cfg.ChunkSize / 4
if minChunkSize < 50 { minChunkSize = 50 }
```

从左到右扫边界:
- `blockLen > ChunkSize` → 超大 block,flush 当前累积,委托 Tier 3 递归切
- `accumulated > ChunkSize && curEnd-chunkStart >= minChunkSize` → 封口,开新 chunk
- 否则继续累积

**Step 7:超大 Block 递归切**(`appendOversizeBlock`)
- 调 `SplitText`(Tier 3)内切
- 子 chunk 位置 `Start = start + s.Start` 还原绝对偏移

**Step 8:Overlap 对齐**(`applyOverlapAligned`)
- 3 级退化:
  1. 窗口内最近的边界(语义对齐)
  2. target 附近最近的换行符(行对齐)
  3. raw target(实在没办法)
- 永不切在词中间

### Tier 2 独有
- **8 个正则**找结构信号
- **minChunkSize = ChunkSize/4,最小 50** 防过早封口
- **无面包屑栈**(ContextHeader 空,因为"第一章""1.1"信号不够稳定,强维护会错乱)
- **无 coalesceTinyChunks**(没语义信息可保留,合并收益小)

### Tier 2 降级条件
- 无切点(bounds==0)
- 校验失败

---

## 八、Tier 3:recursive(递归切,SplitText)★ 兜底

### 适用
- Tier 1 失败
- Tier 2 失败
- Tier 1/Tier 2 内部超大 block 二次切
- 文档完全无结构信号

### 核心流程(3 步)

```go
func SplitText(text, cfg) []Chunk {
    protected := protectedSpans(text)                          // Step 1
    units := buildUnitsWithProtection(text, protected, ...)    // Step 2
    return mergeUnits(units, chunkSize, chunkOverlap)          // Step 3
}
```

**Step 1:找 7 种保护区间**(`protectedSpans`)

```
$$...$$       LaTeX 块公式
![](url)       Markdown 图片
[](url)        Markdown 链接
| A | B |\n| --- | --- |  表格头+分隔行
| data |       表格行
```...```      代码块
`code`         行内代码
```

保护区间**作为原子单元**,不参与切分。超 `maxProtectedSize=7500` 才硬切(换行/空格对齐)。

**Step 2:递归切+保原子**(`buildUnitsWithProtection`)
- 保护区间之间的普通文本 → `splitBySeparators` 递归切
- 保护区间本身 → 单个原子 unit
- 输出 `[]splitUnit`,每个带 `{text, start, end}`(rune 偏移)

**递归切分**(`splitBySeparators`):
- 默认分隔符 `["\n\n", "\n", "。"]`
- 先按 `\n\n` 切段落
- 段落 > chunkSize → 用 `["\n", "。"]` 内切(递归)
- 行 > chunkSize → 用 `["。"]` 内切
- **递归是对"还太大的 piece"用下一级分隔符**,不是全文重切

**Step 3:贪心合并 + overlap + 表头追踪**(`mergeUnits`)

```go
const absoluteMaxSize = 7500   // 硬上限
ht := newHeaderTracker()        // 表头追踪器

for _, u := range units {
    // 1. 单 unit 超 7500:硬切
    // 2. ht.update(u.text):更新表头状态
    //    表头结束 → flush(防两表糊一起)
    // 3. 累积超 chunkSize:flush + computeOverlap
    //    有活跃表头 → prepend 到新 chunk(去重 + 列数检查)
    // 4. 累积超 7500:硬 flush 无 overlap
    // 5. 加入累积
}
flush 剩余
```

**表头追踪**(`headerTracker`):
- 检测表头开始(`| A | B |\n| --- | --- |\n`)
- 检测表头结束(空行/非表行/列数不匹配)
- 空表头补全(MarkItDown 的 `||\n| --- | --- |\n` 用第一数据行补)
- 超大表格切多 chunk 时,每个 chunk 开头 prepend 表头
- hUnit 是零宽(`start==end`),不破坏 `End-Start = runeCount(Content)` 位置不变量

**Overlap 对齐**(`computeOverlap`):
- 默认 `DefaultChunkOverlap = 80`(约 15% of 512)
- 受 `chunkSize - nextLen` 限制(防下个 chunk 超)
- 搜索窗口 = `maxOverlap + 4`(lookbehind for `\r\n\r\n`)
- 3 级优先级:
  1. 段落 `\n\n` / `\r\n\r\n`(最强)
  2. 行 `\n` / `\r\n`
  3. 句末 `。？！` / `. ` `? ` `! `
- 同优先级取最早(让 overlap 尽量大)
- 边界后必须有非空白内容
- 边界不在保护区间内
- **找不到边界 → 不 overlap,绝不切词中间**

### Tier 3 独有
- **7 种保护模式**(防切表格/代码/LaTeX/图片/链接)
- **递归分隔符切分**(`\n\n` → `\n` → `。`)
- **表头追踪 + prepend**(超大表格不丢列语义)
- **overlap 3 级优先级对齐**(段落>行>句末)
- **两个硬上限 7500**(`maxProtectedSize` + `absoluteMaxSize`,防 embedding API 爆)
- **位置不变量** `End-Start = runeCount(Content)` 贯穿所有设计

### Tier 3 降级条件
- **不降级**!Tier 3 是兜底,校验失败也返回结果(strategy.go:50-57)
- 实在没办法,返回 `SplitText(text, cfg)` 的结果

---

## 九、三层协作的关键:位置不变量

**所有 tier 都遵守同一个不变量**:
```go
// Chunk struct 注释
// End-Start == utf8.RuneCountInString(Content)
```

每个 chunk 的 Content 是**原文精确切片**,`End - Start` 严格等于 Content 的 rune 数。

**为什么这个不变量重要**:
- 下游能重建原文(按 Start 排序拼接)
- UI 高亮能定位(Start/End 直接对应原文位置)
- 版本 diff(编辑后位置还能对上)
- 重新索引能重建一样的输入

**各层怎么保住它**:
- Tier 1:`cur.End == next.Start`(coalesceTinyChunks 合并条件)
- Tier 2:`appendOversizeBlock` 子 chunk 位置 `Start = start + s.Start` 还原
- Tier 3:hUnit 零宽(`start==end`),overlap 保留原偏移,buildChunk 用 `units[0].start` 和 `units[-1].end`

---

## 十、向量化链路(切完之后)

切完拿到 `[]Chunk`,后续向量化:

```go
// knowledge_process.go:519
indexContent := buildKnowledgeIndexContent(knowledge, chunk.EmbeddingContent())

// chunk.EmbeddingContent() = ContextHeader + "\n\n" + TrimSpace(Content)
// buildKnowledgeIndexContent = knowledge.Title + "\n" + content
```

**最终喂给 embedding 模型的输入**:
```
文档标题(knowledge.Title)
面包屑(ContextHeader,可能空)

正文(Content,trim 空白)
```

- **Tier 1 切的 chunk**:ContextHeader 有面包屑(`# XX\n## YY\n### ZZ`),向量带章节语义
- **Tier 2/Tier 3 切的 chunk**:ContextHeader 空,向量只有正文 + 文档标题

**向量库 vs 关系库**:
- chunks 表存 `Content + ContextHeader + Start/End + Seq`(业务用:编辑、版本、重建索引)
- 向量库存 `向量 + chunk_id`(检索用:最近邻查找)

---

## 十一、一个完整例子:从文档到 chunk

**输入文档**:
```markdown
# 公司报销制度

## 差旅报销
### 飞机票
经济舱凭票报销,公务舱需提前审批。

### 酒店住宿
标准间按城市限额。

## 日常报销
### 出租车
凭发票报销,30元/次以内无需审批。
```

### Step 1:体检

```
MdHeadingCounts = {1:1, 2:2, 3:4}
MdHeadingTotal = 7
HeadingDensity = 7/15 ≈ 0.47
其他结构信号:无
```

### Step 2:SelectStrategy

```
MdHeadingTotal=7 >= 3 ✓
HeadingDensity=0.47 > 0.005 ✓
DominantHeadingLevel()=3(H1 才 1 <3,H2 才 2 <3,H3 4 ≥3)
→ chain = [TierHeading, TierLegacy]
```

### Step 3:runTier(TierHeading)

- primaryLevel=3,按 H3 切
- 切点:0, "### 飞机票", "### 酒店住宿", "## 日常报销", "### 出租车"
- 栈走一遍:
  - section 0:"# 公司报销制度\n## 差旅报销"(2个标题层),面包屑=`# 公司报销制度\n## 差旅报销`
  - section 1:"### 飞机票\n经济舱凭票报销...",面包屑=`# 公司报销制度\n## 差旅报销\n### 飞机票`
  - section 2:"### 酒店住宿\n标准间按城市限额",面包屑=`# 公司报销制度\n## 差旅报销\n### 酒店住宿`
  - section 3:"## 日常报销",面包屑=`# 公司报销制度\n## 日常报销`
  - section 4:"### 出租车\n凭发票报销...",面包屑=`# 公司报销制度\n## 日常报销\n### 出租车`

- 每个 section 都 < ChunkSize=512,无需二次切
- coalesceTinyChunks:section 3 太短(就 1 行),但跟相邻的共享前缀不同(`## 差旅` vs `## 日常`),不合并

### Step 4:ValidateChunks

5 个 chunk,每个 30-50 字,< 50 的有 section 3(6字)。
- tinyCount=1(不算 section 4,它是最后一个,跳过)
- 1 > 5/4=1 ✓ 且 1 > 2 ✗ → **不触发"too many tiny chunks"**
- maxLen=50 > 512/4=128?否 → 不触发"all chunks far below"
- maxLen=50 < 2*512 → 不触发"chunk exceeds 2x"
- 5 chunks,不触发"single chunk for large document"
- **校验通过!** 返回 Tier 1 结果

### 最终结果

| chunk | Content | ContextHeader | Start | End |
|-------|---------|--------------|-------|-----|
| 0 | # 公司报销制度\n## 差旅报销 | # 公司报销制度\n## 差旅报销 | 0 | X1 |
| 1 | ### 飞机票\n经济舱凭票报销... | # 公司报销制度\n## 差旅报销\n### 飞机票 | X1 | X2 |
| 2 | ### 酒店住宿\n标准间按城市限额 | # 公司报销制度\n## 差旅报销\n### 酒店住宿 | X2 | X3 |
| 3 | ## 日常报销 | # 公司报销制度\n## 日常报销 | X3 | X4 |
| 4 | ### 出租车\n凭发票报销... | # 公司报销制度\n## 日常报销\n### 出租车 | X4 | 末尾 |

### 向量化(以 chunk 1 为例)

```
喂给 embedding 模型的输入:
"公司报销制度              ← knowledge.Title
# 公司报销制度              ← ContextHeader
## 差旅报销
### 飞机票

### 飞机票                  ← Content(含标题行)
经济舱凭票报销,公务舱需提前审批。"
```

(注:Content 里包含 section 内的标题行,因为切点在 H3 位置,前面的标题是 section 内容的一部分。)

---

## 十二、三层对比总表

| 维度 | Tier 1 (heading) | Tier 2 (heuristic) | Tier 3 (recursive) |
|------|-----------------|--------------------|--------------------|
| 触发 | 有 `#` 标题 | 有结构信号 | 兜底/超大内切 |
| 找切点 | MarkdownHeadingPattern 1 个 | 8 个正则 | 分隔符递归 `\n\n` `\n` `。` |
| 标题栈 | 6 级栈 + 面包屑 | 无 | 无 |
| 保护区间 | 跳过 ``` 代码块 | 7 种(dropBoundsInsideSpans) | 7 种(buildUnitsWithProtection) |
| 二次切 | SplitText + sectionBreadcrumbs | SplitText(位置还原) | 自己就是 SplitText |
| overlap | 边界对齐 | applyOverlapAligned 3 级退化 | computeOverlap 3 级优先级 |
| 碎处理 | coalesceTinyChunks 合并 | minChunkSize 防过早封口 | 无(靠 overlap 平滑) |
| 表格头 | 无特殊处理 | 无特殊处理 | headerTracker 追踪 + prepend |
| 硬上限 | 无 | 无 | 7500(absoluteMaxSize) |
| 校验失败 | 降级 Tier 2 | 降级 Tier 3 | 不降级,返回最后结果 |
| ContextHeader | 有面包屑 | 空 | 空 |

---

## 十三、你应该带走的 10 件事

1. **3 层是瀑布降级链**:Tier 1 失败 → Tier 2 → Tier 3,第一层通过校验的直接用。Tier 3 是兜底,失败也返回结果。

2. **入口是 ProfileDocument + SelectStrategy**:扫一遍文档统计结构信号,根据信号动态选 chain(可能 `[T1,T2,T3]` / `[T2,T3]` / `[T3]`)。Tier 3 永远在链尾。

3. **ValidateChunks 4 个判据**:碎 chunk 多、最大 chunk 太小、chunk 超 2x、大文档只 1 chunk。失败就整层作废降级。

4. **Tier 1 = 标题切**:按主导层级(≥3 次的最低层级)切,6 级栈维护面包屑,超大 section 委托 Tier 3 + sectionBreadcrumbs 反查子标题,FAQ 类碎 chunk 用 coalesceTinyChunks 合并(共享前缀降级是接受的副作用)。

5. **Tier 2 = 启发式切**:8 个正则找结构信号(分页符/编号/章节/全大写/分隔线/页脚/多空行),贪心装箱,minChunkSize=ChunkSize/4 防过早封口,无面包屑栈,超大 block 委托 Tier 3。

6. **Tier 3 = 递归切(SplitText)**:7 种保护模式 + 递归分隔符(`\n\n`→`\n`→`。`)+ 表头追踪 prepend + overlap 3 级优先级对齐 + 两个硬上限 7500。是最底层安全网,也是 T1/T2 内部超大内切的工具。

7. **递归切 = "对还太大的 piece 用下一级分隔符内切"**:不是全文重切,保留层级。递归终止 = piece ≤ chunkSize。

8. **overlap 默认 80 字,3 级优先级对齐**:段落 `\n\n` > 行 `\n` > 句末 `。？！/. `。受 `chunkSize - nextLen` 限制。找不到边界就不 overlap,绝不切词中间。overlap 是下个 chunk 的开头。

9. **位置不变量 `End-Start = runeCount(Content)` 贯穿所有设计**:T1 coalesceTinyChunks 的 `cur.End==next.Start`、T2 的 `Start=start+s.Start` 还原、T3 的 hUnit 零宽 + overlap 保留原偏移 —— 都是为了保住它,下游能重建原文、UI 高亮、版本 diff。

10. **向量化是三层拼接**:`文档标题 + 面包屑 + 正文`(buildKnowledgeIndexContent + EmbeddingContent)。Tier 1 的 chunk 带面包屑,向量带章节语义;Tier 2/3 的 chunk 面包屑空,向量只有正文 + 文档标题。