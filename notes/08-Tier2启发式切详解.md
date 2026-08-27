# 08 · Tier 2 启发式切(heuristic)详解

> Tier 1 处理"有清晰 `# ## ###` Markdown 标题的文档";Tier 2 处理"没 Markdown 标题,但有'第一章''1.1''Page 2 of 10''\f''---'这些结构信号的文档"。

---

## 一、Tier 2 是干嘛的

**适用场景**:文档没有规范的 `#` 标题,但有人类可识别的"章节标记"——比如:

- PDF 转换后的纯文本(分页符 `\f`、页脚 "Page 2 of 10")
- 中文学术文档("第一章""第 3 节""第二部分")
- 英文学术文档("Chapter 1""Section 2.3""Part IV")
- 德语文档("Kapitel 1""Abschnitt 2""Teil 3")
- 编号标题("1. Intro""2.3 Methods""IV. Results")
- 全大写标题("INTRODUCTION""SUMMARY")
- 分隔线("---""===""***")
- 多空行(\n\n\n,≥3 个连续换行)

这些信号"不是 Markdown 标题,但暗示语义边界"。Tier 2 用一组正则找这些信号当切点。

---

## 二、Tier 2 的 8 个正则(patterns.go)

### 1. FormFeed(分页符 `\f`)
```go
PrioFormFeed = 100  // 最高优先级
FormFeedPattern = `\f`
```
PDF 转换常把分页转成 `\f`。这是"最强的单字符边界"。

### 2. NumberedSection(编号标题)
```go
PrioNumberedHead = 90
NumberedSectionPattern = `(?m)^[ \t]*(?:\d+(?:\.\d+){1,3}\.?|(?:\d+|[IVX]{1,5})\.)[ \t]+\S.{0,200}$`
```
匹配:
- `1. Intro` / `2.3 Methods` / `2.2.1 用户与权限`(1-4 级数字编号,末尾点可选)
- `IV. Results`(罗马数字)
- `1. Intro`(简单数字)

### 3. ChapterMarker(章节标记,中英德三语)
```go
PrioChapterMarker = 85
ChineseChapterPattern  = `(?m)^[ \t]*第[ \t]*[一二三四五六七八九十百千零〇0-9]+[ \t]*(?:章|节|節|部分|篇)[ \t]?.{0,200}$`
EnglishChapterPattern  = `(?m)^[ \t]*(?:Chapter|Section|Part)\s+(?:[0-9]+|[IVX]{1,5})[\.: ].{0,200}$`
GermanChapterPattern   = `(?m)^[ \t]*(?:Kapitel|Abschnitt|Teil)\s+(?:[0-9]+|[IVX]{1,5})[\.: ].{0,200}$`
```
匹配:
- `第一章` / `第 3 节` / `第二部分`(中文,数字和单位间允许空格)
- `Chapter 1` / `Section IV` / `Part 2`(英文)
- `Kapitel 1` / `Abschnitt 2` / `Teil 3`(德文)

### 4. AllCapsHeading(全大写标题)
```go
PrioAllCapsHeading = 70
AllCapsHeadingPattern = `(?m)^[ \t]*([A-ZÄÖÜ][A-ZÄÖÜ \-]{3,80}):?\s*$`
```
匹配:
- `INTRODUCTION`
- `SUMMARY:`
- `METHODS AND MATERIALS`

要求:至少 4 个字母(防 `OK` `API` 这种缩写),最多 80 字符,容忍末尾冒号。

### 5. VisualSeparator(视觉分隔线)
```go
PrioVisualSep = 60
VisualSeparatorPattern = `(?m)^[ \t]*(?:-{3,}|={3,}|\*{3,}|_{3,})[ \t]*$`
```
匹配:
- `---`(Markdown 水平线)
- `===`
- `***`
- `___`

至少 3 个相同字符连续。

### 6. PageFooter(页脚)
```go
PrioPageFooter = 50
PageFooterPattern = `(?mi)^[ \t]*(?:Seite|Page|页码?)\s+\d+(?:\s*(?:von|of|/)\s*\d+)?[ \t]*$`
```
匹配:
- `Page 2` / `Page 2 of 10`
- `Seite 3 von 15`(德语)
- `页 5` / `页码 5`

页脚通常是分页位置,适合当切点。

### 7. ExcessiveBlanks(多空行)
```go
PrioBlankBlock = 40  // 最低优先级
ExcessiveBlanksPattern = `\n{3,}`
```
匹配 3 个或更多连续换行,通常表示段落间大间隔(硬分节)。

### 8. 优先级总结

