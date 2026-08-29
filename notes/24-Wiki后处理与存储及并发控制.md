# 24 · Wiki 后处理与存储(Map-Reduce 流水线、不进向量库、三重并发控制、为什么悲观+乐观锁互补)

> 用户问:wiki 具体怎么存的?为什么用乐观锁不用悲观锁?为什么不能跳过 LLM 只改版本号?
> 本篇讲 wiki 存储 + wiki/graph 两套并发控制的根本区别。

---

## 一句话

> **wiki 存在 PG 的 wiki_pages 表(不进向量库),页面内容由 LLM 生成(不是 chunk 拼接),走 Map-Reduce 流水线:Map 每文档独立抽实体/概念/摘要+判断 chunk 引用,Reduce 按 slug 合并多文档贡献的 chunk 逐字内容喂 LLM 编辑成页。并发靠三重防护:row claiming(Map 防重复处理)+ per-slug 悲观锁(Reduce 串行化)+ 乐观锁(锁失效兜底)。为什么两套锁互补:悲观锁挡在 LLM 之前省算力(让后到的等着不白跑 LLM),乐观锁防 Redis 挂(纯 DB 不依赖外部)。**

---

## 一、wiki 存哪:PG wiki_pages 表,不进向量库

这是最反直觉的一点。**wiki 页面不向量化、不进向量库**,只存 PG 的 `wiki_pages` 表。

证据:
- wiki ingest 全程**没有任何 embedding 调用**(Map 调 LLM 抽实体/摘要/引用分类,Reduce 调 LLM 生成页面,没有 embed 步骤)
- `wiki_pages` 表**没有 embedding 列**(migrations/000037 建表 SQL 确认)
- wiki 搜索走 PG 全文检索 `to_tsvector` + GIN 索引,或正则 `~*`,**纯 SQL 不走向量**

**为什么不进向量库**:wiki 用途是"浏览/阅读式"(按 slug 直接读整页),不是"问答式"(有 query 才搜)。Agent 用 `wiki_read_page` 工具直接从 DB 按 slug 取页面,不需要语义相似度。它要的是"完整一篇",不是"相似一段"。

### 表结构核心字段(wiki_page.go:167)

| 字段 | 作用 |
|------|------|
| `Slug` | KB 内唯一标识,如 `entity/docreader` |
| `PageType` | summary/entity/concept/index/synthesis/comparison |
| `Content` | 完整 Markdown(LLM 生成) |
| `SourceRefs` | 文档级溯源(哪些文档贡献这页) |
| `ChunkRefs` | chunk 级溯源(具体哪些 chunk UUID) |
| `InLinks`/`OutLinks` | 反向/正向链接(页面关联) |
| `Version` | 乐观锁版本号 |

**对比图谱**:图谱存 Neo4j(图遍历用),wiki 存 PG wiki_pages 表(直接读用)。两者都不进向量库——向量库只存 chunks 和生成的问句。

---

## 二、页面内容由 LLM 生成,不是 chunk 拼接(最关键认知)

wiki 不是把相关 chunk 拼起来。是 Map-Reduce 流水线,最终内容由 LLM 当"wiki 编辑器"生成。

### Map 阶段(每文档独立并行)

每文档做 3 件事,都调 LLM:
1. **抽候选 slug**(Pass 0):LLM 从文档提取实体/概念,输出 `{entities:[],concepts:[]}`,每个带 name/slug/aliases/description
2. **chunk 引用分类**(Pass 1..N):LLM 判断每个候选 slug 被**哪些 chunk 实质性讨论**,产 `map[slug][]chunkID`
3. **摘要生成**:LLM 基于全文生成文档摘要,注入 `[[slug]]` wiki 链接

产出 `[]SlugUpdate`,每个携带:slug + 源文档ID + 贡献的 chunk UUID 列表 + 文档摘要。

### Reduce 阶段(按 slug 分组合并)

