# 10 · Tier 3 递归切(SplitText)详解 —— 兜底机制

> Tier 3 是最底层的安全网。Tier 1 失败、Tier 2 失败,都会落到这里。如果这里再失败,文档就真的切不出来了。它也是 Tier 1/Tier 2 内部超大 block 的"内切工具"。

---

## 一、Tier 3 是啥地位

```
Tier 1(标题切)失败  ─┐
                      ├─→ fall through ─→ Tier 3(SplitText)
Tier 2(启发式)失败 ─┘
```

- **Tier 1 超大 section 二次切**:调 `SplitText`
- **Tier 2 超大 block 二次切**:调 `SplitText`
- **Tier 1/Tier 2 校验失败降级**:调 `SplitText`
- **Tier 3 自己校验失败**:**不再降级**,返回最后结果(strategy.go 的兜底逻辑)

代码入口:`splitter.go::SplitText`(915 行的文件,核心就是这一个函数 + 它的辅助函数)。

---

## 二、SplitText 的 3 步主流程

```go
func SplitText(text string, cfg SplitterConfig) []Chunk {
    // Step 1: 找保护区间(LaTeX/图片/链接/表格/代码/行内代码)
    protected := protectedSpans(text)
    // Step 2: 把非保护区按分隔符切,保护区当原子单元
    units := buildUnitsWithProtection(text, protected, separators, chunkSize)
    // Step 3: 把 units 贪心合并成 chunk,带 overlap
    return mergeUnits(units, chunkSize, chunkOverlap)
}
```

3 步:找保护 → 切+保原子 → 合并+overlap。

---

## 三、7 种保护模式(protectedPatterns)

