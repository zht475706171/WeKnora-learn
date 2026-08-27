# 05 · Tier 1 大白话详解(回答 5 个问题)

> 用户问了 5 个很尖锐的问题,每个都戳中 Tier 1 设计的核心。这篇按问题逐个讲透。

---

## 问题 1:EmbeddingContent 拼了又不用?不是白做吗?

**这是最有价值的一个问题。先看代码事实:**

### `ContextHeader` 字段的定义(`internal/types/chunk.go`)

```go
type Chunk struct {
    Content       string `json:"content"`           // ← 进数据库,前端能看到
    ...
    ContextHeader string `json:"-" gorm:"type:text"` // ← 进数据库,前端看不到
}
```

注意两个 tag:
- `json:"-"` → 不序列化到 API 响应(前端拿不到)
- `gorm:"type:text"` → **持久化到数据库**(后端能读出来)

所以 ContextHeader **存进了数据库**,只是不给前端看。

### 注释原文(关键证据)

```go
// ContextHeader is a Markdown heading breadcrumb prepended when indexing.
// It is persisted so a later content edit can rebuild the same index input.
```

翻译:**"面包屑在索引时拼接。它被持久化,是为了让以后的 content 编辑能重建出一样的索引输入。"**

### 完整链路(代码实证)

```go
// knowledge_process.go:519
indexContent := buildKnowledgeIndexContent(knowledge, chunk.EmbeddingContent())
//                                  ↑
//   这里把 ContextHeader + Content 拼起来,变成"索引输入"

// knowledge_index_content.go:12
func buildKnowledgeIndexContent(knowledge, content string) string {
    title := knowledge.Title
    return title + "\n" + content   // 再前面拼上文档标题
}

// 所以最终喂给 embedding 模型的文本是:
// "文档标题\n# 公司报销制度\n## 差旅报销\n### 飞机票\n\n正文 A:经济舱凭票报销..."
```

这个 `indexContent` 就是**喂给 embedding 模型的输入**,生成的向量进**向量数据库**(pgvector / Milvus 等)。

### 回答你的疑问

**你的疑问**:拼完直接入向量数据库,Content 进数据库又是另一份,那不是白做吗?

**真相**:数据库里其实存了**两份不同的东西**,服务不同目的:

```
┌─────────────────────────────────────────────────────────────┐
│  PostgreSQL(关系数据库,业务数据)                          │
│  ────────────────────────────────────────────────────────  │
│  chunks 表:                                                │
│    Content       = "正文 A:经济舱凭票报销..."  ← 纯正文     │
│    ContextHeader = "# 公司报销制度\n## 差旅报销\n### 飞机票"│
│    StartAt/EndAt = 位置                                   │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ 入库时 EmbeddingContent() 临时拼接
                          ▼
┌─────────────────────────────────────────────────────────────┐
  喂给 embedding 模型:
  "文档标题\n# 公司报销制度\n## 差旅报销\n### 飞机票\n\n正文 A..."
                          │
                          │ 生成向量
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  向量数据库(pgvector / Milvus / Qdrant 等)                 │
│  ────────────────────────────────────────────────────────  │
│  chunk_id → 向量 [0.012, -0.034, ...]                      │
│  (向量是基于"拼接后的全文"生成的,带章节上下文语义)        │
└─────────────────────────────────────────────────────────────┘
```

**两个库存的不是一回事**:
- **关系库 chunks 表**:存 `Content` + `ContextHeader` + 位置,用于业务(分块编辑、版本历史、UI 高亮、回滚、重建索引)
- **向量库**:存 `向量 + chunk_id`,用于检索(根据 query 找最近邻的 chunk)

### 为啥要 ContextHeader 单独存,不直接拼进 Content

如果直接拼进 Content:
- ❌ `End - Start ≠ 字符数(Content)`,位置不变量被破坏,无法做"按位置 diff、按位置回滚、UI 高亮"
- ❌ 用户编辑 chunk 时,看到的会是 `# 公司报销制度\n## 差旅报销\n### 飞机票\n\n正文 A...`,编辑完根本不知道改了啥
- ❌ 重新索引时,如果标题层级变了,旧的拼接结果丢失,没法重建

单独存 ContextHeader:
- ✅ Content 保持原文精确切片,位置不变量稳定
- ✅ 用户编辑只动 Content,ContextHeader 不变
- ✅ 重新索引时,从数据库读 `ContextHeader + Content` 拼回去,跟第一次一致(代码注释里就是这么写的:"rebuild the same index input")

