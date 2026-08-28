# 14 · Tier 3 递归切与合并机制 + 碎处理详解

> 用户问:Tier 3 按句号递归切,一句话才多少字,会不会一句一个 chunk?Tier 2 怎么保证不切太碎?碎了的处理?Tier 2 碎兜不住,Tier 3 也会碎吧?

---

## 核心破除误解:递归切不是"一句一 chunk"

**担心的**:按句号切,那不就一句话一个 chunk?

**实际不是**。递归切分(splitBySeparators)只是**"把文本切成 unit"**,**不是"直接把 unit 当 chunk"**。

两步:
1. **splitBySeparators**:递归切出很多小 unit(句子级)
2. **mergeUnits**:把小 unit **贪心合并**到 ~512 字一个 chunk

**关键**:句子只是"unit",不是"chunk"。mergeUnits 会把很多句子拼成一个 chunk,直到快超 512 才封口。

### 例子

```
原文:30 个句子,每句 20 字,共 600 字,没有段落分隔

splitBySeparators 切成 60 个 unit(30 句 + 30 个句号):
  [句1, 。, 句2, 。, ..., 句30, 。]

mergeUnits 贪心合并:
  装 u0 句一(20)+ u1 。(1)+ u2 句二(20)+ ...
  一直装到累积快超 512 字 → 封口
  → chunk 0 ≈ 25 句(500 字)
  → chunk 1 = 剩下 5 句 + overlap(100 字)
```

**30 个句子合并成 2 个 chunk**,不是 30 个。

---

## Tier 3 怎么避免切得太碎 —— 3 道机制

### 机制 1:mergeUnits 贪心合并(主防线)

`splitter.go:470` 的封口条件:
```go
if curLen+uLen+headersLen > chunkSize && len(current) > 0 {
    // flush + 开新 chunk
}
```

unit **一直往盒子里装**,累积到快超 512 才封口。即使 unit 是一句话(20 字),也会**等装满 512 字才封口**,不会"一句一封"。

**结果**:chunk 大小 ≈ ChunkSize(512),不会碎成一句一个。

### 机制 2:overlap 平滑(防止碎 chunk 语义断裂)

封口后,下个 chunk 的开头是上个 chunk 的尾部(默认 80 字,3 级优先级对齐到段落/行/句末)。

即使切分边界不理想,相邻 chunk 共享 80 字,**检索时相邻内容都能命中**,缓解"碎"的体感。

### 机制 3:校验器兜底(ValidateChunks)

Tier 3 切完跑校验器 4 判据,失败也返回(`strategy.go`:`if tier == TierLegacy && i == len(chain)-1 { lastOut = out }`)。

**为什么 Tier 3 失败也返回**:Tier 3 已经是最底层,没地方再降了。返回碎 chunk 也比返回空强 —— 至少检索能命中点什么。而且 overlap 已经做了平滑,实际可用性不会太差。

---

## Tier 2 怎么避免切得太碎 —— 3 道防线

### 防线 1:minChunkSize 防过早封口(主防线)

```go
minChunkSize := cfg.ChunkSize / 4
if minChunkSize < 50 { minChunkSize = 50 }
```

封口双条件:
1. `accumulated > ChunkSize`(累积超 512)
2. `curEnd-chunkStart >= minChunkSize=128`(已经累积够 128 字)

**关键**:即使下一个 block 会让累积超 512,**但当前累积还不到 128**,就**不封口,继续往里塞**。避免"刚到 minChunkSize 就封口,下一个 block 又很大"导致的碎 chunk。

### 防线 2:appendChunk 丢弃纯空白

```go
if strings.TrimSpace(raw) == "" {
    return out  // 纯空白不进结果
}
```

边界聚集会产生"两个边界之间全是空行"的片段,直接丢弃。

### 防线 3:校验失败降级到 Tier 3

Tier 2 切完跑 `ValidateChunks`,失败 → 降级 Tier 3。

**注意**:Tier 3 没合并机制,**Tier 2 降级到 Tier 3 不一定更好**!Tier 3 是递归切 + overlap 平滑,可能切得比 Tier 2 更细。所以校验器在"碎"和"降级"之间是个权衡。

---

## "Tier 2 碎兜不住,Tier 3 也会碎" 的判断是对的

**为什么 T2 碎了降级到 T3 可能更碎**:

| 层 | 切分粒度 | 最细能切到 |
|----|---------|-----------|
| Tier 2 | 结构信号(第一章/1.1/Page/---) | 结构信号之间(可能很大,委托 Tier 3) |
| Tier 3 | 递归分隔符(`\n\n` → `\n` → `。`) | **句子级**(几十字) |

Tier 3 的最细粒度是句子。如果文档结构信号密集,Tier 2 切得碎;降级到 Tier 3,如果文档段落也短,Tier 3 也会切得碎。

### 但实际情况有几个缓解

**缓解 1:T3 的 mergeUnits 会合并**
- 句子是 unit,mergeUnits 会把句子合并到 512 字一个 chunk

**缓解 2:T3 的 overlap 平滑**
- 相邻 chunk 共享 80 字,即使切得碎,检索时相邻内容都能命中

**缓解 3:T3 失败也返回,有总比无强**
- 真要碎到不能用,校验器判失败 —— 但 Tier 3 是兜底,失败也返回

**缓解 4:父子分块(SplitTextParentChild)—— 系统设计层面的兜底**
- parentSize=4096(大块,提供上下文)
- childSize=384(小块,精确检索)
- 检索时:用 child 精确命中 → 拉出 parent 给 LLM(上下文完整)
- 即使 child 切得碎,parent 仍然是大块,语义完整

**这是 WeKnora 的真实做法**:默认就走父子分块,child 切碎不怕,parent 兜住上下文。

---

## 3 层碎处理对比总表

| 层 | 主防线 | 辅助防线 | 校验失败 |
|----|--------|---------|---------|
| Tier 1 | coalesceTinyChunks **合并** | sectionBreadcrumbs 反查 | 降级 Tier 2 |
| Tier 2 | minChunkSize **防过早封口** | appendChunk 丢空白 | 降级 Tier 3 |
| Tier 3 | mergeUnits **贪心合并** | overlap 平滑 | **不降级,返回结果** |

---

## 你应该带走的 4 件事

1. **Tier 3 按句号切不会一句一 chunk**:递归切只是切出 unit,mergeUnits 会把 unit 合并到 512 字一个 chunk。句子是粒度,不是 chunk 大小。

2. **Tier 2 防碎 3 道防线**:minChunkSize(防过早封口)+ appendChunk(丢空白)+ ValidateChunks(降级 T3)。

3. **Tier 2 碎降级 T3 可能更碎**(T3 最细能切到句子),但有 3 个缓解:T3 mergeUnits 合并、T3 overlap 平滑、父子分块 parent 兜上下文。

4. **核心设计哲学**:Tier 3 失败也返回,有总比无强。真要碎到不能用,靠父子分块的 parent 兜住语义完整性。