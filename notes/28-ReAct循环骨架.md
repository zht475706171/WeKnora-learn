# 28 · ReAct 循环骨架(executeLoop + runReActIteration)

> 配套:笔记 27 讲了嵌在循环里的 4 道上下文管理闸(trim/redact/Consolidator/Compress,4 道防 token 爆炸的"减压阀"),本篇讲承载这 4 道闸的**主循环骨架**——循环怎么转、一轮 4 步怎么走、3 种退出信号怎么控制流转、4 类异常怎么兜底。
> 本篇不展开 Think 阶段(callLLMWithRetry 重试退避,LLM 调用出错时重试的机制)和 Act 阶段(executeToolCalls 串并行分发,工具怎么并行执行)的内部细节,留后续单独深挖。
> 配套:finalize.go 的 handleMaxIterations(跑满 20 轮还没答案时的兜底)/ streamFinalAnswerToEventBus(把已有工具结果合成最终答案)/ emitCompletionEvent(发完成事件告诉前端"我跑完了")本篇一起讲,它们是循环的退出兜底,跟骨架强耦合。

---

## 一句话

> **AgentEngine.Execute 是入口(用户提问的入口函数),装配好 state(本次执行的临时状态)/systemPrompt(系统提示词,告诉 AI 你是谁)/messages(发往 LLM 的消息数组,含历史+当前问题)/tools(工具清单)后调 executeLoop(主循环)。executeLoop 是 ReAct 主循环(ReAct = Reason+Act,边想边做的循环),for `CurrentRound < MaxIterations`(从第 0 轮转到第 19 轮,20 是默认上限),每轮调 runReActIteration(跑一轮完整流程:想→分析→做→观察)拿一个 iterOutcome(这一轮的退出信号:进下一轮/重跑本轮/退出循环)switch 控制流转。runReActIteration 一轮 4 步:Step 0 manageContextWindow(管 context 窗口,笔记 27 的 4 道闸在这里跑,防 token 爆)→ Step 1 Think callLLMWithRetry(调 LLM 想一下,带重试)→ Step 1.5 stuck loop 检测(防 LLM 卡死循环,连续 2 轮说一模一样的话还不调工具就强停)→ Step 1.6 ctx 取消保 partial step(用户按 stop 时保留半截思考)→ Step 2 Analyze analyzeResponse(分析 LLM 回应:是停了/被内容过滤了/继续调工具)→ Step 3 Act executeToolCalls(执行 LLM 要调的工具)→ Step 4 Observe appendToolResults(把工具结果回灌进 messages,按 OpenAI 协议 assistant 带 tool_calls + tool 带 ToolCallID 配对不拆)。4 类异常路径(ctx 取消 salvage / stuck loop / empty retry / content_filter)各自兜底。循环结束 handleMaxIterations 跑满 20 轮没 natural stop(LLM 自己说"我答完了")时 streamFinalAnswerToEventBus 合成答案,ctx 已取消时不跑(防"用户已停止却冒出答案")。defer emitCompletion 用 WithoutCancel 保证 exactly-one(恰好一次,不多不少)完成事件,ctx 取消时也发。★AgentEngine 是 per-turn 实例(每次提问新建销毁),跨轮状态全在 DB(数据库是唯一真相来源),4 道闸是视图层(只改发给 LLM 的 messages,不动 DB 原版)——这是 WeKnora 的 DB-centered 架构。★跟 claude-code 对比:claude-code 是 in-memory state 架构,state.messages 是内存里的活账本(整个 session 活着,不是每次重建),压缩破坏性改 state.messages(microcompact 把旧 tool_result 换成 "[Old tool result cleared]"),需要冻结决策(ContentReplacementState 的 seenIds/replacements Map)保字节一致保 prompt cache 命中。WeKnora 不需要冻结决策(DB 不变 + 算法确定 → 字节天然一致,靠架构而不是靠机制)。最根本差异是设计哲学:WeKnora 牺牲性能换简单和完整(DB 是主角),claude-code 牺牲复杂度换性能和省 token(内存是主角)。**

---

## 一、AgentEngine 的设计原则:跨轮无状态

`engine.go:28-34` 注释:

> History persistence note: the engine is stateless across turns. Conversation history is rebuilt from the DB once per turn by the caller (see service.LoadAgentHistory) and passed into Execute as llmContext. The engine therefore does not maintain its own cache, system-prompt store, or cross-turn buffer.

**大白话**:AgentEngine 是**一次性用品**——用户每次提问新建一个,跑完销毁。**跨轮状态全在 DB**(数据库是唯一的真相来源),Engine 内存里不存:
- 不存 conversation history(对话历史,每轮从 DB 重新载入 `llmContext`)
- 不存 system-prompt store(系统提示词仓库,每轮重新 `buildSystemPrompt`)
- 不存跨 turn buffer(跨轮缓冲区)

这是笔记 27 第 9.4.2 节讨论"为啥不需要冻结决策"(冻结决策 = 给会被改的消息贴标签记"改成啥了",下次按标签复原,保字节多轮一致)的根因——**靠架构而不是靠机制**保证字节稳定。Engine 是无状态执行器(只跑不管存),所有持久状态委托 DB。

**但 Engine 在单次 Execute 内是有状态的**——`state *types.AgentState` 累积 `RoundSteps`(每轮的思考 + 工具调用记录)、`CurrentRound`(当前第几轮)、`IsComplete`(是否完成)、`FinalAnswer`(最终答案)、`TurnUsage`(本次 turn 的 token 用量)。这些状态只在本次 Execute 存活,Execute 结束随 Engine 一起销毁。

**但 RoundSteps 不会丢**:`emitCompletionEvent` 把 `state.RoundSteps` 写进 `EventAgentComplete` 事件,handler(事件处理者)拿去持久化到 `Message.AgentSteps`(消息表里的步骤字段)。下次用户继续对话时,`LoadAgentHistory`(从 DB 载入历史)读这些 AgentSteps 重建历史。这是单次 Execute 的产出落盘,不是 Engine 跨 turn 状态。

---

## 二、Execute 入口(装配 + 调度)

位置:`engine.go:206-317`。

### 2.1 装配阶段(4 步,大白话:准备好上战场)

```
Execute(ctx, sessionID, messageID, query, llmContext, imageURLs...)
│
├─ 1. defer toolRegistry.Cleanup                            // 工具资源回收,跑完清场
├─ 2. 开 Langfuse top-level span "agent.execute"              // 追踪整个执行(像给这次执行装个行车记录仪)
├─ 3. 初始化 state = &AgentState{RoundSteps:[], CurrentRound:0, IsComplete:false}
│                                                              // 临时状态:0 步、第 0 轮、没完成
├─ 4. 装配(准备 3 样东西):
│    ├─ systemPrompt = buildSystemPrompt(ctx)                 // 系统提示词:"你是 WeKnora 知识助手..."
│    ├─ messages = buildMessagesWithLLMContext(systemPrompt, query, sessionID, llmContext, imgs)
│    │                                                          // ★消息数组:[system, ...历史, 当前问题]
│    │                                                          // ★笔记 27 闸② redact(历史 KB 结果涂黑)在这里跑
│    └─ tools = buildToolsForLLM()                            // 工具清单:knowledge_search/grep_chunks/shell_exec 等
│
└─ 5. executeLoop(ctx, state, query, messages, tools, sessionID, messageID)
     └─ 失败时 Emit EventError + finishAgentSpan(err) + return nil, err
                                                                  // 失败发错误事件 + 关行车记录仪 + 返回错误
```

### 2.2 buildSystemPrompt 的 3 段拼接(engine.go:123)

```go
prompt := BuildSystemPromptWithOptions(knowledgeBasesInfo, WebSearchEnabled, opts, systemPromptTemplate)
return strings.TrimRight(prompt, " \t\r\n") + e.memoryPrompt + e.modelContext.ProtocolPrompt()
```

**3 段**(大白话:3 层夹心):
1. `BuildSystemPromptWithOptions` 渲染 YAML 模板(配置文件,含 KB 信息 / 工具开关 / skills metadata 技能元数据)——主体
2. `e.memoryPrompt` 拼长期记忆信封(用户偏好,像"该用户喜欢简洁回答")——第二层
3. `e.modelContext.ProtocolPrompt()` 拼协议补丁(模型特定指令,不同 LLM 厂家的小补丁)——第三层

**注释里的关键说明**(engine.go:130-133):
> Memory has to ride in the system prompt: buildMessagesWithLLMContext drops system messages coming from history, so a separate memory message would be silently discarded from the second turn onward.

**大白话**:记忆(长期用户偏好)必须塞进 system prompt(系统提示词),不能单独发一条 system 消息。因为 `buildMessagesWithLLMContext`(装配消息数组)装配历史时**丢弃历史里的 system 消息**(observe.go:726 `if msg.Role == "system" { continue }`)——如果记忆单独发一条 system,第二轮装配历史时这条会被丢,LLM 失忆(忘了用户偏好)。塞进 system prompt 才能跨轮保留。