### 拼的意义在哪?

**拼是为了让向量带有"我在哪一节"的语义**。比如"经济舱凭票报销"这一段,如果没有面包屑,向量只知道"这是关于经济舱报销的";加上面包屑后,向量知道"这是公司报销制度 > 差旅报销 > 飞机票"。

效果差别:用户问"差旅报销怎么算",裸 chunk 也能召回,但排名可能输给"日常报销"的同义文本。带面包屑的 chunk 因为向量空间里"差旅报销"和 query 更近,召回更准、排名更高。

**所以"拼"不是白做,而是把章节上下文注入向量,让检索更准。拼的产物只用于喂 embedding 模型,不存回 Content。**

### 一句话总结

数据库里**两份存储**:
1. `chunks` 表的 `Content`(纯正文)+ `ContextHeader`(面包屑)—— 给业务用(编辑、版本、重建索引)
2. 向量库的"向量"—— 给检索用(基于 `ContextHeader + Content + 文档标题` 生成,带章节语义)

EmbeddingContent() 是临时拼接函数,**只用于生成向量**,不持久化拼接结果(因为下次还要重建)。

---

## 问题 2:这个栈是干啥的,有啥用,只有正则提取到标题才有用对吧?

### 你的理解对一半

栈的作用:**追踪当前 chunk 在文档树里"从根到当前位置"的路径**,不是"最深层那章"。

举例:
```
文档:
# 引言
## 背景
正文 a          ← 此时栈=[引言,背景],路径="# 引言\n## 背景"
## 方法
正文 b          ← 此时栈=[引言,方法],路径="# 引言\n## 方法"
```

走到"正文 b"时,栈是 `[引言, 方法]`,**两层数据都有**。面包屑输出 `# 引言\n## 方法`,不是只有"## 方法"。

### 栈是干嘛用的——具体用途

**栈的唯一用途:在每个 chunk 切出来的时候,把"当前栈状态"快照下来,变成这个 chunk 的 `ContextHeader`。**

代码里就这么一处用:
```go
breadcrumb := hierarchy.BreadcrumbWithHashes()  // 读当前栈
// 然后:
Chunk{
    Content:       sectionContent,   ← 纯正文
    ContextHeader: breadcrumb,       ← 栈快照(就是面包屑)
}
```

走完整个文档,每个 chunk 都拿到自己"出生时刻"的栈快照。这就是栈的全部用途——**给每个 chunk 盖一个"我属于哪个章节"的章**。

### 为啥用栈,不是直接查"这个 chunk 上面的标题"

理论上你也可以在切完每个 chunk 后,扫一遍 chunk 上面的所有标题算出面包屑。但栈的好处:**一次遍历,边走边记,不用回头扫**。整个文档扫一遍,O(n)。

栈的本质是**维护"当前活跃的标题路径"状态**,扫到 H1 就更新 stack[0],扫到 H2 就更新 stack[1],这样任意时刻栈里都是"从根到当前位置"的完整路径。

### 你的最后一个问题:只有正则能提取到标题才用对吧?

**对,完全对。**

`HeadingHierarchy.Observe(line)` 函数第一行就是:
```go
m := MarkdownHeadingPattern.FindStringSubmatch(line)
if m == nil {
    return 0, ""   // 不是标题,栈不变,直接返回
}
```

如果这一行不匹配 `^#{1,6}\s+...` 这个正则,栈根本不动。所以:
- 文档完全没有 Markdown 标题 → 栈永远是空 → 面包屑永远是空字符串 → Tier 1 直接降级到 Tier 3
- 文档只有 H1 → 栈只有 stack[0] 有值 → 面包屑只到 H1 这层
- 文档有完整 H1/H2/H3 → 栈最多到 stack[2] → 面包屑最深 3 层

Tier 2(启发式)和 Tier 3(递归)都不用这个栈,因为它们假设文档没有规范 Markdown 标题,或者干脆不维护章节上下文。

---

## 问题 3:">=3 的最低层级"是 H3 吗?MdHeadingCounts 啥时候记录的?

### "最低层级"指的是哪一层

**这里的"低"指层级数字,不是视觉上的"深"**。H1 是最低层级(数字最小,最靠根),H6 是最高层级。

