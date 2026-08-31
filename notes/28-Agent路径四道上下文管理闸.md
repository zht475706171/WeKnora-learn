# 28 · Agent 路径四道上下文管理闸

> 用户细问:Agent 路径的 trimCurrentTurnToolResults / Consolidator 怎么做裁剪?展开说一下。
> 本篇细讲 Agent 路径(`AgentEngine`)在每次 LLM 调用前跑的 4 道上下文管理闸。KnowledgeQA 路径没有这 4 道(笔记 27 讲过,这是笔记 27 优化点②的对照面)。
> 配套:`redactHistoryKBResults`(涂黑)和 `CompressContext`(硬截)本篇给框架,留作后续单独深挖。

---

## 一句话

> **Agent 路径每次 LLM 调用前跑 `manageContextWindow`(observe.go:26),4 道闸串行:①`trimCurrentTurnToolResults` 当前轮工具结果按预算裁(占位符+倒序补全+头尾预览,配对不拆)②`redactHistoryKBResults` 历史轮 KB 工具结果涂黑(默认开)③`Consolidator` 旧历史 LLM 摘要成 system 消息(token>10万触发,tool 组不拆,失败 rawArchive 兜底)④`CompressContext` 硬砍最旧消息(token>16万兜底,group 不可分)。4 道全是视图层——只改发给 LLM 的 messages 数组,不动 DB 原版。KnowledgeQA 一道都没有,靠 maxRounds 硬兜。**

---

## 一、为什么 Agent 路径需要 4 道闸

Agent 是 ReAct 循环:思考→调工具→观察→再思考,多轮多步。token 增长模式跟 KnowledgeQA 完全不同:

- **KnowledgeQA**:单轮单次 RAG,token 量 = system + 历史(固定5轮) + 当前检索 context + 当前 query。靠 maxRounds 硬兜就够,不需要运行时压缩。
- **Agent**:多轮 tool 调用,每轮 tool 结果可能巨大(grep 几 MB、Read 大文件几十 KB),历史累积爆炸。必须有运行时压缩闸,否则 context 必爆。

4 道闸针对 4 个不同的 token 增长源:

| 闸 | 针对什么 | 触发时机 | 做法 | 语义保留 |
|---|---|---|---|---|
| ① trim | 当前轮单个工具结果太大 | 每次 LLM 调用前都跑 | 占位符+补全+头尾预览 | 中(头尾留) |
| ② redact | 历史轮 KB 工具结果可能过时 | 装配历史进 context 时 | 整条 content 换占位符 | 低(全丢) |
| ③ Consolidator | 旧历史累积太多 | token > 0.5(10万) | LLM 摘要成 system 消息 | 高(LLM 浓缩) |
| ④ Compress | 极端兜底 | token > 0.8(16万) | 硬砍最旧 group | 零(直接删) |

**层次关系**:①②是无脑预防(每次都跑或装配时跑),③④是阈值触发(达到才跑)。③在 0.5 先触发优先用(LLM 摘要语义保留好),④在 0.8 兜底(③摘要后还涨才硬砍)。

---

## 二、trimCurrentTurnToolResults(闸①)深挖

位置:`observe.go:79-143` + `compactToolMessage`(observe.go:152) + `compactedToolResultMarker`(observe.go:145)。

### 2.1 作用范围:只动当前轮的 tool 消息

从最后一个 `role=user` 消息往后扫,只收集 `role=tool` 的消息。历史轮 tool 消息不碰(②管)。一个 ReAct 循环里用户问完后 LLM 可能调多轮工具才给答案,这些 tool 消息都属于"当前轮",全在裁剪范围。

### 2.2 预算(observe.go:65 currentTurnToolResultBudget)

```go
budget = MaxContextTokens / 5   // 上下文的 20%
下限 8K (minCurrentTurnToolTokens = 8*1024)
上限 32K (maxCurrentTurnToolTokens = 32*1024)
```

默认 20 万上下文 → budget=40000,被上限卡到 32K。**当前轮所有工具结果加起来最多 32K token**。防当前轮工具结果把后续推理空间吃爆——LLM 还要留空间思考、调下个工具、生成最终答案。

### 2.3 算法核心:两阶段"先占位→后补全"

**阶段 A:先把所有当前轮 tool 消息 content 全替换成最短 marker**(observe.go:115-126)

```go
out := append([]chat.Message(nil), messages...)  // ★拷贝,不原地改
for _, idx := range toolIndexes {
    out[idx].Content = compactedToolResultMarker(messages[idx].Content)
    baseCosts[idx] = 估算替换后的 token
    remaining -= baseCosts[idx]
}
```

marker 是固定字符串(observe.go:145):
```
[Tool result compacted: original_bytes=N. Re-run the tool with narrower filters or a smaller range if more detail is needed.]
```

N 是原始内容字节数,给 LLM 留线索"原来多大"并暗示"要细节重查"。这步先把所有 tool 消息压到最短,保证"最差也能塞下"——保底。`remaining` 是预算减去所有 marker 成本,剩空间用来补全。

**阶段 B:从新到旧倒序补全**(observe.go:128-141)

```go
for i := len(toolIndexes) - 1; i >= 0; i-- {  // ★倒序,最新最先
    idx := toolIndexes[i]
    fullCost := 估算原始 tool 结果 token
    extra := fullCost - baseCosts[idx]  // 从 marker 恢复成全文要补的差额
    if extra <= remaining {
        out[idx] = messages[idx]  // 预算够,用原文
        remaining -= extra
        continue
    }
    // 预算不够装全文,装"头+尾预览"
    out[idx] = compactToolMessage(messages[idx], baseCosts[idx]+remaining, estimator)
    remaining = 0
}
```

**关键取舍:最新先补全**。当前轮里最新工具结果跟 LLM 当前推理最相关,旧的能省则省。跟"历史轮老的先丢"是同一个远近原则。

### 2.4 头尾预览:compactToolMessage(observe.go:152)

当某条 tool 结果原文太长预算只够装一部分,做的不是简单截断,而是 **marker + 头部 + 省略号 + 尾部**:

```go
func compactToolMessage(msg, maxTokens, estimator) chat.Message {
    runes := []rune(msg.Content)
    base := msg
    base.Content = compactedToolResultMarker(msg.Content)  // marker 永远在前
    if len(runes) == 0 || estimator.EstimateMessage(&base) >= maxTokens {
        return base  // marker 自己就超预算,只发 marker
    }

    best := base
    low, high := 1, len(runes)
    for low <= high {  // ★二分搜索找最大 keep
        keep := low + (high-low)/2
        head := keep / 4           // ★头部 1/4
        tail := keep - head        // ★尾部 3/4
        candidate := base
        candidate.Content = fmt.Sprintf(
            "%s\n\n%s\n...[tool result preview omitted]...\n%s",
            base.Content,
            string(runes[:head]),
            string(runes[len(runes)-tail:]),
        )
        if estimator.EstimateMessage(&candidate) <= maxTokens {
            best = candidate
            low = keep + 1
        } else {
            high = keep - 1
        }
    }
    return best
}
```

**3 个细节**:

1. **marker 永远在前面**:即使能装部分原文,marker 也保留。LLM 看到"这条被压缩了,原文 N 字节"同时能看到头尾,知道"要完整就重查"。

2. **头尾比例 1:3**(`head := keep/4; tail := keep-head`):尾部留多。工具结果关键信息常在末尾——错误码、最终返回值、状态行、堆栈根因。头部一般是开场(命令回显、初始输出),信息密度低。

3. **二分搜索 keep**:找"总 token 不超预算"的最大 keep。`EstimateMessage` 非线性(多一个字符不一定是固定 token),不能直接算只能二分。性能优化避免每加一个 rune 都重新估算。

### 2.5 关键保证:配对不拆

注释 observe.go:81-82:
> Assistant tool-call messages remain untouched so every compacted tool result retains its provider-required call/result pairing.

**只动 tool 消息的 Content,不删消息、不动 assistant 消息(带 tool_calls 的那条)**。LLM provider(OpenAI 兼容协议)要求每个 tool_call 必须有对应 tool_result,否则请求被拒。裁剪**永远只改 Content、不删消息**。

### 2.6 不原地改的原因(observe.go:80 注释)

> It never mutates ToolResult objects used by SSE, diagnostics, or persistence.

拷贝一份再改。原始 tool 结果对象还被 SSE 流式输出(前端实时显示)、诊断日志、DB 持久化在用,原地改会污染所有这些地方。"视图层"操作的典型约束——发往 LLM 的是视图,原版不动。

### 2.7 Worked Example:4 个工具各 10000 token,budget=32K

**初始**:
```
[system, ..., user(当前问题),
  assistant(tool_calls: [T1,T2,T3,T4]),  ← 不动
  tool(T1, 10000),
  tool(T2, 10000),
  tool(T3, 10000),
  tool(T4, 10000)
]
```

total=40000 > 32000,触发。

**阶段 A**:全替换成 marker(~50 token/条),total=200,remaining=31800。

**阶段 B** 倒序补全:
- T4(最新):extra=9950 ≤ 31800 → 用原文,remaining=21850
- T3:extra=9950 ≤ 21850 → 用原文,remaining=11900
- T2:extra=9950 ≤ 11900 → 用原文,remaining=1950
- T1(最旧):extra=9950 > 1950 → 装不下,走 compactToolMessage,maxTokens=50+1950=2000
  - 二分找最大 keep:假设 keep=1900 时 token=1980 ≤ 2000
  - head=1900/4=475, tail=1900-475=1425
  - 最终:`[marker]\n\n<前475字符>\n...[tool result preview omitted]...\n<后1425字符>`

**结果**:T4/T3/T2 原文(各 10000),T1 头尾预览(~2000)。总 32000,配对全保留,最新 3 条完整,最旧 1 条头尾预览。

---

## 三、redactHistoryKBResults(闸②)框架

位置:`observe.go:686`。

### 3.1 做什么

遍历**历史轮**(不是当前轮)消息,凡是 `role=tool` 且工具名在 `kbToolNames` 集合(KB 检索类工具)的,把 content 替换成:

```
[Previous retrieval result omitted — knowledge base may have changed. Please perform a fresh search.]
```