```go
var protectedPatterns = []*regexp.Regexp{
    `(?s)\$\$.*?\$\$`,                       // 1. LaTeX 块级公式 $$...$$
    `!\[[^\]]*\]\([^)]+\)`,                   // 2. Markdown 图片 ![alt](url)
    `\[[^\]]*\]\([^)]+\)`,                    // 3. Markdown 链接 [text](url)
    "(?m)[ ]*(?:\\|[^|\\n]*)+\\|[\\r\\n]+\\s*(?:\\|\\s*:?-{3,}:?\\s*)+\\|[\\r\\n]+",  // 4. 表格头+分隔行
    "(?m)[ ]*(?:\\|[^|\\n]*)+\\|[\\r\\n]+",   // 5. 表格行
    "(?s)```(?:\\w+)?[\\r\\n].*?```",         // 6. 代码块 ```...```
    "`[^`\\r\\n]+`",                          // 7. 行内代码 `code`
}
```

**为啥要保护**:
- LaTeX 公式被切碎 → 公式失效,向量没语义
- 图片/链接被切碎 → URL 被切断,渲染失败
- 表格被切碎 → 列对齐丢失,无法理解
- 代码块被切碎 → 代码不完整,可能从函数中间截断

**保护机制**:保护区内的文本**作为一个原子 unit**,不参与 split,整段塞进一个 chunk。如果保护单元自己超 `maxProtectedSize=7500`,才强制硬切(在换行或空格处对齐)。

---

## 四、递归分隔符切分(splitBySeparators)

默认分隔符:
```go
Separators: []string{"\n\n", "\n", "。"}
```

**递归语义**(注释强调 "Python-parity recursive split"):

```go
func splitBySeparators(text, separators, chunkSize) []string {
    for i, sep := range separators {
        // 用第 i 个分隔符切
        // 切完的 piece 如果 > chunkSize,用剩下的分隔符(separators[i+1:])递归内切
        for _, p := range pieces {
            if len(p) > chunkSize && len(remaining) > 0 {
                out = append(out, splitBySeparators(p, remaining, chunkSize)...)
            } else {
                out = append(out, p)
            }
        }
        return out
    }
}
```

**递归顺序**:
1. 先按 `\n\n` 切(段落级)
2. 段落还太大 → 按 `\n` 切(行级)
3. 行还太大 → 按 `。` 切(句子级,中文句号)
4. 都切不动 → 保持原样(后面 mergeUnits 会硬切)

**关键**:递归是**对"还太大的 piece"用下一级分隔符**,不是"对整个文本换分隔符重切"。这样保留层级关系:大段内的句子不会跑到别的段去。

### 例子

```
原文(假设 ChunkSize=100):
段落A(150字,>100)
段落B(80字,<100)
```

Step 1:按 `\n\n` 切 → `[段落A, 段落B]`
Step 2:段落A > 100,用剩余分隔符 `["\n", "。"]` 递归内切
  - 按 `\n` 切段落A → `[行A1(70字), 行A2(80字)]`(都 <100,不再递归)
Step 3:段落B < 100,保留
最终:`[行A1, 行A2, 段落B]`

---

## 五、buildUnitsWithProtection —— 保护+切分组合

这个函数把"保护区间"和"递归切分"组合起来:

```go
for _, p := range protected {
    // 1. 保护区间之前的普通文本:用 splitBySeparators 切
    if p.start > bytePos {
        pre := text[bytePos:p.start]
        parts := splitBySeparators(pre, separators, chunkSize)
        // 每个 part 变成一个 splitUnit(rune 偏移)
    }
    // 2. 保护区间本身:作为单个原子 unit
    if protRuneLen > maxProtectedSize {
        // 太大,硬切(换行/空格对齐)
    } else {
        // 整段当一个 unit
    }
}
// 3. 最后一个保护区间之后的文本:用 splitBySeparators 切
```

**输出**:`[]splitUnit`,每个 unit 记录 `{text, start, end}`,start/end 是 **rune 偏移**(不是 byte 偏移)。

### 例子

```
原文:"前言\n\n$$E=mc^2$$\n\n后语"
```

1. 找保护:`$$E=mc^2$$` 是 LaTeX,保护区间 [byte 8:19]
2. 保护前:"前言\n\n" → 按 `\n\n` 切 → `["前言", "\n\n"]`
3. 保护内:"$$E=mc^2$$" → 原子 unit
4. 保护后:"\n\n后语" → 按 `\n\n` 切 → `["\n\n", "后语"]`

units:
```
{前言, 0, 2}
{\n\n, 2, 4}
{$$E=mc^2$$, 4, 13}      ← 原子,不切
{\n\n, 13, 15}
{后语, 15, 17}
```

---

## 六、mergeUnits —— 贪心合并 + overlap + 表格头追踪

这是 SplitText 最复杂的一步。核心逻辑:

```go
func mergeUnits(units, chunkSize, chunkOverlap) []Chunk {
    const absoluteMaxSize = 7500   // 硬上限,防超 embedding API 限制
    ht := newHeaderTracker()        // 表格头追踪器

    for _, u := range units {
        uLen := runeLen(u.text)

        // 1. 单个 unit 超 absoluteMaxSize:硬切
        if uLen > absoluteMaxSize { ... 硬切对齐换行/空格 ... }

        // 2. 更新表格头状态
        ht.update(u.text)
        // 表格结束 → flush 当前累积(防两表糊一起)
        if ht.headerEndedThisUnit && len(current) > 0 { flush }

        // 3. 累积会超 ChunkSize → flush + overlap
        if curLen+uLen+headersLen > chunkSize && len(current) > 0 {
            flush
            current, curLen = computeOverlap(current, chunkOverlap, chunkSize, uLen)
            // 如果有活跃表格头,prepend 到下一个 chunk
            if headers != "" && !headerAlreadyPresent(...) {
                current = append([]splitUnit{hUnit}, current...)
            }
        }

        // 4. 累积会超 absoluteMaxSize:flush 无 overlap
        if curLen+uLen > absoluteMaxSize { flush; current=nil; curLen=0 }

        // 5. 加入累积
        current = append(current, u)
        curLen += uLen
    }
    flush 剩余
}
```

### 6.1 表格头追踪(headerTracker)—— 关键设计

**问题**:一个超大 Markdown 表格被切到多个 chunk,第二个 chunk 开始就是数据行 `| 数据1 | 数据2 |`,**没有表头**,检索时用户看不出这数据是哪一列的。

**解决**:`headerTracker` 追踪活跃表头,每次切新 chunk 时把表头 **prepend** 到新 chunk 开头。

```go
type headerTracker struct {
    activeHeaders map[int]string  // priority -> 表头文本
    endedHeaders  map[int]bool
    pendingExtend map[int]bool
    pendingTableBreak bool
    headerEndedThisUnit bool
}
```

工作机制:
1. **检测表头开始**:遇到 `| A | B |\n| --- | --- |\n` 这样的表头+分隔行,记为活跃表头
2. **检测表头结束**:遇到空行或非 `|` 开头的行,表头失效
3. **检测新表开始**:列数不匹配 → 旧表头失效,新表头开始
4. **空表头补全**:MarkItDown 转换器有时生成空表头 `||\n| --- | --- |\n`,遇到第一个数据行时用它补全表头

### 6.2 表格头 prepend 的去重

```go
// 防止重复 prepend
if !headerAlreadyPresent(headers, overlapText, u.text) &&
    !headerColumnMismatch(headers, u.text) {
    // prepend 表头到新 chunk
}
```

- `headerAlreadyPresent`:overlap 或下一个 unit 已经包含表头 → 不重复 prepend
- `headerColumnMismatch`:下一个 unit 是不同列数的表 → 不 prepend 旧表头(会启动新表头追踪)

### 6.3 表格边界 flush

```go
if ht.headerEndedThisUnit && len(current) > 0 {
    chunks = append(chunks, buildChunk(current, len(chunks)))
    current = nil
    curLen = 0
}
```

表头结束(空行/非表行)时 flush,防止"上一个表的数据"和"下一个表的数据"糊进同一个 chunk 但表头只带第一个的。

---

## 七、overlap 对齐(computeOverlap)

**问题**:naive overlap 是"取末尾 chunkOverlap 个字符",但可能切在词中间/行中间。

**解决**:`computeOverlap` 用语义边界对齐:

```go
func computeOverlap(current, chunkOverlap, chunkSize, nextLen) ([]splitUnit, int) {
    if chunkOverlap <= 0 { return nil, 0 }
    // overlap 受 chunkSize - nextLen 限制(防下个 chunk 超)
    maxOverlap := chunkOverlap
    if remaining := chunkSize - nextLen; remaining < maxOverlap {
        maxOverlap = remaining
    }
    // 从 current 末尾取 maxOverlap + 4 个 rune 的窗口(4 = \r\n\r\n 最长分隔符)
    window := semanticOverlapWindow(current, maxOverlap+4)
    // 在窗口里找语义边界,优先级:
    //   1. 段落分隔(\n\n / \r\n\r\n)
    //   2. 行分隔(\n / \r\n)
    //   3. 句末(。？！, ". " "? " "! ")
    boundaryEnd, ok := findSemanticOverlapBoundaryEndingAtOrAfter(windowText, originalWindowStart)
    if !ok { return nil, 0 }  // 找不到语义边界 → 无 overlap
    // 对齐到边界
    overlap := trimUnitsPrefix(window, boundaryEnd)
    return overlap, overlapLen
}
```

### 3 级优先级

```
优先级 1:段落分隔 \n\n / \r\n\r\n    ← 最强,段落边界
优先级 2:行分隔 \n / \r\n            ← 次强,行边界
优先级 3:句末 。？！ / ". " "? " "! "  ← 最弱,句子边界
```

### 关键约束

1. **边界必须在窗口内**:窗口 = `maxOverlap + 4` 个 rune,4 是 `\r\n\r\n` 的最长分隔符长度(lookbehind)
2. **边界后必须有内容**:`hasMeaningfulTail` 检查边界后不是纯空白
3. **边界不在保护区间内**:`insideProtected` 排除 LaTeX/代码/表格里的边界
4. **同优先级取最早**:让 overlap 尽量大(保留更多上下文)
5. **找不到边界 → 无 overlap**:宁可不 overlap 也不切词中间

### 例子

```
chunk 末尾:"...这是第一段。\n\n这是第二段开头"
maxOverlap=80,lookbehind=4
窗口 = "...这是第一段。\n\n这是第二段开"(84 字符)

