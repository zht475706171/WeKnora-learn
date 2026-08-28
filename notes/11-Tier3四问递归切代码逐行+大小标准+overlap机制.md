# 11 · Tier 3 四问:递归切代码逐行 + 大小标准 + overlap 机制

> 用户问了 4 个核心问题:递归切代码逐行翻译、"太大"标准、"快满"标准、overlap 怎么留。这篇按代码逐个讲透。

---

## 问题 1:递归切分代码逐行翻译(splitBySeparators)

```go
func splitBySeparators(text string, separators []string, chunkSize int) []string {
```
**翻译**:函数名 `splitBySeparators`,输入:文本、分隔符列表(默认 `["\n\n", "\n", "。"]`)、chunkSize(每块最大字符数)。返回切好的字符串数组。

```go
    if text == "" || len(separators) == 0 {
        return []string{text}
    }
```
**翻译**:边界情况 —— 文本空或没分隔符,直接返回原文本(包一层 slice)。

```go
    if chunkSize > 0 && runeLen(text) <= chunkSize {
        return []string{text}
    }
```
**翻译**:**递归终止条件**。文本字数 ≤ chunkSize,**不再切**,直接返回。这是递归的"出口"。

```go
    for i, sep := range separators {
        if sep == "" {
            continue
        }
```
**翻译**:从左到右遍历分隔符。i=0 是 `\n\n`(最高优先级),i=1 是 `\n`,i=2 是 `。`。空分隔符跳过。

```go
        re := regexp.MustCompile("(" + regexp.QuoteMeta(sep) + ")")
```
**翻译**:把分隔符编译成正则,加括号变成**捕获组**。`QuoteMeta` 转义特殊字符(防 `\` 被当正则语法)。

```go
        splits := re.Split(text, -1)
```
**翻译**:用正则切文本,`-1` 切所有。结果**不含分隔符本身**。`"句一。句二。"` 用 `。` 切得 `["句一", "句二", ""]`。

```go
        matches := re.FindAllString(text, -1)
```
**翻译**:找出所有匹配的分隔符。`"句一。句二。"` 找 `。` 得 `["。", "。"]`。

```go
        if len(matches) == 0 {
            continue
        }
```
**翻译**:**这个分隔符在文本里不存在 → 试下一个分隔符**。

```go
        var pieces []string
        for j, s := range splits {
            if s != "" {
                pieces = append(pieces, s)
            }
            if j < len(matches) && matches[j] != "" {
                pieces = append(pieces, matches[j])
            }
        }
```
**翻译**:**把切出的片段和分隔符交错拼回去**,空片段丢弃。**分隔符被当作独立 unit 保留**,这样后面 mergeUnits 可以决定"把分隔符塞进哪个 chunk"。

举例 `"句一。句二。"` 用 `。` 切:
- splits = `["句一", "句二", ""]`,matches = `["。", "。"]`
- 交错拼接 → `["句一", "。", "句二", "。"]`

```go
        if len(pieces) <= 1 {
            continue
        }
```
**翻译**:切完只有 1 块(分隔符没起作用)→ 试下一个分隔符。

```go
        var out []string
        remaining := separators[i+1:]
```
**翻译**:准备输出数组。`remaining` = **当前分隔符之后的所有分隔符**(优先级更低的)。用 `\n\n`(i=0)切,remaining = `["\n", "。"]`。

```go
        for _, p := range pieces {
            if chunkSize > 0 && runeLen(p) > chunkSize && len(remaining) > 0 {
                out = append(out, splitBySeparators(p, remaining, chunkSize)...)
            } else {
                out = append(out, p)
            }
        }
```
**翻译**:**递归核心**。piece 字数 > chunkSize **且**还有更低级分隔符 → **递归调用自己**,用 `remaining` 内切这个 piece。

**关键**:传入的是 `remaining`(不是原 separators),所以递归时**只用更低级分隔符**,不会回头用 `\n\n` 切已经切过的 piece。

```go
        return out
    }
    return []string{text}
}
```
**翻译**:第一个能切出 >1 块的分隔符切完就 return。所有分隔符都切不动 → 返回原文。

### 递归核心思想一句话

**"用最高级分隔符切 → 谁还太大就单独对它用下一级分隔符内切,不切全文"**。保留层级:大段里的句子不会跑到别的段去。

### 走一遍例子

```
text = "段落A(150字)\n\n段落B(80字)"
separators = ["\n\n", "\n", "。"]
chunkSize = 100
```

1. text 230字 > 100,继续
2. i=0, sep=`\n\n`:
   - pieces = `["段落A", "\n\n", "段落B"]`
   - 遍历 pieces:
     - "段落A" 150字 > 100,remaining=`["\n", "。"]` → 递归
       - "段落A" 里没 `\n`,跳到 `\n`,没 `\n`,跳到 `。`
       - "段落A" 有句号 → pieces = `["句A1", "。", "句A2", "。", ...]`
       - 每个句 < 100,返回
     - "\n\n" 2字 < 100,直接加入
     - "段落B" 80字 < 100,直接加入
3. 返回 `["句A1", "。", "句A2", "。", ..., "\n\n", "段落B"]`

---

## 问题 2:大块太大的标准是什么?

**标准就是 `chunkSize`**(`cfg.ChunkSize`,默认 512)。

```go
if chunkSize > 0 && runeLen(p) > chunkSize && len(remaining) > 0 {
    // 递归内切
}
```

**3 个条件全满足才递归内切**:
1. `chunkSize > 0`:chunkSize 有效(=0 是"切完不检查大小"的特殊模式)
2. `runeLen(p) > chunkSize`:piece 字数 > chunkSize
3. `len(remaining) > 0`:还有更低级分隔符可用

**注意是 `>`,不是 `>=`**。等于 chunkSize 就不切了,刚好够。

### 配置链路

```
用户配的 ChunkSize(默认 512)
  ↓