**不删消息、不破配对**,只改 content。占位符带行动建议"Please perform a fresh search",明确告诉 LLM "要查重查"。

### 3.2 触发时机

装配历史进 LLM context 时(`buildMessagesWithLLMContext` observe.go:704),由 `RetainRetrievalHistory` 开关控制:

```go
if e.config.RetainRetrievalHistory {
    sanitized = llmContext  // 保留全量历史检索结果
} else {
    sanitized = redactHistoryKBResults(llmContext)  // ★默认走涂黑
}
```

**默认 false → 默认涂黑**。

### 3.3 为什么

注释 observe.go:684:
> This prevents the LLM from reusing stale retrieval data when the knowledge base has been modified or switched between turns.

历史轮检索结果可能已过时(KB 被改、chunk 变了、换 KB)。占位符让 LLM "别信旧结果,要查重查",但保留消息骨架让 LLM 知道"上一轮确实调过这个工具、拿到了某种结果"。

### 3.4 留作深挖

KB 工具集合具体有哪些、占位符精确语义、跟 KnowledgeQA 丢 RenderedContent 的关系、`RetainRetrievalHistory=true` 的使用场景——本篇不展开,留后续单独深挖。

---

## 四、Consolidator(闸③)深挖

位置:`internal/agent/memory/consolidator.go`。**用 LLM 把旧历史摘要成一条 system 消息**。跟闸①"占位符机械替换"是两种思路——闸①保结构丢内容,Consolidator 丢结构保语义。

### 4.1 触发条件(consolidator.go:63 ShouldConsolidate)

```go
triggerAt = MaxContextTokens * 0.5   // 默认 20万*0.5 = 10万 token
currentTokens > triggerAt → 触发
```

比 CompressContext 的 0.8(16万)早触发。**Consolidator 是"先礼"优先用,语义浓缩信息损失小;CompressContext 是"后兵"兜底**。

开关:`maxTokens <= 0` → `ShouldConsolidate` 永远 false,Consolidator 完全不工作,只剩 CompressContext 兜底。

### 4.2 算法核心:三段切割 + LLM 摘要 + 装配

**Step 1:消息太少直接返回**(consolidator.go:84-86)

```go
if len(messages) <= 3 { return messages, nil }
```
少于 4 条没东西可摘要。

**Step 2:切三段**(consolidator.go:88-109)

```go
systemMsg = messages[0]               // system prompt,保留
lastUserIdx = 找最后一个 role=user 的 index  // 当前轮起点
history = messages[1:lastUserIdx]     // ★要被摘要的旧历史
tail = messages[lastUserIdx:]         // ★当前轮,完整保留
```

当前轮(最后一个 user + 后续所有 assistant/tool)**完整保留不动**。原因:当前轮 LLM 正在用,摘要它会破坏进行中的推理。

边界:`lastUserIdx <= 1`(旧历史不到1条)或 `len(history) < 2`(少于2条) → 直接返回。

**Step 3:算目标 token**(consolidator.go:111)

```go
targetTokens = MaxContextTokens * 0.5 * 0.6 = MaxContextTokens * 0.3
```
摘要后想压到**上下文的 30%**(默认 6 万)。给当前轮 + system + 后续新增留 70% 余量。

**Step 4:找保留边界 `findKeepBoundary`**(consolidator.go:118)——关键算法

决定**旧历史里最近 N 条保留原文、更早的才送 LLM 摘要**。逻辑(consolidator.go:158-212):

```go
budget = targetTokens - systemMsg成本 - tailTokens - 500  // 500 预留给摘要消息本身
if budget <= 0 { return 0 }  // 当前轮就把预算吃光,全摘要

tokens := 0
keepCount := 0
i = len(history) - 1  // 从最新开始往前数

for i >= 0 {
    msg = history[i]
    msgTokens = 估算 msg 的 token

    if msg.Role == "tool" {
        // ★关键:把连续 tool 消息 + 触发它们的 assistant(tool_calls) 当一组
        groupTokens = msgTokens
        groupSize = 1
        j = i - 1
        for j >= 0 && history[j].Role == "tool" {  // 往前扫连续 tool
            groupTokens += 估算 history[j]
            groupSize++
            j--
        }
        if j >= 0 && history[j].Role == "assistant" {  // 再往前那个 assistant
            groupTokens += 估算 history[j]
            groupSize++
        }
        if tokens + groupTokens > budget { break }  // ★整组超了就停
        tokens += groupTokens
        keepCount += groupSize
        i -= groupSize
    } else {
        if tokens + msgTokens > budget { break }
        tokens += msgTokens
        keepCount++
        i--
    }
}
return keepCount
```

**核心不变量:tool_call/tool_result 组不可分**。遇到 tool 消息时,往前把所有连续 tool 消息 + 触发它们的 assistant(带 tool_calls)当一个组。要么整组保留,要么整组摘要,**绝不拆**。

跟闸①"配对不拆"同一个原因——assistant 的 tool_calls 和对应 tool 结果必须成对出现,否则 LLM 看到"调了工具没结果"会困惑。摘要时把 assistant(tool_calls) 留下但把对应 tool 结果摘要掉,配对破坏。

**Step 5:切出要摘要的和要保留的**(consolidator.go:120-125)

```go
if keepFromEnd >= len(history) { return messages, nil }  // 全能保留,不摘要
toConsolidate = history[:len(history)-keepFromEnd]  // ★送 LLM 摘要
toKeep = history[len(history)-keepFromEnd:]         // ★原文保留(最近的)
```

**Step 6:调 LLM 摘要**(consolidator.go:127 summarizeWithRetry)

prompt 构造(`buildConsolidationPrompt` consolidator.go:253)把 `toConsolidate` 格式化成纯文本:

```
Summarize the following conversation history, preserving:
1. Key facts and decisions made
2. Tool execution results and their outcomes
3. User's original intent and requirements
4. Any errors encountered and how they were resolved

Conversation to summarize:

**User**: <user 内容,截 2000 字符>
**Assistant** [called tools: tool1, tool2]: <assistant 内容,截 1000 字符>
**Tool [tool1]**: <tool 结果,截 1000 字符>
**Tool [tool2]**: <tool 结果,截 1000 字符>
**Assistant**: <assistant 内容,截 2000 字符>
...
```

system prompt(`consolidationSystemPrompt` consolidator.go:326):

```
You are a conversation summarizer. Your task is to create a concise but comprehensive summary of a conversation between a user and an AI assistant.

The summary should:
- Be written in the same language as the original conversation
- Preserve all key facts, numbers, and specific details
- Include the outcomes of any tool executions
- Note any errors or issues encountered
- Be structured with clear sections if the conversation covered multiple topics
- Be concise — aim for 30% or less of the original length

Output only the summary, no preamble or explanation.
```

调用参数:
- `Temperature: 0.3`(事实性摘要要稳,不要发散)
- `MaxTokens: 2000`(摘要本身不能太长,否则失去压缩意义)
- 超时 60 秒(`consolidationTimeout`)
- **重试 3 次**(`maxConsolidationAttempts`)

**Step 7:LLM 失败兜底**(consolidator.go:128-132)

```go
if err != nil {
    summary = rawArchive(toConsolidate)  // 纯文本 dump 兜底
}
```

`rawArchive`(consolidator.go:288)把每条消息截 500 字符拼成:

```
Raw conversation archive (LLM summarization unavailable):

- User: <截 500 字符>
- Assistant [tools: tool1,tool2]: <截 500 字符>
- Tool[tool1]: <截 500 字符>
- Tool[tool2]: <截 500 字符>
- Assistant: <截 500 字符>
```

**无 LLM 理解**,保底能塞下,但语义保留差。是"摘要失败也比原文全塞强"的兜底。

**Step 8:组装结果**(consolidator.go:134-146)

```go
result = [
    systemMsg,                                              // 原 system
    {Role: "system", Content: "[Memory Summary - N earlier messages consolidated]\n\n" + summary},
    ...toKeep,                                              // 最近的旧历史(原文)
    ...tail,                                                // 当前轮(原文)
]
```

摘要作为**一条新的 system 消息**插在 system prompt 之后、保留的历史之前。用 system role 而不是 user/assistant——它是"机器生成的背景说明"不是真实对话。用 user role 会混淆"用户真的说过这话",用 assistant role 会混淆"LLM 真的答过"。

### 4.3 Worked Example:全摘要场景

```
[0] system (2000)
[1] user_old1 (500)
[2] assistant_old1 (800)
[3] user_old2 (500)
[4] assistant_old2_toolcall (300)  ← 带 tool_calls
[5] tool_old2_result (15000)
[6] user_old3 (500)
[7] assistant_old3 (700)
[8] user_current (1000)   ← lastUserIdx = 8
[9] assistant_current_toolcall (400)
[10] tool_current_result (12000)
```

当前 token=33700,假设 MaxContextTokens=40000,触发阈值 0.5=20000,已超。

**切三段**:
- systemMsg = [0]
- history = [1]..[7](token: 500+800+500+300+15000+500+700 = 18300)
- tail = [8]..[10](token: 1000+400+12000 = 13400)

**算目标**:targetTokens = 40000*0.3 = 12000

**findKeepBoundary**:budget = 12000 - 2000(system) - 13400(tail) - 500 = -3900 ≤ 0 → return 0

当前轮 + system 就把预算吃光了,旧历史一条都不保留,**全部送 LLM 摘要**。

**toConsolidate = [1]..[7]**,toKeep = 空。

**调 LLM**:把 [1]..[7] 格式化成纯文本,调 LLM 摘要,假设返回 800 token summary。

**装配**:
```
[0] system (2000)
[new] system: [Memory Summary - 7 earlier messages consolidated]\n\n<800 token 摘要> (850)
[8] user_current (1000)
[9] assistant_current_toolcall (400)
[10] tool_current_result (12000)
```

总 token = 16250,从 33700 降到 16250,降 52%。降到 0.4(target 0.3 没达标但接近,摘要本身有开销)。

### 4.4 Worked Example:有保留场景

改 tail 只 5000 token:
- budget = 12000 - 2000 - 5000 - 500 = 4500