```
PrioFormFeed       = 100  \f
PrioNumberedHead   = 90   1.1 Foo
PrioChapterMarker  = 85   第一章 / Chapter 1
PrioAllCapsHeading = 70   INTRODUCTION
PrioVisualSep      = 60   ---
PrioPageFooter     = 50   Page 2 of 10
PrioBlankBlock     = 40   \n\n\n
```

**优先级不影响切的顺序**(切点都是按位置升序排),**只影响同一位置多个 pattern 命中时的去重保留**。

---

## 三、Tier 2 完整流程(8 步)

### Step 1:短文本直接走 Tier 3

```go
if totalRunes <= cfg.ChunkSize {
    return SplitText(text, cfg)  // 短文档不需要找边界,直接 Tier 3 切
}
```

### Step 2:找所有候选切点(findHeuristicBoundaries)

扫一遍文本,按上面 8 个正则找切点。返回 `[]boundary`,每个 boundary 记录:
```go
type boundary struct {
    runeStart int  // rune 偏移
    priority int   // 优先级
}
```

关键点:
- **跳过 ``` 代码块内的行**(`inFence` 状态机),防代码注释里的 `#1` `---` 被误识别
- **逐行扫** chapterPatterns / NumberedSectionPattern / AllCapsHeadingPattern / VisualSeparatorPattern / PageFooterPattern,每行最多命中一个(用 `added` 标志)
- **全文扫** FormFeed 和 ExcessiveBlanks(这两个不是行级,是字符级/多行级)

### Step 3:排除保护区间内的切点(dropBoundsInsideSpans)

保护区间 = 表格 / 代码块 / LaTeX / 图片 / 链接 / 行内代码等"原子内容",切点落在保护区间内部的**丢弃**,落在区间边缘的**保留**。

```go
if prot := protectedSpansRune(text, protectedSpans(text)); len(prot) > 0 {
    bounds = dropBoundsInsideSpans(bounds, prot)
}
```

例子:表格里某行恰好是 `---`(Markdown 表格分隔行),这个 `---` 在表格区间内,被丢弃,不当切点。

### Step 4:无切点 → 降级 Tier 3

```go
if len(bounds) == 0 {
    return SplitText(text, cfg)
}
```

文档完全没有结构信号 → Tier 2 无能为力,交给 Tier 3 递归切。

### Step 5:加哨兵

```go
// 末尾加哨兵,让 bin-packer 能 flush 最后一段
bounds = append(bounds, boundary{runeStart: totalRunes})
// 开头如果没 0 位置切点,补一个
if bounds[0].runeStart != 0 {
    bounds = append([]boundary{{runeStart: 0}}, bounds...)
}
```

### Step 6:贪心装箱(greedy bin-packing)—— 核心算法

```go
minChunkSize := cfg.ChunkSize / 4
if minChunkSize < 50 { minChunkSize = 50 }
```

`minChunkSize` = ChunkSize/4(最小 50),是"封口的最小长度"——避免切出太碎的 chunk。

然后从左到右扫边界:
```go
for i := 1; i < len(bounds); i++ {
    nextEnd := bounds[i].runeStart
    blockLen := nextEnd - curEnd  // 当前 block 长度

    if blockLen > cfg.ChunkSize {
        // Block 自己就超 ChunkSize,flush 当前累积,Block 递归切
        flush 当前累积
        appendOversizeBlock(递归 SplitText 切这个超大 Block)
        重置 chunkStart = curEnd = nextEnd
        continue
    }

    accumulated := nextEnd - chunkStart
    if accumulated > cfg.ChunkSize && curEnd-chunkStart >= minChunkSize {
        // 加这个 block 会超 ChunkSize,且当前累积已经够长(minChunkSize)
        flush 当前累积为一个 chunk
        chunkStart = applyOverlapAligned(runes, curEnd, cfg.ChunkOverlap, bounds)
        // ↑ overlap 对齐到最近的边界或换行符,不切在词中间/行中间
    }
    curEnd = nextEnd
}
// 循环结束 flush 剩余
```

**核心思想**:**在边界之间累积 block,累积到快超 ChunkSize 就封口**。封口后下个 chunk 从 `curEnd - overlap` 附近开始,但要对齐到最近的边界或换行符(applyOverlapAligned)。

### Step 7:超大 Block 递归切(appendOversizeBlock)

如果两个边界之间的 block 自己就 > ChunkSize(比如某个章节特别长),这个 block 单独递归调 `SplitText`(Tier 3)内切:

```go
func appendOversizeBlock(out, runes, start, end, cfg, seq) []Chunk {
    subText := string(runes[start:end])
    subs := SplitText(subText, cfg)  // Tier 3 递归切
    for _, s := range subs {
        out = append(out, Chunk{
            Content: s.Content,
            Seq:     *seq,
            Start:   start + s.Start,  // 位置偏移还原
            End:     start + s.End,
        })
        *seq++
    }
    return out
}
```

