# 20 · 父子分块 SplitTextParentChild 详解(跟 Tier1-3 的区别、为什么要两层、全流程 8 步)

> 用户问:父子模块跟之前的 tier1-3 区别是什么?为什么要多分一层?怎么切的全流程是什么?传进去的大小到底影响什么?

---

## 一句话

> **父子分块不是新刀法,是「把 SplitText 套个壳,用不同 size 跑两次」。先切 4096 父块、再对每个父块切 384 子块。子块进向量库负责被搜到,父块不进向量库只存表负责提供上下文。检索命中子块后用 ParentChunkID 拉父块拼到前面喂 LLM。它解决的是「检索要小块 / 理解要大块」的矛盾。**

---

## 先纠正一个最易搞混的点:父子分块和 Tier1/2/3 不是同一层的东西

**Tier1/2/3 是「刀法」**(怎么切):Tier1 按标题切、Tier2 启发式切、Tier3 递归兜底,三者是**同一种工作(单层切分)的三个备选方案**,由体检 ProfileDocument 选一把用。

**父子分块是「层数」**(切几次):它不发明新刀法,**用的还是 SplitText(也就是那套 Tier 刀)**,只是套了个外壳——先切一刀大的,再对每个大的切一刀小的。

一句话区分:

> **Tier1/2/3 = 怎么切(刀法);父子分块 = 切几次(一层还是两层)。不是并列关系,父子分块内部用的就是 Tier1/2/3。**

---

## 跟之前 Tier1-3 的具体区别(4 条)

| 维度 | 普通 SplitText(Tier1-3) | 父子 SplitTextParentChild |
|------|--------------------------|---------------------------|
| **切几次** | 1 次 | 2 次(父一次、子一次) |
| **产出** | 一套 chunk,都进向量库 | 父块(不进向量库)+ 子块(进向量库)两套 |
| **目标大小** | 都 ~512 | 父 ~4096、子 ~384 |
| **刀法选择** | 整篇体检一次选刀 | 父层体检选一次,**每个父块子层再体检选一次**,刀法可以不同 |

第三条是重点:**普通模式所有 chunk 地位平等,都既要被检索又要喂给 LLM**;父子模式把职责拆开了——**子块负责「被搜到」,父块负责「提供上下文」**。

---

## 为什么要多分一层?—— 核心:检索与理解的矛盾

### 矛盾:检索要小块,理解要大块

RAG 有两个互相打架的需求:
- **向量检索**希望块**小**:embedding 把文本压成一个向量,块越小、语义越聚焦,检索越准。一块 4096 字混 5 个话题,搜「第 3 个话题」时向量被其他 4 个稀释,可能搜不到。所以 **384 字子块检索最准**。
- **LLM 理解**希望块**大**:LLM 拿到 384 字碎片看不到上下文,答得干巴巴。它想要的是「这 384 字所在的那整段」,也就是 **4096 字父块**。

**普通模式只有一个 size(512),只能两头妥协**——检索不算特别准,上下文也不算特别全。

### 父子模式怎么解

**各司其职**:
- **子块 384 字进向量库** → 检索时命中精准(语义聚焦)
- **父块 4096 字不进向量库**,只存 chunks 表 → 命中子块后,用 `ParentChunkID` 把它爹拉出来拼在前面

merge.go:317 那段干的事:

```go
// 检索命中的是子块 r
parent := parentMap[r.ParentChunkID]     // 按 ID 把父块拉出来
r.Content = JoinChunkContent(parent.Content, r.Content, "\n\n")  // 父+子拼一起喂给 LLM
r.ContentRewritten = true
```

**效果**:搜的时候精准命中 384 字小块,喂给 LLM 时带着 4096 字爹一起进。**检索的准 + 理解的全,两个都要。**

### 用人话打比方

普通模式 = 图书馆每张卡片写 500 字,查到一张给一张。

父子模式 = 小卡片写 384 字(方便快速翻到相关那张),卡片背面写「详见第 X 卷」;翻到相关卡片后,**顺着编号把那整卷 4096 字搬出来一起看**。检索快(翻小卡片),理解深(看整卷)。

---