### 2.3 buildMessagesWithLLMContext(observe.go:704)

笔记 27 闸② redact(历史 KB 结果涂黑,防 LLM 用陈旧检索结果)的入口,这里只点一下装配结构:

```go
messages := [{Role: "system", Content: systemPrompt}]   // [0] 永远是 system(系统提示词)
// 历史段(从 llmContext 载入,即从 DB 读出的历史消息)
if len(llmContext) > 0 {
    sanitized = redactHistoryKBResults(llmContext)  // ★笔记 27 闸②:历史里 KB 工具结果换成占位符
    for _, msg := range sanitized {
        if msg.Role == "system" { continue }     // ★丢历史里的 system(防历史 system 污染当前)
        if msg.Role in {user, assistant, tool} { messages = append(messages, msg) }
    }
}
// 当前轮 user(用户这一轮的问题)
messages = append(messages, {Role: "user", Content: RenderUserTurnContent(...), Images: imageURLs})
```

**装配产物**:`[system, ...历史(user/assistant/tool 交替)..., 当前 user]`(大白话:开头是系统提示词,中间是历史对话,结尾是当前问题)。这是发给第一轮 LLM 的初始 messages,后续每轮 runReActIteration 往尾部追加 assistant(带 tool_calls)+tool 消息。

### 2.4 Langfuse span 层级(大白话:追踪的树结构)

```
agent.execute(顶层)              ← 整个执行
└─ agent.round.1                ← 第 1 轮
│   ├─ chat call (think)         ← LLM 调用
│   └─ tool spans                ← 工具执行
└─ agent.round.2                ← 第 2 轮
│   ├─ chat call
│   └─ tool spans
└─ ...
```

Execute 开顶层 span(像顶层的行车记录仪),runReActIteration 每轮开 round span(每轮一个小记录仪)。所有 LLM 调用和工具执行通过 ctx(上下文)自动归到对应 round span 下,Langfuse UI 看到清晰的 trace(追踪)→ agent.execute → agent.round.N → (chat + tools) 树结构。运维查问题时能精确定位"哪一轮的哪个工具慢"。

---

## 三、executeLoop 主循环(顶层控制)

位置:`engine.go:362-444`。这是 ReAct 的"齿轮",一圈一圈转。

### 3.1 主循环骨架(大白话:for 循环 + 3 种退出信号)

```go
emptyRetries := 0              // 空 content 重试计数(上限 2)
consecutiveSameContent := 0    // 连续相同内容计数(stuck loop 检测,上限 2)
lastResponseContent := ""      // 上一轮内容(用于比较是否相同)
loop:
for state.CurrentRound < e.config.MaxIterations {    // ★从 0 转到 19(默认 20 轮上限)
    // Step A: 检查 ctx 取消(用户按 stop)
    select {
    case <-ctx.Done():
        // salvage 路径(尽力抢救):已有 tool 结果时合成最终答案
        if totalTC := countTotalToolCalls(state.RoundSteps); totalTC > 0 {
            _ = e.streamFinalAnswerToEventBus(ctx, query, state, sessionID)
            state.IsComplete = true
        }
        return state, ctx.Err()
    default:
    }

    // Step B: 跑一轮(完整的 think→analyze→act→observe)
    outcome, iterErr := e.runReActIteration(ctx, state, &messages, tools,
        sessionID, messageID, query, &emptyRetries, &consecutiveSameContent, &lastResponseContent)
    if iterErr != nil {
        return state, iterErr    // 出错直接退出
    }

    // Step C: 根据 outcome(这一轮的退出信号)控制流转
    switch outcome {
    case iterOutcomeContinue:
        continue loop    // 重跑本轮,CurrentRound 不 advance(空 content 重试用)
    case iterOutcomeBreak:
        break loop       // 退出循环(自然停/stuck loop/content_filter/final answer)
    case iterOutcomeNext:
        state.CurrentRound++  // advance 到下一轮(正常 ReAct 推进)
    }
}

// Step D: 循环结束兜底(跑满 20 轮没 natural stop)
if !state.IsComplete && ctx.Err() == nil {
    e.handleMaxIterations(ctx, query, state, sessionID)
}
```

### 3.2 3 个跨轮状态(指针传递,大白话:3 个跨轮计数器)

`emptyRetries` / `consecutiveSameContent` / `lastResponseContent` 都是**跨轮共享的可变状态**(上一轮改了,下一轮能看到),通过指针传给 runReActIteration:

| 状态 | 类型 | 作用(大白话) | 重置时机 |
|---|---|---|---|
| `emptyRetries` | int | 空 content 重试计数(LLM 答了空白时重试几次了,上限 2) | 自然停 + 有内容时归零 |
| `consecutiveSameContent` | int | 连续相同内容计数(LLM 连续几轮说一模一样的话,上限 2,防 stuck loop) | 有 tool_call 时归零(engine.go:585) |
| `lastResponseContent` | string | 上一轮的内容(用来比较"这轮跟上轮是不是一字不差") | 有 tool_call 时归零(engine.go:586) |

**为啥用指针不用返回值**:runReActIteration 需要跨轮**累积**这些状态,不是单轮的纯函数(纯函数 = 输入一样输出就一样,没副作用)。用指针让多个 iteration(轮次)共享同一份计数器,上一轮改了下一轮直接看到。

### 3.3 defer emitCompletion 的 exactly-one 保证(engine.go:383-391)

```go
completionEmitted := false
emitCompletion := func() {
    if completionEmitted { return }    // 已发过就不再发
    completionEmitted = true
    e.emitCompletionEvent(context.WithoutCancel(ctx), state, sessionID, messageID, startTime)
}
defer emitCompletion()    // ★所有 return 路径都跑 defer
```

**3 个关键设计**(大白话:保证"完成事件"恰好发一次):

1. **`completionEmitted` bool 防 double emit**(防重发)——多种退出路径(自然停/ctx 取消/iterErr)都可能触发 defer,布尔保证只发一次
2. **`context.WithoutCancel(ctx)`**——ctx 取消时,用"不可取消的 ctx"发事件,防 ctx 已取消导致事件发不出去(像"即使被开除了也要把最后一份报告交出去")
3. **defer 而非每条 return 路径手动调**——所有 return 路径自动触发,不怕漏(像门口的自动关门器,不用手动关)

**为啥要 exactly-one**(恰好一次,不多不少):下游 `agent_stream_handler.handleComplete`(事件处理者)是把 `state.RoundSteps` 写进 `assistantMessage.AgentSteps`(消息表的步骤字段)的入口,漏发会导致 steps 不持久化(前端刷新就丢了),多发会导致重复写。defer+bool 是 Go 实现 exactly-once 的标准模式。

### 3.4 ctx 取消的 salvage 路径(engine.go:398-412)

```go
select {
case <-ctx.Done():
    if totalTC := countTotalToolCalls(state.RoundSteps); totalTC > 0 {
        logger.Infof(ctx, "[Agent] Synthesizing final answer from %d existing tool results", totalTC)
        _ = e.streamFinalAnswerToEventBus(ctx, query, state, sessionID)  // ★用已取消的 ctx
        state.IsComplete = true
    }
    return state, ctx.Err()
default:
}
```

**2 个分支**(大白话:用户按 stop 后,尽力抢救已有产出):
- **已有 tool 结果**(totalTC > 0):调 `streamFinalAnswerToEventBus` 合成答案。注意这里用**已取消的 ctx**——`streamFinalAnswerToEventBus` 内部的 LLM 调用会立即失败,但流式框架可能已经发出了部分内容,用户至少看到"已生成的部分"。注释明说"Try to salvage existing results"(尽力抢救已有结果)
- **没有 tool 结果**:直接 return,LLM 什么都没产出,用户看到空答或前端 stop 提示

**关键**:`state.IsComplete = true` 标记完成,defer emitCompletion 会发完成事件,前端停止 loading(转圈圈)。`ctx.Err()` 返回让上层知道是取消而非正常完成。

### 3.5 3 种 iterOutcome 的精确语义(engine.go:449-458)

```go
type iterOutcome int
const (
    iterOutcomeNext     iterOutcome = iota  // advance CurrentRound, loop again
    iterOutcomeContinue                      // 重跑本轮不 advance(empty retry 用)
    iterOutcomeBreak                         // 退出循环(final answer/stuck/end)
)
```

| outcome | CurrentRound 动作 | 大白话 | 谁返回 |
|---|---|---|---|
| **next** | +1 | 正常一轮结束,进入下一轮 | runReActIteration 末尾(执行了工具,继续 ReAct) |
| **continue** | 不变 | 重跑本轮(不算一轮,因为 LLM 答了空白要重试) | empty retry 重新让 LLM 生成内容 |
| **break** | 不变 | 退出循环(不转了) | 自然停/final answer/stuck loop/content_filter/ctx 取消保 partial |