同一 slug(如 `entity/docreader`)可能被 5 文档的 Map 产出,Reduce 合并成一页:
1. `GetPageBySlug`:不存在建 draft,存在加载现有内容
2. `resolveCitedChunks`:从 DB 批量加载被引用 chunk 的**逐字内容**
3. 组装 `<new_information>` 块(实体名+描述+chunk逐字内容)
4. **调 LLM(WikiPageModifyUserPrompt)**:现有页面内容 + 新增 chunk 逐字内容 + 被删文档内容喂 LLM,生成更新后页面 Markdown。约束:基于源 chunk 逐字内容、不幻觉、不内联 chunk ID、首行 SUMMARY:
5. `CreatePage`/`UpdatePage`

**关键**:chunk 逐字内容是喂 LLM 的"证据",最终页面是 LLM 转写的 Markdown,不是 chunk 原文拼接。**这是 wiki 和图谱的根本区别**——图谱直接存抽出来的实体/关系(原始数据),wiki 让 LLM 把多文档 chunk 综合成可读页面(加工数据)。

---

## 三、slug 怎么来 + 页面怎么关联

### slug 生成

- **entity/concept slug**:**LLM 直接生成**,格式 `entity/<lowercase-hyphenated-name>`,非拉丁名用罗马化/拼音("腾讯"→`entity/tencent`)。prompt 要求**复用之前的 slug**(实体在之前提取过就用原 slug),保证跨文档更新 slug 稳定
- **summary slug**:代码生成 `summary/<slugify(knowledgeID)>`。故意用 UUID 不用文件名避免泄漏
- **index slug**:固定 `"index"`
- **去重**:`deduplicateExtractedBatch` 用 pg_trgm 三元组相似度找已有页面,再调 LLM 判断是否合并(新 slug 映射到已有 slug),防同一实体生成两 slug

### 页面关联:`[[slug]]` 链接 + InLinks/OutLinks

三种建立途径:
1. **LLM 内容里嵌 `[[slug]]`**:摘要/页面 prompt 指示 LLM 插入 wiki 链接
2. **Cross-link 注入**(纯文本替换,不调 LLM):扫描页面内容找其他 wiki 页面标题/别名出现位置,自动注入 `[[slug|title]]`
3. **死链清理**:OutLinks 目标不存在了,尝试 fuzzy resolve,修不了降级纯文本

解析:`CreatePage`/`UpdatePage` 时 `parseOutLinks(content)` 从 Markdown 解析 `[[slug]]` 填 OutLinks;`updateInLinks` 反向维护——A 的 OutLinks 每个目标 B,把 A slug 加到 B 的 InLinks。**双向链接自动维护。**

### SourceRefs + ChunkRefs 双层溯源

- **SourceRefs**(文档级):`"knowledgeID"` 或 `"knowledgeID|docTitle"`,记录这页由哪些源文档贡献
- **ChunkRefs**(chunk 级):具体源 chunk UUID 列表。summary 页面清空 ChunkRefs(文档级摘要不带 chunk 级引用)

**跟图谱节点 chunks 区别**:图谱节点 chunks 是"提到这实体的 chunk";wiki ChunkRefs 是**跨文档聚合**——一个 entity 页可能由 5 文档的 chunk 贡献,ChunkRefs 是所有 chunk UUID 并集。图谱是"实体→片段",wiki 是"主题→多文档片段聚合"。

---

## 四、并发控制:wiki vs 图谱两套机制的根本区别

这是本篇重点。wiki 和图谱并发控制完全不同,因为**写入模型**不同。

### 根因:写入模型决定并发策略

**图谱 = 节点累加模型**:每个 chunk 抽出的实体往同一节点"加东西"(加 chunks 引用)。多 chunk 都提"RAG框架",就是往同一节点的 chunks 属性加 chunk ID。**只增不减,纯累加。** 天然好并发——谁先谁后加都行,最后 union 去重,不覆盖彼此劳动。