## 一个关键误区纠正:占用更多 token?—— 错

❌ 「父子分块让 LLM 了解更多上下文,但占用更多 token」这个直觉是反的。

**「入库存储量」和「每次检索喂给 LLM 的 token」是两回事,不能混着算。**

### 喂给 LLM 的 token:有同父去重,可控

```
普通模式(8000字文档, top-3 命中):
  16个512块都进向量库 → 命中3个 → 喂 LLM = 3×512 ≈ 1500字(碎片,无上下文)

父子模式:
  2个父块(4096)+21个子块(384),向量库只有子块
  命中3个子块,但可能同属1个父块 → 去重后只拉1个父块
  → 喂 LLM ≈ 1×4096 ≈ 4096字(完整段落)
```

**3 个命中的子块如果同属一个父块,拉出来的是同一个父块,去重后只算一份 4096**,不会 3×4096 重复。父块内容已包含子块内容,拼接是「父块+这个子块」,不是叠加。去重代码在 merge.go:240(map 收集 ParentChunkID)。

### 真正的代价(不在 token,在这 3 处)

1. **存储和写入成本略增**:父+子两套都存 chunks 表,行数比单层多;子块要向量化,父块不向量化但也要存;写入多切一刀,CPU 多花点。
2. **单次检索上下文更大 → 单次喂 LLM token 确实更多**:但这是 feature 不是 bug,换来质量。有同父去重,可控,不爆炸。
3. **父块太大可能塞进无关内容**(真正的坑):4096 父块里有 5 个子话题,用户只问第 3 个,整个父块喂进去,另外 4 个是噪声。所以父块 size 不能无限大,4096 是权衡值。

### 修正后的正确表述

> 父子分块用「子块精准检索 + 父块扩上下文」的分工,让单次喂 LLM 的内容从「无上下文碎片」变成「完整段落」。代价不是 token 爆炸(有同父去重,可控),而是 ①存储/写入成本略增 ②父块可能带入相邻话题噪声。它拿「存储+可能噪声」换「检索精度+上下文完整度」,长文档问答场景是赚的。

**反直觉点**:RAG 成本大头是「每次问答喂 LLM 的上下文」,不是「库里存了多少块」。库里存 1 万块不影响每次问答成本,每次只取 top-k。父子分块让库里存的块变多了一点,但让每次问答命中更准、上下文更全——前者一次性写入成本,后者每次问答质量收益。

---

## 全流程 8 步(从开关到检索闭环)

```
用户上传文档
   │
   ▼
① 开关判断: EnableParentChild == true ?  (knowledge_process.go:3533)
   ├─ false → 走普通 Split (单层,前 19 篇讲的)
   └─ true  → 走父子分块 ↓
        ▼
② 造两套配置: DeriveParentChildConfigs  (strategy.go:220)
     parentCfg: ChunkSize=4096, Overlap=用户配的
     childCfg : ChunkSize=384,  Overlap=384/5≈76
        ▼
③ 第一刀切父: parents = Split(text, parentCfg)   (strategy.go:159)
     整篇 → ~4096 父块(走 Tier1/2/3 选刀)
        ▼
④ 第二刀切子: for 每个 parent:
                  subs = Split(parent.Content, childCfg)  (strategy.go:169)
                每个父块 → ~384 子块(重新体检选刀)
        ▼
⑤ 父块过滤: 只留「真正被切开」的父块  (splitter.go:873)
        ▼
⑥ 偏移换算 + 面包屑合并  (strategy.go:177-180)
        ▼
   ┌────┴────┐
   ▼         ▼
父块们     子块们(带ParentIndex)
   ▼         ▼
⑦ 入库(先父后子):
   父块: 写chunks表, ChunkType=ParentText, 不进向量库  (knowledge_process.go:376)
   子块: 写chunks表, ChunkType=Text, 进向量库          (knowledge_process.go:415)
   子块 ParentIndex → 父块真实DB ID → ParentChunkID     (knowledge_process.go:432)
        ▼
⑧ 检索闭环: 提问 → 向量库搜子块(384) → 命中 → 拉父块(4096) → 拼一起喂LLM
                                                          (merge.go:317)
```

