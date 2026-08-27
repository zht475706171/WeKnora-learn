# 07 · Tier 1 用户描述纠错(对照代码逐条核对)

> 用户给出自己对 Tier 1 的总结描述,问"我描述的有问题吗"。这篇按代码逐条核对,指出对的部分和说反的部分。

---

## ✅ 说对的部分

1. **"判断是否可以通过标题进行切割"** —— 对。`DominantHeadingLevel()==0` 就降级到 Tier 3。

2. **"最多存储 6 级标题"** —— 对。`HeadingHierarchy.stack [6]string` 固定数组。

3. **"面包屑就是从根等级标题拼接到当前等级标题"** —— 对。`BreadcrumbWithHashes()` 把栈里非空的层从上到下拼成 `# XX\n## YY\n### ZZ`。

4. **"弊端是整章短可能都切到一个块里"** —— 对。FAQ 例子展示:7 个 chunk 最终合并成 1 个(187 字 < target 256)。

---

## ❌ 说错或说反的部分

### 错误 1:"标题拼接起来入 title"

**实际**:章节标题拼起来进入的是 **`ContextHeader` 字段**,不是 `title`。

- `title` 在代码里指 **文档标题**(`knowledge.Title`),存在 knowledges 表,不在 chunks 表。
- `ContextHeader` 是 chunk 的字段,持久化到 chunks 表(`gorm:"type:text"`),但不对前端(`json:"-"`)。

两个 title 要分清:
```
文档标题(knowledge.Title)  ←  整篇文档的标题,1 个
面包屑(ContextHeader)     ←  每个 chunk 自己的"我在哪一节",N 个
```

### 错误 2:"向量化使用 title+content"

**实际**:喂给 embedding 模型的是 **文档标题 + 面包屑 + 正文**,三层拼接。

代码链路:
```go
// knowledge_process.go:519
indexContent := buildKnowledgeIndexContent(knowledge, chunk.EmbeddingContent())
// chunk.EmbeddingContent() = ContextHeader + "\n\n" + Content  (面包屑+正文)
// buildKnowledgeIndexContent 再在前面拼 knowledge.Title:
//   return knowledge.Title + "\n" + content
```

最终喂给 embedding 模型的输入:
```
公司报销制度            ← 文档标题(knowledge.Title)
# 公司报销制度          ← 面包屑(ContextHeader)
## 差旅报销
### 飞机票

经济舱凭票报销,公务舱...  ← 正文(Content)
```

**三段,不是两段。**

### 错误 3:"二次切割会先找是否有下一级标题,有则优先根据下一级标题切"

**实际**:超大 section 二次切**直接调用 `SplitText`(Tier 3 递归切)**,不会"优先按下一级标题切"。SplitText 按保护模式 + `\n\n` + `\n` + 句号切,不认 Markdown 标题。

子 chunk 的面包屑怎么带子标题?靠 `sectionBreadcrumbs` 反查:
- 切之前,先扫一遍 section 内部所有子标题,记录 `[(rune偏移, 当时栈快照), ...]`
- SplitText 切完后,每个子 chunk 拿自己的 `Start` 位置,去这个列表里找"最后一个不超过我的栈快照"当面包屑

所以**子 chunk 的面包屑不是"二次切时按子标题切的产物",而是"二次切完按位置反查到的"**。子 chunk 的切分边界还是 Tier 3 的递归切,跟子标题位置无关。

### 错误 4:"没有下一级标题的话,将切割的内容使用同一个面包屑进行标注"

**实际**:无论有没有下一级标题,**每个子 chunk 都用自己起始位置对应的活跃标题当面包屑**(通过 `breadcrumbAtOffset` 反查),不是"同一个面包屑"。

只有当 section 内部完全没有子标题时,所有子 chunk 的反查结果都回落到 section 顶层的栈快照 —— 这时候才"看起来是同一个面包屑",但这是"反查的结果",不是"主动用同一个标注"。

### 错误 5:"合并直到合并后的章节小于限制/2 也就是 200"

**方向说反了**。实际是:**"只要当前累积长度 < target,就继续往里合并;一旦达到 target 就封口"**。