**findKeepBoundary 从末尾往前数**:
- i=7(assistant_old3, 700):700 ≤ 4500,tokens=700,keepCount=1,i=6
- i=6(user_old3, 500):700+500=1200 ≤ 4500,tokens=1200,keepCount=2,i=5
- i=5(tool_old2_result, 15000):role=tool!往前扫连续 tool:[5] 是 tool,[4] 是 assistant(触发者),组=[4]+[5],groupTokens=300+15000=15300。1200+15300=16500 > 4500 → break

**keepCount=2**:保留 [6]+[7]。

**toConsolidate = [1]..[5]**(user_old1, assistant_old1, user_old2, assistant_old2_toolcall, tool_old2_result),**toKeep = [6]+[7]**。

**装配**:
```
[0] system
[new] system: [Memory Summary - 5 earlier messages consolidated]\n\n<summary>
[6] user_old3 (原文)
[7] assistant_old3 (原文)
[8] user_current
[9] assistant_current_toolcall
[10] tool_current_result
```

**关键**:[4]+[5] 这个 tool_call/tool_result 组**整组被摘要**,没拆。如果 findKeepBoundary 没做分组,可能只摘要 [5] tool 结果但保留 [4] assistant(tool_calls)——LLM 看到"调了工具没结果",配对破坏。分组算法防的就是这个。

### 4.5 一个容易忽略的点:摘要会嵌套

Consolidator 跑完后 messages 数组变了——多了 `[Memory Summary]` system 消息,旧历史被摘要掉了。这条变更**会被后续轮次继承**——下一轮 LLM 调用时 messages 里已经有这条 summary 了。

会话很长时 Consolidator 可能触发多次。第二次触发时,第一次的 `[Memory Summary]` 也会被算进"要摘要的旧历史"——**摘要嵌套**,第二次 summary 是对"第一次 summary + 中间消息"的再摘要。

跟 CompressContext 不同——CompressContext 硬砍最旧消息,不会产生"对摘要再摘要"的嵌套。Consolidator 有这个问题,但 LLM 摘要质量还行,嵌套一两层语义损失可控。

### 4.6 失败模式

1. **LLM 调用失败 3 次** → `rawArchive` 兜底(纯文本截断,语义损失大但能塞下)
2. **摘要本身超长** → 受 `MaxTokens 2000` 限制,不会无限长
3. **`maxTokens<=0`**(配置禁用) → `ShouldConsolidate` 永远 false,Consolidator 完全不工作
4. **预算被当前轮吃光** → `findKeepBoundary` 返回 0,全历史送摘要(worked example 第一种)

---

## 五、CompressContext(闸④)框架

位置:`internal/agent/token/compress.go:19`。

### 5.1 做什么

token 超 0.8(16万)时,从最旧 history 开始按 group 砍,砍到 token 降到 0.8 阈值以下。**硬截断兜底**。

### 5.2 触发条件(compress.go:29)

```go
threshold = MaxContextTokens * 0.8   // 默认 16万
currentTokens > threshold → 触发
```

比 Consolidator 的 0.5(10万)晚触发,是 Consolidator 失败或不够用时的兜底。

### 5.3 算法核心:group 不可分 + 保留 system + 当前轮

```go
// compress.go:34-50
systemMsg = messages[0]                       // 保留
lastUserIdx = 找最后一个 role=user 的 index    // 当前轮起点
history = messages[1:lastUserIdx]              // 要被砍的旧历史
tail = messages[lastUserIdx:]                  // 当前轮完整保留

groups = groupToolMessages(history)            // ★分组:assistant(tool_calls)+后续tool 一组

tokensToFree = currentTokens - threshold
freed = 0
removeUpTo = 0
for i, group := range groups {
    freed += group 总 token
    removeUpTo = i + 1
    if freed >= tokensToFree { break }
}

remaining = [systemMsg] + groups[removeUpTo:] + tail
```

**group 不可分**(`groupToolMessages` compress.go:85):assistant 带 tool_calls 的 + 后续连续 tool 结果当一个 group,要么整组留要么整组砍,不拆配对。跟 Consolidator 的 findKeepBoundary 同一个原则。

### 5.4 跟 Consolidator 的层次关系

| 机制 | 触发 | 做法 | 语义保留 | 结构保留 |
|---|---|---|---|---|
| **Consolidator** | 0.5(10万) | LLM 摘要成 system 消息 | ✅ 高(LLM 浓缩) | ❌ 丢(原文消息没了) |
| **CompressContext** | 0.8(16万) | 硬砍最旧的 group | ❌ 零(直接删) | ✅ 保留剩余消息原样 |

正常流程:Consolidator 0.5 触发,摘要后 token 降到 30%,离 0.8 远,CompressContext 不触发。只有极端情况(如当前轮 tool 结果巨大)才两道闸都触发。

### 5.5 留作深挖

`groupToolMessages` 的精确分组规则、跟 Consolidator findKeepBoundary 分组算法的差异、硬截断的失败模式、跟 KnowledgeQA 复用的具体接入点(笔记 27 优化点②)——本篇不展开,留后续单独深挖。

---

## 六、4 道闸的共同特征

### 6.1 全是视图层

4 道闸都只改**发给 LLM 的 messages 数组**,不动 DB 原版。DB 是 single source of truth,每次 LLM 调用前从 DB 载入生成视图,4 道闸在视图上跑。

这跟 Claude Code 的前 4 道压缩(snip/applyToolResultBudget/microcompact/contextCollapse)是视图层的思想一致。但 WeKnora **没有等价于 Claude Code 第 5 道 autocompact(破坏性重置 state.messages)的机制**——因为 WeKnora 的 messages 每次从 DB 重新载入,不存在"换底稿"的概念。

### 6.2 都保留 system + 当前轮

4 道闸都保留:
- system prompt(第一条)
- 当前轮(最后一个 user + 后续所有 assistant/tool)

差别只在"旧历史怎么处理":
- ①trim:不动旧历史,只裁当前轮 tool
- ②redact:旧历史 KB 工具结果涂黑
- ③Consolidator:旧历史 LLM 摘要,留最近 N 条原文
- ④Compress:旧历史硬砍最旧

### 6.3 都遵守 tool_call/tool_result 配对不拆

4 道闸都不破配对:
- ①trim:只改 tool 消息 Content,不删
- ②redact:只改 tool 消息 Content,不删
- ③Consolidator:findKeepBoundary 分组,整组摘要或整组保留
- ④Compress:groupToolMessages 分组,整组留或整组砍

这是 LLM provider(OpenAI 兼容协议)的硬要求——每个 tool_call 必须有对应 tool_result,否则请求被拒。

---

## 七、代码位置速查

### 闸① trimCurrentTurnToolResults
- 主入口:`observe.go:83 trimCurrentTurnToolResults`
- 预算:`observe.go:65 currentTurnToolResultBudget`
- 预算常量:`observe.go:21-23`(min 8K / max 32K / fraction 5)
- marker:`observe.go:145 compactedToolResultMarker`
- 头尾预览:`observe.go:152 compactToolMessage`
- 调用点:`observe.go:31`(manageContextWindow Step 0)

### 闸② redactHistoryKBResults
- 主入口:`observe.go:686 redactHistoryKBResults`
- kbToolNames 集合:`observe.go:679`(附近)
- 开关:`observe.go:715 RetainRetrievalHistory`
- 装配调用:`observe.go:704 buildMessagesWithLLMContext`

### 闸③ Consolidator
- 主入口:`consolidator.go:80 Consolidate`
- 触发判定:`consolidator.go:63 ShouldConsolidate`
- 保留边界:`consolidator.go:158 findKeepBoundary`
- LLM 摘要:`consolidator.go:215 summarizeWithRetry`
- prompt 构造:`consolidator.go:253 buildConsolidationPrompt`
- system prompt:`consolidator.go:326 consolidationSystemPrompt`
- 兜底:`consolidator.go:288 rawArchive`
- 阈值常量:`consolidator.go:21`(0.5) / `consolidator.go:25`(重试3次) / `consolidator.go:28`(超时60s)
- 引擎持有:`engine.go:51 memoryConsolidator` / `engine.go:94 NewConsolidator`
- 调用点:`observe.go:42`(manageContextWindow Step 1)

### 闸④ CompressContext
- 主入口:`token/compress.go:19 CompressContext`
- 分组:`token/compress.go:85 groupToolMessages`
- 阈值常量:`token/compress.go:9 DefaultContextThresholdRatio = 0.8`
- 调用点:`observe.go:55`(manageContextWindow Step 2)

### 总调度
- `observe.go:26 manageContextWindow`:4 道闸串行
- `engine.go:534`:每个 ReAct round 调用前跑 manageContextWindow

---

## 八、深挖补充:4 个边界问题

> 用户研究笔记 28 后追问的 4 个问题,触及 trim 盲区、cache 策略、摘要 prompt、CompressContext 细节。

### 8.1 Q1:trimCurrentTurnToolResults 的 3 个盲区

#### 8.1.1 单个工具有没有限制?——没有,只限总和

`currentTurnToolResultBudget` 只判断**当前轮所有 tool 消息的总 token**,不判断单个:

```go
// observe.go:103-113
total := 0
for i := lastUser + 1; i < len(messages); i++ {
    if messages[i].Role == "tool" {
        toolIndexes = append(toolIndexes, i)
        total += estimator.EstimateMessage(&messages[i])  // ★累加所有 tool
    }
}
if total <= budget || len(toolIndexes) == 0 {
    return messages, false  // 只判断总和
}
```

单个工具结果多大都行,只要总和没超 budget 就一个都不动。

**真实盲区**:budget=32K,调 1 个工具返回 31K(≤32K),不触发裁剪,全量 31K 发出去。单个工具占整个 budget 的 97%,后续推理空间被压到只剩 1K,LLM 几乎没空间思考。

**为什么没单工具限制**(代码无注释,推断):
1. 单工具体积控制应在**工具实现层**(Read 自己有 maxTokens、Bash 自己截 stdout),trim 是"事后兜底"假设工具已做一层控制
2. 加单工具限制触发后只能"压缩这一个"或"丢弃",后者破坏配对,设计者选"宁可单工具大也不破配对"

工具实现层没做好控制时这个假设失效——**真实设计盲区**。

