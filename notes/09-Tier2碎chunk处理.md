# 09 · Tier 2 碎 chunk 怎么处理(3 道防线)

> 用户问:Tier 2 如果切得太小了,是怎么处理的?这篇按代码事实讲清楚 3 道防线。

---

## 防线 1:装箱时 `minChunkSize` 防过早封口(主防线)

`heuristic_splitter.go:68-70`:
```go
minChunkSize := cfg.ChunkSize / 4
if minChunkSize < 50 { minChunkSize = 50 }
```

`heuristic_splitter.go:93`:
```go
if accumulated > cfg.ChunkSize && curEnd-chunkStart >= minChunkSize {
    // 封口
}
```

**关键**:封口条件是**两个都满足**:
1. `accumulated > ChunkSize`(累积超预算)
2. `curEnd-chunkStart >= minChunkSize`(当前累积**已经**够长)

**意思**:如果累积还不到 `minChunkSize`,即使下一个 block 会让累积超 ChunkSize,**也不封口,继续往里塞**。这样避免切出"刚到 minChunkSize 就封口,下一个 block 又很大"的碎 chunk。

- ChunkSize=512 → minChunkSize=128
- ChunkSize=200 → minChunkSize=50

---

## 防线 2:纯空白切片直接丢弃(appendChunk)

`heuristic_splitter.go:237-248`:
```go
func appendChunk(out, runes, start, end, seq) []Chunk {
    if end <= start { return out }
    raw := string(runes[start:end])
    if strings.TrimSpace(raw) == "" {  // ★ 纯空白直接跳过
        return out
    }
    c := Chunk{Content: raw, Seq: *seq, Start: start, End: end}
    *seq++
    return append(out, c)
}
```

边界聚集(多个连续边界)会产生"两个边界之间全是空行"的片段,`appendChunk` 检查 `TrimSpace` 是空就直接丢弃,不进结果。

---

## 防线 3:校验器 `ValidateChunks` 判碎 → 整层降级 Tier 3(兜底)

如果 Tier 2 切完还是有太多碎 chunk,校验器判失败,**整个 Tier 2 结果作废,降级到 Tier 3**。这是 `strategy.go:43-52` 的链式降级机制。

`validator.go` 的 4 个判碎判据:

```go
// 判据 1:碎 chunk 数量超过 1/4 且超过 2 个 → 失败
tinyCount := 0
for i, c := range chunks {
    if i == len(chunks)-1 { continue }  // 最后一个允许小,跳过
    if len([]rune(c.Content)) < 50 {   // < 50 字 = 碎
        tinyCount++
    }
}
if tinyCount > len(chunks)/4 && tinyCount > 2 {
    return ValidationResult{Reason: "too many tiny chunks"}
}

// 判据 2:最大 chunk 都不到 ChunkSize/4 → 切得太碎
if maxLen < chunkSize/4 && totalChars > chunkSize {
    return ValidationResult{Reason: "all chunks far below target size"}
}

// 判据 3:有 chunk 超过 2*ChunkSize → 切得太大
if maxLen > 2*chunkSize && chunkSize > 0 {
    return ValidationResult{Reason: "chunk exceeds 2x target size"}
}

// 判据 4:文档 >> ChunkSize 但只切出 1 个 → 没切
if len(chunks) == 1 && totalChars > 2*chunkSize {
    return ValidationResult{Reason: "single chunk for large document"}
}
```

**注意**:校验器**对最后一个 chunk 宽容**(`if i == len(chunks)-1 { continue }`),因为尾部残留小是正常的,不算碎。

---

## 对比 Tier 1 的碎处理

| 维度 | Tier 1 | Tier 2 |
|------|--------|--------|
| 切完碎的处理 | `coalesceTinyChunks` **合并**相邻碎 chunk | **无合并机制** |
| 防碎预防 | 无(切完再合) | `minChunkSize` 防过早封口 |
| 校验失败 | 降级 Tier 3 | 降级 Tier 3(同) |

**为啥 Tier 2 没有合并机制**:
- Tier 1 切的是标题边界,合并后面包屑用"共享前缀"仍能保住部分语义层级
- Tier 2 没有面包屑栈(ContextHeader 空),合并碎 chunk 只是把文本拼一起,没有任何语义信息可保留
- 合并的收益小,所以 Tier 2 选择"预防为主"(minChunkSize)+ "兜底降级"(校验器),不做合并

---

## 具体例子

假设 ChunkSize=200,minChunkSize=50,边界位置 `[0, 30, 60, 90, 120, 末尾=400]`。

**关键场景**:block 0-1 只有 20 字,block 1-2 有 250 字(超大)。

- **没 minChunkSize**:封口 20 字的 chunk → 碎!
- **有 minChunkSize=50**:20 < 50,不封口,直接走超大 block 路径(appendOversizeBlock),flush 20 字进前一个累积,然后递归切 250 字那块

结果:避免切出 20 字的碎 chunk。

---

## 4 个判碎判据详解

### 判据 1:碎 chunk 数量超标

```go
if tinyCount > len(chunks)/4 && tinyCount > 2 {
    return ValidationResult{Reason: "too many tiny chunks"}
}
```

- `< 50 字` 的 chunk(除最后一个)算"碎"
- 碎 chunk 数 > 总 chunk 数 / 4 **且** > 2 个 → 失败
- 两个条件都满足才判失败,避免小文档误判

例子:10 个 chunk,3 个 < 50 字 → 3 > 10/4=2 且 3 > 2 → 失败

### 判据 2:最大 chunk 都太小

```go
if maxLen < chunkSize/4 && totalChars > chunkSize {
    return ValidationResult{Reason: "all chunks far below target size"}
}
```

- 最大的 chunk 都不到 ChunkSize/4 → 切得太碎
- 加上 `totalChars > chunkSize` 防止短文档误判(短文档切完本来就应该都是小 chunk)

例子:ChunkSize=512,文档 2000 字,最大 chunk 100 字 → 100 < 128 且 2000 > 512 → 失败

### 判据 3:chunk 超大

```go
if maxLen > 2*chunkSize && chunkSize > 0 {
    return ValidationResult{Reason: "chunk exceeds 2x target size"}
}
```

- 任何 chunk > 2*ChunkSize → 切得太大,失败
- ChunkSize=512 时,任何 chunk > 1024 → 失败

### 判据 4:大文档只切出 1 个

```go
if len(chunks) == 1 && totalChars > 2*chunkSize {
    return ValidationResult{Reason: "single chunk for large document"}
}
```

- 文档 >> ChunkSize 但只 1 个 chunk → 根本没切,失败
- 防止某层策略形同虚设

---

## 你应该带走的 3 件事

1. **Tier 2 防碎有 3 道防线**:`minChunkSize`(防过早封口)→ `appendChunk` 丢弃纯空白 → `ValidateChunks` 校验失败整层降级 Tier 3。

2. **`minChunkSize = ChunkSize/4,最小 50`**:封口的**最小累积长度**,累积不到这个数即使下一个 block 会超 ChunkSize 也不封口,继续往里塞。

3. **Tier 2 没有合并机制**(Tier 1 有 `coalesceTinyChunks`):因为 Tier 2 没面包屑栈,合并碎 chunk 没语义信息可保留,收益小,所以选择"预防为主 + 兜底降级"。