```go
target := chunkSize / 2        // ChunkSize=512 → target=256
if target < 200 { target = 200 }  // 防下限
// 循环里:
if curLen < target && curLen+nextLen <= chunkSize { 合并 }
```

- 合并的**继续条件**是 `curLen < target`(还没到 target 就继续合)
- 合并的**停止条件**是 `curLen >= target`(到 target 就停,封口,下一个 chunk 重新开始)
- 200 是**最小 target 下限**,不是"合并到 200 就停"

举例 ChunkSize=512:
- target = 256
- 累积到 200,还 <256,继续合
- 累积到 260,≥256,封口,下一个 chunk 重新开始
- 不会"合到 200 就停"

200 是防御性下限:如果 ChunkSize 配成 100,target 会算成 50,太小了合完还是碎,所以抬到 200。

### 错误 6:"按第一个标题数 >3 的标题等级"

**实际**:`>= 3`,不是 `> 3`。

```go
if p.MdHeadingCounts[level] >= 3 { return level }
```

3 个就够,不用 4 个。

### 顺带纠正一个理解:"最低层级"

"最低层级"在代码里指 **层级数字最小**(最靠根),不是视觉上的"最深"。

H1 是最低层级(数字最小、最靠根),H6 是最高层级。`DominantHeadingLevel` 从 level=1 往 level=6 扫,第一个 ≥3 次的就是。所以"最低层级" = "最粗的标题" = "最靠根的那层"。

---

## 📋 纠正后的完整描述

> 匹配完后,判断是否能按标题切(`DominantHeadingLevel()==0` 则降级 Tier 3)。可以的话,从 H1 往 H6 扫,找第一个出现 **≥3 次** 的层级当主导层级 primaryLevel。
>
> 按 `level ≤ primaryLevel` 的所有标题当切点(`findHeadingBoundaries`,跳过代码块),把文档切成一段段 section。
>
> 对每个 section:栈维护当前从根到当前位置的标题路径(最多 6 级),取栈快照当**面包屑**存到 chunk 的 `ContextHeader` 字段(不是 title!),`Content` 存纯正文。
>
> 向量化时,喂给 embedding 模型的输入 = `文档标题 + "\n" + 面包屑 + "\n\n" + 正文`(三层拼接,通过 `buildKnowledgeIndexContent(knowledge, chunk.EmbeddingContent())` 生成)。
>
> 如果 section 太大(> ChunkSize),调 `SplitText`(Tier 3 递归切)二次切。切之前先扫 section 内子标题记录 `(rune偏移, 栈快照)` 列表,切完后每个子 chunk 按自己的 `Start` 位置反查这个列表,拿"最后一个不超过我的栈快照"当自己的面包屑。所以**每个子 chunk 有自己的面包屑,不是共用**。
>
> 如果切完太碎(FAQ 类),`coalesceTinyChunks` 合并相邻小 chunk:**只要当前累积长度 < target(=ChunkSize/2,最小 200)就继续合,达到 target 封口**。合并后面包屑用两个 chunk 面包屑的"共享前缀"(`commonHeadingPrefix`,按行逐行对比到第一个不同行停)。
>
> 弊端:整章内容短的话,会一直满足 `curLen < target` 一直合并,最终整篇文档合成 1 个 chunk(粒度很粗)。这是接受的取舍 —— 保住 Tier 1 不降级 > 保住面包屑精确层级。

---

## 你应该带走的 6 个纠正

1. **章节标题入 `ContextHeader`,不是 `title`**。`title` 是文档标题,是另一个东西。
2. **向量化输入是三段**:`文档标题 + 面包屑 + 正文`,不是 `title + content` 两段。
3. **二次切调 `SplitText`(Tier 3 递归),不按子标题切**。子 chunk 面包屑是按 `Start` 位置反查 `sectionBreadcrumbs` 得到的。
4. **每个子 chunk 有自己的面包屑**(按位置反查),不是"同一个面包屑"。
5. **合并的停止条件是"达到 target 封口"**,不是"合并到 < 200 停"。200 是 target 的最小下限。
6. **`>=3` 不是 `>3`**。3 个标题就够主导层级。