#### 8.1.2 工具多+内容大还是会塞满?——会,渐进式塞满

trim 是**"保底不超 budget,不保证留多少推理空间"**。

极端例子:调 10 个工具各 10K,budget=32K。
- 阶段 A:全 marker,total=500,remaining=31500
- 阶段 B 倒序:
  - T10(最新):extra=9950 ≤ 31500 → 原文,remaining=21550
  - T9:9950 ≤ 21550 → 原文,remaining=11600
  - T8:9950 ≤ 11600 → 原文,remaining=1650
  - T7:9950 > 1650 → 头尾预览(~1650)
  - T6~T1:remaining=0,全 marker

**结果**:T10/T9/T8 原文(各 10K)+ T7 头尾预览(~1.6K)+ T6~T1 全 marker。总 ≈ 32K,**正好塞满 budget**。

**trim 保证"当前轮 tool 总 token ≤ budget",不保证"留推理空间"**。工具多内容大时 trim 会把 budget 用到极致——塞满 32K,留给 LLM 思考的只有上下文剩余的 80%(20万上下文,剩 168K 给 system+历史+当前 query+生成)。

**权衡取舍**:trim 站在"尽可能保留工具结果原文"这边,相信"LLM 宁可看到更多原文也别看到占位符"。工具结果对推理关键时合理;工具结果其实没用时塞满 budget 反而拖累——但 trim 不做语义判断。

#### 8.1.3 头尾预览丢中间?——会丢,设计取舍

`compactToolMessage` 头尾 1:3 预览**确实丢中间**。10000 token 工具结果预算 2000,最终 `[marker]\n\n<前 475 字符>\n...[tool result preview omitted]...\n<后 1425 字符>`,中间 8000 token 全丢。

**为什么这么设计**(observe.go 无注释,从工具结果形态推断):

工具结果信息分布通常不均匀:
- **头部**:命令回显、初始状态、HTTP 响应头、文件路径列表开头——上下文信息,密度低但提供"这是什么"线索
- **尾部**:错误码、最终返回值、状态行、堆栈根因、总结输出——**关键信息密度最高**
- **中间**:重复数据行、长输出主体、中间过程——**密度最低,丢了损失最小**

头尾 1:3 基于这个假设:尾部多留(关键信息)、头部少留(上下文线索)、中间丢(冗余)。

**假设不总对**:工具结果中间有关键数据(如 JSON 中间关键字段)丢了就真丢了。marker 文本明说"Re-run the tool with narrower filters or a smaller range if more detail is needed"——**设计者预期 LLM 看到预览不够会自己重查**。丢中间不是致命问题,前提是工具可重调。

**真正盲区**:工具不可重调时(一次性副作用操作、时间敏感查询)中间丢了永久丢。trim 算法没区分"可重调工具"和"不可重调工具"——**潜在改进点**。

### 8.2 Q2:prompt cache 怎么做的?——★更正:用了但只覆盖 stable prefix,详见第九节

> 本节初稿曾写"Agent 引擎不维护 prompt cache"——**这是错的**。更正如下,完整机制见第九节《stable prefix 机制详解 + Q1/Q2/Q3 三问答》。

#### 8.2.1 ★更正:Agent 引擎用了 prompt cache,但只覆盖 stable prefix

真实情况:
- ✅ Agent 引擎**用了** prompt cache——`think.go:42-43` 每次调 LLM 前算 `PromptPrefixFingerprint` 贴进 ctx
- ✅ 但**只覆盖 stable prefix**(前导 system 消息 + tools schema),不覆盖对话历史
- ❌ 不维护"哪个前缀缓存了什么"的显式状态(不做缓存本身,只做标签和观测)
- ⚠️ Consolidator 生成的 `[Memory Summary]` 是 system 消息,**进了 stable prefix 段** → 指纹变 → **整个 stable prefix cache 全失效**(比初稿说的"summary 字节变"更严重)

engine.go:33 注释 "engine therefore does not maintain its own cache, system-prompt store, or..." 指的是**不维护跨 round 的 in-memory 状态**,不是说不用 prompt cache。这两件事不一样:
- "不用 prompt cache" ❌(其实用了,通过指纹贴标签)
- "不维护跨 round in-memory 状态" ✅(DB-centered,每轮从 DB 重建)

#### 8.2.2 cache 友好的间接机制(对话历史段不保证)

**机制 1:DB 是 single source of truth,每次从 DB 重新载入**

`LoadAgentHistory`(agent_history.go:50)每次 LLM 调用前从 DB 重新读 messages 表。DB 原版永远不变。视图层(trim/redact/Consolidator/Compress)每次基于 DB 原版重新生成——**同样的 DB 状态 + 同样的确定性算法 = 同样的视图**。

**Consolidator 的 cache 漏洞**(对话历史段,不是 stable prefix):Consolidator 调 LLM 摘要,LLM 输出非确定性(Temperature 0.3 也有随机性)。`manageContextWindow` 返回的 messages 只用于本次 LLM 调用(engine.go:534),**不写回 DB**。下一轮再从 DB 载入,再跑 manageContextWindow——重新调 LLM 摘要同一份旧历史,可能生成不同字节的 summary。

**机制 2:工具注册顺序稳定**

registry.go:90 注释:
> iteration order would otherwise reshuffle the tools block and break cache

工具列表每轮发给 LLM(function calling 的 tools 字段),顺序变了 cache 失效。WeKnora 用稳定排序保证工具列表顺序一致。

**机制 3:视图层操作不原地改 + 算法确定性**

trim 拷贝再改(observe.go:115),redact 拷贝再改(observe.go:687),算法本身确定性(同样输入产生同样输出)。同一轮内多次处理同一份 messages 结果一致。

#### 8.2.3 ★更正后跟 claude-code 的差异

| 维度 | claude-code | WeKnora Agent |
|---|---|---|
| state 持久层 | in-memory state.messages | DB messages 表 |
| 视图生成 | 每次基于 state.messages | 每次从 DB 重新载入 |
| stable prefix 指纹 | 显式 ContentReplacementState Map | **有**(`PromptPrefixFingerprint`)但只哈希 system+tools |
| 对话历史 cache 状态 | 显式 seenIds/replacements | **无** |
| 决策冻结 | 必须(保 cache) | **不做**(stable prefix 不需要,对话历史段放弃) |
| 历史非 KB 工具结果 | 压缩 + 冻结决策保住小体积 | **不压缩**(每轮从 DB 读原版全量) |
| 当前轮工具结果 | per-message 预算全压 | 只压总和 ≤32K,单工具可 31K |
| 压缩触发时机 | **晚**(历史保持小体积,token 涨慢) | **早**(历史全量,token 涨快) |
| cache 命中率(百分比) | 历史段命中率本身两者差不多 | miss 部分(当前轮)更大,百分比可能更低 |
| 实际省的算力 | **多**(总 token 小,绝对省算力多) | **少**(历史大,绝对费算力多) |

**更正后结论**:WeKnora 的 prompt cache 策略是**"stable prefix 显式打指纹贴标签(用),对话历史段不显式管理(放弃),靠 DB-centered 架构间接保证"**。不是"不用 cache",是"只用 prefix 段的 cache,不要历史段的"。

**claude-code 冻结决策的真正价值**:不是"让 cache 命中"(WeKnora 靠 DB 原版+算法确定也能命中历史段 cache),而是**"让压缩后的历史保持小体积"**——这是省 token 的关键,也是压缩触发晚的根因。冻结决策保住的是"压缩成果",不是"cache 命中"。

#### 8.2.4 ★更正:压缩和 prompt cache 是两个不同层面

> 用户戳穿表述失误:触发压缩(层面 A:token 超了)和保 prompt cache(层面 B:命中率)是两个不同层面的问题,不该混在一起。

**两个层面拆开**:

| 层面 | 触发条件 | 核心矛盾 | 该不该管 cache |
|---|---|---|---|
| **层面 A:token 超了,要不要压缩** | token > 10万/16万 | 不压任务死,压任务活 | **不该管**,命都保不住了 |
| **层面 B:不压缩时,怎么保 cache 命中** | token 没超阈值 | stable prefix 字节稳定 → cache 命中 | 管的就是 cache |

**正确关系**:
- 层面 A 触发时,层面 B 的 cache 失效是**必然物理后果**(压缩改了 messages 结构,前缀字节必变),不是 stable prefix 机制的隐患
- 触发压缩时**不考虑 prompt cache**——生存危机面前效率让位
- 我前面把"summary 进 prefix 拖垮 cache"当成 stable prefix 机制隐患来讲,是表述失误——这是压缩的必然物理后果,不是隐患

**重新定位"summary 进 prefix 段"**:
- 这是层面 A 的副作用(压缩必然改 messages)
- 不是 stable prefix 机制的问题
- 不需要"把 summary 排出 prefix 段"作为优化(改 `PromptPrefixFingerprint` 性价比低)

**重新定位"Consolidator 摘要落盘"优化方向**:
- 主价值:**省 LLM 调用**(层面 A 成本优化,同 session 多次触发不重算)
- 顺带好处:summary 字节固定,cache 不反复失效
- **不是"保 cache"的优化,是"省 LLM 调用"的优化**——cache 命中率提升只是顺带好处

### 8.3 Q3:Consolidator 摘要 prompt 翻译

#### 8.3.1 user prompt(buildConsolidationPrompt, consolidator.go:253)

**英文原文**:
```
Summarize the following conversation history, preserving:
1. Key facts and decisions made
2. Tool execution results and their outcomes
3. User's original intent and requirements
4. Any errors encountered and how they were resolved

Conversation to summarize:

**User**: <user 内容,截 2000 字符>
**Assistant** [called tools: tool1, tool2]: <assistant 内容,截 1000 字符>
**Tool [tool1]**: <tool 结果,截 1000 字符>
**Tool [tool2]**: <tool 结果,截 1000 字符>
**Assistant**: <assistant 内容,截 2000 字符>
...
```