举例:
- 文档有 H1 5个,H2 3个,H3 8个 → 算法从 level=1 开始扫,H1 出现 5 次 ≥ 3 → 返回 1,primaryLevel=1
- 文档只有 H1 1个(文档标题),H2 5个,H3 8个 → H1 才 1 次 < 3,跳过;H2 5 次 ≥ 3 → 返回 2,primaryLevel=2

代码原文:
```go
func (p *DocProfile) DominantHeadingLevel() int {
    for level := 1; level <= 6; level++ {     // 从 H1 往 H6 扫
        if p.MdHeadingCounts[level] >= 3 {
            return level                       // 第一个满足"≥3次"的就是
        }
    }
    for level := 6; level >= 1; level-- {     // 实在不行从最深的往回找
        if p.MdHeadingCounts[level] > 0 {
            return level
        }
    }
    return 0
}
```

所以"最低层级"=**层级数字最小的那层**=**最靠根的那层**=**最粗的那层标题**。H1 < H2 < H3,数字小的是父,数字大的是子。

### 为啥要 ≥3 次

防坑:如果文档只有一个 H1(就一个文档标题),按 H1 切只会切成 `[文档标题部分, 全部正文]` 两块,毫无意义。≥3 次保证这个层级的标题数量足够把文档切成多个有意义的 section。

### MdHeadingCounts 啥时候记录的

在 `ProfileDocument(text)` 函数里,扫文档每一行时记录的。

```go
// profiler.go
func ProfileDocument(text string) *DocProfile {
    p := &DocProfile{MdHeadingCounts: make(map[int]int)}
    lines := strings.Split(text, "\n")
    for _, line := range lines {
        ...
        if matchHeading(line, &p.MdHeadingCounts) {  // ★ 关键这一行
            p.MdHeadingTotal++
            continue
        }
        ...
    }
    return p
}

// matchHeading 用 MarkdownHeadingPattern 正则匹配
// 如果这行是 `## XX`,matchHeading 会让 MdHeadingCounts[2]++
```

**记录的就是"每级标题各出现了几次"**:
- 看到 `# XX` → `MdHeadingCounts[1]++`
- 看到 `## XX` → `MdHeadingCounts[2]++`
- 看到 `### XX` → `MdHeadingCounts[3]++`

最终 `MdHeadingCounts` 长这样:
```
map[1:5, 2:3, 3:8, 4:0, 5:0, 6:0]
意思:文档里有 5 个 H1,3 个 H2,8 个 H3,没有 H4-H6
```

`DominantHeadingLevel()` 用这个 map 算"主导层级"。

**所以流程是:**
1. `ProfileDocument` 扫一遍文档,把每级标题出现次数记到 `MdHeadingCounts`
2. `DominantHeadingLevel()` 用 `MdHeadingCounts` 算出主导层级
3. `findHeadingBoundaries` 再扫一遍文档,这次只找 `level ≤ primaryLevel` 的标题当切点

---

## 问题 4:"子 chunk 用子 chunk 起始位置对应的活跃标题"没明白,举例

### 场景

文档:
```
# 第一章 引言              ← H1,primaryLevel=1
## 1.1 背景                ← H2
正文 a (40字)
## 1.2 方法                ← H2
正文 b1 (300字)
正文 b2 (300字)
正文 b3 (300字)
# 第二章 结论              ← H1
正文 c
```

ChunkSize=100,所以"## 1.2 方法"这一段(b1+b2+b3 共 900 字)是超大 section,要二次切。

### 不做精细处理会怎样(错误版本)

如果二次切后所有子 chunk 都用 section 顶层的面包屑 `# 第一章 引言`,会变成:

```
chunk_b1: Content="正文 b1", ContextHeader="# 第一章 引言"
chunk_b2: Content="正文 b2", ContextHeader="# 第一章 引言"
chunk_b3: Content="正文 b3", ContextHeader="# 第一章 引言"
```

但 b1 在"1.1 背景"下面,b2/b3 在"1.2 方法"下面——**子 chunk 丢了它真正所属的子标题**。检索"研究方法是什么",b1 的向量跟 b2/b3 的向量分不清,因为面包屑都是 `# 第一章 引言`。

### WeKnora 实际怎么做(正确版本)

代码里有个函数 `sectionBreadcrumbs`,在 section 内部扫一遍,**记录每个子标题的 rune 偏移 + 当时栈状态**:

```go
// sectionBreadcrumbs 走一遍 section 内部,记录:
// [
//   {runeStart: 0,   breadcrumb: "# 第一章 引言"},              ← section 顶
//   {runeStart: 50,  breadcrumb: "# 第一章 引言\n## 1.1 背景"}, ← 1.1 头
//   {runeStart: 200, breadcrumb: "# 第一章 引言\n## 1.2 方法"}, ← 1.2 头
// ]
```

然后对每个子 chunk,看它的 `Start` 位置,在 `sectionBreadcrumbs` 列表里找"最后一个 runeStart ≤ 我的 Start 的":

```go
func breadcrumbAtOffset(bcs, offset, fallback) string {
    for _, e := range bcs {
        if e.runeStart > offset {
            break
        }
        bc = e.breadcrumb   // 一直更新到"最后一个不超过 offset 的"
    }
    return bc
}
```

### 实际结果

```
section "# 第一章 引言"(0-1000)内部:
  位置 0:   "# 第一章 引言"           ← bcs[0]
  位置 50:  "## 1.1 背景"             ← bcs[1],栈更新为 [引言,背景]
  位置 200: "## 1.2 方法"             ← bcs[2],栈更新为 [引言,方法]

二次切 SplitText 把这段切成 9 个子 chunk(每 100 字一个):
  chunk_b1.start=210  → 落在 bcs[2] 之后,面包屑 = "# 第一章 引言\n## 1.2 方法"
  chunk_b2.start=310  → 同上,面包屑 = "# 第一章 引言\n## 1.2 方法"
  chunk_b3.start=410  → 同上,面包屑 = "# 第一章 引言\n## 1.2 方法"

但 b1 实际是 1.1 下面那段(位置 50-150),为啥 b1 在 210?
```

让我重新对齐下,假设二次切结果是:
- chunk_b1: Start=50, End=150(1.1 背景下的正文 a)
- chunk_b2: Start=200, End=400(1.2 方法下的 b1)
- chunk_b3: Start=400, End=600(1.2 方法下的 b2)
- chunk_b4: Start=600, End=800(1.2 方法下的 b3)

查面包屑:
- chunk_b1.Start=50 → bcs 里找 ≤ 50 的最大 runeStart,是 bcs[1](runeStart=50),面包屑 = `# 第一章 引言\n## 1.1 背景`
- chunk_b2.Start=200 → 找到 bcs[2](runeStart=200),面包屑 = `# 第一章 引言\n## 1.2 方法`
- chunk_b3.Start=400 → 还是 bcs[2],面包屑 = `# 第一章 引言\n## 1.2 方法`
- chunk_b4.Start=600 → 还是 bcs[2],面包屑 = `# 第一章 引言\n## 1.2 方法`

**结果**:
```
chunk_b1: Content="正文 a",  ContextHeader="# 第一章 引言\n## 1.1 背景"
chunk_b2: Content="正文 b1", ContextHeader="# 第一章 引言\n## 1.2 方法"
chunk_b3: Content="正文 b2", ContextHeader="# 第一章 引言\n## 1.2 方法"
chunk_b4: Content="正文 b3", ContextHeader="# 第一章 引言\n## 1.2 方法"
```

**每个子 chunk 都拿到了它真正所属的子标题**,而不是糊成 section 顶层。这就是"子 chunk 起始位置对应的活跃标题"的意思——**根据子 chunk 在原文里的位置,反查那个位置当时活跃的标题栈状态**。

---

## 问题 5:用大白话把 Tier 1 讲透

### 一句话

**Tier 1 = 给有 Markdown `#` 标题的文档切蛋糕,按"主导层级"切,每块蛋糕带上"我在第几层抽屉"的标签。**

### 完整流程(一步步走)

**输入**:一段 Markdown 文本 + 配置(ChunkSize=512)

**Step 1:体检**(`ProfileDocument`)
扫一遍文本,统计:
- 每级标题出现几次 → `MdHeadingCounts = {1:5, 2:3, 3:8}`
- 总行数、平均行长、有没有表格、有没有代码、检测到的语言等

**Step 2:选主导层级**(`DominantHeadingLevel`)
从 MdHeadingCounts 算"用哪一级标题切":
- 从 H1 往 H6 扫,找第一个出现 ≥3 次的层级
- H1 5 次 ≥3 → primaryLevel=1,按 H1 切
- 如果只有 H2 出现 5 次,H1 才 1 次 → primaryLevel=2,按 H2 切
- 如果啥标题都没有 → primaryLevel=0,降级到 Tier 3