**iterOutcomeContinue 是特殊设计**(大白话:特殊重试信号):正常 ReAct 每轮 advance(+1),但 empty retry 时**不 advance**——因为这一轮 LLM 没产出内容,不能算"完整一轮",重跑让 LLM 再试。代价是浪费一次 LLM 调用,收益是避免空答。`maxEmptyResponseRetries = 2` 限制重试次数,不会无限浪费。

**为啥不用 iterOutcomeNext + 计数器**:empty retry 重跑时 messages 加了 nudge 消息(提示 LLM "请给完整答案"),CurrentRound 没 advance,下一轮 LLM 看到的 messages 跟重跑前一模一样(只是多了 nudge),语义上还是"同一轮的重试"而不是"新一轮"。iterOutcomeContinue 显式表达这个语义,代码更清晰。

---

## 四、runReActIteration 一轮 4 步(核心)

位置:`engine.go:468-674`。这是 ReAct 的核心实现,think→analyze→act→observe(想→分析→做→观察)一轮的完整代码。

### 4.1 函数签名 + Langfuse round span(大白话:每轮一个追踪器)

```go
func (e *AgentEngine) runReActIteration(
    parentCtx context.Context,
    state *types.AgentState,
    messagesPtr *[]chat.Message,    // ★指针,跨轮共享 messages(消息数组)
    tools []chat.Tool,
    sessionID, assistantMessageID, query string,
    emptyRetries, consecutiveSameContent *int,  // ★指针,跨轮共享 3 个计数器中的 2 个
    lastResponseContent *string,                // ★指针,第 3 个
) (outcome iterOutcome, retErr error) {
    roundStart := time.Now()
    round := state.CurrentRound + 1

    // 开 round span(每轮一个追踪器),defer Finish 在所有 return 路径都跑
    ctx, roundSpan := langfuse.GetManager().StartSpan(parentCtx, ...)
    defer func() { roundSpan.Finish(..., outcome.String(), ...) }()
    ...
}
```

**messagesPtr 是指针**——messages 是跨轮累积的(每轮往尾部追加),用指针让所有 iteration 共享同一份 messages slice。

**defer roundSpan.Finish 在所有 return 路径都跑**——无论本轮是 next/continue/break,span 都正常关闭(追踪器不能忘关)。`outcome` 和 `retErr` 是命名返回值,defer 里能拿到最终值写进 span metadata(追踪数据)。

### 4.2 Step 0:manageContextWindow(笔记 27 的 4 道闸)

```go
currentTokens := e.estimateCurrentTokens(*messagesPtr)  // ★token 估算优化(见 4.3)
beforeLen := len(*messagesPtr)
*messagesPtr = e.manageContextWindow(ctx, *messagesPtr, round, currentTokens)  // ★笔记 27 4 道闸
if len(*messagesPtr) < beforeLen {
    currentTokens = e.tokenEstimator.EstimateMessages(*messagesPtr)  // 压缩后重算
}
```

**4 道闸串行跑**(笔记 27 详细讲过,这里只点一下):
1. 闸① trimCurrentTurnToolResults(当前轮工具结果按 32K 预算裁,装不下的用占位符+头尾预览)
2. 闸② redactHistoryKBResults(历史 KB 结果涂黑,防陈旧)——★其实在 buildMessagesWithLLMContext 阶段就跑过,这里 manageContextWindow 不重跑
3. 闸③ Consolidator(token > 10万时 LLM 摘要旧历史)
4. 闸④ CompressContext(token > 16万时硬砍最旧消息)

**★更正笔记 27 的一个细节**:笔记 27 第 3.5 节说 redact"每任务跑一次"(装配历史时),manageContextWindow 里**不重跑** redact。看 observe.go:29-63 的 manageContextWindow 实现,确实只跑 trim/Consolidator/Compress 三道,不跑 redact。redact 在 buildMessagesWithLLMContext(observe.go:721)跑一次,后续 round 的 messages 都基于这次装配的结果。

### 4.3 estimateCurrentTokens 的 token 估算优化(engine.go:196)

```go
func (e *AgentEngine) estimateCurrentTokens(messages []chat.Message) int {
    if e.lastUsage.TotalTokens > 0 && e.lastSentMsgCount > 0 && e.lastSentMsgCount < len(messages) {
        delta := e.tokenEstimator.EstimateMessages(messages[e.lastSentMsgCount:])
        return e.lastUsage.TotalTokens + delta
    }
    return e.tokenEstimator.EstimateMessages(messages)
}
```

**2 段优化**(大白话:省 BPE 估算开销):
- **有上一轮 API usage 时**:用上一轮的 `TotalTokens`(LLM API 返回的精确 token 数)作基线,只 BPE 估算新增消息的 delta(`messages[lastSentMsgCount:]`,只算上次之后追加的部分),相加。避免每轮全量 BPE 估算(慢,BPE = Byte Pair Encoding,字节对编码,tokenizer 的算法)
- **没有 usage 时**(第一轮或 API 没返回 usage):全量估算

**条件 `lastSentMsgCount < len(messages)`**:只有 messages 增长了才用 delta 模式。如果 messages 被 manageContextWindow 压缩变短了,这条不成立,走全量估算。

**lastSentMsgCount 在哪赋值**(engine.go:550):`e.lastSentMsgCount = len(*messagesPtr)` 在 Step 1 调 LLM 之前赋值,记录"这次发给 LLM 的 messages 长度"。下一轮 estimateCurrentTokens 用这个数判断"从哪开始算 delta"。

**lastUsage 在哪赋值**(engine.go:560-562):`if response.Usage.TotalTokens > 0 { e.lastUsage = response.Usage; state.TurnUsage.Accumulate(response.Usage) }`。LLM 返回 usage 时存进 lastUsage,下一轮 estimateCurrentTokens 用。

### 4.4 Step 1:Think(callLLMWithRetry,大白话:调 LLM 想一下)

```go
e.lastSentMsgCount = len(*messagesPtr)
resp, err := e.callLLMWithRetry(ctx, *messagesPtr, tools, state, query, state.CurrentRound, sessionID)
if err != nil {
    retErr = err
    return iterOutcomeNext, err   // ★err 时返回 next 不是 break,让循环看 ctx 是否取消
}
if resp == nil {
    return iterOutcomeBreak, nil  // ★resp nil 直接 break(可能是 ctx 取消)
}
response = resp

if response.Usage.TotalTokens > 0 {
    e.lastUsage = response.Usage
    state.TurnUsage.Accumulate(response.Usage)    // 累加用量
}
```

**callLLMWithRetry 内部**(think.go:378,下篇深挖):含 transient error(瞬时错误,如 429 限流/500 服务器错)重试 + 超时(`defaultLLMCallTimeout=120s`)+ 退避(失败后等一会儿再试)。本篇不展开。

**2 个特殊返回**:
- **err != nil**:返回 `iterOutcomeNext, err`。注意**不是 break**——executeLoop 拿到 iterErr 直接 `return state, iterErr` 退出循环,不走 switch。但 runReActIteration 这里用 next 而非 break 是**防御性写法**:万一 executeLoop 改成不直接 return 而是 switch,outcome=next 让 CurrentRound advance,不会死循环
- **resp == nil**:break。callLLMWithRetry 可能在 ctx 取消时返回 nil resp + nil err,这里 break 退出循环,defer emitCompletion 跑

**TurnUsage.Accumulate**(累加用量):每轮 LLM usage 累加进 state.TurnUsage,最后 emitCompletionEvent 把整 turn 用量发出去。多个 round + 最终 synthesis 的 usage 全累加,给持久化层做成本统计。

### 4.5 Step 1.5:Stuck loop 检测(engine.go:570-587,大白话:防 LLM 卡死循环)

```go
if len(response.ToolCalls) == 0 && response.Content != "" {
    // 没调工具 + 有内容 → 可能是 stuck loop
    if response.Content == *lastResponseContent {
        *consecutiveSameContent++    // 跟上一轮内容一模一样,计数+1
    } else {
        *consecutiveSameContent = 0  // 跟上一轮不同,归零
    }
    *lastResponseContent = response.Content    // 记下这轮内容,下轮比较用
    if *consecutiveSameContent >= maxRepeatedResponseRounds {  // ★≥2,连续 2 轮相同
        logger.Warnf(ctx, "[Agent][Round-%d] Detected stuck loop: same content repeated %d times (finish=%s), stopping",
            round, *consecutiveSameContent+1, response.FinishReason)
        state.FinalAnswer = response.Content    // ★重复内容当最终答案,不浪费
        state.IsComplete = true
        return iterOutcomeBreak, nil
    }
} else {
    // 有 tool_call 或没内容 → 重置
    *consecutiveSameContent = 0
    *lastResponseContent = ""
}
```

**触发条件**(3 个 AND,大白话:LLM 连续 2 轮"说一模一样的话还不调工具"):
1. `len(response.ToolCalls) == 0`——没调工具
2. `response.Content != ""`——有内容(不是空答)
3. `response.Content == *lastResponseContent`——跟上一轮内容完全相同

