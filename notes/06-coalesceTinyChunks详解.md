# 06 · coalesceTinyChunks 合并碎 chunk 详解

> 回答用户问题:Step 6 合并碎 chunk 怎么做?"面包屑用共享前缀"啥意思?举例

---

## 一、为啥要合并碎 chunk

FAQ 类文档每段只有几十字,Tier 1 切完会有一堆小 chunk,校验器会判"chunk 太小"失败 → 整个 Tier 1 降级到 Tier 3(纯递归切,不知道语义边界,切得更糟)。

`coalesceTinyChunks` 在降级前抢救:把相邻小 chunk 合并到合理大小,绕过校验,保住 Tier 1 的语义切分成果。

---

## 二、合并规则(4 个条件全满足才合并)

```go
if sharedHeader != "" && cur.End == next.Start && curLen < target && curLen+nextLen <= chunkSize
```

1. **有共享前缀**(`sharedHeader != ""`):两个 chunk 的面包屑至少有一行相同(安全阀,防把不相关的糊一起)
2. **相邻**(`cur.End == next.Start`):保证合并后 Content 仍是原文连续切片,保位置不变量
3. **当前累积还小**(`curLen < target`,target = ChunkSize/2,最小 200):一旦到 target 就封口
4. **合并后不超 ChunkSize**(`curLen+nextLen <= chunkSize`):硬上限,防超 embedding API 限制

---

## 三、"共享前缀"是啥意思 —— commonHeadingPrefix

两个面包屑按 \n 切成行数组,**从上往下逐行对比,到第一个不同的行就停,把相同的前几行拼回来**。

```go
func commonHeadingPrefix(a, b string) string {
    if a == b { return a }
    la := strings.Split(a, "\n")
    lb := strings.Split(b, "\n")
    common := 0
    for i := 0; i < min(len(la), len(lb)); i++ {
        if la[i] != lb[i] { break }
        common = i + 1
    }
    if common == 0 { return "" }
    return strings.Join(la[:common], "\n")
}
```

### 三个例子

**例子 1:完全相同**
```
cur  = "# 常见问题\n## 安装问题\n### Q1: 如何安装?"
next = "# 常见问题\n## 安装问题\n### Q1: 如何安装?"
→ "# 常见问题\n## 安装问题\n### Q1: 如何安装?"
```

**例子 2:前几行相同,后面不同(最常见)**
```
cur  = "# 常见问题\n## 安装问题\n### Q1: 如何安装?"
next = "# 常见问题\n## 安装问题\n### Q2: 依赖是什么?"
逐行对比:
  第 0 行 "# 常见问题" == "# 常见问题" ✓
  第 1 行 "## 安装问题" == "## 安装问题" ✓
  第 2 行 "### Q1..." vs "### Q2..." ✗ 停
common = 2
→ "# 常见问题\n## 安装问题"   ← 共享前缀(Q1/Q2 那行不算)
```

**例子 3:第一行就不同**
```
cur  = "# 文档A\n## 章节1"
next = "# 文档B\n## 章节2"
逐行对比:
  第 0 行 "# 文档A" vs "# 文档B" ✗ 停
common = 0
→ ""   ← 空字符串,不合并
```

---

## 四、完整 FAQ 例子走一遍

文档:
```markdown
# 常见问题

## 安装问题
### Q1: 如何安装?
答:执行 pip install weknora 即可。

### Q2: 依赖是什么?
答:Python 3.9+ 和 Go 1.26。

### Q3: 支持哪些系统?
答:Linux、macOS、Windows 都支持。

## 使用问题
### Q4: 怎么创建知识库?
答:点击右上角"新建知识库"按钮。

### Q5: 支持哪些文件格式?
答:PDF、Word、Excel、Markdown 等。
```

Tier 1 切完(ChunkSize=512,target=256):