为啥 ≥3 次?防止文档只有一个 H1(就一个标题),按 H1 切毫无意义。

**Step 3:找切点**(`findHeadingBoundaries`)
再扫一遍文本,这次:
- 跳过 ``` 代码块(否则代码注释 `#include` 会被误识别)
- 每行用 `MarkdownHeadingPattern` 正则匹配
- 命中且 `level ≤ primaryLevel` 的,记一个切点(rune 偏移 + 那行原文)

结果:切点列表 = `[开头位置, 第一个标题位置, 第二个标题位置, ...]`

**Step 4:初始化标题栈**(`NewHeadingHierarchy`)
```go
stack = ["", "", "", "", "", ""]   // 6 个空
depth = 0
```

**Step 5:逐个 section 切**

每两个相邻切点之间是一段 section。对每个 section:

(a) 如果 section 起始是个标题行,调 `hierarchy.Observe(标题行)` 更新栈

(b) 取出栈当前状态当面包屑:
```
breadcrumb := hierarchy.BreadcrumbWithHashes()
// 比如 "# 引言\n## 背景"
```

(c) 同时扫一遍 section 内部的更深层标题(`observeSubHeadings`),让栈保持同步——为啥?因为下个 section 的标题栈要基于"我已经走过哪些子标题"来更新

(d) 判断 section 大小:
```
if 面包屑字符数 + section 字符数 ≤ ChunkSize(512):
    一个 chunk,Content=section 内容,ContextHeader=面包屑
else:
    调 SplitText(Tier 3)二次切 section,
    每个子 chunk 的面包屑 = 它起始位置对应的活跃标题
```

**Step 6:合并碎 chunk**(`coalesceTinyChunks`)
如果某些 section 太短(比如 FAQ 一段只有 30 字),会有很多碎 chunk,校验会判"chunk 太小"失败。把相邻碎 chunk 合并到 ChunkSize/2,合并后面包屑用两个 chunk 共享的标题前缀。

**Step 7:重新编号 Seq**
最后把所有 chunk 的 Seq 从 0 重新连续编号。

### 完整例子走一遍

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
### 餐饮
工作餐按 80 元/人。
```

**Step 1 体检**:H1=1, H2=2, H3=4

**Step 2 选层级**:H1 才 1 次 <3,跳过;H2 2 次 <3,跳过;H3 4 次 ≥3 → primaryLevel=3,按 H3 切

**Step 3 找切点**:扫一遍,匹配 H1/H2/H3,只记 level ≤ 3 的(也就是全部):
- 切点0:0(开头,占位)
- 切点1:"### 飞机票" 位置
- 切点2:"### 酒店住宿" 位置
- 切点3:"## 日常报销" 位置(level=2 ≤ 3,也算切点!)
- 切点4:"### 出租车" 位置
- 切点5:"### 餐饮" 位置

等等,这就有问题了:primaryLevel=3,但 `level ≤ primaryLevel` 意思是 H1/H2/H3 都当切点?这会把文档切得更碎。

**重新看代码确认下:**
```go
if level >= 1 && level <= primaryLevel && pos > 0 {
    bounds = append(bounds, headingBoundary{...})
}
```
确实,`level <= primaryLevel`,所以 H1/H2 都算切点。

那 primaryLevel=3 时,所有 H1/H2/H3 都是切点,section 会很碎。

实际上 Tier 1 的设计意图是:**按"主导层级及更粗的层级"切**,所以 H3 主导时,H1/H2/H3 都切。每个 section 是"一个标题 + 到下个标题前的内容"。

继续走:
- section 0:开头到 "### 飞机票" 前(包含 H1 "公司报销制度"、H2 "差旅报销"两个标题)
- section 1:"### 飞机票" + "经济舱凭票报销..."
- section 2:"### 酒店住宿" + "标准间按城市限额"
- section 3:"## 日常报销"(只是标题行)
- section 4:"### 出租车" + "凭发票报销..."
- section 5:"### 餐饮" + "工作餐按 80 元/人"

**Step 5 切**:
- section 0:栈走完 H1、H2 后,面包屑="# 公司报销制度\n## 差旅报销",内容 = "# 公司报销制度\n## 差旅报销"(50字 <512)→ 一个 chunk,Content=这两行标题,ContextHeader=面包屑
  - 这里要注意:section 内容里**包含**它内部的标题行,因为切点在 H3 的位置,H1/H2 是 section 0 的内容

- section 1:栈走 H3"飞机票",面包屑="# 公司报销制度\n## 差旅报销\n### 飞机票",内容="### 飞机票\n经济舱凭票报销..."(40字)→ 一个 chunk

- section 2:栈走 H3"酒店住宿",面包屑="# 公司报销制度\n## 差旅报销\n### 酒店住宿",内容="### 酒店住宿\n标准间按城市限额"→ 一个 chunk

- section 3:栈走 H2"日常报销"(顶掉 H3"酒店住宿"),面包屑="# 公司报销制度\n## 日常报销",内容="## 日常报销"(很短)

- section 4:栈走 H3"出租车",面包屑="# 公司报销制度\n## 日常报销\n### 出租车",内容="### 出租车\n凭发票报销..."

- section 5:栈走 H3"餐饮",面包屑="# 公司报销制度\n## 日常报销\n### 餐饮",内容="### 餐饮\n工作餐按 80 元/人"

**Step 6 合并**:section 3 太短(就一行标题),可能跟相邻合并。coalesceTinyChunks 看 sharedHeader,比如 section 2 和 section 3 的面包屑前缀都是 `# 公司报销制度\n## 差旅报销` vs `# 公司报销制度\n## 日常报销`,前缀不同不合并。所以 section 3 可能单独存在或被校验判失败降级。