**注意**:子 chunk 的位置 `Start = start + s.Start`,把 Tier 3 切出的相对位置还原回原文绝对位置。

### Step 8:Overlap 对齐(applyOverlapAligned)

封口后下个 chunk 不直接从 `curEnd - overlap` 开始(可能切在词中间/行中间),而是**对齐到最近的语义边界或换行符**:

```go
func applyOverlapAligned(runes, curEnd, overlap, bounds) int {
    target := curEnd - overlap  // 期望的 overlap 起点
    windowStart := curEnd - 2*overlap  // 搜索窗口

    // 1. 优先在窗口内找最近的边界
    for _, b := range bounds {
        if b.runeStart >= windowStart && b.runeStart < curEnd && b.runeStart > bestBound {
            bestBound = b.runeStart
        }
    }
    if bestBound >= 0 { return bestBound }

    // 2. 退化:从 target 往前扫到最近的换行符
    for i := target; i > windowStart && i < len(runes); i-- {
        if runes[i] == '\n' { return i + 1 }
    }

    // 3. 实在找不到:用 raw target
    return target
}
```

**为什么**:
- 切在词中间 → embedding 输入是半个词,向量质量下降
- 切在行中间 → 失去行边界信息
- 对齐到边界 → overlap 部分仍是完整语义单元

---

## 四、完整例子走一遍

**输入文档**(假设 ChunkSize=200,overlap=40):
```
第一章 引言

这是引言的内容,介绍背景和研究动机。这部分内容比较短,不会超过 ChunkSize。

1.1 研究背景

研究背景部分,详细描述问题来源和现状。这里写了很多内容,超过了 ChunkSize 的限制,需要二次切。这段文字故意写长一点,让它超过 200 字,这样才能触发 appendOversizeBlock 逻辑,让 Tier 2 把这个超大 block 委托给 Tier 3 递归切。继续写,继续写,继续写,继续写,继续写,继续写,继续写,继续写,继续写,继续写。

1.2 研究目标

研究目标部分,简短描述。结束。

第二章 方法

方法部分,描述技术路线。
```

### Step 1-2 找切点

扫描发现:
- 位置 0:开头(补的 0 边界)
- 位置 X1:`第一章 引言`(ChapterMarker,优先级 85)
- 位置 X2:`1.1 研究背景`(NumberedSection,优先级 90)
- 位置 X3:`1.2 研究目标`(NumberedSection,优先级 90)
- 位置 X4:`第二章 方法`(ChapterMarker,优先级 85)
- 位置末尾:哨兵

### Step 3 保护区间

无表格/代码块,保护区间为空,所有切点保留。

### Step 4-5 加哨兵

`bounds = [0, X1, X2, X3, X4, 末尾]`

### Step 6 贪心装箱

假设:
- 0 → X1 长度 = 20(`第一章 引言` 标题 + 空行)
- X1 → X2 长度 = 80(引言正文)
- X2 → X3 长度 = 350(超大!> ChunkSize=200)
- X3 → X4 长度 = 40(研究目标)
- X4 → 末尾 长度 = 30(方法)

从左到右:

**i=1 (X1)**:
- blockLen = X1 - 0 = 20
- accumulated = 20,≤200,继续累积
- curEnd = X1

**i=2 (X2)**:
- blockLen = X2 - X1 = 80
- accumulated = X2 - 0 = 100,≤200,继续
- curEnd = X2

**i=3 (X3)**:
- blockLen = X3 - X2 = 350 > 200 → 超大 block!
- flush 当前累积(0 → X2,长度 100)为一个 chunk
- appendOversizeBlock(X2, X3):这段 350 字递归 SplitText 切成 2 个子 chunk
- 重置 chunkStart = curEnd = X3

**i=4 (X4)**:
- blockLen = X4 - X3 = 40
- accumulated = X4 - X3 = 40,≤200,继续
- curEnd = X4

**i=5 (末尾)**:
- blockLen = 末尾 - X4 = 30
- accumulated = 末尾 - X3 = 70,≤200,继续
- 循环结束,flush 剩余(X3 → 末尾)为一个 chunk

### 最终结果

```
chunk 0: 0 → X2,        Content = "第一章 引言\n\n引言正文\n\n1.1 研究背景"(100字)
chunk 1: X2 → X3 的子1, Content = "研究背景前半部分..."(SplitText 切的,~180字)
chunk 2: X2 → X3 的子2, Content = "研究背景后半部分..."(SplitText 切的,~170字)
chunk 3: X3 → 末尾,      Content = "1.2 研究目标\n\n研究目标部分...\n\n第二章 方法\n\n方法部分..."(70字)
```