| chunk | Content | ContextHeader | 字符数 |
|-------|---------|--------------|------|
| 0 | `# 常见问题\n## 安装问题` | `# 常见问题\n## 安装问题` | 14 |
| 1 | `### Q1: 如何安装?\n答:...` | `# 常见问题\n## 安装问题\n### Q1: 如何安装?` | 33 |
| 2 | `### Q2: 依赖是什么?\n答:...` | `# 常见问题\n## 安装问题\n### Q2: 依赖是什么?` | 31 |
| 3 | `### Q3: 支持哪些系统?\n答:...` | `# 常见问题\n## 安装问题\n### Q3: 支持哪些系统?` | 32 |
| 4 | `## 使用问题` | `# 常见问题\n## 使用问题` | 6 |
| 5 | `### Q4: 怎么创建知识库?\n答:...` | `# 常见问题\n## 使用问题\n### Q4: 怎么创建知识库?` | 35 |
| 6 | `### Q5: 支持哪些文件格式?\n答:...` | `# 常见问题\n## 使用问题\n### Q5: 支持哪些文件格式?` | 36 |

### 合并过程

**cur = chunk0, curLen=14**

**i=1 (chunk1)**:
- sharedHeader = commonHeadingPrefix("# 常见问题\n## 安装问题", "# 常见问题\n## 安装问题\n### Q1: 如何安装?")
  - cur 行 2 个,next 行 3 个
  - 第 0、1 行相同,cur 没第 2 行,循环结束
  - common = 2,返回 "# 常见问题\n## 安装问题"
- 4 条件满足 → 合并
- cur.Content += chunk1.Content
- cur.ContextHeader = "# 常见问题\n## 安装问题"
- curLen = 14+33 = 47

**i=2 (chunk2)**:
- sharedHeader = "# 常见问题\n## 安装问题"(同上)
- 4 条件满足 → 合并
- curLen = 47+31 = 78

**i=3 (chunk3)**:
- sharedHeader = "# 常见问题\n## 安装问题"
- 4 条件满足 → 合并
- curLen = 78+32 = 110

**i=4 (chunk4: "## 使用问题")**:
- sharedHeader = commonHeadingPrefix("# 常见问题\n## 安装问题", "# 常见问题\n## 使用问题")
  - 第 0 行 "# 常见问题" 相同 ✓
  - 第 1 行 "## 安装问题" vs "## 使用问题" ✗ 停
  - common = 1,返回 "# 常见问题"  ← **降级!**
- 4 条件满足(110 < 256,116 ≤ 512)→ 合并
- cur.ContextHeader = "# 常见问题"  ← **从 "## 安装问题" 降到 "# 常见问题"**
- curLen = 110+6 = 116

**i=5 (chunk5)**:
- sharedHeader = commonHeadingPrefix("# 常见问题", "# 常见问题\n## 使用问题\n### Q4...")
  - cur 行 1 个,next 行 3 个
  - 第 0 行 "# 常见问题" 相同,cur 没第 1 行,循环结束
  - common = 1,返回 "# 常见问题"
- 4 条件满足(116 < 256)→ 合并
- curLen = 116+35 = 151

**i=6 (chunk6)**:
- sharedHeader = "# 常见问题"
- 4 条件满足(151 < 256)→ 合并
- curLen = 151+36 = 187

### 最终结果

```
out[0]:
  Content = "# 常见问题\n## 安装问题" + chunk1 + chunk2 + chunk3 + chunk4 + chunk5 + chunk6
  ContextHeader = "# 常见问题"
  Start = 0, End = 文档末尾
  字符数 = 187
  Seq = 0
```

整个文档被合并成了**一个 chunk**(因为 187 < target 256,一直满足 curLen < target)。

---

## 五、共享前缀降级—— 重要副作用

跨章节合并时,**面包屑会被降级到更浅的层级**:

```
chunk0-3 都是 "## 安装问题" 下的 → 面包屑 "# 常见问题\n## 安装问题"
chunk4 是 "## 使用问题" 下的 → 面包屑 "# 常见问题\n## 使用问题"
两个的共享前缀 = "# 常见问题"  ← 从 H2 降到 H1
```