连续 2 次满足这 3 条 → stuck loop(卡死循环),强停。

**为啥要这个检测**(const.go:42-46 注释):
> maxRepeatedResponseRounds catches stuck loops caused by unhandled finish reasons (e.g., content_filter not caught elsewhere).

content_filter(内容过滤,模型安全策略拦截)在 analyzeResponse 里有专门处理(下面 Step 2 讲),但万一有别的 finish reason(LLM 停止原因)没被 isNaturalStopFinishReason(判断是否自然停的函数)识别,LLM 反复返回相同内容不调工具,循环会空转 20 轮才停。stuck loop 检测提前终止,省 18 轮 LLM 调用。

**重置逻辑**:有 tool_call 时 `consecutiveSameContent = 0; lastResponseContent = ""`。LLM 调工具说明在推进(查新东西),不算 stuck。

**强停时的处理**:`state.FinalAnswer = response.Content`——把重复的内容当最终答案,不浪费已有的 LLM 产出(像"虽然卡了,但好歹有句话给用户")。`state.IsComplete = true`,defer emitCompletion 发完成事件。

### 4.6 Step 1.6:ctx 取消保 partial step(engine.go:598-615,大白话:用户按 stop 时保留半截思考)

```go
if ctx.Err() != nil {
    logger.Warnf(ctx, "[Agent][Round-%d] Context cancelled during LLM call; preserving partial step", round)
    if step.Thought != "" || len(step.ToolCalls) > 0 {
        state.RoundSteps = append(state.RoundSteps, step)    // ★半截思考写进步骤
    }
    return iterOutcomeBreak, nil
}
```

**场景**(大白话:LLM 流式输出过程中用户按 stop):stream driver(流式驱动)返回 partial content(半截内容)+ finish_reason="stop" + 无 tool calls。**不**让 analyzeResponse 把 partial thinking(半截思考)当最终答案(会污染 Message.Content,显示成"半截思考")。

**做法**:
1. 保留 partial 内容作为 `step`(Thought + ToolCalls)
2. 把 step 追加进 `state.RoundSteps`——defer emitCompletion 会把它写进 `assistantMessage.AgentSteps`,前端能看到"这一轮思考到一半被停了"
3. `return iterOutcomeBreak, nil` 退出循环,**不**设 `state.IsComplete = true`

**不设 IsComplete 的原因**:executeLoop 的 Step D `if !state.IsComplete && ctx.Err() == nil` 会跑 handleMaxIterations,但 `ctx.Err() != nil` 所以也不跑。最终走 ctx 取消 salvage 路径(如果有 tool 结果)或直接退出。前端看到的是"用户主动停止",不是"AI 给了答案"。

**对比 Step 1 err 路径**:err 时返回 next 让 executeLoop 退出;ctx 取消保 partial 时返回 break。两种路径都退出循环,但语义不同——err 是"出错退出",ctx 取消是"用户主动停止但保留已生成的思考"。

### 4.7 Step 2:Analyze(analyzeResponse,大白话:分析 LLM 回应:停了/被过滤了/继续调工具)

位置:`observe.go:211-322`。判断 LLM 是否自然停止。

```go
verdict := e.analyzeResponse(ctx, response, step, state.CurrentRound, sessionID, roundStart)
if verdict.isDone {
    // Guard against empty content(防空答)
    if verdict.emptyContent {
        *emptyRetries++
        if *emptyRetries <= maxEmptyResponseRetries {  // ★≤2 重试
            *messagesPtr = append(*messagesPtr, chat.Message{
                Role:    "user",
                Content: "Please provide your complete answer now as plain text.",    // ★nudge 消息(提示 LLM 给完整答案)
            })
            return iterOutcomeContinue, nil  // ★重跑本轮不 advance
        }
        // 重试耗尽用 fallback
        state.FinalAnswer = "I'm sorry, I was unable to generate a response. Please try again."
        state.IsComplete = true
        state.RoundSteps = append(state.RoundSteps, verdict.step)
        return iterOutcomeBreak, nil
    }
    // 正常自然停
    state.FinalAnswer = verdict.finalAnswer
    state.IsComplete = true
    state.RoundSteps = append(state.RoundSteps, verdict.step)
    return iterOutcomeBreak, nil
}
// 非 done,继续 Step 3
```

**analyzeResponse 的 3 种 verdict**(判断结果):

| 情况 | 触发条件 | verdict | 处理 |
|---|---|---|---|
| **content_filter**(内容过滤) | `FinishReason=="content_filter" && len(ToolCalls)==0` | isDone=true, finalAnswer=内容或 fallback | 直接 break |
| **natural stop**(自然停) | `isNaturalStopFinishReason(reason) && len(ToolCalls)==0` | isDone=true, finalAnswer=内容, emptyContent=(内容=="") | empty 时重试,否则 break |
| **继续 ReAct** | 有 tool_calls 或非自然停 | isDone=false | 走 Step 3 |

**isNaturalStopFinishReason**(observe.go:194,判断是否自然停):`stop` / `end_turn` / `stop_sequence` 三种 finish reason(LLM 停止原因)视为自然停。不同 LLM 厂家用不同字段,这里统一识别。

**content_filter 处理**(observe.go:218-256,大白话:被模型安全策略拦了):
- 发 EventAgentFinalAnswer 事件(内容或 fallback 文本)
- 直接返回 isDone=true,不走重试
- 跟 natural stop 的区别:content_filter 是被模型安全策略拦(像"这话不能说"),natural stop 是 LLM 主动停(话说完了)。content_filter 直接给 fallback,natural stop 走 emptyContent 判断

**emptyContent 重试**(engine.go:623-641,大白话:LLM 答了空白时重试):
- `emptyRetries <= maxEmptyResponseRetries`(≤2)时:追加 nudge 消息(提示 LLM "请给完整答案")"Please provide your complete answer now as plain text.",返回 `iterOutcomeContinue` 重跑本轮
- 重试耗尽:用 fallback 文本 "I'm sorry, I was unable to generate a response. Please try again.",设 IsComplete,break

**const.go:35-40 注释解释为啥要重试**:
> maxEmptyResponseRetries guards against the agent completing with an empty answer when the LLM fails to produce content (e.g., thinking-only loops without KB). Trade-off: each retry costs ~2s of LLM latency; 2 retries = max 4s extra.

**thinking-only loop**(只生成思考不生成答案):LLM 只生成 thinking 内容(ReasoningContent,推理过程)没有 Content(最终回答),finish_reason="stop"。没有重试的话直接空答,用户看到空白。重试加 nudge 让 LLM 生成 Content。代价最多 4 秒延迟,收益是避免空答。

**为啥 nudge 是 user 消息**:LLM 看到这条 user 消息后会生成 Content 作为 assistant 回复。追加成 user 消息符合对话协议,user 消息明示"现在请给完整答案"。

### 4.8 Step 3:Act(executeToolCalls,大白话:执行 LLM 要调的工具)

```go
e.executeToolCalls(ctx, response, &step, state.CurrentRound, sessionID, assistantMessageID)
toolCallCount = len(step.ToolCalls)
```

**executeToolCalls 内部**(act.go:217,下篇深挖):串并行分发(`ParallelToolCalls` 配置控制,是否允许并行调多工具)、单工具执行、tool span 追踪(每个工具一个追踪器)、参数脱敏(敏感参数不进日志)、错误处理。本篇不展开。

**step.ToolCalls 被填**:`executeToolCalls` 把每个 tool call 的执行结果填进 `step.ToolCalls[i].Result`。step 是指针,直接改。

### 4.9 Step 4:Observe(appendToolResults,大白话:把工具结果回灌进 messages)

```go
state.RoundSteps = append(state.RoundSteps, step)
*messagesPtr = e.appendToolResults(*messagesPtr, step)
```

**appendToolResults**(observe.go:602-658):把本轮的 thought(LLM 思考)+ tool calls(工具调用)+ tool results(工具结果)按 OpenAI 兼容格式回灌 messages。

**关键:OpenAI tool-calling 格式要求配对不拆**(大白话:assistant 说"我要调工具 X" 必须跟着 tool 消息说"工具 X 返回了啥")——笔记 27 第 2.5 节讲过,OpenAI 兼容协议要求每个 tool_call 必须有对应 tool_result。appendToolResults 的实现严格遵守:

```go
// 1. 先追加 assistant 消息(含 tool_calls,LLM 说"我要调这些工具")
if step.Thought != "" || len(step.ToolCalls) > 0 || step.ReasoningContent != "" {
    assistantMsg := chat.Message{
        Role:             "assistant",
        Content:          step.Thought,           // LLM 的思考
        ReasoningContent: step.ReasoningContent,  // 推理过程(DeepSeek 等模型有)
    }
    if len(step.ToolCalls) > 0 {
        assistantMsg.ToolCalls = ...  // ★tool_calls 填这里(LLM 要调哪些工具)
    }
    messages = append(messages, assistantMsg)
}

// 2. 再追加每个 tool 结果(role=tool,带 ToolCallID,说"工具 X 返回了啥")
for _, toolCall := range step.ToolCalls {
    resultContent := e.modelContext.ModelToolResultForTool(toolCall.Name, toolCall.Result)
    toolMsg := chat.Message{
        Role:       "tool",
        Content:    resultContent,
        ToolCallID: toolCall.ID,   // ★配对关键(靠 ID 匹配"哪个 tool_call 对应哪个 tool_result")
        Name:       toolCall.Name,
    }
    messages = append(messages, toolMsg)
}
```

**配对机制**:assistant 消息的 `ToolCalls[i].ID`(工具调用 ID)跟 tool 消息的 `ToolCallID`(工具调用 ID)对应。LLM provider(LLM 服务方)靠 ID 匹配"哪个 tool_call 对应哪个 tool_result"。

**★stepContainsMarkdownImage 的特殊处理**(observe.go:606-611,大白话:工具结果含图片时追加"图片要求"):
```go
if stepContainsMarkdownImage(step) {
    messages = appendAgentRetrievedImageRequirement(messages)
}
```

如果 tool 结果包含 markdown 图片(`![](url)` 格式),追加一条"图片要求"消息(强制 LLM 在最终答案里包含图片)。这条在 assistant + tool 消息之前追加。注释说"Keep the requirement at system priority even when a custom Agent prompt replaces the built-in template"——防止自定义 system prompt 覆盖默认图片要求。

### 4.10 返回 iterOutcomeNext

```go
return iterOutcomeNext, nil
```

本轮执行了工具,ReAct 继续。executeLoop switch 拿到 next → `state.CurrentRound++` → 下一轮。

---

## 五、4 类异常路径深挖

### 5.1 ctx 取消 salvage(executeLoop + runReActIteration,大白话:用户按 stop 后尽量抢救)

**2 个 ctx 取消检查点**:

| 位置 | 时机 | 处理 |
|---|---|---|
| **executeLoop 循环头**(engine.go:399) | 每轮开始前 | select `<-ctx.Done()`:有 tool 结果时 streamFinalAnswerToEventBus 合成,无结果直接 return |
| **runReActIteration Step 1.6**(engine.go:608) | LLM 调用后 | `ctx.Err() != nil`:保 partial step,break |

**为什么 2 个检查点**:
- 循环头检查:用户在两轮之间按 stop,本轮还没开始就退出
- Step 1.6 检查:LLM 流式输出过程中用户按 stop,partial 内容保留

**salvage 的设计哲学**(大白话:用户主动停止时,尽量保留已有产出):
- 有 tool 结果 → 合成最终答案(用户至少看到"基于已有检索的总结")
- 无 tool 结果 → 空答(用户看到"还没检索就停了")
- partial thinking(半截思考) → 写进 AgentSteps(用户看到"思考到一半被停了")

**不悄悄给答案**:ctx 取消保 partial 时**不设 IsComplete=true**,handleMaxIterations 也不跑(ctx.Err() != nil),用户看到的是"已停止"不是"AI 给了答案"。

### 5.2 Stuck loop 检测(Step 1.5,大白话:防 LLM 卡死循环)

见 4.5。**触发条件**:连续 2 轮相同内容无 tool_call。**目的**:防 unhandled finish reason(没被识别的停止原因)导致空转 20 轮。**强停处理**:重复内容当 final answer,不浪费 LLM 产出。

### 5.3 Empty retry(Step 2 emptyContent 分支,大白话:LLM 答空白时重试)

见 4.7。**触发条件**:natural stop(自然停)+ empty content(空白内容)。**重试**:最多 2 次,追加 nudge user 消息(提示"请给完整答案"),iterOutcomeContinue 重跑本轮不 advance。**耗尽**:fallback 文本 + IsComplete + break。

### 5.4 content_filter(analyzeResponse Case 0,大白话:被内容过滤拦了)

见 4.7。**触发条件**:`FinishReason=="content_filter" && len(ToolCalls)==0`。**处理**:直接 isDone=true,内容或 fallback 文本作 final answer,break。**不重试**:content_filter 是模型安全策略,重试大概率还是被拦。

**4 类异常路径的关系**:

| 异常 | 触发 | 是否重试 | 是否保产出 | 退出方式 |
|---|---|---|---|---|
| ctx 取消 | 用户操作 | 否 | 是(salvage/partial) | return ctx.Err() / break |
| stuck loop | 连续 2 轮相同 | 否 | 是(重复内容当答案) | break |
| empty retry | natural stop + empty | 是(≤2 次) | 重试成功保内容,失败用 fallback | continue / break |
| content_filter | finish_reason | 否 | 是(内容或 fallback) | break |

**共同点**:都尽量保留 LLM 已产出,都避免无限循环,都不悄悄给"AI 主动答案"(ctx 取消除外,因为用户主动停)。

---

## 六、循环结束后的兜底(handleMaxIterations)

位置:`engine.go:439-441` + `finalize.go:166-184`。

```go
// executeLoop 末尾
if !state.IsComplete && ctx.Err() == nil {
    e.handleMaxIterations(ctx, query, state, sessionID)
}
```

**2 个条件 AND**(大白话:跑满 20 轮 + ctx 没取消):
1. `!state.IsComplete`——循环跑完 20 轮没 natural stop(LLM 自己说"我答完了")
2. `ctx.Err() == nil`——ctx 没取消(用户没按 stop)

**为什么 ctx 取消时不跑**:注释明说(engine.go:434-438):
> skip this when the context was cancelled (user pressed stop). In that case the fallback LLM call would fail on the already-cancelled ctx and set state.FinalAnswer to the generic "Sorry, I was unable to generate a complete answer." message, which then leaks to the UI as the final answer for a conversation the user deliberately stopped.

**大白话**:ctx 取消时跑 handleMaxIterations → streamFinalAnswerToEventBus 用已取消的 ctx 调 LLM → LLM 调用失败 → fallback 文本 "Sorry, I was unable to generate a complete answer." → 这个文本被当 final answer 泄露到 UI,用户看到"AI 给了答案"——但用户明明是主动停止的,这个文本会让用户困惑(像"我都喊停了你还在答")。

**handleMaxIterations 内部**(finalize.go:166):
```go
func (e *AgentEngine) handleMaxIterations(ctx, query, state, sessionID) {
    logger.Info(ctx, "Reached max iterations, generating final answer")
    if err := e.streamFinalAnswerToEventBus(ctx, query, state, sessionID); err != nil {
        state.FinalAnswer = "Sorry, I was unable to generate a complete answer."
    }
    state.IsComplete = true    // ★无论合成成功失败,循环已结束,必须标完成
}
```

### 6.1 streamFinalAnswerToEventBus(finalize.go:27,大白话:把已有工具结果合成最终答案)

**场景**:循环跑满 20 轮没 natural stop,但累积了 N 轮 tool 结果。用一个独立 LLM 调用把这些结果合成成最终答案(像"查了 20 轮还没答完,最后用一个总结性 LLM 调用把所有结果汇总给用户")。

**messages 装配**(finalize.go:47-93):
```go
messages := []chat.Message{
    {Role: "system", Content: systemPrompt},
    {Role: "user", Content: RenderUserTurnContent(sessionID, query)},
}
// 把所有 tool 结果作为 user 消息追加(注意:不是 tool 消息,是 user 消息)
for stepIdx, step := range state.RoundSteps {
    for toolIdx, toolCall := range step.ToolCalls {
        modelOutput := e.modelContext.ModelToolResultForTool(toolCall.Name, toolCall.Result)
        messages = append(messages, chat.Message{
            Role:    "user",
            Content: fmt.Sprintf("Tool %s returned: %s", toolCall.Name, modelOutput),
        })
    }
}
// 追加 final prompt(最后一条 user 消息,明示"基于上面工具结果生成答案")
finalPrompt := fmt.Sprintf(`Based on the above tool call results, generate a complete answer for the user's question.

User question: %s

Requirements:
1. Answer based on the actually retrieved content
2. Organize the answer in a structured format
3. If information is insufficient, honestly state so
4. IMPORTANT: Respond in the same language as the user's question
%s

Now generate the final answer:`, query, imageRequirement)
messages = append(messages, chat.Message{Role: "user", Content: finalPrompt})
```

**关键设计**:
- **tool 结果作为 user 消息**(不是 tool 消息)——这是独立 LLM 调用,不是 ReAct 循环里的一轮,不需要遵守 OpenAI tool-calling 配对协议(像"这是给一个新 LLM 看的总结任务,不是继续 ReAct")。用 user 消息承载 tool 结果更直接
- **final prompt 明确要求**:基于实际检索内容 + 结构化 + 信息不足要诚实 + 同语言 + 图片要求(如果有)
- **imageRequirement**(finalize.go:17):如果 tool 结果含 markdown 图片,追加"答案必须包含至少一张图片"的硬要求