注意:chunk 1、2 是 Tier 3 递归切出来的,位置已还原到原文绝对偏移。

---

## 五、几个关键设计点

### 1. 优先级不决定切的顺序,只决定去重保留

```go
sort.Slice(bounds, func(i, j int) bool {
    if bounds[i].runeStart != bounds[j].runeStart {
        return bounds[i].runeStart < bounds[j].runeStart  // 先按位置升序
    }
    return bounds[i].priority > bounds[j].priority  // 同位置,优先级高的排前
})
// 去重:同位置只留第一个(优先级最高的)
```

例子:某行同时匹配 `NumberedSectionPattern`(90)和 `VisualSeparatorPattern`(60),只保留 NumberedSection。

### 2. 为什么逐行扫,不是全文正则

逐行扫可以用 `inFence` 状态机跳过 ``` 代码块内的行。全文正则做不到这点(代码块内的 `---` 会被误识别)。

### 3. minChunkSize = ChunkSize/4,最小 50

防止"累积刚到 minChunkSize 就封口,下一 block 又很大"导致 chunk 太碎。封口前检查 `curEnd-chunkStart >= minChunkSize`。

### 4. applyOverlapAligned 的 3 级退化

1. **优先**:窗口内最近的边界(语义对齐)
2. **退化**:target 附近最近的换行符(行对齐)
3. **最后**:raw target(实在没办法)

3 级退化保证 overlap 永远不会切在词中间。

### 5. 超大 Block 委托 Tier 3

Tier 2 不自己处理超大 block,委托 SplitText(Tier 3)内切,这样保护模式、重叠对齐等逻辑都能复用。

### 6. Tier 2 不维护面包屑栈

Tier 1 维护标题栈给每个 chunk 盖"我在哪一节"的章;**Tier 2 没这个机制**。Tier 2 切出来的 chunk `ContextHeader` 是空的(或者只有文档标题)。

为什么?
- Tier 2 处理的文档没有规范的层级标题,"第一章""1.1"这类信号不够稳定,很难维护栈
- 强行维护栈会出现"第一章 + 1.1 + 第二章"这种错乱状态
- 接受的取舍:用结构边界切,但放弃面包屑

### 7. 降级条件

- 无切点(bounds==0)→ Tier 3
- 校验失败(某 chunk 太大或太小)→ Tier 3

---

## 六、Tier 1 vs Tier 2 对比

| 维度 | Tier 1 (heading) | Tier 2 (heuristic) |
|------|-----------------|--------------------|
| 触发条件 | 有 Markdown `#` 标题 | 有结构信号(第一章/1.1/Page/\f/---) |
| 找切点 | `MarkdownHeadingPattern` 1 个正则 | 8 个正则,按优先级去重 |
| 标题栈 | 维护 6 级栈,带面包屑 | 不维护,ContextHeader 空 |
| 二次切 | SplitText + sectionBreadcrumbs 反查 | SplitText 递归切,位置还原 |
| 碎 chunk 处理 | coalesceTinyChunks 合并 | 无(靠 minChunkSize 防过碎) |
| overlap 对齐 | 边界/换行符 | applyOverlapAligned(3 级退化) |
| 保护区间 | 跳过 ``` 代码块 | dropBoundsInsideSpans(7 种保护模式) |
| 降级 | 无标题/边界≤1/校验失败 → Tier 3 | 无边界/校验失败 → Tier 3 |

---

## 七、你应该带走的 6 件事

1. **Tier 2 用 8 个正则找结构信号**:`\f`(100)、`1.1 Foo`(90)、`第一章/Chapter 1`(85)、`INTRODUCTION`(70)、`---`(60)、`Page 2 of 10`(50)、`\n\n\n`(40)。优先级只影响同位置去重,不影响切顺序。

2. **核心算法是贪心装箱**:在边界之间累积 block,累积到快超 ChunkSize 就封口,超大 block 委托 Tier 3 递归切。

3. **minChunkSize = ChunkSize/4,最小 50**:封口的最小长度,防 chunk 太碎。

4. **overlap 用 applyOverlapAligned 对齐**:3 级退化——边界 > 换行符 > raw target,永不切在词中间。

5. **Tier 2 不维护面包屑栈**:ContextHeader 是空的(或只有文档标题),因为"第一章""1.1"这类信号不够稳定,强行维护栈会错乱。

6. **降级条件**:无切点 → Tier 3;校验失败 → Tier 3。Tier 2 是 Tier 1 和 Tier 3 之间的中间层,处理"有结构信号但没 Markdown 标题"的文档。