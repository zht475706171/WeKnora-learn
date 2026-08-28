# 13 · 体检(ProfileDocument)详解 —— 切分前的文档扫描

> 回答用户问题:第一次体检是怎么做的?收集哪些信息?后面干嘛用?

---

## 体检是干嘛的

**一句话**:在切分前扫一遍文档,**统计 11 种结构信号**,决定"这篇文档有资格走哪几层切分策略"。

入口:`profiler.go::ProfileDocument(text)` → 返回 `*DocProfile`。

---

## 收集哪些信息(11 种信号)

### A. 基础统计(3 个)

```go
p.TotalChars = len([]rune(text))           // 总字符数(rune 计数)
p.TotalLines = len(lines)                   // 总行数
p.AvgLineLen = sum / float64(len(lengths))   // 平均行长
p.StdLineLen = math.Sqrt(variance)           // 行长标准差
```

**后面干嘛用**:
- `TotalChars`:跟 ChunkSize 比较,短文档直接走 Tier 3 不折腾
- `TotalLines`:算 HeadingDensity(标题密度)= `MdHeadingTotal / TotalLines`
- `AvgLineLen / StdLineLen`:调试用,目前没直接影响策略(预留信号)

### B. Markdown 标题结构(2 个)

```go
MdHeadingCounts map[int]int  // level(1..6) → 出现次数
MdHeadingTotal  int          // 标题总数
```

**怎么记的**:
```go
if matchHeading(line, &p.MdHeadingCounts) {
    p.MdHeadingTotal++
    continue  // ★ 标题行不再匹配其他 pattern
}
```

`matchHeading` 用 `MarkdownHeadingPattern = (?m)^(#{1,6})\s+(.+?)\s*#*\s*$` 匹配:
- `# 标题` → `MdHeadingCounts[1]++`
- `###### 标题` → `MdHeadingCounts[6]++`

**后面干嘛用**:
- `MdHeadingCounts` → 算 `DominantHeadingLevel`(Tier 1 的主导切分层级)
- `MdHeadingTotal` → SelectStrategy 判断 Tier 1 候选资格(≥3 且密度 > 0.5%)
- `HeadingDensity()` = `MdHeadingTotal / TotalLines` → 防止"标题多但占比极低"误判

### C. 启发式结构信号(8 个)

```go
NumberedSectionCount     // "1.1 Foo" 编号标题数
AllCapsShortLineCount    // "INTRODUCTION" 全大写标题数
VisualSepCount           // "---" 视觉分隔线数
FormFeedCount            // \f 分页符数
GermanChapterCount       // "Kapitel 1" 德文章节数
EnglishChapterCount      // "Chapter 1" 英文章节数
ChineseChapterCount      // "第一章" 中文章节数
RepeatedFooterCount      // "Page 2 of 10" 页脚数
```

**怎么记的**:逐行匹配 8 个正则(见 patterns.go)。

**关键**:跳过 ` ``` ` 代码块内的行(`inFence` 状态机),防代码注释里的 `#1` `---` 被误识别。

**后面干嘛用**:
- `HeuristicMarkerTotal()` = 这些信号的总和 → SelectStrategy 判断 Tier 2 候选资格(≥5 个,或有分页符,或有中英德章节)
- 各个 count 目前**不单独决定策略**,只汇总成 total

### D. 多空行 + 表格 + 代码(3 个)