**wiki = 页面重写模型**:每文档贡献要**改写整页 Markdown**。A 写"docreader 是解析模块",B 要写"docreader 是解析模块,依赖 unstructured"。若 A B 同时写,**后写覆盖先写**——B 没看到 A 的内容,把 A 写的部分丢了。**这就是 lost-update,纯累加解决不了。**

| 维度 | 图谱(merge+union) | wiki(三重锁) |
|------|-------------------|---------------|
| 写入模型 | 只增累加(加 chunks 引用) | 改写整页 |
| 并发风险 | 重复节点 | lost-update(覆盖) |
| 并发安全靠 | 数据库原子操作(merge去重+节点写锁串行化union) | 应用层串行化(per-slug锁)+DB兜底(乐观锁) |
| 要不要应用加锁 | 不要 | 要 |
| 冲突了怎么办 | 数据库自动合并 | 拒绝+重试 |

**加法天然并发友好(union 去重),替换天然并发危险(要锁)。** 图谱靠数据库原子合并就够,wiki 必须应用层串行化+多重兜底。

### 图谱:merge 去重 + union 累加(数据库原子操作)

**merge 去重**(repository.go:68):
```cypher
CALL apoc.merge.node(labels, {name, kg}, props, {}) YIELD node
```
`merge`="有则更新无则创建",按 `{name,kg}` 唯一判断。两 chunk 同时抽"RAG框架",merge 保证只建一个节点(`CREATE` 会建俩)。**Neo4j 内部加锁保证同名节点只建一个**,不用应用层锁。

**union 累加**(repository.go:69):
```cypher
SET node.chunks = apoc.coll.union(node.chunks, row.chunks)
```
`union(现有,新增)`=合并去重。防 chunks 覆盖靠 **Neo4j 节点写锁串行化**:B 事务要等 A 提交后才能拿节点锁,这时 B 读到的 `node.chunks` 已是 A 写完的,再 union 不丢。**全靠数据库层,应用层不加锁。**

### wiki:三重防护(本篇核心)

#### 机制 1:Row Claiming(Map 防重复,悲观锁-数据库行锁)

**解决**:Map 阶段两 batch 都拿文档 #5 会重复处理。`ClaimBatch` 用 `FOR UPDATE SKIP LOCKED` + `claimed_at` 原子标记一批文档"已认领":
- A 锁住文档[1,2,3]
- B 几乎同时执行,[1,2,3]被锁→SKIP LOCKED 跳过→拿到[4,5,6]
**两 batch 文档集不相交**,Map 不重复。Stale claim 90 分钟超时可重认领(>asynq timeout 60分钟)。异常退出 defer 释放 claimed rows。**这层防 Map 重复,不防 Reduce 覆盖。**

#### 机制 2:Per-slug Lock(Reduce 串行化,悲观锁-Redis分布式锁,核心)

**解决**:Reduce 是"读页面→LLM生成→写回",两 batch 同时改同一 `entity/docreader` 会 lost-update。`withSlugLock`(wiki_ingest.go:874)用 Redis SetNX 抢锁:
```go
key := "wiki:slug:{kbID}:{slug}"
ok := redisClient.SetNX(key, "1", 5min)  // 不存在才设置
if ok { break }  // 抢到
// 没抢到→轮询等2分钟
```
**SetNX=Set if Not eXists**,原子操作,只有一个成功。Reduce 在锁内执行:
```go
s.withSlugLock(ctx, kbID, slug, func() error {
    page := GetPageBySlug(slug)        // 读
    newContent := LLM生成(page, chunks) // 改
    UpdatePage(page, newContent)        // 写
    return nil
})
defer redisClient.Del(key)  // 写完释放
```
**时序**:A 抢锁→读(v=1)→LLM→写(v=2)→释放;B 等待→A释放→抢锁→读(v=2)→LLM→写(v=3)。**B 读到 A 写完的 v=2,不覆盖 A**。没抢到等2分钟超时就放弃,记录 unappliedSlugKIDs,requeue 重试不丢数据。**这层防 Reduce 覆盖,是 wiki 并发核心。**