### ① 开关:EnableParentChild

```go
// knowledge_process.go:3533
if eff.ChunkingConfig.EnableParentChild {
    // 父子分块
} else {
    splitChunks := chunker.Split(convertResult.MarkdownContent, chunkCfg)  // 普通
}
```

一个 bool 开关二选一,整个流程的岔路口。来自知识库 ChunkingConfig(建库时配,initialization.go:335)。

### ② 造两套配置:DeriveParentChildConfigs

```go
// strategy.go:220
parentSize = 4096  (没配用默认)
childSize  = 384   (没配用默认)

parent = { ChunkSize: 4096, Overlap: base.Overlap,         // overlap 沿用用户配的
           Separators, Strategy, Languages }
child  = { ChunkSize: 384,  Overlap: 384/5 ≈ 76,           // 子块 overlap 自动算
           Separators, Strategy, TokenLimit, Languages }
```

三个要点:
1. **父块 overlap 沿用用户配的**,子块 overlap 自动 = `childSize/5`(≈76)。子块要 overlap 防止切在句中劈两半。
2. **Strategy 两层都复制**——strategy.go:236 注释强调:Strategy 空 → 解析成 legacy tier → **永远不跑 heading 切分器** → 父子块静默丢标题对齐和面包屑。必须显式传。
3. **TokenLimit 只传子块,不传父块**:父块不进向量库、不喂 embedding,不受 embedding token 上限约束;子块要进向量库,必须卡 token。

### ③ 第一刀:切父块

```go
// strategy.go:159
parents = Split(text, parentCfg)   // 整篇 markdown, ChunkSize=4096
```

整篇用 4096 跑一次 Split。Split 内部 = resolveChainWithProfile 体检选刀 → 跑刀 → ValidateChunks 验收 → 不过关降级下一把。产出 ~4096 父块,每个带 Start/End(全文偏移)、ContextHeader、Seq。

### ④ 第二刀:切子块(对每个父块再切一次)

```go
// strategy.go:168
for _, parent := range parents {
    subs := Split(parent.Content, childCfg)   // ChunkSize=384
}
```

**传进去的是 parent.Content(父块正文),不是全文。** 所以 Split **对这段父块内容重新体检选刀**——strategy.go:130 注释说的「Re-profiling each parent」。

**为什么每个父块要重新体检**:父块切出后,内部结构可能跟全文不同。全文标题清晰(父层选 Tier1),某父块内部没子标题(子层降级 Tier3 递归)。**每层独立选刀,刀法可以不同。** 成本 O(sum(parent_size)) ≈ O(N),跟父层同阶,不额外炸。

### ⑤ 父块过滤:只留「真正被切开」的父块

```go
// splitter.go:873
parentIndex := -1
if len(subs) > 1 || (len(subs) == 1 && subs[0].Content != parent.Content) {
    parentIndex = len(newParents)
    newParents = append(newParents, parent)   // 留
}
```

- 切出 ≥2 子块 → 真切开 → 留
- 切出 1 子块但内容跟父块不一样(trim/overlap 改过) → 留
- 切出 1 子块,内容跟父块一模一样 → **没切开 → 丢父块**

为什么丢:父块就 3000 字 < 4096 且内部没更细边界,384 子块切下来一整块跟父块同。这种父块留着纯浪费——子块就是它本身,检索命中子块时拉个一模一样父块没意义。`parentIndex=-1` 标记「没爹」,将来检索单独用子块不拉父。**省空间优化,不是所有父块都进库。**

### ⑥ 偏移换算 + 面包屑合并

子块 Start/End 是**相对父块内容的偏移**(第④步传的是 parent.Content),要换算回**全文偏移**:

```go
// strategy.go:177-180
sub.Seq = childSeq            // 子块在全文全局唯一编号
sub.Start += parent.Start     // 子块偏移 + 父块起始 = 全文偏移
sub.End += parent.Start
sub.ContextHeader = mergeBreadcrumbs(parent.ContextHeader, sub.ContextHeader)
```