```go
BlankParagraphBreaks = strings.Count(text, "\n\n\n")  // 3+ 连续换行
HasTables  // 有没有表格行(|...|)
HasCode    // 有没有 ``` 代码块
CodeRatio  // 代码字符占比 = codeChars / TotalChars
```

**后面干嘛用**:
- `BlankParagraphBreaks`:目前**没**直接用(预留信号)
- `HasTables / HasCode`:目前**没**直接用(预留给下游 UI 展示)
- `CodeRatio`:目前**没**直接用(预留信号)

### E. 语言检测(1 个)

```go
sample := text
if len(sample) > 4096 {
    sample = sample[:4096]  // ★ 采样 4096 字符,防大文档 O(N)
}
lang := DetectLanguage(sample)
p.DetectedLangs = []string{lang}
if lang == LangMixed {
    p.DetectedLangs = []string{LangEnglish, LangGerman, LangChinese}
}
```

**怎么检测**(`DetectLanguage`):
- 数 CJK 字符数(中日韩)、Latin 字符数、德语变音字符(äöüß)
- CJK 占比 ≥30% → 中文
- CJK 和 Latin 都 ≥15% → 混合
- 有德语变音 或 含德语停用词("der die das und ist") → 德语
- 否则 → 英文

**后面干嘛用**:
- 选 Tier 2 的章节正则(`ChapterPatternsForLangs`):中文文档只用 `ChineseChapterPattern`
- 算 token 预算(`CharsForTokenLimit`):中文 1.7 字/token,英文 4.0 字/token,德语 4.5 字/token,混合 3.0 字/token
- 默认分隔符:不同语言句末不同(中文 `。`,英文 `. `)

---

## 体检完整流程(profiler.go:88-190)

```go
func ProfileDocument(text string) *DocProfile {
    p := &DocProfile{MdHeadingCounts: make(map[int]int)}
    if text == "" { return p }

    p.TotalChars = len([]rune(text))
    p.FormFeedCount = strings.Count(text, "\f")

    lines := strings.Split(text, "\n")
    p.TotalLines = len(lines)

    // 第一遍:逐行扫
    inFence := false
    codeChars := 0
    for _, line := range lines {
        trimmed := strings.TrimSpace(line)
        // 1. 代码块状态机
        if strings.HasPrefix(trimmed, "```") {
            inFence = !inFence
            p.HasCode = true
            continue
        }
        if inFence {
            codeChars += len([]rune(line))
            continue  // ★ 代码块内的行不匹配任何 pattern
        }
        // 2. 行长统计
        runeLen := len([]rune(line))
        lengths = append(lengths, float64(runeLen))
        // 3. 标题匹配(命中就 continue,不再匹配其他)
        if matchHeading(line, &p.MdHeadingCounts) {
            p.MdHeadingTotal++
            continue
        }
        // 4. 8 个启发式正则
        if NumberedSectionPattern.MatchString(line) { p.NumberedSectionCount++ }
        if GermanChapterPattern.MatchString(line)   { p.GermanChapterCount++ }
        if EnglishChapterPattern.MatchString(line)  { p.EnglishChapterCount++ }
        if ChineseChapterPattern.MatchString(line)   { p.ChineseChapterCount++ }
        if AllCapsHeadingPattern.MatchString(line)   { p.AllCapsShortLineCount++ }
        if VisualSeparatorPattern.MatchString(line)  { p.VisualSepCount++ }
        if PageFooterPattern.MatchString(line)       { p.RepeatedFooterCount++ }
        // 5. 表格行检测
        if strings.HasPrefix(trimmed, "|") && strings.HasSuffix(trimmed, "|") {
            p.HasTables = true
        }
    }

    // 算平均行长 + 标准差
    if len(lengths) > 0 { ... p.AvgLineLen ... p.StdLineLen ... }

    // 代码占比
    if p.TotalChars > 0 {
        p.CodeRatio = float64(codeChars) / float64(p.TotalChars)
    }

    // 多空行数(全文 count,不是逐行)
    p.BlankParagraphBreaks = strings.Count(text, "\n\n\n")

    // 语言检测(采样 4096 字符)
    sample := text
    if len(sample) > 4096 { sample = sample[:4096] }
    lang := DetectLanguage(sample)
    p.DetectedLangs = []string{lang}
    if lang == LangMixed {
        p.DetectedLangs = []string{LangEnglish, LangGerman, LangChinese}
    }
    return p
}
```

---

## 算主导层级(DominantHeadingLevel)

体检后,根据 `MdHeadingCounts` 算主导切分层级:

```go
func (p *DocProfile) DominantHeadingLevel() int {
    if p.MdHeadingTotal == 0 { return 0 }
    // 首选:从 H1 往 H6 扫,第一个出现 ≥3 次的层级
    for level := 1; level <= 6; level++ {
        if p.MdHeadingCounts[level] >= 3 { return level }
    }
    // 退化:从最深的往回找,有任何出现的就用
    for level := 6; level >= 1; level-- {
        if p.MdHeadingCounts[level] > 0 { return level }
    }
    return 0
}
```

**2 段逻辑**:
1. **首选**:≥3 次的最低层级(防文档只有 1 个 H1 切了没意义)
2. **退化**:实在没 ≥3 次的,用最深的层级(有切点总比没切点好)

**后面干嘛用**:Tier 1 的 `splitByHeadingsImpl` 用这个 `primaryLevel` 找切点(`level <= primaryLevel` 的标题当切点)。

---

## 体检结果怎么用(SelectStrategy)

`profiler.go::SelectStrategy(p)` 用 DocProfile 选策略链:

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

### 3 个判断条件

**Tier 1 候选**(3 个全满足):
- `MdHeadingTotal >= 3`(至少 3 个标题)
- `HeadingDensity() > 0.005`(标题密度 > 0.5%,防"3 个标题但 10 万行正文")
- `DominantHeadingLevel() > 0`(有主导层级,=0 表示没标题)

**Tier 2 候选**(3 个满足任一):
- `HeuristicMarkerTotal() >= 5`(8 种启发式信号总和 ≥5)
- `FormFeedCount > 0`(有分页符)
- `中英德章节 > 0`(有章节标记)

**Tier 3 候选**:**永远加**(兜底)

### 3 种典型链

| 文档特征 | chain |
|---------|------|
| 有 `#` 标题 + 有 `第一章` | `[Tier1, Tier2, Tier3]` |
| 无 `#`,但有 `第一章/1.1/\f` | `[Tier2, Tier3]` |
| 啥结构都没(纯流水账) | `[Tier3]` |