SplitterConfig.ChunkSize
  ↓
SplitText 传给 splitBySeparators 当 chunkSize 参数
```

如果配 `TokenLimit`,会先算 `charBudget = CharsForTokenLimit(TokenLimit, lang)`,可能把 ChunkSize 改小(strategy.go::ensureDefaults)。

---

## 问题 3:快满的标准是什么?

**标准还是 `chunkSize`**,但加了表头预留。

`mergeUnits` 里:
```go
if curLen+uLen+headersLen > chunkSize && len(current) > 0 {
    // flush 当前累积,开新 chunk
}
```

**翻译**:
- `curLen` = 当前盒子已经装的字数
- `uLen` = 这次想装进来的 unit 字数
- `headersLen` = 活跃表头的字数(预留空间给表头 prepend)

**封口条件**(2 个全满足):
1. `curLen + uLen + headersLen > chunkSize`:装进来后总字数(含表头预留)会超 chunkSize
2. `len(current) > 0`:当前盒子非空(空盒子直接装,不存在"封口")

**所以"快满"= "装下一个会让总字数(含表头预留)超过 chunkSize"**。

### 例子

- ChunkSize=512,当前 480 字,下个 unit 50 字:480+50=530 > 512 → 封口!当前 480 字定下来,下个 unit 从新盒子开始
- ChunkSize=512,当前 400 字,下个 unit 80 字:400+80=480 ≤ 512 → 继续

### 表头预留的意义

有活跃表头(`headersLen > 0`)时,封口要给表头 prepend 留空间,确保新 chunk 装表头后也不超 chunkSize。

### 还有第二个硬上限:absoluteMaxSize=7500

```go
if curLen+uLen > absoluteMaxSize {
    // 硬 flush,无 overlap
}
```

**embedding API 的硬限制**。7500 字 ≈ 8000 token,卡在大多数 API 的 8K-10K 上限以内。任何 chunk 超 7500 强制 flush,不留 overlap。

---

## 问题 4:overlap 是怎么留的?

### 这就是"防止断语义"的做法

**overlap 让相邻 chunk 共享一部分内容**,防止语义在 chunk 边界被切断:

```
chunk 0: "...飞机票经济舱凭票报销"      ← 末尾
chunk 1: "经济舱凭票报销,公务舱需审批..." ← 开头
                  ↑↑↑↑↑↑↑↑
                  overlap,两个 chunk 都有
