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
17. `notes/17-rune偏移是什么意思.md` —— rune vs 字节 vs 行号
18. `notes/18-全流程纠错与三层数字差异.md` —— 全流程纠错 + 256/128/无差异
19. `notes/19-保护机制7500硬切详解.md` —— 7500 硬切(切成什么、是不是单独 chunk)
20. `notes/20-父子分块SplitTextParentChild详解.md` —— 父子分块(跟 Tier1-3 区别、为什么要两层、全流程 8 步、切点与大小分离)
21. `notes/21-父子分块优化分析与检索双路径.md` —— 父块强塞噪声诊断、自动/Agent 双检索路径、为什么不推荐 LLM 摘要、3 个真正优化方向

### 阶段 3:向量化与入库

22. `notes/22-向量化与入库全流程.md` —— processChunks 12 步(幂等清理→建DB Chunk→写chunks表→3层拼接IndexInfo→BatchIndex批量embed→异步后处理→三态状态机)、隐藏的二次 BatchIndex(问题生成)
23. `notes/23-知识图谱后处理与Neo4j存储原理.md` —— 图谱抽实体+关系存Neo4j、图库vs关系库、定长记录偏移寻址+关系双向链表、index-free adjacency、apoc.merge去重+union累加chunks+Label按KB隔离
24. `notes/24-Wiki后处理与存储及并发控制.md` —— wiki存PG不进向量库、Map-Reduce流水线LLM生成内容、slug/SourceRefs/链接、图谱vs wiki并发根本区别(累加vs重写)、三重防护(claiming+per-slug悲观锁+乐观锁)、为什么悲观+乐观互补、冲突不能跳过LLM

## 当前进度

✅ **已完成**:
- 阶段 1:架构与文档解析(笔记 01-02)
- 阶段 2:切块(笔记 03-21,完整覆盖 Tier 1/2/3 + 体检 + overlap + 保护区间 + rune + 7500 硬切 + 父子分块 + 父子分块优化分析)—— **阶段 2 收尾,两条切分路线(普通 SplitText / 父子分块)都讲透,含优化方向**
- 阶段 3:向量化与入库(笔记 22)—— **阶段 3 闭环,入库 12 步全流程 + 二次 BatchIndex + 三态状态机**
- 后处理:知识图谱(笔记 23,Neo4j 存储)+ Wiki(笔记 24,存储与并发控制)—— **入库后 4 大异步后处理讲透 2 个(图谱/wiki),含图谱vs wiki 并发模型根本区别**

🚧 **待落盘**:
- (暂无)

⏳ **还没进行(按后续讲解顺序)**:

1. **检索与后处理 enrichment** —— 查到 chunk 后怎么加工给 LLM(笔记 21 已摸过 resolveParentChunks,顺势讲完整检索链路)
2. **端到端走一个具体场景** —— 拿真实文档从体检到检索全跑一遍
3. 后续:知识图谱 / 摘要生成 / Wiki / FAQ / 多模态(图片 OCR/Caption)/ 检索后处理 / 答案生成 等

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

**关键**:两条路线都用 SplitText,区别只是切一次还是切两次。父子分块不是独立算法,是"在普通切分外面套一层"。**切点不由 size 定,封口由 size 定,选刀受 size 间接影响**——ChunkSize 只管装箱封口阈值,下刀靠 Tier1/2/3 的 separators/体检/保护区间/7500。详见笔记 20。

## 我们怎么学

- 每一步都从外到内,先看懂"这是啥",再下钻"怎么实现的"
- 每个笔记都用大白话,听不懂就让我重写
- 原始对话存 `sessions/`,整理后的笔记存 `notes/`
- 不直接改源码,只读和记;有歧义先查代码再下结论,不瞎点头