**LLM 调用参数**:`Temperature: e.config.Temperature`(默认 0.7,温度参数控制随机性),**Thinking disabled**(注释:finalize.go:103 "Thinking disabled for final answer synthesis")。合成答案是事实性总结,不需要 thinking 发散(像"总结事实别瞎发挥")。

**流式输出**:通过 `streamLLMToEventBus`(下篇深挖)流式发 EventAgentFinalAnswer 事件,前端实时看到答案生成(像打字机一样一字一字出来)。

**StripThinkBlocks 兜底**(finalize.go:154,剥掉 ` IMDetached` 块):即使 LLM 返回的内容里有 ` IMDetached` 块(DeepSeek/Qwen 等模型会嵌入"思考过程"块),也剥掉。防 thinking 内容泄露到最终答案。

**TurnUsage 累加**(finalize.go:149-151):synthesis 调用的 usage 也累加进 state.TurnUsage。注释说"The synthesis call is often the largest of the turn"——合成答案往往是整 turn 最大的一次 LLM 调用(输入所有 tool 结果),用量必须计入(像"最后一次总结虽然不是 ReAct 一轮,但用量算整次任务")。

### 6.2 跟 ctx 取消 salvage 的关系

ctx 取消 salvage 也调 streamFinalAnswerToEventBus,但**用的是已取消的 ctx**(engine.go:407 `_ = e.streamFinalAnswerToEventBus(ctx, ...)`)。这次调用大概率失败,但流式框架可能已经发出了部分内容,用户看到"已生成的部分"。

**两个调用点的区别**:

| 调用点 | ctx 状态 | 目的 | 失败处理 |
|---|---|---|---|
| **executeLoop salvage**(engine.go:407) | 已取消 | 尽量输出已生成内容 | 忽略 err,IsComplete=true |
| **handleMaxIterations**(finalize.go:176) | 正常 | 合成最终答案 | fallback 文本,IsComplete=true |

**共同点**:都设 IsComplete=true,都通过 streamFinalAnswerToEventBus 流式输出。

### 6.3 emitCompletionEvent(finalize.go:187,大白话:发完成事件告诉前端"我跑完了")

位置:executeLoop defer 里调,所有退出路径都跑。

```go
func (e *AgentEngine) emitCompletionEvent(ctx, state, sessionID, messageID, startTime) {
    knowledgeRefsInterface := []interface{}{...}
    e.eventBus.Emit(ctx, event.Event{
        Type: event.EventAgentComplete,
        Data: event.AgentCompleteData{
            FinalAnswer:     state.FinalAnswer,        // 最终答案
            KnowledgeRefs:   knowledgeRefsInterface,  // 引用的知识
            AgentSteps:      state.RoundSteps,        // ★步骤详情(每轮的思考+工具调用)
            Usage:           turnUsage(state),        // token 用量
            TotalSteps:      len(state.RoundSteps),   // 总步数
            TotalDurationMs: time.Since(startTime).Milliseconds(),  // 总耗时
            MessageID:       messageID,               // 消息 ID(用于持久化)
        },
    })
}
```

**AgentSteps 是关键**(大白话:步骤详情是历史的来源):state.RoundSteps(每轮的 thought + tool calls)通过事件传给下游 handler(事件处理者),handler 写进 `assistantMessage.AgentSteps`(消息表的步骤字段)持久化。下次用户继续对话时,LoadAgentHistory(从 DB 载入历史)从 DB 读这些 AgentSteps 重建历史(笔记 26 讲过)。

**turnUsage**(finalize.go:217):state.TurnUsage.TotalTokens > 0 时返回用量,否则返回 nil。防 0 用量字段泄露到事件(像"没用量就别发用量字段")。

**WithoutCancel ctx**:executeLoop 用 `context.WithoutCancel(ctx)` 调 emitCompletionEvent,ctx 取消时事件也能发出去(见 3.3,像"即使被开除了也要把最后一份报告交出去")。

---

## 七、跟笔记 27 的衔接

### 7.1 4 道闸嵌在哪

| 闸 | 嵌入点 | 触发频率 |
|---|---|---|
| 闸② redactHistoryKBResults(历史 KB 结果涂黑) | buildMessagesWithLLMContext(observe.go:721) | 每任务跑一次(装配历史时) |
| 闸① trimCurrentTurnToolResults(当前轮工具结果裁剪) | manageContextWindow(observe.go:31) | 每 round 跑一次(Step 0) |
| 闸③ Consolidator(LLM 摘要旧历史) | manageContextWindow(observe.go:45) | 每 round 判断,token > 10万时触发 |
| 闸④ CompressContext(硬砍最旧消息) | manageContextWindow(observe.go:55) | 每 round 判断,token > 16万时触发 |

**注意**:redact 不在 manageContextWindow 里,在装配阶段跑一次。笔记 27 第 3.5 节讲过这个区别——redact 处理历史(固定的,历史不会变),trim/Consolidator/Compress 处理当前轮(每轮有新工具结果,需要每轮跑)。

### 7.2 appendToolResults 的输出是下一轮 trim 的输入(大白话:循环关系)

```
本轮 Step 4: appendToolResults → messages 追加 assistant(tool_calls)+tool 结果
  ↓ (下一轮看到这些新消息)
下一轮 Step 0: manageContextWindow
  ↓
  闸① trim 看到这些新 tool 消息,判断当前轮 tool 总 token 是否超 32K
  ↓
  超了就裁(占位符+倒序补全+头尾预览,笔记 27 闸①)
```

**循环关系**:appendToolResults 往 messages 尾部追加 → 下一轮 trim 检查这些追加的内容 → trim 后的 messages 发给 LLM → LLM 可能再调工具 → appendToolResults 再追加 → ...

### 7.3 token 估算优化跟 4 道闸的关系

estimateCurrentTokens 用 lastUsage + delta 优化(4.3 节),这个估算值传给 manageContextWindow。4 道闸的触发判断(Consolidator ShouldConsolidate / CompressContext 阈值)用这个估算值。优化避免每轮全量 BPE 估算,降低 Engine CPU 开销。

**潜在盲区**:delta 只估算新增消息,如果 manageContextWindow 上一轮压缩了 messages(变短了),lastSentMsgCount 可能指向已不存在的位置。4.3 节的条件 `lastSentMsgCount < len(messages)` 处理这个——messages 变短时走全量估算,不用 delta。

---

## 八、跟 KnowledgeQA 的对比(大白话:单轮单次 vs 多轮 ReAct)

| 维度 | KnowledgeQA(知识问答) | Agent(ReAct 智能体) |
|---|---|---|
| **循环** | 单轮单次(无循环,一次检索一次生成) | for 循环,MaxIterations=20(最多 20 轮) |
| **LLM 调用** | 1 次(检索后生成) | N 次(每轮 1 次 + 可能的 synthesis) |
| **工具调用** | 代码控制检索(无 function calling) | LLM 自主调工具(ReAct) |
| **上下文管理** | 无 4 道闸(笔记 26 讲过) | 4 道闸(笔记 27) |
| **历史结构** | 一轮 = user + assistant 两条 | 一轮 = user + assistant(tool_calls) + tool 多条 |
| **异常路径** | 无(单轮没异常路径可走) | 4 类(ctx 取消/stuck loop/empty retry/content_filter) |
| **终态判断** | 生成完就 done | natural stop / content_filter / stuck / max iterations / ctx 取消 |
| **Token 估算** | 不需要(单轮) | lastUsage + delta 优化 |
| **完成事件** | 直接生成完发 | defer emitCompletion exactly-one |

**核心差异**:KnowledgeQA 是"一次性 RAG"(检索增强生成,查一次答一次),Agent 是"多轮 ReAct"(边想边做,查一下看结果再查一下)。多轮带来 3 个复杂度:
1. **上下文管理**(4 道闸)——防 token 爆(消息越攒越多)
2. **异常路径**(4 类)——防循环异常(卡死/空白/被拦/用户停)
3. **终态判断**(多种)——啥时候算完(自然停/被拦/卡死/跑满/用户停)

**KnowledgeQA 没这些复杂度**:单轮单次,token 不会爆,不会有循环异常,生成完就 done。但牺牲了"多步推理"能力——只能基于一次检索的内容生成,不能像 Agent 那样"查一下→看结果→再查一下"。

---

## 九、跟 claude-code 的对比(in-memory state vs DB-centered)

> 本节回答一个核心问题:WeKnora 的 AgentEngine 说"跨轮无状态,记忆全在 DB",这跟 claude-code 的做法差异在哪?不是"存哪里"的差异,是整个设计哲学的差异。
> claude-code 源码位置:`D:/GoProject/claude-code`。

### 9.1 先纠正一个理解

