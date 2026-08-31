# WeKnora-learn

我跟 AI 一起把 `D:/Project/WeKnora`(腾讯开源的 WeKnora 知识管理框架)从头到尾吃透的学习笔记。

## 目录结构

```
WeKnora-learn/
├── README.md            # 本文件,说明进度和怎么用这套笔记
├── notes/               # 按层级整理的学习笔记
└── sessions/            # 每次对话的原始记录(防止 AI 失忆)
```

## 阅读顺序

### 阶段 1:架构与文档解析

1. `notes/01-最外层架构.md` —— 用大白话讲项目是干嘛的、有哪些大模块、之间怎么配合
2. `notes/02-docreader文档解析.md` —— 文档怎么被转成 Markdown

### 阶段 2:切块(切分策略)

3. `notes/03-Go后端切块.md` —— Go 后端切块入口
4. `notes/04-Tier1和Tier2的正则切法.md` —— Tier 1 / Tier 2 正则总览
5. `notes/05-Tier1大白话详解.md` —— Tier 1 按标题切详解
6. `notes/06-coalesceTinyChunks详解.md` —— Tier 1 小 chunk 合并机制
7. `notes/07-Tier1用户描述纠错.md` —— 用户讲 Tier 1 时的 7 个错
8. `notes/08-Tier2启发式切详解.md` —— Tier 2 启发式 8 正则切
9. `notes/09-Tier2碎chunk处理.md` —— Tier 2 防 3 道防线
10. `notes/10-Tier3递归切详解.md` —— Tier 3 递归切(兜底)
11. `notes/11-Tier3四问递归切代码逐行+大小标准+overlap机制.md` —— Tier 3 四问 + overlap
12. `notes/12-Tier1到Tier3全过程串联.md` —— Tier 1 → Tier 3 串起来
13. `notes/13-体检ProfileDocument详解.md` —— 体检机制(11 个信号)
14. `notes/14-Tier3递归切与合并机制详解.md` —— mergeUnits 合并(不会一句一 chunk)
15. `notes/15-两阶段切分思路与overlap机制.md` —— 两阶段切分 + overlap 80 字 3 级优先级
16. `notes/16-保护区间机制详解.md` —— 7 正则保护区间 + span 字节偏移
17. `notes/17-全流程纠错与三层数字差异.md` —— 全流程纠错 + 256/128/无差异
18. `notes/18-保护机制7500硬切详解.md` —— 7500 硬切(切成什么、是不是单独 chunk)
19. `notes/19-父子分块SplitTextParentChild详解.md` —— 父子分块(跟 Tier1-3 区别、为什么要两层、全流程 8 步、切点与大小分离)
20. `notes/20-父子分块优化分析与检索双路径.md` —— 父块强塞噪声诊断、自动/Agent 双检索路径、为什么不推荐 LLM 摘要、3 个真正优化方向

### 阶段 3:向量化与入库

21. `notes/21-向量化与入库全流程.md` —— processChunks 12 步(幂等清理→建DB Chunk→写chunks表→3层拼接IndexInfo→BatchIndex批量embed→异步后处理→三态状态机)、隐藏的二次 BatchIndex(问题生成)
22. `notes/22-知识图谱后处理与Neo4j存储原理.md` —— 图谱抽实体+关系存Neo4j、图库vs关系库、定长记录偏移寻址+关系双向链表、index-free adjacency、apoc.merge去重+union累加chunks+Label按KB隔离
23. `notes/23-Wiki后处理与存储及并发控制.md` —— wiki存PG不进向量库、Map-Reduce流水线LLM生成内容、slug/SourceRefs/链接、图谱vs wiki并发根本区别(累加vs重写)、三重防护(claiming+per-slug悲观锁+乐观锁)、为什么悲观+乐观互补、冲突不能跳过LLM

### 阶段 4:检索与对话

24. `notes/24-检索全链路.md` —— 9阶段pipeline(query改写→双路并行检索→RRF融合→rerank两阶段粗排精排→merge8步增厚→filter_topk→喂LLM)、cross-encoder rerank、复合分+MMR、无token budget裁剪
25. `notes/25-对话历史记忆召回改写意图与数据库选型.md` —— 对话历史(取最近N轮+三层压缩+丢弃检索结果原文+Agent KB结果涂黑)、记忆召回(跨会话5kind+常驻无条件+情境RRF混合+显式+蒸馏写入)、query改写意图(一次LLM三任务+9意图路由+@mention)、数据库选型(pgvector+ParadeDB同库+Redis基础设施+Lite)
26. `notes/26-载入历史对话细讲与优化点.md` —— KnowledgeQA路径载入历史5步(取20行→request_id配对→丢弃RenderedContent+剥think+补图片附件→丢不完整轮→截5轮)+10机制+4优化点(最该做:KnowledgeQA复用Agent压缩闸)
27. `notes/27-Agent路径四道上下文管理闸.md` —— Agent路径每次LLM调用前跑manageContextWindow 4道闸:①trimCurrentTurnToolResults当前轮工具结果按预算裁(占位符+倒序补全+头尾预览1:3+配对不拆,3个盲区:单工具无限制/渐进式塞满/头尾预览丢中间)②redactHistoryKBResults历史轮KB工具结果涂黑(默认开,两层处理:落盘CompactToolOutputForHistory+装配redact,8个KB工具,占位符精确语义,跟KnowledgeQA丢RenderedContent对比)③Consolidator旧历史LLM摘要成system消息(token>0.5触发,findKeepBoundary分组不拆,rawArchive兜底,prompt翻译)④CompressContext硬砍最旧group(token>0.8兜底,groupToolMessages分组,worked example)。4道全视图层不动DB原版。★stable prefix机制(prompt_cache.go):PromptPrefixFingerprint只哈希前导system+tools schema,Agent每round贴指纹观测cache, wiki用BuildPromptCacheKey+awaitWikiPromptWarmup主动协调。★更正:Agent用了prompt cache不是不用,但只覆盖stable prefix不覆盖对话历史。★两层面拆分:触发压缩(层面A生存问题)和保cache(层面B效率问题)是两个不同层面,触发压缩时cache失效是必然物理后果不是隐患。★冻结决策真正价值是保住压缩成果不让历史膨胀,不是保cache命中(WeKnora靠DB-centered已保证字节稳定)。★claude-code对比:历史非KB工具结果claude-code压缩+冻结保小体积/WeKnora不压全量保留→压缩触发WeKnora早claude-code晚,历史段命中率两者差不多,差在总token体积。Q1/Q2/Q3收敛到Consolidator摘要落盘+复用+CompressContext优先保留summary(主价值省LLM调用,顺带cache命中率提升)。4道闸全讲透+stable prefix讲透+两层面拆分讲透

