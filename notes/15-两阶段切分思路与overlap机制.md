# 15 · 两阶段切分思路 + overlap 80 字机制详解

> 用户总结理解:切 + 合并两阶段,unit 小不怕,合并兜底。然后问 overlap 80 字怎么判断的,细节是什么,多个边界怎么选?

---

## 用户的理解(完全正确)

> 实际 tier2 和 tier3 都是先按规则将单 unit 压缩到 512 之内,如果大于 512,则按下一级继续切,这样可能会将内容切成很小的一段一段的,但是这个 unit 虽然可能小,后面还有个合并处理的逻辑,所以基本不用怕他太小对吧?

**完全正确!** 这正是设计的精髓。

```
两阶段:先切 → 再合并

阶段1:切(把大块压到 ≤ 512,可能切出小 unit)
  Tier 3:  \n\n → \n → 。  (按符号递归,大块用下一级内切)
  Tier 2:  8 个正则找切点(大 block 委托 Tier 3 内切)

阶段2:合并(把小 unit 贪心装到 512 一个 chunk)
  mergeUnits:从左到右装,装到快超 512 封口
  → 即使 unit 是 20 字的句子,合并后 chunk ≈ 512 字
```

unit 小不用怕,合并会兜底。除非极端情况(比如文档总长就 < 512,那就一个 chunk,也合理)。

⚠️ 小修正:
- Tier 2 不是递归用正则,8 个正则是**平行找切点**(同位置去重),然后贪心装箱。**超大 block 委托 Tier 3 内切**,不是"用下一个正则切"。
- Tier 3 的"下一个符号"对,递归用下一级分隔符内切。

---

## 80 字 overlap 是配置默认值,不是动态算的

```go
const DefaultChunkOverlap = 80
```

**80 是写死的默认值**,不是动态算的。注释:
> DefaultChunkOverlap = 80 chars (≈15% of DefaultChunkSize): community-recommended sweet spot

**3 个档位**:

| 配置 | 适用场景 | 原因 |
|------|---------|------|
| **0** | FAQ、JSON 记录(原子数据) | 每条记录独立,不需要跨边界检索 |
| **80**(默认) | 通用文档 | 平衡召回率和存储成本 |
| **150-200** | 长叙事、论证类文档 | 推理跨 chunk 多,需要更多 overlap 保上下文 |

---

## overlap 怎么取 —— 完整流程

### Step 1:算 maxOverlap(实际能取多少)

```go
maxOverlap := chunkOverlap  // 默认 80
if remaining := chunkSize - nextLen; remaining < maxOverlap {
    maxOverlap = remaining  // ★ 受下个 chunk 剩余空间限制
}
```

**关键**:下个 chunk 装完下一个 unit(`nextLen` 字)后剩 `chunkSize - nextLen` 空间,overlap 不能超这个。

**例子**:
- 下个 unit 50 字,512-50=462,overlap 最多 80(80 < 462,不限制)
- 下个 unit 480 字,512-480=32,overlap 最多 32(不是 80!)

### Step 2:开搜索窗口

```go
const semanticOverlapLookbehind = 4
window := semanticOverlapWindow(current, maxOverlap+4)
```

**窗口 = maxOverlap + 4**,从 chunk 末尾往前取这么多字。

**为什么多取 4 字**:为了能完整看到 `\r\n\r\n`(4 字符段落分隔符)的 lookbehind。

### Step 3:算 originalWindowStart

```go
originalWindowStart := runeLen(windowText) - maxOverlap
```

**意思**:窗口 84 字,但只有**最后 maxOverlap 字**是"overlap 区域"(前 4 字是 lookbehind)。边界必须在 overlap 区域内,否则 overlap 会超 maxOverlap。

### Step 4:在窗口里找所有语义边界候选

**3 级优先级**(数字越小越强):

```
优先级 1:段落分隔 \n\n / \r\n\r\n    ← 最强
优先级 2:行分隔 \n / \r\n            ← 次强
优先级 3:句末 。？！ / ". " "? " "! "  ← 最弱
```