**中文翻译**:
```
请总结以下对话历史,保留:
1. 关键事实和做出的决策
2. 工具执行结果及其产出
3. 用户的原始意图和需求
4. 遇到的任何错误以及如何解决的

待总结的对话:

**用户**: <用户内容,截 2000 字符>
**助手** [调用了工具: tool1, tool2]: <助手内容,截 1000 字符>
**工具 [tool1]**: <tool1 结果,截 1000 字符>
**工具 [tool2]**: <tool2 结果,截 1000 字符>
**助手**: <助手内容,截 2000 字符>
...
```

**4 个保留点的设计意图**:
1. **关键事实和决策**——避免摘要丢掉"用户选了方案 B"这种关键转折点
2. **工具执行结果**——工具返回的数据是事实依据,不能丢
3. **用户原始意图**——防止多轮后 LLM 忘了用户最初要干啥
4. **错误和修复**——避免 LLM 重复踩坑

#### 8.3.2 system prompt(consolidationSystemPrompt, consolidator.go:326)

**英文原文**:
```
You are a conversation summarizer. Your task is to create a concise but comprehensive summary of a conversation between a user and an AI assistant.

The summary should:
- Be written in the same language as the original conversation
- Preserve all key facts, numbers, and specific details
- Include the outcomes of any tool executions
- Note any errors or issues encountered
- Be structured with clear sections if the conversation covered multiple topics
- Be concise — aim for 30% or less of the original length

Output only the summary, no preamble or explanation.
```

**中文翻译**:
```
你是一个对话总结器。你的任务是创建一份关于用户和 AI 助手之间对话的简明但全面的总结。

总结应该:
- 用与原对话相同的语言编写
- 保留所有关键事实、数字和具体细节
- 包含任何工具执行的产出
- 注明遇到的任何错误或问题
- 如果对话覆盖多个主题,用清晰的章节结构组织
- 要简洁——目标压缩到原长度的 30% 或更少

只输出总结,不要前言或解释。
```

**6 条要求的意图**:
1. **同语言**——中文对话摘要也用中文,避免翻译损失
2. **保留事实/数字/细节**——数字最容易丢,明确强调
3. **工具产出**——跟 user prompt 第 2 点呼应
4. **错误**——跟 user prompt 第 4 点呼应
5. **多主题分章节**——结构化输出,方便后续 LLM 快速定位
6. **30% 以下**——硬压缩目标,配合 MaxTokens 2000 上限

最后"只输出总结,不要前言或解释"——防止 LLM 输出"好的,这是总结:..."这种废话浪费 token。

#### 8.3.3 摘要落盘格式(consolidator.go:134)

```
[Memory Summary - N earlier messages consolidated]

<LLM 生成的 summary 内容>
```

插在 system prompt 之后、保留历史之前。用 system role。N 是被摘要的消息数。

#### 8.3.4 失败兜底 rawArchive(consolidator.go:288)

LLM 失败 3 次后走兜底,每条消息截 500 字符:
```
Raw conversation archive (LLM summarization unavailable):

- User: <截 500 字符>
- Assistant [tools: tool1,tool2]: <截 500 字符>
- Tool[tool1]: <截 500 字符>
- Tool[tool2]: <截 500 字符>
- Assistant: <截 500 字符>
```

机械截断拼接,非 LLM 摘要。语义损失大但保底能塞下。

### 8.4 Q4:CompressContext(闸④)讲透

#### 8.4.1 跟 Consolidator 的关系:先礼后兵

- **Consolidator**(闸③):token > 0.5(10万)触发,LLM 摘要,语义保留好但有成本
- **CompressContext**(闸④):token > 0.8(16万)触发,硬砍,零成本但信息全丢

正常 Consolidator 0.5 触发后 token 降到 0.3,离 0.8 远,CompressContext 不触发。**只有极端情况才两道闸都触发**——Consolidator 摘要后当前轮 tool 结果又暴涨,token 重新涨到 0.8。

CompressContext 是"Consolidator 也救不了"的最后一道防线。

#### 8.4.2 触发条件(compress.go:29)

```go
const DefaultContextThresholdRatio = 0.8   // compress.go:9

threshold = MaxContextTokens * 0.8   // 默认 20万*0.8 = 16万 token
if currentTokens <= threshold { return messages }  // 没超不砍
```

#### 8.4.3 算法(compress.go:34-77,6 步)

**Step 1:切三段**(跟 Consolidator 一样)
```go
systemMsg = messages[0]                       // 保留 system
lastUserIdx = 找最后一个 role=user 的 index    // 当前轮起点
history = messages[1:lastUserIdx]              // ★要被砍的旧历史
tail = messages[lastUserIdx:]                  // ★当前轮完整保留
if len(history) == 0 { return messages }
```

**Step 2:分组**(compress.go:85 groupToolMessages)

```go
groups := groupToolMessages(history)
```

分组规则:
- 遇到 `assistant` 且带 `tool_calls`:这条 + 后续所有连续 `tool` 消息当一个 group
- 其他情况(user、assistant 不带 tool_calls):每条单独一个 group

```go
// 示例
messages = [user, assistant(tool_calls), tool, tool, user, assistant, assistant(tool_calls), tool]
groups = [
  [user],                                    // group 0
  [assistant(tool_calls), tool, tool],       // group 1 ★配对不拆
  [user],                                    // group 2
  [assistant],                               // group 3
  [assistant(tool_calls), tool],             // group 4 ★配对不拆
]
```

**关键**:`assistant(tool_calls)` 和它的 `tool` 结果**永远在同一个 group**,砍的时候整组砍,绝不拆配对。跟 Consolidator 的 findKeepBoundary 分组逻辑**形态一样但用途相反**——Consolidator 用于"保留边界"(从最新往前数保留),Compress 用于"砍除边界"(从最旧开始砍)。

**Step 3:算要释放多少**(compress.go:54)
```go
tokensToFree = currentTokens - threshold   // 超出阈值多少
freed = 0
removeUpTo = 0
```

**Step 4:从最旧开始砍,砍到够为止**(compress.go:58-68)
```go
for i, group := range groups {
    groupTokens = 估算 group 总 token
    freed += groupTokens
    removeUpTo = i + 1
    if freed >= tokensToFree { break }   // ★砍够了就停
}
```

从最旧 group 开始砍(i=0 最旧,history 时间正序),每砍一个 group 累加 freed,直到 freed ≥ tokensToFree 停。`removeUpTo` 记录砍到第几个 group(不含)。

**关键**:只砍"够砍的最旧几个 group",不砍多余的。要释放 5 万 token,最旧 2 个 group 各 3 万,砍这 2 个释放 6 万 ≥ 5 万,停。第 3 个不砍。

**Step 5:组装剩余**(compress.go:70-75)
```go
remaining = [systemMsg]
for i := removeUpTo; i < len(groups); i++ {
    remaining = append(remaining, groups[i]...)   // 砍点之后的 group 全留
}
remaining = append(remaining, tail...)   // 当前轮全留
```

最终:`[system] + [砍点之后的旧历史 groups] + [当前轮 tail]`

**Step 6:返回**(compress.go:77)
```go
return remaining
```

#### 8.4.4 Worked Example

```
[0] system (2000)
[1] user_old1 (500)                              ← group 0
[2] assistant_old1_toolcall (300)                ← group 1 起点带 tool_calls
[3] tool_old1_result (12000)                     ← group 1
[4] user_old2 (500)                              ← group 2
[5] assistant_old2 (700)                         ← group 3
[6] assistant_old3_toolcall (300)                ← group 4 起点带 tool_calls
[7] tool_old3_result (8000)                      ← group 4
[8] user_current (1000)                          ← lastUserIdx = 8
[9] assistant_current_toolcall (400)
[10] tool_current_result (12000)
```

当前 token=37700,假设 MaxContextTokens=40000,threshold=0.8=32000,超了 5700。

**切三段**:systemMsg=[0], history=[1]..[7], tail=[8]..[10]

**分组**:
```
group 0: [1] user_old1 (500)
group 1: [2,3] assistant_old1_toolcall + tool_old1_result (300+12000=12300)  ★配对不拆
group 2: [4] user_old2 (500)
group 3: [5] assistant_old2 (700)
group 4: [6,7] assistant_old3_toolcall + tool_old3_result (300+8000=8300)  ★配对不拆
```

**砍除**:tokensToFree = 37700 - 32000 = 5700
- 砍 group 0(500):freed=500 < 5700,继续
- 砍 group 1(12300):freed=500+12300=12800 ≥ 5700,停!

**removeUpTo = 2**(砍了 group 0 和 group 1)

**组装**:
```
[0] system (2000)
[4] user_old2 (500)                ← group 2
[5] assistant_old2 (700)            ← group 3
[6] assistant_old3_toolcall (300)   ← group 4
[7] tool_old3_result (8000)         ← group 4
[8] user_current (1000)
[9] assistant_current_toolcall (400)
[10] tool_current_result (12000)
```

总 token=24900,从 37700 降到 24900,降 34%。降到 0.62,低于 0.8 阈值。

**关键观察**:
- group 0 和 group 1 整组砍掉,**配对没拆**
- 砍了 12800 token 但只需释放 5700——**多砍了 7100**。这是"按 group 砍"的代价:要砍就得整组砍,不能砍半个 group,所以会多砍一点
- group 1 那 12000 token 的 tool 结果最大,砍它最划算

#### 8.4.5 CompressContext vs Consolidator 精确对比

| 维度 | Consolidator(闸③) | CompressContext(闸④) |
|---|---|---|
| 触发 | 0.5(10万) | 0.8(16万) |
| 处理旧历史 | LLM 摘要成 1 条 system 消息 | **直接删**最旧几个 group |
| 信息保留 | ✅ 高(LLM 浓缩语义) | ❌ 零(消息没了) |
| 结构保留 | ❌ 丢(原文消息没了,变 summary) | ✅ 保留剩余消息原样 |
| 成本 | 1 次 LLM 调用(60s 超时 + 3 次重试) | 零成本(纯内存) |
| 分组算法 | findKeepBoundary(往前扫连续 tool + assistant) | groupToolMessages(往后扫连续 tool + assistant) |
| 分组用途 | 决定"保留边界"(从最新往前数保留 N 条) | 决定"砍除边界"(从最旧开始砍 M 条) |
| 多砍/少砍 | 精确(按 token 预算保留) | 多砍(按 group 砍,整组砍可能超预算) |
| 失败模式 | LLM 失败→rawArchive 兜底 | 不会失败(纯内存) |
| 可逆性 | 摘要后旧历史在 DB 还在 | 砍后旧历史在 DB 还在(视图层) |