#### 机制 3:乐观锁(锁失效兜底,纯 DB version 列)

**解决**:Redis 挂/锁过期,悲观锁失效怎么办。每页有 Version,写时带条件(repository/wiki_page.go:98):
```go
expectedVersion := page.Version  // 读出来是v=2
page.Version = expectedVersion + 1
db.Where("id = ? AND version = ?", page.ID, expectedVersion).Updates(...)
//                                         ↑ 只有当前还是v=2才更新
if result.RowsAffected == 0 {
    return ErrWikiPageConflict  // version不匹配→有人抢先写→拒绝
}
```
**场景**(锁失效):A B 都读到 v=2:A 写成功(v=3);B 写 WHERE version=2→0行受影响(库已v=3)→ErrWikiPageConflict→拒绝写+重试。**B 被拒,不覆盖 A。** 这层防锁失效后覆盖,**不依赖外部系统(不靠 Redis),纯 DB。**

**三重分工**:claiming 防 Map 重复 / per-slug 锁防 Reduce 覆盖(主力) / 乐观锁防锁失效(兜底)。claiming 不防 Reduce(不同 batch 可能 Map 出同 slug),per-slug 锁依赖 Redis,乐观锁纯 DB 是最后防线。

---

## 五、为什么悲观+乐观锁互补,不是二选一

用户问:为什么用乐观锁不用悲观锁?——**前提纠正**:per-slug 锁本质就是悲观锁(SetNX 抢锁才能改)。所以真正问题是"**既然已有悲观锁为什么还要乐观锁**"。

### 为什么需要乐观锁兜底——悲观锁会失效

悲观锁前提是"锁本身可靠"。但 per-slug 锁有3个失效场景:

**1. Redis 挂了→锁 fail-open(直接放行)**
```go
ok, rerr := redisClient.SetNX(...)
if rerr != nil {
    return true, fn()  // ← Redis错误时不加锁直接执行!(fail-open)
}
```
**为什么 fail-open 不 fail-close**:wiki ingest 是批量后台任务,Redis 抖动就 fail-close 会让几十文档处理卡住,体验"导入永不完成"。宁可冒并发风险也要让任务继续。但 fail-open 后两 batch 真同时 Reduce,per-slug 锁失效,**只有乐观锁能发现冲突**(version不匹配拒绝写)。

**2. 锁过期但 LLM 没跑完(TTL 超时)**:per-slug 锁 TTL=5分钟,但 Reduce 调 LLM 可能超5分钟。A 锁过期释放→B 抢到→A 的 LLM 返回写回→覆盖 B!乐观锁:B 写成功(v=3),A 写 WHERE version=2 不匹配→拒绝。

**3. 进程崩溃锁残留**:defer 释放锁前被 kill,锁残留到 TTL 过期。

### 为什么不全用乐观锁还要悲观锁——wiki 的 modify 是调 LLM,冲突重试太贵

乐观锁短板:**冲突时代价极高**。wiki 的 Reduce 中间"modify"是**调 LLM 生成内容**(几十秒可能上分钟)。
```
纯乐观锁场景(无悲观锁):
  A: 读(v=2)→调LLM(30秒)→写 WHERE version=2→成功(v=3)
  B: 读(v=2)→调LLM(30秒)→写 WHERE version=2→失败(已v=3)→重读(v=3)→再调LLM(又30秒)→再写...
```
**冲突=LLM 白跑30秒+重跑**。并发高时反复冲突,LLM 算力大量浪费。