## 当前进度

✅ **已完成**:
- 阶段 1:架构与文档解析(笔记 01-02)
- 阶段 2:切块(笔记 03-20,完整覆盖 Tier 1/2/3 + 体检 + overlap + 保护区间 + 7500 硬切 + 父子分块 + 父子分块优化分析)—— **阶段 2 收尾,两条切分路线(普通 SplitText / 父子分块)都讲透,含优化方向**
- 阶段 3:向量化与入库(笔记 21)—— **阶段 3 闭环,入库 12 步全流程 + 二次 BatchIndex + 三态状态机**
- 后处理:知识图谱(笔记 22,Neo4j 存储)+ Wiki(笔记 23,存储与并发控制)—— **入库后 4 大异步后处理讲透 2 个(图谱/wiki),含图谱vs wiki 并发模型根本区别**
- 阶段 4:检索与对话(笔记 24-26)—— **RAG "读"侧闭环,检索 9 阶段全链路 + 对话历史/记忆召回/改写意图/数据库选型 + KnowledgeQA 载入历史细讲**
- Agent 路径上下文管理(笔记 27)—— **Agent 路径 4 道闸全讲透:闸①trim(含 3 盲区)+ 闸②redact(两层处理:落盘 CompactToolOutputForHistory + 装配 redact,8 个 KB 工具,占位符精确语义,跟 KnowledgeQA 丢 RenderedContent 对比)+ 闸③Consolidator(摘要 prompt 翻译)+ 闸④Compress(worked example)。stable prefix 机制讲透(prompt_cache.go 基础设施 + 3 个调用点 + wiki warmup 主动协调)。两层面拆分讲透(触发压缩是生存问题/保cache是效率问题,不该混在一起)。claude-code 对比讲透(冻结决策真正价值是保压缩成果不是保cache命中,WeKnora压缩触发早claude-code晚)。Q1/Q2/Q3 收敛到 Consolidator 摘要落盘(主价值省 LLM 调用)**

🚧 **待落盘**:
- 评估笔记 26 优化点②(KnowledgeQA 复用 Agent 压缩闸)看完 4 道闸 + cache 基础设施 + 两层面拆分后是否还成立
- 评估笔记 27 第九节收敛的优化方向(Consolidator 摘要落盘,主价值省 LLM 调用)要不要深挖成具体方案

⏳ **还没进行(按后续讲解顺序)**:

1. **Agent 路径其他点** —— ReAct 循环 / 工具调度 / finalize / approval 等(笔记 27 只讲了上下文管理)
2. **端到端走一个具体场景** —— 拿真实文档从体检到检索全跑一遍
3. 剩余后处理:摘要生成 / FAQ / 多模态(图片 OCR/Caption)/ 答案生成
4. 用户消化笔记 24-27 后的针对性问题

## 切分阶段两条路线(阶段 2 总结)

```
切分阶段(由 EnableParentChild 开关二选一):

  路线 A(EnableParentChild=false):普通分块
    └─ SplitText(text, cfg) → 内部走 Tier 1/2/3
    → 产出一套 chunk(都进向量库)

  路线 B(EnableParentChild=true):父子分块
    └─ SplitTextParentChild(text, parentCfg, childCfg)
         ├─ SplitText(text, parentCfg)        // 切 parent(4096)
         └─ 对每个 parent:
              └─ SplitText(parent, childCfg)  // 切 child(384)
    → 产出 parent(不进向量库)+ child(进向量库)
```

**关键**:两条路线都用 SplitText,区别只是切一次还是切两次。父子分块不是独立算法,是"在普通切分外面套一层"。**切点不由 size 定,封口由 size 定,选刀受 size 间接影响**——ChunkSize 只管装箱封口阈值,下刀靠 Tier1/2/3 的 separators/体检/保护区间/7500。详见笔记 19。

## 我们怎么学

- 每一步都从外到内,先看懂"这是啥",再下钻"怎么实现的"
- 每个笔记都用大白话,听不懂就让我重写
- 原始对话存 `sessions/`,整理后的笔记存 `notes/`
- 不直接改源码,只读和记;有歧义先查代码再下结论,不瞎点头