# Session 2026-08-29 · 父子分块 SplitTextParentChild

## 用户原始需求

1. 先给我讲讲父子模块,跟之前的 tier1-3 的区别是什么,为什么要多分一层父子模块
2. 这个父子切块的优点就是能够让 llm 了解更多的上下文,但是会占用更多的 token 对吧?(带误区)
3. ok,那我想再看一下这个父子模块是怎么切得,全流程是什么
4. 这个切块的方法的入参是切块的大小对吧?那这个方法是怎么对这个大小处理的?是直接数前 n 个字符,再根据这部分内容是否有切点去分割吗?还是只是将原本的 512 换成传进去的值 4096 和 384?
5. 懂了,这个传进去的值,实际只影响合并和装箱的大小对吧,实际切割的逻辑还是原本 tier1、2、3 的那一套
6. 先落盘

## 我做了什么

1. 读 splitter.go:840-894(SplitTextParentChild legacy 实现)+ strategy.go 全文(Split/SplitParentChild/DeriveParentChildConfigs/mergeBreadcrumbs)
2. 读 knowledge_process.go:3533 入口开关 + 370-449 入库逻辑 + 225 buildParentChildConfigs
3. 读 merge.go:230-360 检索闭环(child 命中拉父 + 图片三层爷爷链)
4. grep EnableParentChild/ParentChunkID/ParentIndex 全仓,确认入口和 ID 串联
5. 三轮讲解:
   - 第1轮:父子 vs Tier1-3 区别(刀法 vs 层数)+ 为什么要两层(检索要小/理解要大矛盾)
   - 第2轮:纠正 token 误区(入库存储 vs 检索喂 LLM 是两回事,有同父去重,代价是存储略增+父块噪声不是 token 爆炸)
   - 第3轮:全流程 8 步(开关→造配置→切父→切子→父过滤→偏移+面包屑→入库先父后子→检索拉父)
   - 第4轮:切点与大小分离(入参是 SplitterConfig 不是大小;先切点后装箱不是先数后切;只是换 512 但有3条间接影响:装箱阈值/选刀/固定7500)
6. 写笔记 notes/20-父子分块SplitTextParentChild详解.md
7. 写本 session 记录

## 关键发现(给 AI 自己看,防失忆)

- **SplitTextParentChild 是套壳,自己 0 行切分逻辑**:入参不是大小,是两个 SplitterConfig,转发给 SplitText。大小只是 cfg 一个字段。
- **父子 vs Tier1-3 不是并列**:Tier1/2/3=刀法(怎么切),父子=层数(切几次)。父子内部用的就是 Tier1/2/3,没有新刀法。
- **职责分工**:子块384进向量库负责被搜到,父块4096不进向量库只存表负责提供上下文。检索命中子块→ParentChunkID拉父→拼接喂LLM。
- **进不进向量库靠 ChunkType**:ParentText 不进,Text 进(knowledge_process.go:447 只收 Text 进 textChunks 向量化)。
- **token 误区纠正**:「占用更多 token」是反的。入库存储 vs 每次检索喂 LLM 是两回事。同父去重(map 收集 ParentChunkID,merge.go:240),多个子块同父只拉一次4096,不爆炸。真代价:存储/写入略增 + 父块可能带入相邻话题噪声。
- **切点与大小分离(总钥匙)**:ChunkSize 只管装箱封口阈值(curLen+unitLen>ChunkSize),不管下刀位置。下刀靠 separators/体检/保护区间/7500。但 size 通过 ValidateChunks 验收反向影响选刀。**切点不由 size 定,封口由 size 定,选刀受 size 间接影响。**
- **不是先数N字符再找切点,是先按语义切unit再按N装箱封口**:顺序反了,切点在前大小在后。
- **大小3条间接影响**:①装箱阈值变→chunk数量变 ②验收标准变→选刀可能变 ③7500硬切/保护区间/体检是固定值不跟size变。
- **每层独立选刀**:第4步传 parent.Content 给 Split,对每个父块重新体检选刀(Re-profiling),成本 O(N)。全文用Tier1,某父块内部没子标题可降级Tier3。
- **父块过滤**:len(subs)>1 或(==1但内容不同)才留父块;==1且内容同=没切开→丢(省空间,parentIndex=-1)。
- **偏移换算 +=parent.Start 不按Content长度**:因为Content可能被trim,长度跟偏移区间不完全对应,加法最稳(splitter.go:881注释)。
- **面包屑合并去重**:父面包屑最后一行==子面包屑第一行(trim后)→删子第一行→拼接,防embedding冗余(strategy.go:250 mergeBreadcrumbs)。
- **ParentIndex→ParentChunkID 转换**:切分阶段子块带数组下标ParentIndex;入库阶段父块先写拿真uuid ID,子块用 parentDBChunks[ParentIndex].ID 换成真ID。必须先父后子(knowledge_process.go:376→432)。
- **图片特例三层**:image→text父→parent_text爷爷。图片父块是text不是ParentText,要再上找爷爷(merge.go:346)。普通文本两层够。

## 代码位置速查

- 入口开关:knowledge_process.go:3533
- 造配置:strategy.go:220 DeriveParentChildConfigs / knowledge_process.go:239 buildParentChildConfigs
- 第一刀切父:strategy.go:159 Split
- 第二刀切子:strategy.go:169
- 父过滤:splitter.go:873(legacy) / strategy.go:172(strategy版)
- 偏移+面包屑:strategy.go:177-180
- 入库父块:knowledge_process.go:376(ChunkType=ParentText)
- 入库子块:knowledge_process.go:415(ChunkType=Text)
- ParentIndex→ID:knowledge_process.go:432
- 向量化只收Text:knowledge_process.go:447
- 检索拉父:merge.go:317-345
- 同父去重:merge.go:240
- 图片爷爷链:merge.go:346-358
- legacy 版 SplitTextParentChild:splitter.go:860

## 下一步建议

阶段 2(切块)至此完整收尾:普通 SplitText(Tier1/2/3,笔记03-19)+ 父子分块(笔记20)两条路线都讲透。

接下来(README 已列):
1. 向量化与入库落盘(阶段3,讲过未落盘→笔记21)
2. 检索与后处理 enrichment
3. 端到端走一个具体场景
4. 后续:知识图谱/摘要/Wiki/FAQ/多模态/答案生成

等用户选。