#### 8.4.6 两道闸都触发的极端场景

假设:
- 第 N 轮:token=11 万,Consolidator 触发(>10万),摘要后降到 3 万
- 第 N+1 轮:当前轮 tool 结果巨大 15 万,token 涨到 18 万,CompressContext 触发(>16万)

这时 CompressContext 砍最旧 group——但最旧的可能就是上一轮 Consolidator 生成的 `[Memory Summary]` 那条 system 消息。**CompressContext 会把 Consolidator 的摘要都砍掉**。两道闸"互相覆盖"——Consolidator 摘要省下的 token 被 CompressContext 砍了,等于白摘要。但极端场景很少见。

#### 8.4.7 CompressContext 设计哲学

1. **保 system + 当前轮**——无论怎么砍,system prompt 和当前轮完整保留。当前轮 LLM 正在用,砍它破坏推理
2. **保配对**——group 不可分,砍就整组砍
3. **从最旧开始砍**——越老的越先丢(远近原则)
4. **多砍不多留**——按 group 砍会多砍一点,但保证 token 降到阈值以下
5. **零成本兜底**——不调 LLM,纯内存,永远不会失败

**跟 Consolidator 分工**:Consolidator 是"语义压缩"(花 LLM 成本换信息保留),CompressContext 是"机械删除"(零成本换 token 空间)。Consolidator 先上,扛不住时 CompressContext 兜底。

---

## 九、stable prefix 机制详解 + Q1/Q2/Q3 三问答

> 用户研究 8.2 后追问 3 个问题:①能不能引入冻结决策保字节级一致 ②每轮从 DB 读再判断压缩,等于每轮 cache 基本失效?能加缓存考量吗 ③有没有定时/条件触发的摘要机制,最终压缩时复用之前摘要?还是每次压缩重调 LLM。
>
> 答这 3 个问题前先得把 **stable prefix 机制**讲透——因为 WeKnora 的 prompt cache 就是建在这个概念上的,不理解它答不清楚。

### 9.1 stable prefix 是什么——发往 LLM 的消息序列里"前段不变"的那一截

#### 9.1.1 LLM 请求的消息序列长啥样

OpenAI 兼容协议每次调 LLM 发的是一个 messages 数组:

```
[
  {role: "system",    content: "你是一个知识助手..."},         ← stable prefix 段
  {role: "system",    content: "<user_memory>用户偏好</user_memory>"},
  {role: "user",      content: "第1轮问题"},
  {role: "assistant", content: "第1轮答案"},
  {role: "user",      content: "第2轮问题"},                    ← 对话历史段(每轮变)
  {role: "assistant", content: "第2轮答案"},
  {role: "user",      content: "当前轮问题"},                    ← 当前轮段(每轮新)
  ...
]
```

LLM provider(OpenAI/Anthropic/DeepSeek 等)按这个数组逐 token 算。**provider 端有一个 KV cache**:已经算过的 token 不重算,直接复用前向传播的中间状态。这是 prompt cache 的物理基础。

#### 9.1.2 哪些段每轮变,哪些不变

| 段 | 内容 | 每轮变吗 | 能不能命中 provider KV cache |
|---|---|---|---|
| **stable prefix 段** | 前导 system 消息(系统提示 + 用户记忆 + 协议补丁)+ tools schema | **不变**(同 session 同 agent 同 KB 同记忆) | ✅ **能命中**(这是 WeKnora 唯一在做的 cache 收益) |
| **对话历史段** | 历史 user/assistant/tool 消息 + 视图层处理结果 | **变**(新 round 追加新消息 / gate ①②③④ 改写) | ❌ **基本失效** |
| **当前轮段** | 当前 user 输入 + 当前轮 tool 消息 | **每轮新内容** | ❌ **必然 miss** |

**关键**:provider 的 KV cache 是**按前缀字节匹配**的——只要前 N 个 token 字节一致,这 N 个 token 就能命中 cache,后面的全 miss。所以:
- stable prefix 字节一致 → 这段命中 cache
- stable prefix 后面有任何字节变化 → 从变化点开始全 miss

#### 9.1.3 WeKnora 怎么知道"前缀稳定"——`PromptPrefixFingerprint`(prompt_cache.go:28)

```go
func PromptPrefixFingerprint(messages []Message, opts *ChatOptions) string {
    type stablePrefix struct {
        System []Message `json:"system,omitempty"`
        Tools  []Tool    `json:"tools,omitempty"`
    }
    prefix := stablePrefix{}
    for _, message := range messages {
        if message.Role != "system" { break }  // ★只取前导 system,碰到第一个非 system 就停
        prefix.System = append(prefix.System, message)
    }
    if opts != nil { prefix.Tools = opts.Tools }
    data, _ := json.Marshal(prefix)
    return FingerprintPromptPrefix(string(data))  // sha256 取 16 字符
}
```

**3 个关键设计**:

1. **只取前导 system 消息**:`for` 循环碰到第一个 `role != "system"` 就 `break`。**前导 system 之间不能夹非 system 消息**——如果中间夹了一条 user,前导就断在那里。WeKnora 的 messages 装配保证 system 全在开头(engine.go:269 buildSystemPrompt 生成一条 system,buildMessagesWithLLMContext 把它放第一条),所以前导 system 段就是 messages[0] 这一条(可能还有第二条记忆或协议补丁,也是 system role)。

2. **包含 tools schema**:`opts.Tools` 是 function calling 的工具定义(参数 JSON schema)。tools 列表变了(注册顺序变、工具增减)指纹就变。registry.go:90 注释明说 "reshuffle tools block break cache" 就是防这个。

3. **显式不包含对话历史**:注释 `Dynamic conversation/user messages intentionally do not participate` 明说——对话历史**故意不参与指纹**。这意味着指纹不反映对话历史变化,对话历史段 cache 命中率 WeKnora 不主动管。

#### 9.1.4 指纹用在哪——3 个用途

**用途 1:贴标签做 cache 观测**(`think.go:42-43` / `common.go:120-130`)

Agent 路径:
```go
// think.go:42-43
prefixFingerprint := chat.PromptPrefixFingerprint(messages, opts)
llmCtx = types.WithLLMCallMetadata(llmCtx, "agent_round", prefixFingerprint)
```

KnowledgeQA 路径:
```go
// chat_completion.go:55 / chat_completion_stream.go:59
ctx = withPromptCacheMetadata(ctx, chatModel, chatMessages, opt, "knowledge_qa")
// common.go:127 内部调 PromptPrefixFingerprint
```

两路径都把指纹贴进 ctx(`LLMPromptPrefixFingerprintContextKey`),下游 `logUsage`(usage.go:18)和 `langfuse_wrapper.go:28` 取出来打日志:

```
[LLM Usage] model=..., purpose=agent_round, prompt_prefix=<16字符指纹>,
            prompt_tokens=..., cached_tokens=..., cache_read_tokens=..., ...
```

**这是指纹的主要用途——观测 cache 命中率**,不参与缓存本身。运维拿这个指纹聚合相同前缀的调用,看 cache 命中情况。

**用途 2:wiki 写入的 warmup 协调**(`wiki_ingest.go:2483-2620`)

这是 WeKnora **唯一一处把指纹用于缓存协调**的地方(不是 Agent 路径,但用同一套基础设施):

```go
// wiki_ingest.go:2483-2493
prefixFingerprint := chat.PromptPrefixFingerprint(messages, opts)
warmupKey := ""
if promptTpl == agent.WikiPageModifyUserPrompt {
    prefixFingerprint = chat.FingerprintPromptPrefix(
        messages[0].Content, maskedData["SharedSourceContexts"],
    )
    if tenantID, ok := types.TenantIDFromContext(ctx); ok {
        warmupKey = chat.BuildPromptCacheKey(
            tenantID, chatModel.GetModelID(), purpose, prefixFingerprint,
        )
    }
}
```

wiki 页面修改场景,多个并发 LLM 调用共用同一个 prefix(系统提示 + 共享上下文)。`BuildPromptCacheKey` 把 (tenantID, modelID, purpose, prefixFingerprint) 拼成 `"wk-<hash>` 作为进程内协调 key,`awaitWikiPromptWarmup`(L2597)用 `sync.Map` 做"第一个请求先跑 warmup,后续并发请求等它跑完"的扇入协调——**让第一个请求把 prefix 写进 provider KV cache,后续请求直接命中**。

**这是主动利用 prompt cache 的唯一场景**。Agent 路径和 KnowledgeQA 路径**都不做这种协调**,只贴标签观测。

**用途 3:provider 计费统计**(`prompt_cache.go:112 applyRawPromptCacheUsage`)

provider 返回的 usage 里有 cache 命中字段(DeepSeek `prompt_cache_hit_tokens` / Anthropic `cache_read_input_tokens` / OpenAI `cached_tokens`),`applyRawPromptCacheUsage` 按各 provider 协议读出来填进 `TokenUsage`。指纹作为标签跟 usage 关联,运维看哪个前缀命中率高、哪个低。

### 9.2 什么时候触发——3 个调用点

| 路径 | 调用点 | 时机 | purpose 标签 |
|---|---|---|---|
| **Agent** | `think.go:42-43 streamLLMToEventBus` | **每个 ReAct round 调 LLM 前**(engine.go:551 callLLMWithRetry → streamThinkingToEventBus → streamLLMToEventBus) | `"agent_round"` |
| **KnowledgeQA 非流式** | `chat_completion.go:55` | 单次 LLM 调用前 | `"knowledge_qa"` |
| **KnowledgeQA 流式** | `chat_completion_stream.go:59` | 单次 LLM 流式调用前 | `"knowledge_qa"` |
| **Wiki 写入**(主动协调) | `wiki_ingest.go:2482, 2494` | 每次 wiki 页面生成 LLM 调用前 | `"wiki_page_modify"` 等 |

**Agent 路径触发频率**:每个 ReAct round 一次。一个 Agent 任务可能跑 5-20 round,每 round 都重算指纹(因为 manageContextWindow 可能改了 system 段——比如 Consolidator 加了 `[Memory Summary]` system 消息,指纹就变)。