"跨轮记忆用 DB 代替内存"——**对,但只对了一半**。WeKnora 确实用 DB 代替了 claude-code 的内存 state 做跨轮记忆,但**关键差距不在"存哪里",而在"什么时候存、什么时候读、什么时候压缩、压缩改什么"**。下面拆开讲。

### 9.2 WeKnora 的做法:DB-centered(数据库中心化)

**AgentEngine 是 per-turn 实例**(每次用户提问新建一个,跑完销毁):

```
用户第 1 次提问 → 新建 AgentEngine → 跑完 → 销毁
用户第 2 次提问 → 新建 AgentEngine → 跑完 → 销毁
用户第 3 次提问 → 新建 AgentEngine → 跑完 → 销毁
```

Engine 内存里**啥都不存**:不存对话历史 / 不存系统提示词仓库 / 不存跨 turn 缓冲区。

**跨 turn 记忆全在 DB**:

```
用户第 1 次提问:
  Engine 从 DB 读历史 → DB 是空的 → 装配 messages = [system, 当前问题]
  → 跑完 → 把 AgentSteps(每轮的思考+工具调用)写进 DB 的 messages 表
  → Engine 销毁

用户第 2 次提问:
  新 Engine 从 DB 读历史 → LoadAgentHistory(笔记 26)→ 重建 messages
  → messages = [system, ...历史(从 DB 读), 当前问题]
  → 跑完 → 把这次 AgentSteps 追加写进 DB
  → Engine 销毁
```

**大白话**:WeKnora 把 DB 当成"硬盘上的记忆本",每次提问前翻开本子看历史,答完把这次记录追加进本子,合上本子。下次提问再翻开。

**4 道闸(笔记 27)是"视图层"**:trim/redact/Consolidator/Compress 只改发给 LLM 的 messages 数组,**不动 DB 原版**:

```
DB 原版:[全部历史消息,完整内容]    ← 永远不变
   ↓ LoadAgentHistory 读出来
视图层:[system, 历史被 4 道闸处理过的版本, 当前问题]   ← 发给 LLM
   ↓ LLM 回答完
新消息追加写进 DB    ← DB 原版继续增长
```

**大白话**:DB 是"完整账本",视图是"给 LLM 看的摘要版"。4 道闸在视图上裁裁剪剪,但账本上一字不改。

### 9.3 claude-code 的做法:in-memory state(内存状态)

**state.messages 是"内存里的活账本"**(REPL.tsx:1182):

```tsx
const [messages, rawSetMessages] = useState<MessageType[]>(initialMessages ?? []);
const messagesRef = useRef(messages);
```

**关键**:claude-code 的 `state.messages` 是 **React useState 维护的内存数组**(活账本,在内存里实时改),整个 session(会话)期间一直活着,**不是每次提问重建**:

```
用户第 1 次提问:
  state.messages = [] (空)
  → 跑一轮 → state.messages.push(assistant 消息) → state.messages.push(tool 消息) → ...
  → state.messages 在内存里越攒越长

用户第 2 次提问:
  state.messages 还是上次那个数组(没销毁)
  → state.messages.push(新 user 消息)
  → 跑一轮 → 继续往 state.messages 尾部 push
  → state.messages 越来越长
```

**大白话**:claude-code 把整个对话历史放在**内存里的一个数组**(`state.messages`),整个 session 期间这个数组一直活着,越攒越长。不像 WeKnora 每次提问重建 Engine。

**DB(JSONL 文件)只做"持久化备份"**(防崩溃恢复):

```
state.messages(内存) ← 主角,LLM 看的就是这个
   ↓ appendEntry(异步写盘,sessionStorage.ts:1128)
session.jsonl 文件 ← 备份,用于:
   - 进程崩溃后恢复(resumeConversation)
   - /rewind 回退
   - /branch 分支
```

`appendEntry` 是异步写盘,不阻塞主流程。LLM 看的是内存里的 `state.messages`,不是从 DB 读。

**对比 WeKnora**:WeKnora 的 DB 是主角,LLM 看的就是从 DB 读出来的。claude-code 的 DB(文件)是备份,LLM 看的是内存数组。

### 9.4 claude-code 的压缩是"破坏性改 state.messages"

这是**最关键的差距**。claude-code 的压缩(microcompact/autocompact)**直接改 state.messages**,不是视图层:

```
state.messages = [user1, assistant1(tool), tool1_result(10KB), user2, assistant2(tool), tool2_result(8KB), ...]
   ↓ microcompact 跑(把旧 tool_result 换成短预览)
state.messages = [user1, assistant1(tool), "[Old tool result cleared]", user2, assistant2(tool), tool2_result(8KB), ...]
   ↑ 内存数组被改了!tool1_result 永久变成占位符
```

`TIME_BASED_MC_CLEARED_MESSAGE = '[Old tool result cleared]'`(microCompact.ts:36)——这就是 claude-code 把旧 tool_result 替换成的占位符。

**大白话**:claude-code 是**在活账本上直接划掉旧内容**,划掉就没了。下次 LLM 调用看到的就是划掉后的版本。

### 9.5 为啥 claude-code 需要"冻结决策"

因为压缩是**破坏性改 state.messages**,带来一个问题:**同一份 tool_result 这次压缩成预览 A,下次压缩成预览 B,字节不一样,prompt cache(提示词缓存)就失效了**。

claude-code 的解法——**冻结决策**(toolResultStorage.ts:390 `ContentReplacementState`):

```ts
type ContentReplacementState = {
  seenIds: Set<string>           // 见过的 tool_result ID(命运已定)
  replacements: Map<string, string>  // ID → 上次替换成的预览字符串
}
```

**机制**:
- 第一次看到 tool_result_1 → 压缩成预览 A → 记 `replacements.set("tool_result_1", "预览 A")`
- 下次再看到 tool_result_1 → 直接 `replacements.get("tool_result_1")` 拿"预览 A" → **字节一模一样**

注释明说(toolResultStorage.ts:375-380):
> Once seen, a result's fate is frozen for the conversation.
> Re-application is a Map lookup — no file I/O, guaranteed byte-identical, cannot fail.

**大白话**:claude-code 给每个 tool_result 贴个标签"这个已经被压成 XXX 了",下次遇到同一个 tool_result 直接拿标签上的内容,不重新压缩。保证字节一模一样,prompt cache 命中。

### 9.6 为啥 WeKnora 不需要冻结决策(笔记 27 第 9.4.2 节)

- WeKnora 的 DB 原版永远不变
- 每轮从 DB 读原版 → 跑确定性算法(同样输入同样输出)→ 视图字节自然一致
- **靠架构(DB 不变 + 算法确定)保证字节稳定,不需要冻结决策**

**除 Consolidator 摘要这一条非确定性**(LLM 摘要 Temperature 0.3 有随机性),其他都确定性。同一 DB 状态 + 同一确定性算法 = 同一视图字节。

**关键对比**:claude-code 需要冻结决策是因为 state 在内存,视图层每次重新生成可能字节不同(如 microcompact 截断有随机性),需要"决策冻结"保字节一致。WeKnora 不需要是因为 DB 原版不变 + 算法确定 → 字节天然一致,**靠架构而不是靠机制**。

### 9.7 核心差异汇总表

| 维度 | WeKnora(DB-centered) | claude-code(in-memory state) |
|---|---|---|
| **主角(LLM 看什么)** | DB(数据库 messages 表) | state.messages(内存数组) |
| **Engine 生命周期** | per-turn(每次提问新建销毁) | per-session(整个会话活着) |
| **历史存哪** | DB messages 表 | 内存 state.messages + JSONL 文件备份 |
| **每轮历史怎么来** | 从 DB 重新读(`LoadAgentHistory`) | 内存数组尾部 push(一直累积) |
| **DB 角色** | 主角(真相来源) | 备份(防崩溃恢复) |
| **压缩改什么** | 视图(不动 DB 原版) | state.messages 本身(破坏性) |
| **压缩后历史还能恢复吗** | 能(DB 原版还在) | 不能(内存里被划掉了,只有 JSONL 备份有) |
| **字节稳定靠啥** | DB 不变 + 算法确定 → 自然一致 | 冻结决策(seenIds/replacements Map) |
| **prompt cache 命中靠啥** | DB-centered 间接保证 | 冻结决策显式保证 |
| **压缩触发早晚** | 早(历史全量,每轮从 DB 读全量) | 晚(历史保持小体积,压缩后不涨) |
| **历史非 KB 工具结果** | 不压缩(每轮从 DB 读原版全量) | 压缩 + 冻结保小体积 |

### 9.8 最直白的比喻

**WeKnora = "每次提问翻账本"**:

像会计每次有人来问账,**翻开账本(DB)从头看**,看完合上。下次再问,**重新翻开**。账本上完整记录永远在,给老板看的时候临时做个摘要(视图层 4 道闸),但账本一字不改。