**加悲观锁的价值**:per-slug 锁让 B **一开始就等着**,不跑那30秒 LLM,直到 A 写完。**悲观锁把"冲突重试的高代价"提前变成"等待的低代价"**:
```
有悲观锁:
  A: 抢锁→读(v=2)→LLM(30秒)→写(v=3)→释放
  B: 抢锁失败→等(30秒,不跑LLM)→抢到→读(v=3)→LLM(30秒)→写(v=4)
```
B 等30秒但 LLM 只跑一次不浪费。**悲观锁省 LLM 算力,乐观锁防 Redis 挂。**

### 分工总结

| | 悲观锁(per-slug) | 乐观锁(version) |
|---|---|---|
| 防什么 | 正常并发,让后到的等着 | 锁失效后的并发覆盖 |
| 依赖 | Redis(外部,可能挂) | DB version列(自带,可靠) |
| 正常代价 | 抢锁开销+等待 | 几乎0(多个WHERE条件) |
| 冲突代价 | 无冲突(已串行) | 重试一次 read-modify-write |
| 失效场景 | Redis挂/锁过期 | 无 |

**互补**:悲观锁正常情况主力(省LLM算力),乐观锁异常情况兜底(Redis挂时防脏)。**对昂贵操作用悲观锁避免重试浪费,同时加乐观锁兜住外部依赖失效——分布式系统经典模式,不二选一。**

---

## 六、为什么冲突时不能跳过 LLM 只改版本号

用户问:为什么不直接用乐观锁,冲突时不重新调 LLM,只修改版本号?——**两个关键误解**:

### 误解1:乐观锁不是"只改版本号"——版本号是内容指纹不是写入目标

写入目的(repository/wiki_page.go:99-119)是把**新内容**(page.Content)落库,version+1 只是顺带递增。**version 是"内容有没有被改过"的检测器,不是写入目标。** 写入目标永远是新内容。"只改版本号不调 LLM"=没有内容可写时的无意义操作,根本不存在。

### 误解2:乐观锁失败=内容已过时=必须重新生成,不能跳过

冲突时序:
```
1. B 读页面→content_v2, version=2
2. B 把 content_v2 + 新chunk喂LLM→生成 new_content_B("v2+B文档内容")
3. 同时 A 已写完: content_v2→content_v3("v2+A文档内容")
4. B 写 WHERE version=2→失败(库已v=3)
```
冲突时 B 手里 new_content_B 是基于**旧 content_v2** 生成的——"v2+B文档内容"。但库里已是 A 写的 content_v3="v2+A文档内容"。

**若 B 强行写 new_content_B(不管冲突)**:
```
库: content_v3="v2+A文档内容"
B写: new_content_B="v2+B文档内容"  ← 基于v2,没A的内容!
结果: A文档内容丢失!这就是 lost-update
```
**版本号冲突就是在告诉你:"你手里新内容过时了,基于它等于覆盖别人。"** 冲突时唯一正确做法:重新读最新(v3)→基于v3重新调LLM→再写。跳过 LLM 就没正确内容可写。

**"只改版本号不调LLM"=放弃生成新内容但递增版本=什么都没干却假装更新,还把别人未来冲突检测基准搞乱。无意义且有害。**

### 为什么 wiki 冲突只能重调 LLM(不能像图谱 union)

能不能"接受冲突但 A B 内容都留住"(像 Git merge)?**wiki 做不了**。A 写"B 的自由文本",B 写"基于v2的自由文本",这是两段 LLM 生成的 Markdown,**没有自动合并算法**——没法让程序判断哪部分留哪弃。要合并只能再调 LLM 综合两段,**绕一圈还是得调 LLM**。

对比图谱为什么没这问题:图谱是结构化数据(节点+边+chunks数组),合并=union 去重,A 加 chunk_12,B 加 chunk_03,union=`[chunk_07,chunk_12,chunk_03]`**能自动合并**。所以图谱不需乐观锁重试,靠 union 天然合并。**wiki 是非结构化文本没法 union,才被迫用乐观锁+重试。**

### 终极结论