```

用户问"经济舱怎么报销",chunk 0 和 chunk 1 都能命中,检索更稳。

### 默认 overlap 是多少?

```go
const DefaultChunkOverlap = 80
```

**默认 80 字**(约 15% of 512)。注释里写明:
- 0:适合原子数据(FAQ、JSON 记录)
- 80:通用默认
- 150-200:适合长叙事文档(推理跨 chunk 多)

### overlap 受"下个 chunk 还能装多少"限制

```go
maxOverlap := chunkOverlap
if remaining := chunkSize - nextLen; remaining < maxOverlap {
    maxOverlap = remaining
}
if maxOverlap <= 0 {
    return nil, 0
}
```

**翻译**:下个 chunk 装完 nextLen 字后剩 `chunkSize - nextLen` 空间,overlap 不能超这个,否则下个 chunk 会超 chunkSize。

比如 chunkSize=100,nextLen=80,剩 20,overlap 最多 20(即使配了 80)。

### 搜索窗口:末尾 maxOverlap + 4 字

```go
const semanticOverlapLookbehind = 4
window := semanticOverlapWindow(current, maxOverlap+semanticOverlapLookbehind)
```

**翻译**:取当前盒子末尾 `maxOverlap + 4` 个字作为搜索窗口。多取 4 字是为了能找到 `\r\n\r\n` 这种 4 字符长分隔符(lookbehind)。

### 算"原始窗口起点"

```go
windowText := unitsText(window)
originalWindowStart := runeLen(windowText) - maxOverlap
if originalWindowStart < 0 {
    originalWindowStart = 0
}
```

**翻译**:`originalWindowStart` = 窗口总字数 - maxOverlap = "overlap 区域的起点"。边界必须在这个点**之后**(end ≥ originalWindowStart),否则 overlap 会超 maxOverlap。

### 在窗口里找语义边界

```go
boundaryEnd, ok := findSemanticOverlapBoundaryEndingAtOrAfter(windowText, originalWindowStart)
if !ok {
    return nil, 0
}
```

**翻译**:在窗口里找语义边界。**找不到 → 不 overlap,返回 nil**。这是关键:找不到合适边界就放弃 overlap,绝不切词中间。

### 3 级优先级(findSemanticOverlapBoundary)

```
优先级 1:段落分隔 \n\n / \r\n\r\n    ← 最强,段落边界
优先级 2:行分隔 \n / \r\n            ← 次强,行边界
优先级 3:句末 。？！ / ". " "? " "! "  ← 最弱,句子边界
```

**选择规则**:
1. 优先级高的赢(段落 > 行 > 句末)
2. 同优先级取**最早**(让 overlap 尽量大)
3. 边界后必须有非空白内容(防 overlap 是空尾巴)
4. 边界不在保护区间内(防切表格/代码)

### 砍掉窗口头部,留下 overlap

```go
overlap := trimUnitsPrefix(window, boundaryEnd)
overlapLen := 0
for _, u := range overlap {
    overlapLen += runeLen(u.text)
}
if overlapLen <= 0 || overlapLen > maxOverlap || strings.TrimSpace(unitsText(overlap)) == "" {
    return nil, 0
}
return overlap, overlapLen
```

**翻译**:从窗口头部砍掉 `boundaryEnd` 个字,剩下的就是 overlap。再检查:overlap 不能空、不能超 maxOverlap、不能纯空白。合格返回。

### 下一个 chunk 怎么用 overlap

`mergeUnits` 里:
```go
current, curLen = computeOverlap(current, chunkOverlap, chunkSize, uLen)
// 现在 current = [overlap 的 units]
// 然后继续往 current 里装后续 unit
current = append(current, u)
curLen += uLen
```

**overlap 的 unit 成为下个 chunk 的开头**,然后继续往里装新 unit。下个 chunk 的 `Start` = overlap 第一个 unit 的 `start`(原文偏移),所以位置不变量保住。

### 具体例子(用之前的)

```
当前盒子 = "句一。句二。句三。"(9字)
chunkOverlap=4, chunkSize=10, nextLen=2(下一个 unit "句四" 2字)

maxOverlap = min(4, 10-2=8) = 4
窗口 = 末尾 4+4=8 字 = "一。句二。句三。"
originalWindowStart = 8 - 4 = 4

找边界:
  "。" 在偏移 1:end=2 < 4,不合格
  "。" 在偏移 4(句二的句号):end=5 ≥ 4 ✓,后面 "句三。" 非空白 ✓,优先级 3
  "。" 在偏移 7(句三的句号):end=8 ≥ 4 ✓,但后面是空,不合格

选中:偏移 4 的句号,end=5
overlap = 窗口[5:] = "句三。"(3字)
```

---

## 4 个问题总结表

| 问题 | 答案 |
|------|------|
| 1. 递归切怎么实现 | `splitBySeparators`:用分隔符列表从高到低试,切出 >1 块就停,每块还 > chunkSize 就用**剩下的分隔符**递归内切 |
| 2. 大块太大的标准 | `runeLen(p) > chunkSize`(默认 512),用 `>` 不用 `>=` |
| 3. 快满的标准 | `curLen + uLen + headersLen > chunkSize`(表头预留),封口开新 chunk;硬上限 `curLen+uLen > 7500` |
| 4. overlap 怎么留 | 默认 80 字;受 `chunkSize - nextLen` 限制;**3 级优先级对齐**(段落>行>句末);找不到边界就**不 overlap**;overlap 是下个 chunk 的开头 |

---

## 你应该带走的 5 件事

1. **递归切 = "对还太大的 piece 用下一级分隔符内切"**:不是全文重切,保留层级。递归终止 = piece ≤ chunkSize。

2. **"太大"标准就是 chunkSize(默认 512)**:`runeLen(p) > chunkSize`,用 `>` 不 `>=`,等于刚好够不切。

3. **"快满"标准 = `curLen + uLen + headersLen > chunkSize`**:含表头预留,封口开新 chunk。硬上限 7500 防 embedding API 爆。

4. **overlap 默认 80 字,3 级优先级对齐**:段落 `\n\n` > 行 `\n` > 句末 `。？！/. `。受 `chunkSize - nextLen` 限制。**找不到边界就不 overlap**,绝不切词中间。

5. **overlap 是下个 chunk 的开头**:computeOverlap 返回的 units 成为新 chunk 的起始内容,`Start` 用 overlap 第一个 unit 的原文偏移,位置不变量保住。