# 23 · 知识图谱后处理与 Neo4j 存储原理(节点/关系/属性、定长偏移寻址、index-free adjacency)

> 用户问:图谱和 wiki 是什么?为什么要生成?存在哪?什么时候用?Neo4j 图库怎么存?存储原理是什么?
> 本篇讲图谱后处理 + Neo4j 存储。Wiki 单独讲(见后续笔记)。

---

## 一句话

> **知识图谱后处理 = 对每个 chunk 调 LLM 抽实体+关系,拼成一张关系网存进 Neo4j。Neo4j 是图数据库,跟关系库根本区别:关系库存"实体"、关系靠查时 JOIN;图库存"节点+边"、关系是写入时就存的一等公民。Neo4j 存储原理=节点/关系定长记录+偏移量即 ID(O(1)寻址)+关系双向链表(物理聚一起),查询成本只随节点连接数增长不随全库数据量增长(index-free adjacency),这是图遍历快的根本,也是它能答"关系型问题"的能力来源。**

---

## 一、图谱解决向量检索答不了的问题

### 向量检索的短板:答不了"关系型问题"

向量检索擅长"语义相似",但回答不了关系型问题:

- ❓ "RAG 框架包含哪些模块?" → 向量搜"RAG 包含模块"可能命中某段话,但**不保证全**
- ❓ "docreader 和切块器是什么关系?" → 向量搜这俩词可能搜到各自段落,但**关系本身没显式存**
- ❓ "WeKnora 依赖哪些外部库?" → 要从 N 个 chunk 里把所有"使用"关系拼出来

向量检索靠语义相似,**关系是隐式的、散在文本里**。图谱把这些关系**显式化**成节点+边,靠图遍历答关系型问题,不靠语义相似。

### 四个后处理各补一种短板

| 后处理 | 解决的问题 | 本质 |
|--------|-----------|------|
| 摘要 | "这文档讲了啥" | 文档级压缩 |
| 问题生成 | "原文陈述句搜不到问句" | 检索增强 |
| **知识图谱** | **"概念之间什么关系"** | **实体+关系网络** |
| **Wiki** | **"知识碎片散在 N 个 chunk,怎么结构化浏览"** | **结构化知识页面** |

向量检索是"搜片段",这四个是"补全貌/补关系/补浏览/补问句"——各自补一种向量检索做不到的能力。

---

## 二、图谱后处理怎么抽:每个 chunk 调一次 LLM

`ExtractChunk`(extract.go:230)做的事:

```go
// 1. 从 chunks 表读回一个 chunk(不回源文档)
chunk := chunkRepo.GetChunkByID(ctx, p.TenantID, p.ChunkID)
// 2. 把这段文本喂给 LLM,让它抽出实体和关系
graph := extractor.Extract(ctx, chunk.Content)
// 3. 每个实体节点挂上"我来自哪个 chunk"
for _, node := range graph.Node {
    node.Chunks = []string{chunk.ID}   // ← 回指源 chunk
}
// 4. 写进图库(Neo4j)
graphEngine.AddGraph(ctx, namespace, []*types.GraphData{graph})
```

### 抽出来的数据结构(extract_graph.go:31)

```go
type GraphData struct {
    Text     string           // 原文
    Node     []*GraphNode     // 实体节点
    Relation []*GraphRelation // 关系
}
type GraphNode struct {
    Name       string   // 实体名
    Chunks     []string // 来自哪些 chunk
    Attributes []string // 属性
}
type GraphRelation struct {
    Node1 string // 实体1(起点)
    Node2 string // 实体2(终点)
    Type  string // 关系类型
}
```

### 案例:一个 chunk 抽出什么

chunk 内容:"RAG 框架由 docreader 负责文档解析,切块器按 Tier1/2/3 切分,最后由 WeKnora 写入向量库。docreader 用 unstructured 库把 PDF 转 Markdown。"

LLM 抽出:
```
节点:
  - "RAG 框架"     属性:[知识管理]      来自:chunk_07
  - "docreader"    属性:[文档解析模块]  来自:chunk_07
  - "切块器"       属性:[切块模块]      来自:chunk_07
  - "WeKnora"     属性:[腾讯开源框架]  来自:chunk_07
  - "unstructured" 属性:[Python库]      来自:chunk_07

关系(有向):
  RAG框架 --包含--> docreader
  RAG框架 --包含--> 切块器
  RAG框架 --包含--> WeKnora
  docreader --使用--> unstructured
  WeKnora --写入--> 向量库
```

每个 chunk 抽一次,**所有 chunk 的实体关系拼起来就是完整关系网**。一个实体在多个 chunk 出现,节点会合并、`Chunks` 字段累加(见下文 union)。

### 前置门:NEO4J_ENABLE

```go
// extract.go:99-102
if strings.ToLower(os.Getenv("NEO4J_ENABLE")) != "true" {
    return false, nil  // 没配 Neo4j 就不入队
}
```

没开 Neo4j 直接跳过图谱抽取,不报错。

---

## 三、Neo4j 存储原理(核心)

### 1. 图数据库 vs 关系数据库:根本区别