**`+= parent.Start` 而不是按 Content 长度算**——splitter.go:881 注释:用加法偏移不按 Content 长度,是为让「带前置 context header 的 chunk」位置追踪仍正确(Content 可能被 trim,长度跟偏移区间不完全对应,加法最稳)。

**面包屑合并 mergeBreadcrumbs**(strategy.go:250):
```
父面包屑: "第1章 > 1.1 安装"
子面包屑: "1.1 安装 > 第一步"   ← 子层重新检测标题,父块开头"1.1 安装"被重复检测
合并后  : "第1章 > 1.1 安装 > 第一步"   ← 去掉重复的"1.1 安装"
```
逻辑:父面包屑最后一行 == 子面包屑第一行(trim 后)→ 删子第一行 → 拼接。**防 embedding 上下文冗余。**

### ⑦ 入库:先父后子,ParentIndex 换真 ID

**阶段 A:先建父块(拿 DB ID)**

```go
// knowledge_process.go:376
parentDBChunks[i] = &types.Chunk{
    ID: uuid.New().String(),                 // ← 真实 DB ID
    ChunkType: types.ChunkTypeParentText,    // ★ 类型标记:父文本
    StartAt, EndAt, ChunkIndex...
}
// 父块之间串 prev/next 链 (392行)
```

父块 ChunkType = `ParentText`,检索时用来识别「这是父块」。

**阶段 B:再建子块,ParentIndex 换父块真 ID**

```go
// knowledge_process.go:415  子块
textChunk := &types.Chunk{
    ID: uuid.New().String(),
    ChunkType: types.ChunkTypeText,          // ★ 普通文本(子块)
    Content, ContextHeader, StartAt, EndAt...
}

// knowledge_process.go:432  ★关键:ParentIndex → ParentChunkID
if hasParentChild && chunkData.ParentIndex >= 0 && chunkData.ParentIndex < len(parentDBChunks) {
    textChunk.ParentChunkID = parentDBChunks[chunkData.ParentIndex].ID
}
```

**「ParentIndex → ParentChunkID」转换点**:
- 切分阶段:子块带 ParentIndex(数组下标 0/1/2...)
- 入库阶段:父块先写拿到真 ID,子块用 `parentDBChunks[ParentIndex].ID` 把下标换成真 ID

**必须先父后子**:子块要引用父块 ID,父块 ID 要先写库才生成。所以父块先全部 append 进 insertChunks(403-407行),子块在后。

**进不进向量库?看 ChunkType**:

```go
// knowledge_process.go:447
for _, chunk := range insertChunks {
    if chunk.ChunkType == types.ChunkTypeText {   // ★ 只收 Text,不收 ParentText
        textChunks = append(textChunks, chunk)
    }
}
```

只把 `ChunkType==Text` 子块收进 textChunks,后续向量化只对 textChunks 做。`ParentText` 父块写了 chunks 表但**不进 textChunks → 不进向量库**。这就是「父块进表不进向量库,子块进表又进向量库」的代码实现——**靠 ChunkType 区分**。

### ⑧ 检索闭环:子块命中 → 拉父块

```
用户 query
   ▼
向量库搜 → 命中若干子块(384字, 带ParentChunkID)
   ▼
merge.go:239  收集所有命中的 ParentChunkID, map 去重
   ▼
merge.go:255  ListChunksByID 批量拉父块(4096)
   ▼
merge.go:327  每个命中子块:
              parent = parentMap[r.ParentChunkID]
              r.Content = JoinChunkContent(parent.Content, r.Content)  // 父+子拼接
              r.ContentRewritten = true
   ▼
喂给 LLM = 4096父块上下文 + 384子块精准命中
```

**去重**:多个子块同属一个父块(merge.go:240 map 去重 ParentChunkID),只拉一次父块,不会 3×4096 重复。这就是「同父去重,token 可控」的代码出处。

**图片特例(merge.go:346)**:图片命中是三层——`image → text父 → parent_text爷爷`。因为图片的父块是 text 块不是 ParentText 块,要再往上找一层爷爷才是大上下文。普通文本两层够,图片要三层。

---

## 全流程 8 步小结表