**KnowledgeQA 触发频率**:每个用户问题一次(单轮单次 LLM 调用)。

### 9.3 ★更正:触发压缩和保 prompt cache 是两个不同层面(不是隐患)

> 用户戳穿表述失误:触发压缩(层面 A:token 超了,生存危机)和保 prompt cache(层面 B:命中率,效率问题)是两个不同层面,不该混在一起。

**Consolidator summary 进 system 段这件事**:
- Consolidator(闸③)生成的 `[Memory Summary]` 是 system role(consolidator.go:135),插在原 system 之后(consolidator.go:134-146)
- `PromptPrefixFingerprint` 的 for 循环会把它算进前导 system 段
- 反复触发时 LLM 非确定性 → summary 字节变 → 指纹变 → provider 找不到 cache → 整个 prefix 段 cache 失效

**但这不是 stable prefix 机制的隐患,是压缩的必然物理后果**:

| 层面 | 触发条件 | 核心矛盾 | 该不该管 cache |
|---|---|---|---|
| **层面 A:token 超了,要不要压缩** | token > 10万/16万 | 不压任务死,压任务活 | **不该管**,命都保不住了 |
| **层面 B:不压缩时,怎么保 cache 命中** | token 没超阈值 | stable prefix 字节稳定 → cache 命中 | 管的就是 cache |

- 层面 A 触发时,压缩改了 messages 结构,前缀字节**必然变**,cache **必然失效**——这是物理后果,不是机制问题
- 触发压缩时**不考虑 prompt cache**——生存危机面前效率让位
- 之前把"summary 进 prefix 拖垮 cache"当成 stable prefix 机制隐患来讲,是表述失误

**所以 stable prefix 机制真正管的是层面 B**(正常 round 之间的 cache 命中),不是层面 A。正常 round 里 stable prefix 字节稳定(见 9.1.2 三变量分析),cache 命中率良好,没有隐患。

### 9.4 Q1:能不能引入冻结决策保字节级一致?

**先分清冻结决策是干啥的**:给"会被改的消息"贴标签记"改成啥了",下次按标签复原,保证改后字节多轮一致。**前提是有消息会被改**。

**分两个层级看**:

| 层级 | WeKnora 现状 | 引入冻结决策的可行性 |
|---|---|---|
| **stable prefix 段**(system+tools) | ✅ 装配完就不动,字节天然一致(同一 session 同一 agent 同一 KB 同一记忆下) | **不需要引入**,已最优 |
| **对话历史段**(user/assistant/tool) | DB 原版不变 + 视图层算法确定性 → 字节也稳定 | **不需要引入**,DB-centered 已保证 |

#### 9.4.1 stable prefix 段不需要引入——装配完就不动,冻结决策无效

stable prefix 是**装配**出来的不是**改**出来的:
- `buildSystemPrompt`(engine.go:123)从 YAML 模板渲染 + 拼记忆 + 拼协议 → 一次性装配
- `buildToolsForLLM`(engine.go:280)从 registry 静态读 → 一次性装配
- 装配完后,4 道闸都不动它(trim 只动当前轮 tool / redact 只动历史 KB tool / Consolidator 在 system 之后插 summary 不改原 system / Compress 砍 history 不动 system)

**冻结决策的工作前提是"消息被改"——stable prefix 没被改,冻结决策检测不到"被改"事件,根本不触发**。给它装冻结决策 = 给不会动的零件装定位器,定位器永远不报警,白装。

字节一致性已经由"输入确定 + 算法确定 → 输出确定"**免费保证**,不需要任何额外机制。加冻结决策复杂度上去收益零,负收益。

#### 9.4.2 对话历史段也不需要引入——DB-centered 已保证

WeKnora 历史段字节稳定靠的是 DB-centered:
- DB 是 single source of truth,原版永远不变
- 每轮 `LoadAgentHistory`(agent_history.go:50)从 DB 重新读原版
- 视图层算法确定性:trim 二分搜索+头尾 1:3 / redact 固定占位符 / Consolidator LLM 摘要(★这条非确定性)

**除了 Consolidator 摘要这一条非确定性**,其他都确定性。同一 DB 状态 + 同一确定性算法 = 同一视图字节。

**关键对比**:claude-code 需要冻结决策是因为它的 state 在内存(in-memory state.messages),视图层每次重新生成可能字节不同(如 rawArchive 截断有随机性),所以需要"决策冻结"保字节一致。WeKnora 不需要是因为 DB 原版不变 + 算法确定 → 字节天然一致,**靠架构而不是靠机制**。

#### 9.4.3 ★冻结决策解决不了 summary 字节变的问题

summary 的字节变**不是"被改"的问题,是"源头就变"的问题**——Consolidator 每次调 LLM 摘要,LLM Temperature 0.3 有随机性,同一份旧历史两次摘要字节就不一样。

冻结决策能记"round N 摘要是字节 A",但 **round N+1 再调 LLM 摘要时,LLM 还是会生成字节 B**——冻结决策没法让 LLM 生成跟上次一样的字节。除非把上次的 summary 缓存起来,但那叫"摘要落盘复用"(Q3 方向),不叫"冻结决策"。

**所以 summary 隐患的解法不是冻结决策,是"摘要落盘复用"**——下次不调 LLM,直接读上次的 summary,字节自然一致。

#### 9.4.4 ★为什么对话历史段也不需要冻结决策

**假设方案**(claude-code 式):在 AgentEngine 持有 `historyReplacements Map[messageID]replacement`,trim 把某 tool message 替换成 marker 时不直接改 slice,而是记"messageID → marker"映射;下一轮从 DB 载入原始 message 时按映射重新 apply。

**代价大**:
1. **破坏 engine.go:33 设计原则**——"engine therefore does not maintain its own cache, system-prompt store, or persistent state"。DB-centered 架构基石,改了就是架构层调整
2. **跨 round 状态一致性**——round 1 冻结了 marker,round 5 预算变了 marker 内容"original_bytes=N"也得变,冻结决策要处理"重新冻结"语义,复杂度上升
3. **跨 session 不生效**——Engine 是 per-session 实例,session 结束状态没了

**收益边际**(关键):
- WeKnora 历史段字节稳定靠 DB-centered 已保证(DB 原版不变 + 算法确定性 → 字节稳定)→ 不需要冻结决策也能命中 cache
- 冻结决策的真正价值是"保住压缩成果不让历史膨胀"(claude-code 用法),不是"保 cache 命中"
- WeKnora 的 trim 不压历史非 KB 工具结果(每轮从 DB 读原版全量),没有"压缩成果"要保,冻结决策无处发力

**结论**:Q1 两个层级都不需要引入冻结决策。stable prefix 段不需要(装配完就不动,字节天然一致);对话历史段不需要(DB-centered 已保证字节稳定);summary 字节变的隐患用"摘要落盘复用"解决,不是冻结决策。

### 9.5 Q2:每轮从 DB 读再判断压缩,等于每轮 cache 基本失效?能加缓存考量吗?

#### 9.5.1 先精确化你的理解

"每一轮的缓存基本都是失效的"——**这话只对对话历史段,不对 stable prefix 段**。精确版:

| 段 | 每轮变化吗 | provider cache 命中吗 |
|---|---|---|
| system+tools 前缀 | 不变(同 session 同 KB 同 agent,且没触发 Consolidator) | ✅ **命中** |
| `[Memory Summary]`(Consolidator 触发后) | LLM 非确定性,反复触发时字节变 | ❌ **失效**,且拖垮整个 prefix(见 9.3) |
| 对话历史(非 summary 部分) | 变(新 round 追加 + gate ①②③④ 改写) | ❌ **基本失效** |
| 当前轮 user 新输入 | 每轮新内容 | ❌ **必然 miss** |

**精确说法**:**"每一轮 stable prefix 能命中 cache(前提是没触发 Consolidator),对话历史段基本失效,触发 Consolidator 后整个 prefix 段也失效"**。

#### 9.5.2 能加缓存考量——3 个方向,按可行性排序

**方向 1(最可行,小改):Consolidator 摘要落盘 + 下次复用**

现状:Consolidator 摘要完只在本轮 LLM 调用的 messages slice 里用,不写回 DB(engine.go:534 view-only)。下一轮再触发再调 LLM 摘要同一份旧历史——既费 token 又制造 cache 失效。

改法:messages 表加 `consolidated_summary` 字段(或新表 `session_consolidations`),Consolidator 成功后落盘;下次 LoadAgentHistory 时优先读已摘要版本,不再调 LLM。

收益:
- 省 LLM 调用(同一 session 多次触发 Consolidator 不重算)
- summary 字节固定 → stable prefix 段 cache 命中率提升(解决 9.3 的拖垮问题)
- 防 CompressContext 白摘要(见 9.6.3)

代价:DB schema 改动 + 一致性处理(Agent 历史是 append-only,改了就是新行,旧 summary 仍有效,代价可控)

**这个我推荐,是单点优化不动架构**。

**方向 2(中可行,中改):Consolidator Temperature 调到 0**

现状 Temperature 0.3 破坏字节一致性。改 0 → 同一输入多次摘要字节大概率一致(不保证 100%,LLM 仍可能非确定性)。

收益:不彻底但能提升命中率,改动最小(一行代码)。
代价:摘要质量可能略降。
**可跟方向 1 叠加**。

**方向 3(大改,不建议):引入 in-memory state + 显式冻结决策**

就是 9.4.2 讨论的方案,破坏 DB-centered 原则,大手术,不推荐。

**结论**:Q2 你的理解对(精确化后),加缓存考量**推荐方向 1**(Consolidator 摘要落盘)。**注意:方向 1 的主价值是省 LLM 调用(层面 A 成本优化),cache 命中率提升只是顺带好处**——因为触发压缩时 cache 失效是必然物理后果(9.3 已澄清),不该当成"保 cache"的优化来讲。

### 9.6 Q3:有没有定时/条件触发的摘要机制,最终压缩时复用之前摘要?还是每次压缩重调 LLM?

#### 9.6.1 现状:WeKnora 没有后台/定时摘要机制