合并后:
- chunk4("## 使用问题"那个标题行)被糊进了安装问题的 chunk
- 整个合并 chunk 的面包屑从 "## 安装问题" 降到 "# 常见问题"
- Q4/Q5 被错误地归到"安装问题那批"的延续里,语义层级丢失

**但这是接受的取舍**:保住 Tier 1 不被降级到 Tier 3 更重要。Tier 3 不知道 Q1-Q5 是语义边界,会把 Q2 的答案切成两半,更糟。

---

## 六、为啥共享前缀是必须的(安全阀)

如果没有 `sharedHeader != ""` 检查:

```
文档:
# 文档A
内容 X
# 文档B
内容 Y
```

切出来两个 chunk,面包屑分别是 `# 文档A` 和 `# 文档B`。直接合并:
- Content = "内容 X 内容 Y"
- 面包屑用哪个?用 `# 文档A` 错(内容 Y 不是文档A 的),用 `# 文档B` 也错

`sharedHeader != ""` 至少保证两个 chunk **属于同一个顶层文档/章节**。例子里文档A 和文档B 的 commonHeadingPrefix 是 ""(第一行就不同),所以不合并,各自独立。

---

## 七、几个关键设计点

### 1. target = ChunkSize/2,最小 200

```go
target := chunkSize / 2
if target < 200 { target = 200 }
```

ChunkSize=512 → target=256;ChunkSize=100 → target=200(不是 50),防止 target 太小导致合并完还是碎。

### 2. 重新编号 Seq

```go
for i := range out {
    out[i].Seq = i
}
```

合并后 chunk 数量变了,Seq 要重新连续 0..N-1。下游代码(knowledge.go)依赖 Seq 是连续的。

### 3. 位置不变量保持

```go
cur.Content += next.Content      // 文本拼接
cur.End = next.End                // End 扩展
// Start 不变
// 因为 cur.End == next.Start 保证,Content 仍是原文连续切片
// End - Start = 新字符数,不变量保住
```

### 4. 共享前缀降级是接受的副作用

跨章节合并会让面包屑从深层降级到浅层,代码接受这个降级。

---

## 八、对比效果

### 不合并(Tier 1 失败 → Tier 3)

Tier 3 用递归切,按 \n\n \n 。 切,可能切出:
- Q1 和 Q2 糊一起(检索"如何安装"返回 Q1+Q2,噪声大)
- 没有面包屑(向量不带章节语义)
- "## 使用问题" 标题被切到 Q3 的 chunk 里(语义边界被破坏)

### 合并后(coalesceTinyChunks)

- 整个 FAQ 一个 chunk(粒度粗,但 chunk 大小合理不触发校验)
- 面包屑 "# 常见问题"(降级了但还在)
- 位置不变量保住(能重建文档)

**取舍**:粒度粗可以靠"父子分块"解决(parent 大块、child 小块);面包屑降级比没面包屑好;位置不变量比 Tier 3 切完不能重建强。

---

## 九、你应该带走的 4 件事

1. **`coalesceTinyChunks` 是 Tier 1 的"抢救机制"**:FAQ 类碎 chunk 会触发校验"chunk 太小"降级到 Tier 3,合并碎 chunk 能保住 Tier 1 的语义切分成果。

2. **合并 4 个条件**:有共享前缀 + 相邻(cur.End == next.Start)+ 当前累积 < ChunkSize/2 + 合并后 ≤ ChunkSize。

3. **"共享前缀"=两个面包屑按行切分逐行对比,到第一个不同的行停,把相同的前几行拼回来**。例:`# 常见问题\n## 安装问题\n### Q1` 和 `...\### Q2` 共享前缀是 `# 常见问题\n## 安装问题`(Q1/Q2 那行不同,停)。

4. **共享前缀降级是接受的副作用**:跨章节合并会让面包屑从深层降级到浅层(如 `## 安装问题` 降到 `# 常见问题`),因为保住 Tier 1 不降级 > 保住面包屑精确层级。