**Step 7 编号**:0,1,2,3,4,5

最终 6 个 chunk,每个都带正确的标题面包屑。

### 为啥要这么设计(灵魂三问)

**为啥按标题切?**
- 同一章节的内容语义相近,跨章节的内容语义跳跃
- chunk 应该"一个章节一段",不要"半章半章"
- 检索时,query 命中一个 chunk,这个 chunk 就是完整的一个小节,上下文连续

**为啥要面包屑?**
- 向量检索本质是相似度比较,query"差旅报销怎么算"和 chunk"经济舱凭票报销"向量不一定很近
- 加上"# 公司报销制度\n## 差旅报销\n### 飞机票"前缀后,query 跟"差旅报销"在向量空间更近,召回更准
- 简单说:**面包屑让向量带着上下文语义,提升检索准确率**

**为啥要把 ContextHeader 和 Content 分开存?**
- Content 必须是原文精确切片,保位置不变量(用于 diff、回滚、UI 高亮)
- ContextHeader 是额外上下文,只用于 embedding,不该污染 Content
- 分开后,用户编辑 Content 不影响 ContextHeader,重新索引能重建一样的输入

---

## 这一节你应该带走的 5 件事(对应你的 5 个问题)

1. **EmbeddingContent() 拼接是为了喂给 embedding 模型生成向量,向量存向量库;Content 和 ContextHeader 分别存在 chunks 表里给业务用**。两份存储,各司其职,不是白做。

2. **栈是"当前从根到当前位置的标题路径"状态,一次遍历边走边记,只有匹配标题正则才更新**。用途:给每个 chunk 盖"我属于哪一节"的章(面包屑)。文档没标题 → 栈空 → 面包屑空 → Tier 1 降级。

3. **">=3 的最低层级"=层级数字最小(最靠根)的层级**。`MdHeadingCounts[level]` 在 `ProfileDocument` 扫文档时记录,记的是"每级标题各出现几次"。≥3 次防坑(文档只有一个 H1 切不出有意义的 chunk)。

4. **"子 chunk 起始位置对应的活跃标题"**:超大 section 二次切后,扫一遍 section 内的子标题记录 (rune偏移, 当时的栈快照),每个子 chunk 按它的 Start 位置查"最后一个不超过它的栈快照"当面包屑。例:chunk_b1.Start=50 对应 `# 引言\n## 1.1 背景`,chunk_b2.Start=200 对应 `# 引言\n## 1.2 方法`。

5. **Tier 1 流程**:体检 → 选主导层级 → 找切点(跳过代码块)→ 维护标题栈 → 每 section 一 chunk(超大二次切)→ 合并碎 chunk → 编号。核心:按 Markdown 标题切,每块带"我在哪一节"标签,让向量带上下文语义。