检查 `internal/agent/memory/` 和 `internal/agent/token/` 两个目录——**只有 consolidator.go 和 compress.go 两个文件**,没有 claude-code 那种 sessionMemory / 后台 summarizer / 定时任务。

WeKnora 的摘要机制是**纯被动触发**:
- **Consolidator**:token > 0.5(10万)被动触发,本轮调 LLM 摘要,本轮用完丢弃
- **CompressContext**:token > 0.8(16万)被动触发,硬砍不调 LLM

**没有**"定时摘要"、"条件摘要预生成"、"摘要复用"任何机制。

#### 9.6.2 每次压缩都重调 LLM 吗——分两种情况

| 触发 | 是否调 LLM | 是否复用之前摘要 |
|---|---|---|
| Consolidator(token>0.5) | ✅ **每次都重调** | ❌ 不复用(上一轮 summary 没存) |
| CompressContext(token>0.8) | ❌ 不调 LLM(硬砍) | 部分——它砍 message groups,如果某个 group 恰好是 Consolidator 上一轮生成的 `[Memory Summary]`,会被当普通 system 消息处理(可能被砍掉,等于白摘要) |

**关键问题**:Consolidator 反复触发会重新调 LLM 摘要**同一份旧历史**——因为上一轮摘要结果不持久化(不写 DB)。这就是 9.5.2 方向 1 要解决的问题。

#### 9.6.3 能引入"预摘要 + 复用"机制吗——能,正是 Q2 方向 1 的延伸

完整方案:

**A. 摘要落盘**(Q2 方向 1):Consolidator 成功摘要 → 写 `session_consolidations` 表`(session_id, consolidated_until_message_id, summary_text, created_at)`

**B. 下次 Consolidator 触发时先查表**:如果 `consolidated_until_message_id` 已覆盖要摘要的范围,直接读 DB 的 summary_text,不调 LLM

**C. 增量摘要**:上次摘要到 message 50,本次要摘要到 message 80,只对 51-80 调 LLM,summary 拼成"旧 summary + 新增量摘要"

**D. CompressContext 触发时优先保留 summary**:groupToolMessages 分组时给 `[Memory Summary]` 打"高优先级"标记,砍的时候最后才砍它(等于优先复用)

收益:
- 省重复 LLM 调用(同 session 多轮触发 Consolidator 时)
- summary 字节固定 → stable prefix cache 命中率提升
- CompressContext 不再"白摘要"

代价:
- DB schema 改动(新表)
- 一致性处理(虽然 Agent 历史是 append-only,但万一有删除/编辑场景要处理)
- 增量摘要的拼接策略需要设计

**这个方案是 WeKnora 当前架构下能做的最大优化**——不动 DB-centered 原则,只是给"被动触发"机制加一个"持久化层"。

### 9.7 三问串起来看——收敛到"Consolidator 摘要落盘"(主价值是省 LLM 调用,不是保 cache)

> ★更正:按两个层面拆开后,优化方向重新定位。主价值是层面 A(省 LLM 调用),不是层面 B(保 cache)。

Q1/Q2/Q3 收敛到同一个痛点:**Consolidator 重复劳动**——同 session 多次触发时反复调 LLM 摘要同一份旧历史。

- Q1 问"能不能冻结字节" → stable prefix 不需要(装配完就不动)+ 对话历史段不需要(DB-centered 已保证)+ summary 字节变用冻结决策解决不了(源头问题)→ 不需要冻结决策
- Q2 问"能不能加 cache 考量" → 推荐方向 1(摘要落盘),但主价值是省 LLM 调用不是保 cache(触发压缩时 cache 失效是必然物理后果)
- Q3 问"有没有预摘要复用" → 现在没有,可以引入,且正好是 Q2 方向 1 的延伸

**收敛到优化方向**:Consolidator 摘要落盘 + 下次复用 + CompressContext 优先保留 summary。

**收益清单(按价值排序)**:
1. ★**省 LLM 调用**(主价值,层面 A 成本优化)——同 session 多次触发 Consolidator 不重算
2. **防 CompressContext 白摘要**——CompressContext 砍时优先保留 summary,不把上轮摘要砍掉
3. **顺带 cache 命中率提升**(层面 B 副作用收益)——summary 字节固定,触发压缩时 cache 失效面积小一点(但触发压缩时 cache 本来就必然失效,这条收益边际)

**去掉的优化方向**:
- ~~"把 summary 排出 `PromptPrefixFingerprint` 前导 system 段"~~——性价比低,触发压缩占比本来就少,改指纹逻辑有复杂度有风险,顺带收益边际

**单点优化、不动 DB-centered 架构、收益清晰(主省 LLM 调用,次防白摘要)**。

### 9.8 代码位置速查(第九节)

#### stable prefix 基础设施
- `prompt_cache.go:16 FingerprintPromptPrefix`:sha256 取 16 字符
- `prompt_cache.go:28 PromptPrefixFingerprint`:★只哈希前导 system + tools schema
- `prompt_cache.go:49 BuildPromptCacheKey`:"wk-"+hash(tenantID, modelID, purpose, prefixFingerprint)
- `prompt_cache.go:55 providerCacheAccountingStatus`:OpenAI/Azure/DeepSeek/Aliyun/Anthropic 支持
- `prompt_cache.go:112 applyRawPromptCacheUsage`:DeepSeek hit/miss / Anthropic cache_read/creation / OpenAI cached_tokens

#### 3 个调用点
- Agent:`think.go:42-43 streamLLMToEventBus`(每 round 一次,purpose="agent_round")
- KnowledgeQA 非流式:`chat_completion.go:55`(purpose="knowledge_qa")
- KnowledgeQA 流式:`chat_completion_stream.go:59`(purpose="knowledge_qa")
- Wiki 主动协调:`wiki_ingest.go:2482, 2494`(purpose="wiki_page_modify"等)

#### 标签传递
- `context_helpers.go:275 WithLLMCallMetadata`:贴 (purpose, prefixFingerprint) 进 ctx
- `context_helpers.go:287 LLMCallMetadataFromContext`:取出来
- `const.go:79 LLMPromptPrefixFingerprintContextKey`:ctx key

#### 下游用途
- `usage.go:18 logUsage`:打日志带指纹
- `langfuse_wrapper.go:28, 63`:langfuse 观测带指纹
- `wiki_ingest.go:2597 awaitWikiPromptWarmup`:★唯一主动协调场景,sync.Map 扇入
- `wiki_ingest.go:362-364 promptWarmups`:协调 Map

#### Consolidator summary 进 prefix 的隐患
- `consolidator.go:135`(Role: "system"——summary 是 system role)
- `consolidator.go:134-146`(插在 system 之后、保留历史之前)
- `prompt_cache.go:34-39`(for 循环碰到非 system 才 break,会把 summary 算进前导 system)

---

## 十、接续信息(给下次 session 用)

本篇把 4 道闸 + prompt cache 基础设施全部讲透:
- 闸①trimCurrentTurnToolResults:第七节深挖 + 第八节 8.1 补充 3 个盲区(单工具无限制/渐进式塞满/头尾预览丢中间)
- 闸②redactHistoryKBResults:第三节框架
- 闸③Consolidator:第四节深挖 + 第八节 8.3 补充 prompt 翻译
- 闸④CompressContext:第五节框架 + 第八节 8.4 讲透(算法 6 步 + worked example + 跟 Consolidator 对比)
- ★第九节 stable prefix 机制详解 + Q1/Q2/Q3 三问答 + 更正 8.2/8.3 表述

**重要更正(三轮)**:
1. 8.2 初稿"Agent 引擎不维护 prompt cache"是错的——真实是用了(think.go:42-43 算指纹贴标签),只覆盖 stable prefix(前导 system + tools schema),不覆盖对话历史
2. ★**触发压缩(层面 A)和保 prompt cache(层面 B)是两个不同层面的问题**——触发压缩时 cache 失效是必然物理后果,不是 stable prefix 机制隐患。之前把"summary 进 prefix 拖垮 cache"当隐患讲是表述失误
3. ★**"Consolidator 摘要落盘"主价值是省 LLM 调用**(层面 A 成本优化),不是保 cache——cache 命中率提升只是顺带好处
4. ★**冻结决策解决不了 summary 字节变的问题**——summary 字节变是"源头问题"(LLM 非确定性),不是"被改问题"。冻结决策是给"会被改的消息"用的,summary 用"摘要落盘复用"解决
5. ★**claude-code 冻结决策的真正价值是"保住压缩成果不让历史膨胀"**,不是"保 cache 命中"——WeKnora 历史段靠 DB-centered 已保证字节稳定,不需要冻结决策也能命中 cache

**收敛的优化方向**(Q1/Q2/Q3 收敛):Consolidator 摘要落盘 + 下次复用 + CompressContext 优先保留 summary。主收益省 LLM 调用,次防白摘要,顺带 cache 命中率提升。~~把 summary 排出 prefix 段~~ 性价比低去掉。

下次接续方向:
- 深挖闸②redactHistoryKBResults:kbToolNames 集合具体有哪些、占位符精确语义、跟 KnowledgeQA 丢 RenderedContent 的关系、`RetainRetrievalHistory=true` 的使用场景
- 评估笔记 27 优化点②(KnowledgeQA 复用 Agent 压缩闸):看完 4 道闸 + cache 基础设施后,判断那个优化点是真该做还是 KnowledgeQA 根本不需要
- 评估第九节收敛的优化方向(Consolidator 摘要落盘)要不要深挖成具体方案
- Agent 路径其他点:ReAct 循环、工具调度、finalize、approval 等
- 不要主动继续,等用户提问

## 十一、下一步建议

用户已研究完 4 道闸(含 8.1-8.4 四个深挖点)+ stable prefix 机制(第九节 Q1/Q2/Q3 + 两层面拆分),可能下一步:
1. 深挖闸②redactHistoryKBResults(唯一还只有框架的闸)
2. 重新评估笔记 27 优化点②
3. 评估第九节收敛的优化方向(Consolidator 摘要落盘)要不要深挖成具体方案
4. Agent 路径其他点(ReAct 循环、工具调度、finalize 等)
5. 端到端走具体场景
6. 剩余后处理(摘要/FAQ/多模态/答案生成)

不主动继续,等用户提问。