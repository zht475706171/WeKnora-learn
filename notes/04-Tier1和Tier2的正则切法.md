# 04 · Tier 1 和 Tier 2 的正则切法(大白话版)

> 学习目标:回答用户问题"前两步是不是正则匹配?代码怎么写?"——是的,两层都是正则,但用法完全不同。
>
> Tier 1 用**一个正则找 Markdown 标题**,然后按标题切分,顺便维护一个"标题栈"做面包屑。
> Tier 2 用**一堆正则找各种结构标记**,把它们都当成"候选切点",按优先级排好后贪心装箱。

---

## 一句话总结

| Tier | 用什么正则 | 怎么切 |
|------|----------|------|
| Tier 1 (heading) | 1 个:`MarkdownHeadingPattern` 匹配 `# 标题` | 按"主导标题级别"找切点,每个 chunk 自带"我在第几章第几节"的面包屑 |
| Tier 2 (heuristic) | 8 个:章节/编号/全大写/视觉分隔/页脚/\f 等 | 把所有"结构标记"位置当候选切点,按优先级装箱,超 512 字就封口 |

代码分别在 `heading_splitter.go` 和 `heuristic_splitter.go`,正则都集中定义在 `patterns.go`。

---

## 所有正则定义(`patterns.go`)

```go
// 1. Markdown 标题(Tier 1 主角)
//    匹配行首 1-6 个 # + 空格 + 标题文本
MarkdownHeadingPattern = `(?m)^(#{1,6})\s+(.+?)\s*#*\s*$`
//    (?m) 多行模式,^ 匹配行首
//    (#{1,6}) 捕获组1:1-6 个 #(决定层级)
//    (.+?)    捕获组2:标题文字
//    \s*#*\s*$ 末尾可选的关闭 # 和空白

// 2. 章节标记(Tier 2 用,多语言)
GermanChapterPattern  = `(?m)^[ \t]*(?:Kapitel|Abschnitt|Teil)\s+(?:[0-9]+|[IVX]{1,5})[\.: ].{0,200}$`
EnglishChapterPattern = `(?m)^[ \t]*(?:Chapter|Section|Part)\s+(?:[0-9]+|[IVX]{1,5})[\.: ].{0,200}$`
ChineseChapterPattern = `(?m)^[ \t]*第[ \t]*[一二三四五六七八九十百千零〇0-9]+[ \t]*(?:章|节|節|部分|篇)[ \t]?.{0,200}$`
//    第 + 数字(中文或阿拉伯) + 章/节/部分/篇 + 最多 200 字标题

// 3. 编号标题(Tier 2)
NumberedSectionPattern = `(?m)^[ \t]*(?:\d+(?:\.\d+){1,3}\.?|(?:\d+|[IVX]{1,5})\.)[ \t]+\S.{0,200}$`
//    "1. Intro"、"2.3 Methods"、"2.2.1 用户与权限"、"IV. Results"

// 4. 全大写短行(像 INTRODUCTION 这种没标 # 的标题)
AllCapsHeadingPattern = `(?m)^[ \t]*([A-ZÄÖÜ][A-ZÄÖÜ \-]{3,80}):?\s*$`
//    至少 4 个字母,最多 80 字符,允许德语变音

// 5. 视觉分隔符 ---
VisualSeparatorPattern = `(?m)^[ \t]*(?:-{3,}|={3,}|\*{3,}|_{3,})[ \t]*$`

// 6. 页脚 Page X of Y
PageFooterPattern = `(?mi)^[ \t]*(?:Seite|Page|页码?)\s+\d+(?:\s*(?:von|of|/)\s*\d+)?[ \t]*$`

// 7. \f 分页符(PDF 转换器常用)
FormFeedPattern = `\f`

// 8. 多余空行(3+ 个连续 \n)
ExcessiveBlanksPattern = `\n{3,}`
```

每个模式还带一个"优先级"(用在 Tier 2 装箱时):

```go
PrioFormFeed       = 100  // \f 分页符最强
PrioNumberedHead   = 90   // 1.1 这种编号
PrioChapterMarker  = 85   // 第一章/Chapter/Kapitel
PrioAllCapsHeading = 70   // INTRODUCTION
PrioVisualSep      = 60   // ---
PrioPageFooter     = 50   // Page X of Y
PrioBlankBlock     = 40   // 多空行
```

---

## Tier 1 怎么切(`heading_splitter.go`)

### 流程

```
1. 算出"主导标题层级" primaryLevel
   (ProfileDocument.DominantHeadingLevel():找"≥3次出现的最低层级",
    比如 H1 出现5次、H2出现3次 → primaryLevel=1;
    只有 H1出现1次、H2出现3次 → primaryLevel=2)

2. findHeadingBoundaries(text, primaryLevel)
   扫每一行,凡是匹配 MarkdownHeadingPattern 且 level ≤ primaryLevel 的,
   记录为一个"切点"(rune 偏移 + 那行原文)
   
   ⚠️ 关键:跳过 ``` 代码块里的 # 标题(否则代码注释里的 # 会被误识别)
   
   结果:[边界0(开头), 边界1(H1标题A), 边界2(H1标题B), ...]

3. 对每个 section(相邻两个边界之间):
   - 用 HeadingHierarchy 维护一个标题栈
   - 取出当前栈的"面包屑"(如 "# 引言\n## 背景")
   - 如果 section 内容 ≤ ChunkSize:整块当一个 chunk,面包屑存进 ContextHeader
   - 如果 section 太大:对它内部再调 SplitText(legacy)切,
     每个子 chunk 用"chunk 起始位置对应的子标题"当面包屑

4. coalesceTinyChunks:合并太碎的 chunk
   (FAQ 类文档一段只有几十字,会触发校验器的"chunk 太小"规则,
    把相邻小 chunk 合并到 ChunkSize/2 以下)
```

### HeadingHierarchy:标题栈怎么维护

```go
type HeadingHierarchy struct {
    stack [6]string  // stack[0]=H1, stack[1]=H2, ...
    depth int
}

func (h *HeadingHierarchy) Observe(line string) {
    m := MarkdownHeadingPattern.FindStringSubmatch(line)
    level := len(m[1])           // # 的个数
    heading := strings.TrimSpace(m[2])
    
    h.stack[level-1] = heading   // 这层压栈
    for i := level; i < 6; i++ { // 清掉更深的(同级新标题会顶掉旧标题的子标题)
        h.stack[i] = ""
    }
}

func (h *HeadingHierarchy) BreadcrumbWithHashes() string {
    // 输出 "# 引言\n## 背景\n### 子标题"
    // 这就是每个 chunk 的 ContextHeader
}
```

举例,文档长这样:
```
# 引言
## 背景
正文 A
## 方法
正文 B
```

走到"正文 A":栈 = ["引言","背景"],面包屑 = `# 引言\n## 背景`
走到"正文 B":栈 = ["引言","方法"],面包屑 = `# 引言\n## 方法`(方法顶掉了背景)

### 一个非常聪明的设计:子 chunk 也带子标题的面包屑

如果 section 太大要二次切,不是用 section 的标题当所有子 chunk 的面包屑,而是**记录每个子 chunk 起始位置对应的"最深活跃标题"**。这样:
```
# 第一章 引言     ← primaryLevel=1,section 太大要二次切
## 1.1 背景
正文 a
## 1.2 方法
正文 b
```
- "正文 a" 的子 chunk 面包屑 = `# 第一章 引言\n## 1.1 背景`
- "正文 b" 的子 chunk 面包屑 = `# 第一章 引言\n## 1.2 方法`

不会都糊成 `# 第一章 引言`。

### 降级条件

- 文档没有任何标题(`DominantHeadingLevel` 返回 0) → 直接降级到 `SplitText`
- 找到的边界 ≤ 1 个(只有一个标题,等于没切) → 降级
- 切完校验不过 → strategy.go 自动降到 Tier 2 或 Tier 3

---

## Tier 2 怎么切(`heuristic_splitter.go`)

### 流程

```
1. findHeuristicBoundaries(text, langs)
   扫一遍文本,把所有"结构标记"位置都记成候选切点:
   
   - \f 分页符 → priority=100
   - 逐行扫(跳过 ```代码块```)
     * 章节标记(中/英/德,按 langs 选) → 85
     * 编号标题 "1.1 Foo" → 90
     * 全大写短行 "INTRODUCTION" → 70
     * 视觉分隔 "---" → 60
     * 页脚 "Page X of Y" → 50
   - 多空行 \n{3,} → 40
   
   排序 + 去重(同一位置只保留最高优先级)

2. dropBoundsInsideSpans(bounds, protectedSpans)
   把落在保护区间(LaTeX/表格/代码块)内的切点踢掉
   ⚠️ 边界正好对齐保护区间起止的保留(不会切坏保护内容)

3. 贪心装箱:
   chunkStart = bounds[0]
   curEnd = chunkStart
   for 每个下一个边界 nextEnd:
       blockLen = nextEnd - curEnd
       
       如果 blockLen > ChunkSize:
           这个块本身就超了 → flush 当前累积
           对这个超大块调 SplitText(legacy) 内切
       
       如果累积 nextEnd - chunkStart > ChunkSize 且当前已够长(≥ChunkSize/4):
           flush 一个 chunk
           下一个 chunk 起点用 applyOverlapAligned 对齐到最近的语义边界
       
       curEnd = nextEnd
   
   末尾 flush 剩余

4. applyOverlapAligned:overlap 对齐
   不是傻切 curEnd - overlap 字符,而是:
   - 在 [curEnd - 2*overlap, curEnd) 窗口里找最近的"边界"
   - 找不到就回退到最近的换行符
   - 都找不到才硬切
```

### 一个特别聪明的设计:超大块自动委托给 legacy

如果两个相邻边界之间的块本身就 > ChunkSize(比如"第一章"下面"第二章"上面有 2000 字),Tier 2 不硬切,而是把这个超大块丢回 `SplitText` 用 Tier 3 的递归切法处理。三层是协作的,不是排斥的。

### 降级条件

- 没找到任何边界 → 降级到 `SplitText`
- 切完校验不过 → strategy.go 降到 Tier 3

---

## 两层对比

| 维度 | Tier 1 (heading) | Tier 2 (heuristic) |
|------|----------------|-------------------|
| 用啥正则 | 1 个:`MarkdownHeadingPattern` | 8 个:章节/编号/全大写/分隔符/页脚/\f/空行 |
| 切点来源 | Markdown `#` 标题行 | 各种"看起来像章节"的结构信号 |
| 适合文档 | 有规范 Markdown 标题的 | 没标题但有结构标记的(PDF 转 Markdown、纯文本) |
| 面包屑 | 有(HeadingHierarchy 标题栈) | 无(只切不带上下文) |
| 二次切 | 调 SplitText | 调 SplitText(超大块委托) |
| 优先级排序 | 不需要(标题层级本身决定) | 8 级 priority,同位置去重保留最高 |
| 保护区间 | 跳过代码块里的伪标题 | 排除落在保护区间内的切点 |

---

## 几个关键代码细节(值得记)

### 1. `(?m)` 多行模式到处都是

所有"行首匹配"的正则都带 `(?m)`,让 `^` 匹配每行行首而不是整个文本开头。这是写行级正则的标准做法。

### 2. 代码块 ``` 的处理两层都做了

```
if strings.HasPrefix(trimmed, "```") {
    inFence = !inFence
}
if !inFence {
    // 这里才尝试匹配标题正则
}
```

否则代码注释里的 `#include <stdio.h>` 会被当成 H1 标题,代码里的 `## 这是注释` 会被当 H2。这是常见坑,两层都防了。