关系数据库(PG/MySQL)存**表+行**,表达"关系"靠 JOIN:
```
要查"RAG 包含什么":entities 表找RAG → JOIN relations表 → JOIN entities表
关系是查的时候临时算出来的,不是存的
```

图数据库把**节点和关系都当一等公民存**:
```
(RAG框架) --[包含]--> (docreader)
    \--[包含]--> (切块器)
每条线都是数据库里真实存在的一条记录,带方向带类型
```

查"RAG 包含什么"不是 JOIN,是**沿着 RAG 节点的"包含"边走一圈**——图遍历,天生为关系查询设计。

**一句话**:关系数据库存"实体",关系靠查时 JOIN;图库存"节点+边",关系靠边遍历。

### 2. 存储模型:节点+关系+属性三件套

| 概念 | 存什么 | 例子 |
|------|--------|------|
| **Node(节点)** | 一个实体 | `(RAG框架)`、`(docreader)` |
| **Relationship(关系/边)** | 节点连接,**有方向有类型** | `[包含]`、`[使用]` |
| **Property(属性)** | 挂节点或关系上的键值对 | name="docreader", kg="知识ID_07", chunks=[...] |

**关系是一等公民**:一条 `(docreader)-[使用]->(unstructured)` 在库里是独立记录,有自己的ID、类型、属性,不是临时算的。

### 3. 物理存储:定长记录+偏移量寻址(Neo4j快的根本)

Neo4j 数据分几个文件存,最核心两个:节点存储、关系存储。

**节点存储——定长记录,偏移量即ID:**
```
节点存储文件(固定每条15字节,简化示意):
偏移量0  | [节点0: 指向属性 + 第1条关系指针]  ← 节点ID=0
偏移量15 | [节点1: ...                    ]  ← 节点ID=1
偏移量30 | [节点2: ...                    ]  ← 节点ID=2
```
**每个节点固定15字节。节点ID=它在文件的偏移量。** 找节点#2直接跳偏移量30,O(1)定位,不用扫表。跟关系库B-tree索引查找完全不同——图库**直接算地址**。

**关系存储——定长,带双向链表:**
```
关系存储文件(固定每条34字节,简化):
[类型指针 | 起点节点 | 终点节点 | 上一条关系指针 | 下一条关系指针]
```
**一个节点的所有关系用双向链表串起来**。节点记录存"第一条关系指针",顺着链一路走拿到这个节点的所有边——不用扫全表。
```
docreader 节点
  └─第1条关系指针──→ [使用→unstructured] ──下一条──→ [被RAG包含←] ──下一条──→ nil
```
查"docreader 有哪些关系"=顺着链表走,**每个节点的关系物理上聚一起**。图遍历=走链表,O(度数)而非O(全表)。

**属性存储:** 属性短的(name/kg)内联进节点记录附近;属性长的(chunks数组)外存到属性文件,节点记指针指过去。所以 WeKnora 的 chunks 数组能 union 累加——它是挂节点上的属性。

### 4. 为什么这么存就快:index-free adjacency(无索引邻接)

- 关系库查"RAG包含什么":entities表找RAG(B-tree查 O(log n))→ JOIN relations表(又查 O(log n)),n大了慢
- Neo4j查"RAG包含什么":直接跳到RAG节点(偏移量 O(1))→ 顺着"包含"类型关系链表走一圈(O(度数)),**跟全库有多少节点无关**

**关系库查询成本随数据量增长,图库查询成本只随"这个节点的连接数"增长。** 1亿节点里查RAG的3个包含关系,跟100节点里查一样快——因为只走RAG这3条边。这是图库对关系查询的根本优势。

---

## 四、WeKnora 往 Neo4j 写了什么(真实 Cypher)

### 节点写入(repository.go:66-71)

```cypher
UNWIND $data AS row
CALL apoc.merge.node(row.labels, {name: row.name, kg: row.knowledge_id}, row.props, {}) YIELD node
SET node.chunks = apoc.coll.union(node.chunks, row.chunks)
```

三个关键点:

**1. `apoc.merge.node`——MERGE不是CREATE**
`merge`="有则更新,无则创建"。同一实体"RAG框架"在10个chunk出现,`merge`保证只创建1个节点,不变成10个重复。靠`{name, kg}`唯一性判断(同名+同知识库=同一节点)。

**2. 节点Label是知识库ID拼的**
```go
// repository.go:22
nodePrefix: "ENTITY"
// Labels() 把 namespace 转成 "ENTITY_<知识库ID>"
```
节点打Label `ENTITY_<KB_ID>`,**不同知识库节点隔离**。删某知识图谱按Label删,不误伤别的KB。

**3. `chunks`属性用union累加**
```cypher
SET node.chunks = apoc.coll.union(node.chunks, row.chunks)
```
"RAG框架"节点第1次来自chunk_07,第2次来自chunk_12——`union`合并去重存进chunks属性。**一个节点挂它出现的所有源chunk,这是"回指源文档"机制。**

### 关系写入(repository.go:87-92)