每个候选检查 3 个约束:
- `end ≥ originalWindowStart`(边界在 overlap 区域内)
- `hasMeaningfulTail(end)`(边界后非空白)
- `!insideProtected(start)`(不在保护区间内)

### Step 5:选最优(关键!)

```go
if candidate.priority < best.priority ||
   (candidate.priority == best.priority && candidate.start < best.start) {
    best = candidate
}
```

**2 条规则**:
1. **优先级高的赢**(1=段落 > 2=行 > 3=句末)
2. **同优先级取最早**(让 overlap 尽量大,保留更多上下文)

### Step 6:找不到合格边界 → 不 overlap

```go
if !ok {
    return nil, 0  // ★ 找不到就别 overlap
}
```

**宁可不 overlap 也不切词中间**。

### Step 7:overlap 成为下个 chunk 的开头

下个 chunk Start = overlap 第一个 unit 的原文偏移,位置不变量保住。

---

## 你的疑问:84 字里既有 。又有 \n 又有 \n\n,怎么选?

**答案:不是取第一个,不是取最后一个,是"按优先级选"**。

### 具体例子

假设窗口 84 字里:
```
位置 10: 。      ← 优先级 3
位置 20: \n      ← 优先级 2
位置 30: \n\n    ← 优先级 1
位置 40: \n      ← 优先级 2
位置 50: 。      ← 优先级 3
位置 60: \n\n    ← 优先级 1
```

**遍历所有候选,选最优**:

| 候选 | 优先级 | 是否选中 |
|------|--------|---------|
| 位置 10: 。 | 3 | ✗ 被优先级高的淘汰 |
| 位置 20: \n | 2 | ✗ 被优先级高的淘汰 |
| **位置 30: \n\n** | **1** | **✓ 当前最优** |
| 位置 40: \n | 2 | ✗ 优先级低于 30 |
| 位置 50: 。 | 3 | ✗ 优先级低于 30 |
| 位置 60: \n\n | 1 | ✓ 但位置 30 更早,选 30 |

**最终选位置 30 的 `\n\n`**(段落分隔,优先级最高,且同优先级取最早)。

### 为什么这么设计

- **优先级高 = 语义边界强**:段落分隔比行分隔强,行分隔比句末强。能对齐到段落就不要对齐到句末。
- **同优先级取最早**:既然都是段落分隔,取最早的那个能让 overlap 包含更多内容(更大),上下文更完整。

---

## 3 个约束总结

### 约束 1:边界在 overlap 区域内

```go
end >= originalWindowStart
```

防止 overlap 超 maxOverlap。

### 约束 2:边界后必须有内容(hasMeaningfulTail)

```go
hasMeaningfulTail := func(end int) bool {
    return end >= 0 && end < len(runes) && strings.TrimSpace(string(runes[end:])) != ""
}
```

防止 overlap 是空尾巴(比如 chunk 末尾正好是 `\n\n`,边界后是空)。

### 约束 3:边界不在保护区间内(insideProtected)

```go
insideProtected := func(pos int) bool {
    for _, p := range protected {
        if pos >= p.start && pos < p.end { return true }
    }
    return false
}
```

防 overlap 切在表格/代码/LaTeX 中间。

---

## 你应该带走的 4 件事

1. **你的两阶段理解对了**:切 + 合并。切可能切出小 unit,但合并兜底装到 512,不用怕小。除非极端情况,可忽略。

2. **80 字是配置默认值**(DefaultChunkOverlap=80,约 15% of 512),不是动态算的。受 `chunkSize - nextLen` 限制,实际 overlap 可能 < 80。

3. **不是"找到第一个就停"**:遍历所有候选,按优先级 + 位置规则选最优。优先级高的赢,同优先级取最早。

4. **3 级优先级**:段落 `\n\n` > 行 `\n` > 句末 `。？！/. `。**数字越小越强**(1=段落 < 2=行 < 3=句末)。3 个约束:在 overlap 区域内、边界后非空白、不在保护区间内。找不到合格边界就**不 overlap**,绝不切词中间。