**乐观锁保护的是"内容正确性"不是"版本号本身"**:
- 版本号=检测器:告诉你"新内容是不是基于过时数据"
- 新内容=目标物:LLM 生成要写进库的真正东西
- 跳过 LLM=没正确新内容可写=要么写废内容(覆盖别人)要么无操作(假更新)

**冲突时重调 LLM 不是浪费是必须**——冲突证明"你基于旧内容生成的东西已不适用,得基于最新内容重新生成才写对"。这是乐观锁+昂贵操作的固有代价,也是前面叠悲观锁尽量不走到这一步的原因。

---

## 七、图谱 vs Wiki 终极对比

| 维度 | 知识图谱 | Wiki |
|------|---------|------|
| 形态 | 节点+边 | Markdown 页面 |
| 内容来源 | LLM **抽取**(实体/关系原样存) | LLM **生成**(多文档 chunk 综合成篇) |
| 存哪 | Neo4j | PG `wiki_pages` 表 |
| 进向量库 | ❌ | ❌ |
| 回指源 | 节点 chunks 属性 | SourceRefs + ChunkRefs |
| 关联 | 边(有向有类型) | `[[slug]]` + InLinks/OutLinks |
| 粒度 | 实体+关系(细) | 主题聚合页(粗,跨文档) |
| 检索 | 图遍历(query_knowledge_graph) | slug读/正则搜(wiki_read_page) |
| 并发控制 | merge去重+union累加(数据库原子) | per-slug锁+claiming+乐观锁(应用串行+DB兜底) |

**一句话**:图谱是 LLM 抽取后**原样存结构化数据**(给程序遍历,能 union 合并所以并发简单),wiki 是 LLM 把多文档 chunk **综合加工成可读页面**(给人/模型读,非结构化文本没法合并所以并发复杂)。图谱答"连接",wiki 答"全貌"。

---

## 你应该带走的 6 件事

1. **wiki 存 PG wiki_pages 表不进向量库**:页面是"浏览/阅读式"(按 slug 直接读),不是"问答式"(有 query 才搜)。Agent 用 wiki_read_page 工具从 DB 按 slug 取,不走向量。向量库只存 chunks 和问句。

2. **wiki 内容是 LLM 生成不是 chunk 拼接**:Map-Reduce 流水线。Map 每文档调 LLM 抽实体/概念/摘要+判断 chunk 引用;Reduce 按 slug 合并多文档贡献的 chunk 逐字内容喂 LLM 编辑成页。chunk 是"证据",页面是 LLM 转写的 Markdown。

3. **图谱 vs wiki 并发根本区别=写入模型**:图谱是只增累加(加 chunks 引用,union 去重天然并发友好),wiki 是改写整页(lost-update,必须串行)。图谱靠数据库原子操作(merge+union),wiki 必须应用层串行(per-slug锁)+DB兜底(乐观锁)。

4. **wiki 三重并发防护**:row claiming(Map 防重复处理,数据库行锁)+ per-slug 悲观锁(Reduce 串行化,Redis SetNX,核心)+ 乐观锁(锁失效兜底,纯 DB version 列)。claiming 防 Map,per-slug 锁防 Reduce 覆盖,乐观锁防 Redis 挂。

5. **为什么悲观+乐观锁互补不二选一**:悲观锁挡在 LLM 之前让后到的等着不白跑 LLM(省算力,正常主力),乐观锁不依赖 Redis 防锁失效(兜底异常)。对昂贵操作用悲观锁避免重试浪费,加乐观锁兜外部依赖失效——分布式经典模式。

6. **冲突时不能跳过 LLM 只改版本号**:版本号是内容指纹不是写入目标(写的是新内容);冲突=新内容已过时=必须基于最新版重新生成(跳过 LLM 就没正确内容可写,写的就是会覆盖别人的废内容);wiki 文本没法自动合并(不像图谱能 union),所以冲突只能重调 LLM。**乐观锁保护的是内容正确性,重调 LLM 是必须不是浪费。**