```cypher
UNWIND $data AS row
CALL apoc.merge.node(row.source_labels, {name: row.source, kg: ...}) YIELD node as source
CALL apoc.merge.node(row.target_labels, {name: row.target, kg: ...}) YIELD node as target
CALL apoc.merge.relationship(source, row.type, {}, row.attributes, target) YIELD rel
```
先merge出两个端点节点(保证存在),再在它们之间merge一条`row.type`类型关系。关系有**方向**(source→target)和**类型**(如"包含")。

### 真实例子落库后长什么样

```
节点:
  (ENTITY_kb123:RAG框架)        chunks:[chunk_07,chunk_12]  kg:知识ID  attributes:[知识管理]
  (ENTITY_kb123:docreader)     chunks:[chunk_07]            kg:知识ID
  (ENTITY_kb123:切块器)        chunks:[chunk_07]            kg:知识ID
  (ENTITY_kb123:WeKnora)       chunks:[chunk_07,chunk_03]   kg:知识ID
  (ENTITY_kb123:unstructured)  chunks:[chunk_07]            kg:知识ID

关系(有向):
  (RAG框架) -[包含]-> (docreader)
  (RAG框架) -[包含]-> (切块器)
  (RAG框架) -[包含]-> (WeKnora)
  (docreader) -[使用]-> (unstructured)
  (WeKnora) -[写入]-> (向量库)
```
注意`RAG框架`和`WeKnora`的chunks有多个——多chunk出现被union累加。`向量库`是节点(关系终点,merge关系时临时建的端点)。

### 删除:按Label精准删(DelGraph,repository.go:118)

```cypher
MATCH (n:ENTITY_kb123 {kg: $knowledge_id})-[r]-(m:ENTITY_kb123 {kg: $knowledge_id}) RETURN r  // 先删关系
MATCH (n:ENTITY_kb123 {kg: $knowledge_id}) RETURN n  // 再删节点
```
删某知识图谱:先删KB Label下所有关系,再删节点。靠Label+kg隔离,不删到别的知识库。这是入库前敢清旧图谱(幂等)的原因——删得干净且安全。

---

## 五、什么时候用图谱:query_knowledge_graph 工具

Agent调用`query_knowledge_graph`工具(query_knowledge_graph.go)。代码注释列三种用法(query_knowledge_graph.go:51-53):

1. **关系探索**:query_knowledge_graph → list_knowledge_chunks(先找实体关系,再拉详细内容)
2. **网络分析**:query_knowledge_graph → knowledge_search(先看关系网,再语义搜细节)
3. **主题研究**:knowledge_search → query_knowledge_graph(先语义搜到入口,再深挖实体关系)

**图谱和向量检索互补,不是替代**。向量搜"相似内容",图谱搜"实体关系"。Agent按问题类型选哪条路。

---

## 六、图谱 vs Wiki:别搞混(为下篇铺垫)

| 维度 | 知识图谱 | Wiki |
|------|---------|------|
| **形态** | 节点+边(图结构) | Markdown页面(文档结构) |
| **给谁用** | 程序遍历(图查询) | 人/模型阅读 |
| **存哪** | Neo4j | wiki_pages表(PG) |
| **回答什么** | "A和B什么关系""A关联什么" | "讲讲docreader这个东西" |
| **粒度** | 实体+关系(细) | 实体聚合页(粗) |
| **检索方式** | 图遍历(query_knowledge_graph) | slug读/正则搜(wiki_read_page) |
| **开关** | NEO4J_ENABLE | 默认随入库生成 |

**一句话区分**:图谱是"关系网"(存A-B之间的边,给图遍历用),Wiki是"实体档案"(把一个实体所有信息聚成一篇,给人读)。两者数据来源相同(都从chunks抽),但组织方式不同:图谱拆成点和边,wiki聚成页面。**图谱回答"连接",wiki回答"全貌"。**

---

## 你应该带走的 5 件事

1. **图谱后处理**:对每个chunk调LLM抽实体+关系,拼成关系网存Neo4j。每个chunk抽一次,同实体多chunk出现用merge+union合并去重。读已落库的chunks表不回源文档。前置门NEO4J_ENABLE。

2. **图库vs关系库根本区别**:关系库存"实体"、关系靠查时JOIN;图库存"节点+边"、关系是写入时存的一等公民。查关系图库靠边遍历不靠JOIN。

3. **Neo4j存储原理=定长记录+偏移量寻址+关系双向链表**:节点/关系固定大小记录,节点ID=文件偏移量(O(1)定位);一个节点所有关系用链表物理聚一起,顺着链走O(度数)。这就是index-free adjacency,查询成本只随连接数增长不随全库数据量增长——1亿节点查RAG的3个包含关系跟100节点一样快。

4. **WeKnora写Neo4j写的什么**:实体节点(带chunks属性回指源chunk)+有向关系(带类型)。用apoc.merge(有则更新无则创建,去重防重复节点)+union累加chunks(同实体多chunk来源合并)。节点按KB Label隔离,删除按Label精准删不误伤。

5. **为什么用Neo4j**:向量检索靠语义相似答不了"关系型问题"。图库把实体关系显式存成边,靠图遍历答关系型问题,查询成本只随连接数增长不随数据量增长——这是query_knowledge_graph工具的能力来源。图谱和向量检索互补不替代。