---

## 一个完整例子

**输入文档**:
```markdown
# 公司报销制度

## 差旅报销
### 飞机票
经济舱凭票报销。

### 酒店住宿
标准间按城市限额。

## 日常报销
### 出租车
凭发票报销。
```

### 体检结果

```go
DocProfile {
    TotalChars: 80,
    TotalLines: 12,
    AvgLineLen: 6.7,
    StdLineLen: 5.2,

    MdHeadingCounts: {1:1, 2:2, 3:4},  // 1 个 H1, 2 个 H2, 4 个 H3
    MdHeadingTotal: 7,

    NumberedSectionCount: 0,
    AllCapsShortLineCount: 0,
    VisualSepCount: 0,
    FormFeedCount: 0,
    GermanChapterCount: 0,
    EnglishChapterCount: 0,
    ChineseChapterCount: 0,
    RepeatedFooterCount: 0,

    BlankParagraphBreaks: 0,
    HasTables: false,
    HasCode: false,
    CodeRatio: 0,

    DetectedLangs: ["zh"],
}
```

### SelectStrategy 判断

```
Tier 1 候选:
  MdHeadingTotal=7 >= 3 ✓
  HeadingDensity = 7/12 ≈ 0.58 > 0.005 ✓
  DominantHeadingLevel() = 3(H1 才 1 <3,H2 才 2 <3,H3 4 ≥3)
  → 加入 TierHeading

Tier 2 候选:
  HeuristicMarkerTotal() = 0 < 5 ✗
  FormFeedCount = 0 ✗
  中英德章节 = 0 ✗
  → 不加入 TierHeuristic

Tier 3 候选:
  永远加 → 加入 TierLegacy

chain = [TierHeading, TierLegacy]
```

### 最终走 Tier 1,失败降级 Tier 3

Tier 1 用 `primaryLevel=3` 切 H1/H2/H3,每段带面包屑,失败就落 Tier 3。

---

## 体检的意义总结

| 信号 | 后面干嘛用 |
|------|-----------|
| `TotalChars` | 短文档直接 Tier 3 |
| `TotalLines` | 算标题密度 |
| `MdHeadingCounts` + `MdHeadingTotal` | 算 DominantHeadingLevel + 判断 Tier 1 候选 |
| `HeuristicMarkerTotal`(8 种信号汇总) | 判断 Tier 2 候选 |
| `FormFeedCount` + `中英德章节` | Tier 2 候选的"或"条件 |
| `DetectedLangs` | Tier 2 选章节正则 + 算 token 预算 + 句末分隔符 |
| `AvgLineLen/StdLineLen/HasTables/HasCode/CodeRatio/BlankParagraphBreaks` | 预留信号,目前没直接影响策略 |

**核心**:体检是**一次性 O(N) 扫描**,采集 11 种信号,驱动 3 件事:
1. **选策略链**(SelectStrategy)
2. **选主导切分层级**(DominantHeadingLevel)
3. **选语言相关配置**(分隔符、token 预算、章节正则)

---

## 你应该带走的 5 件事

1. **体检 = 一次性 O(N) 扫描**,采集 11 种信号,不重复扫。入口 `ProfileDocument(text)`,返回 `*DocProfile`。

2. **11 种信号分 5 类**:基础统计(3) + Markdown 标题(2) + 启发式结构(8) + 表格代码空行(3) + 语言(1)。其中**真正影响策略的是前 3 类**(标题、启发式、语言)。

3. **Markdown 标题用 `MdHeadingCounts[1..6]` 记每级数量**,用来算 `DominantHeadingLevel`(≥3 次的最低层级,退化是最深的层级)。

4. **8 个启发式正则逐行匹配**,跳过 ``` 代码块内的行。汇总成 `HeuristicMarkerTotal` 判断 Tier 2 候选资格(≥5 或有分页符或有中英德章节)。

5. **语言检测采样 4096 字符**,算 CJK/Latin/德语变音比例。用来选 Tier 2 章节正则、算 token 预算(中文 1.7 字/token)、定句末分隔符。