- 优点:账本完整,随时能查原始记录;架构简单,不需要冻结决策
- 缺点:每次翻账本要时间(DB 读 + 重新跑 4 道闸);历史大了 token 涨得快(因为每轮从 DB 读全量,历史非 KB 工具结果不压缩)

**claude-code = "活账本上直接划"**:

像会计桌上摊着一本**活账本**(state.messages),每笔账实时写上去。账本太满的时候,**直接在旧账上划掉**(microcompact 把旧 tool_result 换成"[Old tool result cleared]"),划掉的就没了。为了防止同一笔账这次划成 A 下次划成 B,**给每笔账贴个标签记"已经划成啥了"**(冻结决策),下次直接照着划。

- 优点:不用每次重翻,内存里直接拿;压缩后历史保持小体积(划掉了不涨),压缩触发晚
- 缺点:划掉就没了(内存里),要恢复得翻 JSONL 备份;需要冻结决策这套额外机制保字节一致

### 9.9 设计哲学的差异(最根本的差距)

**WeKnora:DB-centered 架构**(数据库是主角)——**牺牲性能换简单和完整**:
- DB 是唯一真相,视图层随便改,改坏了下次从 DB 重读就行
- 不需要冻结决策(架构保字节稳定)
- 但每轮从 DB 读全量历史,压缩触发早,历史非 KB 工具结果不压缩(每轮全量发)

**claude-code:in-memory state 架构**(内存状态是主角)——**牺牲复杂度换性能和省 token**:
- 内存里直接改,快,不需要每次重读
- 但需要冻结决策这套机制兜底保字节一致
- 压缩后历史保持小体积,压缩触发晚,省 token

**不仅是"存哪里"的差异,是整个系统设计哲学的差异**:
- WeKnora 的 4 道闸做成视图层(不动 DB 原版),正是因为 DB 是主角——视图层随便改,改错了下次从 DB 重读
- claude-code 的压缩做成破坏性(直接改 state.messages),正是因为内存是主角——没有"DB 原版"可恢复,改了就是改了,所以需要冻结决策保"改的字节多轮一致"

### 9.10 代码位置速查(claude-code 侧)

- state.messages 内存数组:`REPL.tsx:1182`(`useState<MessageType[]>`)
- state.messagesRef(实时同步):`REPL.tsx:1183`(`useRef(messages)`)
- setMessages 包装(保 ref 实时):`REPL.tsx:1198`(Zustand 模式,ref 是真相,React state 是渲染投影)
- query loop state(type State):`query.ts:204`(`messages: Message[]` 是 State 字段之一)
- appendEntry 异步写盘:`sessionStorage.ts:1128`
- materializeSessionFile(首次写盘触发):`sessionStorage.ts:976`
- pendingEntries(缓冲到首次消息才写盘):`sessionStorage.ts:552`
- ContentReplacementState(冻结决策):`toolResultStorage.ts:390`(seenIds Set + replacements Map)
- cloneContentReplacementState(cache-sharing fork 克隆):`toolResultStorage.ts:405`
- provisionContentReplacementState(新会话初始化):`toolResultStorage.ts:447`
- TIME_BASED_MC_CLEARED_MESSAGE(占位符):`microCompact.ts:36`("[Old tool result cleared]")
- COMPACTABLE_TOOLS(可压缩工具集合):`microCompact.ts:41`(File/Shell/Grep/Glob/WebFetch/WebSearch 等)
- SessionMemory(后台 markdown 笔记):`services/SessionMemory/sessionMemory.ts`(forked subagent 定期提取)

---

## 十、代码位置速查

### 主循环
- Execute 入口:`engine.go:206`
- executeLoop:`engine.go:362`
- runReActIteration:`engine.go:468`
- iterOutcome 类型(3 种退出信号):`engine.go:449-458`
- 3 个跨轮状态(emptyRetries 等):`engine.go:393-395`

### 4 步
- Step 0 manageContextWindow 调用:`engine.go:534`
- Step 0 manageContextWindow 实现:`observe.go:29`
- Step 0 estimateCurrentTokens(token 估算优化):`engine.go:196`
- Step 1 callLLMWithRetry 调用:`engine.go:551`
- Step 1 callLLMWithRetry 实现:`think.go:378`(下篇深挖)
- Step 1.5 stuck loop 检测(防卡死):`engine.go:570-587`
- Step 1.6 ctx 取消保 partial(用户停时保留半截):`engine.go:598-615`
- Step 2 analyzeResponse 调用:`engine.go:618`
- Step 2 analyzeResponse 实现:`observe.go:211`
- Step 2 emptyContent 重试(空答重试):`engine.go:623-641`
- Step 3 executeToolCalls 调用:`engine.go:660`
- Step 3 executeToolCalls 实现:`act.go:217`(下篇深挖)
- Step 4 appendToolResults 调用:`engine.go:665`
- Step 4 appendToolResults 实现:`observe.go:602`

### 退出兜底
- handleMaxIterations 调用:`engine.go:440`
- handleMaxIterations 实现:`finalize.go:166`
- streamFinalAnswerToEventBus(合成最终答案):`finalize.go:27`
- emitCompletionEvent(发完成事件):`finalize.go:187`
- turnUsage(用量统计):`finalize.go:217`
- defer emitCompletion(保证恰好一次):`engine.go:383-391`

### 异常路径
- ctx 取消 salvage(executeLoop,尽力抢救):`engine.go:398-412`
- ctx 取消保 partial(runReActIteration,保留半截):`engine.go:598-615`
- stuck loop(防卡死):`engine.go:570-587`(maxRepeatedResponseRounds=2)
- empty retry(空答重试):`engine.go:623-641`(maxEmptyResponseRetries=2)
- content_filter(内容过滤):`observe.go:218-256`

### 常量
- DefaultAgentMaxIterations=20(默认 20 轮上限):`const.go:15`
- defaultLLMCallTimeout=120s(LLM 调用超时):`const.go:22`
- maxLLMRetries=2(LLM 重试上限):`const.go:33`
- maxEmptyResponseRetries=2(空答重试上限):`const.go:40`
- maxRepeatedResponseRounds=2(stuck loop 阈值):`const.go:46`
- isNaturalStopFinishReason(判断自然停:stop/end_turn/stop_sequence):`observe.go:194`

### 装配
- buildSystemPrompt:`engine.go:123`
- buildMessagesWithLLMContext:`observe.go:704`
- buildToolsForLLM:engine.go:280 附近
- redactHistoryKBResults 入口(笔记 27 闸②):`observe.go:721`

---

## 十一、接续信息(给下次 session 用)

本篇把 ReAct 循环骨架讲透:
- Execute 入口装配(state/systemPrompt/messages/tools,4 样东西准备上战场)
- executeLoop 主循环(for + 3 种退出信号 + defer emitCompletion 保证恰好一次 + ctx 取消 salvage 尽力抢救)
- runReActIteration 一轮 4 步(Step 0 4 道闸 + Step 1 Think + Step 1.5 stuck 检测 + Step 1.6 ctx 取消保 partial + Step 2 Analyze + Step 3 Act + Step 4 Observe)
- 3 种 iterOutcome 精确语义(next 进下一轮/continue 重跑本轮/break 退出)
- 4 类异常路径(ctx 取消/stuck loop/empty retry/content_filter)
- 循环结束兜底(handleMaxIterations 跑满 20 轮 + streamFinalAnswerToEventBus 合成 + emitCompletionEvent 发完成事件)
- 跟笔记 27 4 道闸的衔接
- 跟 KnowledgeQA 的对比(单轮单次 vs 多轮 ReAct)
- ★跟 claude-code 的对比(DB-centered vs in-memory state,设计哲学差异):WeKnora 每次提问翻账本(DB 是主角,4 道闸是视图层不动 DB 原版,不需要冻结决策);claude-code 活账本上直接划(内存 state.messages 是主角,压缩破坏性改 state.messages,需要冻结决策 ContentReplacementState 保字节一致)。最根本差异是设计哲学:WeKnora 牺牲性能换简单和完整,claude-code 牺牲复杂度换性能和省 token

**没展开的点**(留后续):
- **Think 阶段**:callLLMWithRetry 重试退避(transient error 识别 / 超时 / 退避策略)、streamLLMToEventBus 流式输出机制、streamThinkingToEventBus thinking 分流
- **Act 阶段**:executeToolCalls 串并行分发(ParallelToolCalls 配置)、executeSingleToolCall 单工具执行、tool span 追踪、参数脱敏、formatToolHint 提示词注入
- **Approval**:MCP 工具的人审闸(Redis pub/sub 跨实例审批、OAuth 授权、pending 等待)
- **Agent 路径其他**:image_requirement.go(图片要求)、skills(Progressive Disclosure 渐进式披露)、modelContext(协议补丁)

下次接续方向:
- Think 阶段(callLLMWithRetry + streamLLMToEventBus)深挖
- Act 阶段(executeToolCalls + executeSingleToolCall)深挖
- Approval(MCP 审批闸)深挖
- 不要主动继续,等用户提问