| 步 | 干啥 | 代码位置 | 关键点 |
|---|---|---|---|
| ① | 开关判断 | knowledge_process.go:3533 | `EnableParentChild` bool 二选一 |
| ② | 造父子配置 | strategy.go:220 | 父4096/子384,子overlap=76,Strategy两层都传 |
| ③ | 第一刀切父 | strategy.go:159 | 整篇用4096切,体检选刀 |
| ④ | 第二刀切子 | strategy.go:169 | 每个父块用384再切,**重新体检选刀** |
| ⑤ | 父块过滤 | splitter.go:873 | 只留真正切开的父块,没切的丢 |
| ⑥ | 偏移+面包屑 | strategy.go:177 | 子块偏移`+=parent.Start`换全文偏移,面包屑去重合并 |
| ⑦ | 入库 | knowledge_process.go:376/415/432 | 先父后子,**ParentIndex→ParentChunkID**,父`ParentText`不进向量库 |
| ⑧ | 检索拉父 | merge.go:317 | 子块命中→按ParentChunkID拉父→拼一起喂LLM,同父去重 |

---

## 切点与大小分离(理解整个切块系统的总钥匙)

### 入参不是「大小」,是两个完整配置

```go
func SplitTextParentChild(text string, parentCfg, childCfg SplitterConfig) ParentChildResult
```

入参是两个 `SplitterConfig`,大小(ChunkSize)只是其中一个字段。SplitTextParentChild 拿到配置后:

```go
parents := SplitText(text, parentCfg)        // 配置原样丢给 SplitText
subs := SplitText(parent.Content, childCfg)  // 配置原样丢给 SplitText
```

**它是套壳——把两个配置分别转发给 SplitText,自己一行切分逻辑都没有。** 真正处理「怎么按大小切」的是 SplitText(和背后 Tier1/2/3)。

### 真实流程:先切点后装箱,不是先数后切

以 Tier3 递归切为例:

```
SplitText(text, cfg=ChunkSize:384):

  第1步: 找切点(跟 size 无关)
    用 separators=[\n\n, \n, 。] 一层层递归劈文本
    → 得到一串「单元 unit」(语义片段)
    这些 unit 按语义边界切,不是按大小切

  第2步: 装箱(size 在这里才登场)
    for 每个 unit:
      if curLen + unit.len > ChunkSize:   // ← 384 在这里用
         封口,开新chunk
      else:
         装进去, curLen += unit.len
    → 产出 ~384 chunk
```

**切点和大小是两个分离阶段**:
- 切点由 **separators 正则 + 体检 + 保护区间** 决定,跟 ChunkSize 无关
- 大小只在第2步**装箱封口**起作用:`curLen + unit.len > ChunkSize` 就封口

**不是「数前 N 个字符再找切点」,是「先按语义切成一堆碎块,再按 N 把碎块装进箱子,装到接近 N 就封口」。大小控制「箱子容量」,不控制「下刀位置」。**

### 「只是换 512 成 4096/384」——算法层面是,但有三条间接影响

算法路径是同一个 SplitText,ChunkSize 换值而已,**没有「大块专用算法、小块专用算法」**。但这个换法不是无脑替换,大小会通过三条间接路径改变结果:

**路径 1:装箱封口阈值变了 → chunk 数量变了**
```
同样一堆 unit:
  ChunkSize=512 → 装到 512 封口 → 少而大的 chunk
  ChunkSize=384 → 装到 384 封口 → 多而密的 chunk
  ChunkSize=4096 → 装到 4096 封口 → 几乎不封口,很多 unit 合一坨
```
切点(语义边界)没变,但「哪些 unit 被合并进同一个 chunk」变了。4096 把一整章揉进一个 chunk,384 把一段话切成几块。

**路径 2:体检选刀的结果可能变(size 反向影响选刀)**
```
Split 内部:
  for tier in [Tier1, Tier2, Tier3]:
    out = runTier(tier, text, cfg)
    v = ValidateChunks(out, totalChars, cfg.ChunkSize)  // ★ 按当前 size 验收
    if v.OK: return out
```
`ValidateChunks` 按 `cfg.ChunkSize` 验收。同批 Tier1 切出的 chunk:
- ChunkSize=4096 时「大小合适」→ 验收过 → 用 Tier1
- ChunkSize=384 时同样的块「太大」→ 验收不过 → 降级 Tier2/Tier3