### 3. Tier 1 的"主导标题级别"算法很讲究

```go
func DominantHeadingLevel() int {
    for level := 1; level <= 6; level++ {
        if MdHeadingCounts[level] >= 3 {
            return level  // 优先找"出现≥3次的最低层级"
        }
    }
    for level := 6; level >= 1; level-- {
        if MdHeadingCounts[level] > 0 {
            return level  // 实在不行取"出现的最深层级"
        }
    }
}
```

为啥要 ≥3 次?防止文档只有一个 H1(标题)和一堆 H2(章节),被 H1 切成"开头 vs 全文"两块毫无意义。要找"真正能切分文档的层级"。

### 4. Tier 2 的"边界优先级"不是用来排序,是用来去重

```go
sort.Slice(bounds, ...)
// 同一位置多个 pattern 命中时,保留 priority 最高的那个
deduped := bounds[:0]
prev := -1
for _, b := range bounds {
    if b.runeStart != prev {
        deduped = append(deduped, b)
        prev = b.runeStart
    }
}
```

priority 不影响切的顺序(切的顺序永远是位置升序),只影响"同一行被多个 pattern 命中时记哪个标签"。

### 5. Tier 2 的 overlap 对齐

`applyOverlapAligned` 不是硬切字符,而是优先对齐到已有边界,找不到再找换行符。这跟 Tier 3 的 `computeOverlap` 思路一致——overlap 要落在语义边界上,不要切断单词/句子。

---

## 这一节你应该带走的 4 件事

1. **Tier 1 用 1 个正则**(`MarkdownHeadingPattern` 匹配 `# 标题`),按"主导标题层级"切,每个 chunk 带标题栈面包屑。
2. **Tier 2 用 8 个正则**(\f/章节/编号/全大写/分隔符/页脚/空行),把所有结构信号当候选切点,按优先级去重后贪心装箱。
3. **两层都跳过 ``` 代码块**,防止代码里的 `# 注释` 被误识别为标题。
4. **超大 section/超大 block 都委托给 SplitText**(Tier 3),三层是协作的 fallback 链。

---

## 下一节可选

- **A. 向量化与入库** —— chunk 怎么变向量?怎么存 pgvector?父子怎么存?
- **B. 校验和诊断** —— `validator.go` 怎么判断"切得不好"?debug 端点暴露啥?
- **C. 切块后处理** —— 问题生成、VLM 描述图片等 enrichment 怎么做?
- **D. 一种具体场景走一遍** —— 比如"用户上传一个带 # 标题的 PDF"端到端走一遍

你想走哪条?