找边界:
  位置 X1:"。"(句末,优先级 3)
  位置 X2:"\n\n"(段落,优先级 1)  ← 最强
  位置 X3:"\n"(行,优先级 2,但被 \n\n 标记跳过)

选 X2(优先级 1 最强)
overlap = "这是第二段开"(X2 之后的部分)
下一个 chunk 从 "这是第二段开" 开始
```

---

## 八、两个硬上限

```go
const maxProtectedSize = 7500   // buildUnitsWithProtection 里
const absoluteMaxSize = 7500    // mergeUnits 里
```

**为啥是 7500**:留余量给 embedding API 的 token 限制(通常 8K-10K token)。7300 字符 + 表头/前缀 ≈ 8000 token,卡在 API 上限以内。

**两个硬上限的分工**:
- `maxProtectedSize`:**保护单元**的最大尺寸,超了就硬切保护单元(LaTeX/代码可能很长)
- `absoluteMaxSize`:**任何 chunk**的最大尺寸,超了就硬切(防御性)

---

## 九、完整例子走一遍

**输入**(ChunkSize=100,overlap=20):
```
这是引言段落,介绍研究背景,内容比较短。

$$E = mc^2$$

这是方法段落,描述技术路线,内容也比较短。
```

### Step 1:找保护区间

`$$E = mc^2$$` 是 LaTeX,保护区间 = [byte 偏移 X1:X2]

### Step 2:buildUnitsWithProtection

- 保护前:"这是引言段落,介绍研究背景,内容比较短。\n\n" → 按 `\n\n` 切 → `["这是引言...", "\n\n"]`
- 保护内:"$$E = mc^2$$" → 原子 unit
- 保护后:"\n\n这是方法段落,描述技术路线,内容也比较短。" → 按 `\n\n` 切 → `["\n\n", "这是方法..."]`

units(rune 偏移):
```
0: {这是引言段落...短。, 0, 20}
1: {\n\n, 20, 22}
2: {$$E = mc^2$$, 22, 33}      ← 原子
3: {\n\n, 33, 35}
4: {这是方法段落...短。, 35, 55}
```

### Step 3:mergeUnits(ChunkSize=100,overlap=20)

**遍历 units**:

u0(20字):curLen=0+20=20 < 100,current=[u0]
u1(2字):curLen=22 < 100,current=[u0,u1]
u2(11字,LaTeX):curLen=33 < 100,current=[u0,u1,u2]
u3(2字):curLen=35 < 100,current=[u0,u1,u2,u3]
u4(20字):
  - 检查:curLen+uLen = 35+20 = 55 < 100,不 flush
  - current=[u0,u1,u2,u3,u4],curLen=55

循环结束,flush:
- chunk 0:Content = "这是引言段落...短。\n\n$$E = mc^2$$\n\n这是方法段落...短。"(55字),Start=0,End=55

### 假设 ChunkSize=40 重走

u0(20字):curLen=20 < 40,current=[u0]
u1(2字):curLen=22 < 40,current=[u0,u1]
u2(11字):curLen=33 < 40,current=[u0,u1,u2]
u3(2字):35 < 40,current=[u0,u1,u2,u3]
u4(20字):
  - 检查:35+20=55 > 40,且 current 非空 → flush!
  - chunk 0:Content = "这是引言段落...短。\n\n$$E = mc^2$$\n\n"(35字),Start=0,End=35
  - computeOverlap(current, overlap=20, chunkSize=40, nextLen=20):
    - maxOverlap = min(20, 40-20=20) = 20
    - 窗口 = 末尾 24 个 rune(20+4)
    - 找边界:窗口里有 `\n\n`(优先级 1)→ 选它
    - overlap = `\n\n`(2字)
  - current = [overlap=`\n\n`],curLen=2
  - 无活跃表头,不 prepend
  - 加入 u4:current=[\n\n, u4],curLen=22

循环结束 flush:
- chunk 1:Content = "\n\n这是方法段落...短。"(22字),Start=33,End=55

### 最终(ChunkSize=40)

```
chunk 0: "这是引言段落...短。\n\n$$E = mc^2$$\n\n"(35字)
chunk 1: "\n\n这是方法段落...短。"(22字,含 2 字 overlap)
```

注意 chunk 1 的 Start=33,跟 chunk 0 的 End=55 有 22 字符的 overlap 区间(33~55),Content 长度 = 22 = End-Start,**位置不变量保住**。

---

## 十、几个关键设计点

### 1. Chunk 位置不变量

```go
// Chunk struct 注释
// End-Start == utf8.RuneCountInString(Content)
```

每个 chunk 的 Content 必须是原文精确切片,`End - Start` 严格等于 Content 的 rune 数。这是下游重建文档、UI 高亮、版本 diff 的基础。

**保护机制**:
- splitUnit 的 start/end 是 rune 偏移
- buildChunk 用 `units[0].start` 和 `units[-1].end`
- 表头 prepend 的 hUnit 是 `start=end=startPos`(零宽),不破坏不变量
- computeOverlap 的 overlap 单元保留原 start/end

### 2. 递归切分的 Python 同源

```go
// Mirrors the recursive priority semantics of the Python reference splitter
// (docreader/splitter/splitter.py:_split)
```

Tier 3 是从 Python `docreader/splitter/splitter.py` 移植的,递归语义保持一致:大 piece 用下一级分隔符内切,不是全文换分隔符重切。

### 3. 表格头 prepend 的位置陷阱

```go
hUnit := splitUnit{text: headers, start: startPos, end: startPos}  // start == end!
```

表头是"额外 prepend 的上下文",**不是原文的一部分**,所以 `start == end`(零宽)。这样它参与 chunk 的 Content 拼接,但不影响 `End - Start = runeCount(Content)` 不变量。

但 `buildChunk` 用 `units[0].start` 当 chunk.Start —— 如果 hUnit 是 units[0],chunk.Start 会是 `startPos`,看起来对,但 hUnit 的 start 是"伪造"的(用 overlap 或下个 unit 的 start)。这是接受的:位置不变量只要求 `End - Start = runeCount(Content)`,hUnit 的零宽特性保证这点。

### 4. absoluteMaxSize = 7500 是 embedding API 限制

留余量给表头、文档标题等前缀。7300 字符 ≈ 8000 token,卡在大多数 embedding API 的 8K-10K 限制以内。

### 5. 保护单元的硬切对齐

```go
for i := chunkEnd - 1; i > offset && i > chunkEnd-200; i-- {
    if runes[i] == '\n' || runes[i] == ' ' {
        chunkEnd = i + 1
        break
    }
}
```

保护单元超 7500 时硬切,在末尾 200 字符内找最近的 `\n` 或空格对齐,避免切在词中间。

---

## 十一、Tier 3 的校验

Tier 3 切完同样跑 `ValidateChunks`(validator.go),4 个判据:
1. 碎 chunk 数 > 1/4 且 > 2 → 失败
2. maxLen < ChunkSize/4 且文档 > ChunkSize → 失败
3. maxLen > 2*ChunkSize → 失败
4. 1 chunk 且文档 > 2*ChunkSize → 失败

**但 Tier 3 失败后不再降级**(strategy.go:50-57):
```go
if tier == TierLegacy && i == len(chain)-1 {
    lastOut = out  // 即使校验失败,也保留 Tier 3 的结果
}
// 实在没办法,返回 SplitText(text, cfg)
return SplitText(text, cfg)
```

**Tier 3 是最终兜底**:即使切得不完美,也要返回个结果,不能空。

---

## 十二、Tier 1 / Tier 2 / Tier 3 对比

| 维度 | Tier 1 (heading) | Tier 2 (heuristic) | Tier 3 (recursive) |
|------|-----------------|--------------------|--------------------|
| 触发 | 有 `#` 标题 | 有结构信号 | 兜底/超大内切 |
| 找切点 | MarkdownHeadingPattern | 8 个正则 | 分隔符递归(\n\n \n 。) |
| 标题栈 | 6 级栈 + 面包屑 | 无 | 无 |
| 保护区间 | 跳过 ``` 代码块 | 7 种(dropBoundsInsideSpans) | 7 种(buildUnitsWithProtection) |
| 二次切 | SplitText + sectionBreadcrumbs | SplitText(位置还原) | 自己就是 SplitText |
| overlap | 边界对齐 | applyOverlapAligned 3 级退化 | computeOverlap 3 级优先级 |
| 碎处理 | coalesceTinyChunks 合并 | minChunkSize 防过早封口 | 无(靠 overlap 平滑) |
| 表格头 | 无特殊处理 | 无特殊处理 | headerTracker 追踪 + prepend |
| 硬上限 | 无 | 无 | 7500(absoluteMaxSize) |
| 校验失败 | 降级 Tier 2 | 降级 Tier 3 | 不降级,返回最后结果 |

---

## 十三、你应该带走的 7 件事

1. **Tier 3 是最底层安全网**:Tier 1/Tier 2 失败都落到这里,Tier 3 失败也返回结果不空。它也是 Tier 1/Tier 2 内部超大 block 的内切工具。

2. **3 步主流程**:`protectedSpans` 找保护 → `buildUnitsWithProtection` 切+保原子 → `mergeUnits` 贪心合并+overlap。

3. **7 种保护模式**:LaTeX `$$`、图片 `![]()`、链接 `[]()`、表格头+分隔行、表格行、代码块 ` ``` `、行内代码 `` ` ``。保护单元作原子,超 7500 才硬切。

4. **递归分隔符切分**:默认 `["\n\n", "\n", "。"]`,大 piece 用下一级分隔符内切,不是全文重切。保留层级关系。

5. **表格头追踪(headerTracker)**:超大表格切到多 chunk 时,每个 chunk 开头 prepend 表头(去重 + 列数检查),防数据行丢失列语义。表头单元是零宽(`start==end`),不破坏位置不变量。

6. **overlap 用 computeOverlap 对齐**:3 级优先级(段落 \n\n > 行 \n > 句末 。？！/. ),窗口 = maxOverlap + 4(lookbehind),找不到语义边界就无 overlap,绝不切词中间。

7. **两个硬上限 7500**:`maxProtectedSize`(保护单元)+ `absoluteMaxSize`(任何 chunk),留余量给 embedding API 的 token 限制(8K-10K)。

8. **位置不变量 `End - Start = runeCount(Content)`**:所有设计都围绕这个不变量,hUnit 零宽、overlap 保留原偏移、buildChunk 用 units[0].start 和 units[-1].end —— 都是为了保住它。