**size 不改刀法实现,但通过验收反向影响「最终用哪把刀」。**

**路径 3:overlap 和保护区间硬切阈值没换(固定值不跟 size 变)**

| 参数 | 是否随 ChunkSize 变 | 说明 |
|---|---|---|
| 装箱封口阈值 | ✅ 变 | = ChunkSize 本身 |
| overlap | ⚠️ 部分 | 普通沿用用户配的;父子子块=childSize/5 |
| **7500 硬切** | ❌ **固定** | maxProtectedSize=7500 写死,不随 4096/384 变 |
| 保护区间正则 | ❌ 固定 | LaTeX/表格/代码那 7 个正则,跟 size 无关 |
| 体检信号 | ❌ 固定 | 11 个信号,跟 size 无关 |

**「只是换 512」在「装箱阈值」上成立,但在「7500硬切/保护区间/体检」上不成立——这些是 size 之外的固定机制。** 父子分块用 4096/384 时,7500 硬切该触发还是触发(保护内容超 7500 照样硬切),不会因为父块是 4096 就放宽到 8192。

### 结论(锁死)

> **ChunkSize = 装箱封口的容量阈值(只管箱子多大、何时封口),不管下刀位置。下刀靠 tier1/2/3 的切点机制(separators/体检/保护/7500),跟 size 无关。但 size 会通过验收标准,间接决定「最终用 tier1/2/3 里的哪一把」。**

**切点不由 size 定,封口由 size 定,选刀受 size 间接影响。** 这三层分清楚,整个切块系统彻底通。

---

## 父子分块「新增」的特有逻辑只有 4 处

父子分块复用了前 19 篇学的全部 Tier 机制——体检、保护区间、7500 硬切、overlap、rune 偏移,在父层和子层各跑一遍。**没有新刀法。**

唯一「父子特有」的逻辑就 4 处,都是「两层协作」才需要、单层切分用不上的:
1. **⑤父块过滤**:只留真正切开的父块,没切的丢(省空间)
2. **⑥偏移换算 + 面包屑合并**:子块偏移 `+=parent.Start` 换全文偏移;父面包屑最后一行==子面包屑第一行就去重拼接
3. **⑦ParentIndex→ParentChunkID**:父块先写库拿真 ID,子块用下标换成真 ID;靠 ChunkType 区分进不进向量库(ParentText 不进,Text 进)
4. **⑧检索拉父**:命中子块按 ParentChunkID 拉父块拼一起喂 LLM,同父去重;图片特例三层

---

## 你应该带走的 6 件事

1. **父子分块不是新刀法,是「SplitText 套壳跑两次」**:先切 4096 父块,再对每个父块切 384 子块。两次各自独立体检选刀,刀法可以不同。

2. **跟 Tier1/2/3 的区别是「层数」不是「刀法」**:Tier1/2/3 是怎么切(刀法),父子是切几次(一层两层)。父子内部用的就是 Tier1/2/3。

3. **子块进向量库负责被搜到,父块不进向量库只存表负责提供上下文**:检索命中子块后用 ParentChunkID 拉父块拼到前面喂 LLM。靠 ChunkType 区分(ParentText 不进向量库,Text 进)。

4. **为什么多分一层**:解决「检索要小块 / 理解要大块」的矛盾。普通 512 两头妥协,父子 384+4096 两头都要。代价是存储略增 + 父块可能带入相邻话题噪声,**不是 token 爆炸**(有同父去重,可控)。

5. **切点与大小分离是总钥匙**:ChunkSize 只管装箱封口阈值(箱子多大、何时封口),不管下刀位置。下刀靠 separators/体检/保护区间/7500。但 size 通过 ValidateChunks 验收,反向影响「最终用哪把刀」。**切点不由 size 定,封口由 size 定,选刀受 size 间接影响。**

6. **父子特有逻辑只有 4 处**:父块过滤、偏移换算+面包屑合并、ParentIndex→ID、检索拉父。其余全是复用前 19 篇的 Tier 机制。