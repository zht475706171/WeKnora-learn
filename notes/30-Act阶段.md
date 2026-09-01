# 笔记 30 · Act 阶段(executeToolCalls 串并行分发)

> 接笔记 29(Think 阶段)。笔记 29 讲的是模型怎么"想",本篇讲模型"想完要动手"那一步:
> `runReActIteration` 的 Step 3(Act 阶段)——拿到模型返回的 `tool_calls`(工具调用列表),怎么真正执行这些工具。
>
> 风格延续:专业术语 + 白话解释双轨。

---

## 一句话总结

**Act 阶段 = `executeToolCalls` 顶门立户**(外层入口,决定串行还是并行),
**`executeToolCallsParallel` / `executeSingleToolCall` 二选一**(并行用 errgroup,串行用 for 循环),
**`runToolCall` 干实事**(单个工具的完整执行流程:解析参数 → 修 JSON → 发工具提示 → 开 langfuse span → 设超时 → 调 registry → 收结果)。
**`formatToolHint` 管前端展示**(把工具调用变成"搜索网页("xxx")"这种白话提示给用户看)。

---

## 一、四个函数的定位

| 函数 | 定位 | 干什么 |
|---|---|---|
| `executeToolCalls` | 外层入口(act.go:217) | 看 `ParallelToolCalls` 开关和工具数,决定走并行还是串行 |
| `executeToolCallsParallel` | 并行分支(act.go:242) | 用 errgroup 并发跑所有工具,结果按原序收集 |
| `executeSingleToolCall` | 串行分支(act.go:310) | for 循环依次跑每个工具 |
| `runToolCall` | 核心实现(act.go:356) | 单个工具的完整执行流程,串并行都用它 |
| `formatToolHint` | UI 提示(act.go:194) | 把工具调用变成人能看懂的提示 |

**白话比喻**:
- `executeToolCalls` 像调度员(看任务多少,决定一个个干还是一起干)
- `executeToolCallsParallel` / `executeSingleToolCall` 像两种施工模式(并行工程 vs 串行流水线)
- `runToolCall` 像单个工人(从接任务到交差的完整流程)
- `formatToolHint` 像翻译(把技术活翻译成用户能看懂的"在搜网页")

---

## 二、外层 `executeToolCalls`(act.go:217-238)

### 2.1 干什么的

Act 阶段的入口,`runReActIteration` Step 3 直接调它。
拿到模型返回的 `response.ToolCalls`(工具调用列表),决定怎么执行。

### 2.2 分支逻辑

```
executeToolCalls(ctx, response, step, iteration, sessionID, assistantMessageID):
  if len(response.ToolCalls) == 0:
    return  ← 没工具调用,直接返回(理论上 Analyze 阶段已过滤,这是兜底)

  round = iteration + 1
  n = len(response.ToolCalls)

  if e.config.ParallelToolCalls && n >= 2:
    executeToolCallsParallel(...)  ← 并行分支
    return

  for i, tc := range response.ToolCalls:
    executeSingleToolCall(...)  ← 串行分支,一个个跑
```

**两个条件决定并行**:
1. `ParallelToolCalls` 配置开(笔记 29 讲过,默认开)
2. 工具数 ≥ 2(只有 1 个工具没必要并行,直接串行)

**白话**:配置开了且有 2 个以上工具,一起跑(并行);否则一个个跑(串行)。

---

## 三、并行分支 `executeToolCallsParallel`(act.go:242-307)

### 3.1 干什么的

用 `errgroup`(Go 的并发库,带错误管理的 goroutine 组)并发跑所有工具调用,
结果按原序收集到 `results` 数组,最后统一推事件。

### 3.2 关键代码

```go
results := make([]types.ToolCall, n)
var mu sync.Mutex
g, gCtx := errgroup.WithContext(ctx)

for i, tc := range response.ToolCalls {
    i, tc := i, tc  // 捕获循环变量(Go 闭包经典坑)
    g.Go(func() error {
        toolCall := e.runToolCall(gCtx, tc, i, iteration, round, sessionID, assistantMessageID)
        mu.Lock()
        results[i] = toolCall  ← 按原序写入,不是按完成顺序
        mu.Unlock()
        return nil  ← best-effort:不在失败时取消兄弟任务
    })
}

_ = g.Wait()  ← 等所有跑完
```

### 3.3 三个关键设计

**① `i, tc := i, tc` 捕获循环变量**:
- Go 1.22 之前闭包捕获循环变量会共享同一个变量(经典坑),这里显式拷贝
- 保证每个 goroutine 拿到的是自己的 i 和 tc,不会串号

**② `results[i] = toolCall` 按原序写入**:
- 并行执行完成顺序可能乱(先发起的后完成),但结果按原序存
- 后续推事件时按原序推,前端看到工具结果顺序跟模型发起的顺序一致

**③ `return nil` 不取消兄弟任务**:
- `errgroup` 默认行为:任一 goroutine 返回 error,取消所有兄弟
- 这里 `return nil`(永远返回 nil):某个工具失败不影响其他工具
- **白话**:工具之间是独立的(查知识库 A 和查知识库 B 互不影响),一个失败不该连累其他

### 3.4 结果统一推送

`g.Wait()` 后,所有工具结果都在 `results` 数组里,按原序遍历,每个推两个事件:
- `EventAgentToolResult`:工具结果(给前端展示工具产出)
- `EventAgentTool`:工具执行(给前端展示执行过程,含输入输出和耗时)

**白话**:并行跑完后,按模型发起的顺序,一个一个把工具结果告诉前端。

---

## 四、串行分支 `executeSingleToolCall`(act.go:310-352)

### 4.1 干什么的

串行执行,for 循环里每个工具调一次 `runToolCall`,立刻推事件,然后跑下一个。

### 4.2 跟并行的区别

| 维度 | 串行 | 并行 |
|---|---|---|
| 调用方式 | for 循环依次 `runToolCall` | errgroup 并发 `runToolCall` |
| 推事件时机 | 每个工具跑完立刻推 | 全部跑完后按原序统一推 |
| 前端体验 | 工具 A 完成 → 工具 B 开始 → 工具 B 完成 | 几个工具同时"正在执行",同时完成 |
| 总耗时 | 所有工具耗时之和 | 最慢工具的耗时(理论) |

**白话**:串行像排队结账(一个一个来),并行像多个收银台同时开(一起办)。

---

## 五、核心 `runToolCall`(act.go:356-553)——单个工具完整流程

这是 Act 阶段最核心的函数,串行和并行都调它。13 步流程:

### 5.1 步骤总览

```
runToolCall(ctx, tc, i, iteration, round, sessionID, assistantMessageID):
  Step 1:  NormalizeToolCallID  ← 规范化工具调用 ID
  Step 2:  json.Unmarshal 解析参数
  Step 3:  解析失败 → RepairJSON 修复 → 再解析 → 再失败报错返回
  Step 4:  formatToolHint 生成 UI 提示
  Step 5:  发 EventAgentToolCall 事件(告诉前端"要调工具了")
  Step 6:  PipelineInfo 日志(tool_call_start)
  Step 7:  buildToolSpanInput + 开 langfuse span(可观测性追踪)
  Step 8:  取 principal(调用者身份)+ 设 execTimeout(执行超时)
  Step 9:  装工具执行上下文 ToolExecContext(SessionID/EventBus/ToolCallID/UserID/ApprovalCtx/ExecTimeout)
  Step 10: 查 UnresolvedHandles(未解析的模型句柄)→ 有就拒绝执行
  Step 11: context.WithTimeout 起超时 ctx → toolRegistry.ExecuteTool 真正执行 → cancel
  Step 12: 装结果 ToolCall{ID/Name/Args/Result/Duration/ProviderMetadata}
  Step 13: finishToolSpan 关 langfuse span + PipelineInfo/Warn/Error 日志
```

### 5.2 Step 1:NormalizeToolCallID(规范化工具调用 ID)

```go
tc.ID = agenttools.NormalizeToolCallID(tc.ID, tc.Function.Name, i)
```

**白话**:模型返回的 ID 可能格式不统一(有的长有的短,有的带前缀有的不带),这里规范成统一格式,方便后续追踪和配对。

### 5.3 Step 2-3:参数解析 + JSON 修复

模型返回的 `tc.Function.Arguments` 是 JSON 字符串,但模型可能输出**残缺 JSON**(漏逗号、引号不配对、尾随逗号):

```go
if err := json.Unmarshal([]byte(argsStr), &args); err != nil {
    repaired := agenttools.RepairJSON(argsStr)  ← 尝试修复
    if repairErr := json.Unmarshal([]byte(repaired), &args); repairErr != nil {
        // 修复也失败,返回错误给模型,让模型换思路
        return types.ToolCall{...Result: &types.ToolResult{Success: false, Error: "..."}}
    }
    // 修复成功,用修复后的参数继续
    decoded := tc
    decoded.ModelArguments = ""
    decoded.Function.Arguments = repaired
    decodedCalls := []types.LLMToolCall{decoded}
    e.modelContext.DecodeToolCalls(decodedCalls)  ← 修复后重新走模型上下文解码
    ...
}
```

**两层处理**:
1. 先直接解析,成功就用
2. 失败调 `RepairJSON`(JSON 修复器,补残缺)再解析
3. 还失败:返回错误给模型,错误信息末尾加 `[Analyze the error above and try a different approach.]`(提示模型换思路)

**为什么修 JSON**:模型输出不是百分百规范(尤其长参数),修复后能挽救大部分,减少不必要的失败。

**修复后重新走 `modelContext.DecodeToolCalls`**:
- 修复改变了参数内容,模型上下文协议补丁(笔记 28 提过的 modelContext)要重新跑一遍
- 解析里面的临时句柄(cN/dN/bN/wN/iN/res:// 等模型引用资源的简写)

### 5.4 Step 4-5:formatToolHint + 发工具提示事件

`formatToolHint`(act.go:194)把工具调用变成人能看懂的提示:

```go
func formatToolHint(name string, args map[string]any) string {
    displayName := name
    if dn, ok := toolDisplayNames[name]; ok {
        displayName = dn  ← 用中文显示名
    }
    if len(args) == 0 || toolHintSensitiveArgs[name] {
        return displayName  ← 无参数或敏感工具,只返回名字
    }
    for _, v := range args {
        if s, ok := v.(string); ok {
            if len(s) > 40 {
                s = s[:40] + "…"  ← 截断到 40 字
            }
            return fmt.Sprintf(`%s("%s")`, displayName, s)  ← 拼成"搜索网页("xxx")"
        }
    }
    return displayName
}
```

**两个细节**:
1. **用中文显示名**:`toolDisplayNames` map 把内部工具名(如 `web_search`)映射成中文名(如 `搜索网页`)
2. **敏感参数不展示**:`toolHintSensitiveArgs` map(目前只有 `database_query`,因为 SQL 暴露实现细节),只展示参数的 key 不展示值

**白话**:`formatToolHint` 把"调 web_search 工具,参数 query='腾讯'"翻译成"搜索网页("腾讯")"给用户看,前端展示成"AI 正在:搜索网页("腾讯")"。

然后发 `EventAgentToolCall` 事件,前端看到"AI 要调工具了"。

### 5.5 Step 6-7:PipelineInfo + langfuse span

**PipelineInfo**(act.go:429):内部管道日志,记录"tool_call_start",带 iteration/round/tool/tool_call_id/tool_index。

**langfuse span**(act.go:441):可观测性追踪,开一个名为 `agent.tool.<name>` 的 span。
- span 是追踪系统的一个"段",在 Langfuse UI 里能看到 trace → agent.execute → agent.round.N → agent.tool.<name> 的层级
- `buildToolSpanInput`(act.go:69)构造 span 的输入,记录两边:模型原始参数(`model_arguments`)和解析后参数(`resolved_arguments`),还记 `argument_resolution`(参数解析方式)
- 敏感工具(database_query)的 span 输入也脱敏(只记 key 不记值)

### 5.6 Step 8-9:超时 + 工具执行上下文

**`toolExecutionTimeout`**(const.go:49):
```go
func toolExecutionTimeout(toolName string) time.Duration {
    if toolName == "shell_exec" {
        return shellExecToolTimeout  ← 10 分 5 秒
    }
    return defaultToolExecTimeout  ← 60 秒
}
```

**白话**:大部分工具 60 秒超时,`shell_exec`(沙箱命令执行)给 10 分 5 秒(用户跑长时间命令不该被打断)。

**`ToolExecContext`**(act.go:464):工具执行时能拿到的上下文,装 6 样:
- `SessionID`:会话 ID
- `AssistantMessageID`:助手消息 ID
- `EventBus`:事件总线(工具内部要推事件用)
- `ToolCallID`:工具调用 ID
- `UserID`:用户 ID
- `ApprovalCtx`:审批上下文(注意:用的是 `toolCtx` 不是 `toolExecCtx`,**不带执行超时**,因为 MCP 工具审批可能要等很久)
- `ExecTimeout`:执行超时时长

**关键设计**:`ApprovalCtx` 用 `toolCtx`(只有 round 级 ctx),不用 `toolExecCtx`(带 60s 超时)。
**为什么**:MCP 工具的人审闸(人审批,可能几分钟)不该被 60s 工具超时打断,所以审批用更长的 ctx。

### 5.7 Step 10:UnresolvedHandles 检查

```go
if len(tc.UnresolvedHandles) > 0 {
    err = fmt.Errorf("tool arguments contain unresolved model handles: %v", tc.UnresolvedHandles)
}
```

**白话**:模型上下文解码后,如果还有未解析的句柄(cN/dN/bN/wN/iN/res:// 等临时引用),拒绝执行。
**为什么**:这些句柄是"临时引用"(模型引用之前工具返回的资源),不是真实 ID。如果不解析就执行,可能把幻觉或过期的句柄写到数据库或外部服务,有安全风险。

### 5.8 Step 11:真正执行

```go
execCtx, toolCancel := context.WithTimeout(toolExecCtx, execTimeout)
result, err = e.toolRegistry.ExecuteTool(
    execCtx, tc.Function.Name,
    json.RawMessage(tc.Function.Arguments),
)
toolCancel()  ← 立刻释放定时器资源
```

**白话**:用带超时的 ctx 调 `toolRegistry.ExecuteTool`(工具注册表的执行入口),传工具名和参数。`toolCancel()` 立刻取消定时器,防资源泄漏。

### 5.9 Step 12-13:装结果 + 关 span + 日志

**装结果**:`types.ToolCall{ID, Name, Args, Result, Duration, ProviderMetadata}`
- `Duration` 是 `time.Since(toolCallStartTime).Milliseconds()`,从 Step 4 开始算
- `Result` 是 `*types.ToolResult`,里面有 `Success`/`Output`/`Error`/`Data`/`Images`

**`finishToolSpan`**(act.go:105):关 langfuse span,记结果(success/duration_ms/output/output_len/error/data_keys/image_count)。
- 错误分类:`execErr != nil` 是执行错误;`Result.Success == false` 是工具返回失败(两者都算 span 错误)

**Pipeline 日志**:
- 成功:`PipelineInfo`(普通日志)
- 失败但执行没报错(工具返回 Success=false):`PipelineWarn`(警告)
- 执行报错:`PipelineError`(错误)

---

## 六、`formatToolHint` 详解(UI 提示)

### 6.1 `toolDisplayNames` 映射表

act.go:164 把内部工具名映射成中文显示名,部分摘录:

| 内部名 | 中文显示名 |
|---|---|
| `thinking` | 深度思考 |
| `todo_write` | 制定计划 |
| `grep_chunks` | 关键词搜索 |
| `knowledge_search` | 知识搜索 |
| `list_knowledge_chunks` | 查看文档分块 |
| `query_knowledge_graph` | 查询知识图谱 |
| `get_document_info` | 获取文档信息 |
| `search_conversations` | 回顾历史对话 |
| `search_memory` | 查询长期记忆 |
| `database_query` | 查询数据 |
| `data_analysis` | 数据分析 |
| `data_schema` | 查看数据结构 |
| `web_search` | 搜索网页 |
| `web_fetch` | 获取网页 |
| `execute_skill_script` | 执行技能脚本 |
| `read_skill` | 读取技能 |
| `list_sandbox_files` | 列出沙箱文件 |
| `read_sandbox_file` | 读取沙箱文件 |
| `shell_exec` | 执行沙箱命令 |

### 6.2 `toolHintSensitiveArgs` 敏感工具

act.go:188:
```go
var toolHintSensitiveArgs = map[string]bool{
    agenttools.ToolDatabaseQuery: true,  ← 只有 database_query
}
```

**为什么 database_query 敏感**:它的参数是 SQL,SQL 里可能有表名、字段名、查询逻辑,暴露这些等于暴露数据库实现细节,不该给前端用户看。

### 6.3 三个例子

| 工具 | 参数 | 提示 |
|---|---|---|
| `web_search` | `{"query": "腾讯"}` | `搜索网页("腾讯")` |
| `knowledge_search` | `{"kb_id": "x", "query": "长查询..."}` | `知识搜索("长查询…")` (截断到 40 字) |
| `database_query` | `{"sql": "SELECT * FROM users"}` | `查询数据` (只显示名字,不展示 SQL) |

---

## 七、关键设计要点

### 7.1 串并行的取舍

- **并行**:总耗时短(最慢工具的耗时),但并发可能压垮下游(比如同时查 3 个知识库,3 个向量检索同时跑)
- **串行**:总耗时长(所有工具耗时之和),但简单稳妥,不压下游
- **WeKnora 默认开并行**:提速优先,有 `ParallelToolCalls` 开关可关

### 7.2 并行的结果按原序

`results[i] = toolCall` 用索引保证原序,不是用完成顺序。
**为什么**:前端展示要跟模型发起顺序一致,否则用户看乱(模型先查 A 再查 B,前端不能先显示 B 结果再显示 A 结果)。

### 7.3 并行不连坐

`return nil` 不取消兄弟任务。
**为什么**:工具之间独立(查 A 和查 B 不互相依赖),一个失败不该影响其他。
**反例**:如果工具之间有依赖(比如先查 ID 再用 ID 查详情),就不该并行,但这种依赖应该由模型在多轮 ReAct 里处理,不在一轮里并行。

### 7.4 JSON 修复的两层兜底

模型输出不百分百规范,`RepairJSON` 补救。
**为什么不直接报错**:模型输出残缺 JSON 很常见(尤其长参数),直接报错用户体验差,修复后能挽救大部分。

### 7.5 UnresolvedHandles 严格拒绝

未解析的模型句柄(cN/dN/bN/wN/iN/res://)绝不能进工具执行。
**为什么**:这些是"临时引用",不是真实 ID,幻觉或过期的句柄进数据库或外部服务有安全风险。
**设计哲学**:在执行前最后一道防线拦截,而不是执行后回滚。

### 7.6 ApprovalCtx 不带工具超时

`ApprovalCtx` 用 `toolCtx`(round 级)不用 `toolExecCtx`(带 60s 超时)。
**为什么**:MCP 工具的人审闸(人审批)可能要几分钟,60s 超时会打断审批。
**对比**:`ExecTimeout` 还是 60s,工具本身执行还是 60s 超时,但审批等待不算在 60s 内。

### 7.7 langfuse span 的双层记录

`buildToolSpanInput` 记 `model_arguments`(模型原始输出)和 `resolved_arguments`(解析后真实执行参数)两边。
**为什么**:可观测性要能看出"模型说了啥"和"实际执行了啥"的差异,排查参数解析问题。

---

## 八、跟笔记 28(ReAct 循环骨架)的衔接

- 笔记 28 讲 `runReActIteration` 一轮 4 步,Step 3 Act 就是调 `executeToolCalls`
- 本篇展开 `executeToolCalls` 内部:串并行分发 + `runToolCall` 13 步
- 笔记 28 讲的 `appendToolResults`(Step 4 Observe)是 Act 之后的事,把工具结果加到 messages 里
- **Act 阶段的边界**:从 `executeToolCalls` 入口到所有工具执行完,不含 `appendToolResults`(那是 Observe 阶段)

---

## 九、跟 Think 阶段(笔记 29)的对比

| 维度 | Think 阶段 | Act 阶段 |
|---|---|---|
| 入口 | `callLLMWithRetry` | `executeToolCalls` |
| 串并行 | 单次 LLM 调用(内部流式) | 多个工具调用(可并行) |
| 重试 | `isTransientError` 2 次退避 | 无(单工具失败不重试,让模型换思路) |
| 降级 | 优雅降级合成答案 | 无(失败就失败,工具错误返回给模型) |
| 超时 | 120s chunk 间 | 60s 工具执行(shell_exec 10 分 5 秒) |
| 消毒 | SanitizeMessages | 无(messages 不动,只动参数) |
| 可观测性 | prompt cache 指纹 | langfuse span + PipelineInfo |
| 错误处理 | 临时错重试,大错降级 | 解析失败修复 JSON,修复失败返回错误给模型 |

**白话**:
- Think 阶段是"跟模型对话"(有重试有降级,模型卡了想办法给用户一个答案)
- Act 阶段是"干活"(工具失败就告诉模型,让模型换思路,不替模型兜底)

---

## 十、跟 KnowledgeQA 的对比

| 维度 | KnowledgeQA | Agent Act 阶段 |
|---|---|---|
| 工具调用 | 无(纯生成) | 多工具并行/串行 |
| 参数解析 | 不需要 | JSON 解析 + 修复 |
| 超时 | 无 | 60s / 10 分 5 秒(shell_exec) |
| 可观测性 | 无 | langfuse span + PipelineInfo |
| 错误处理 | 整体失败 | 单工具失败不影响其他 |

**白话**:KnowledgeQA 不调工具(纯文本生成),没有 Act 阶段;Agent 的 Act 阶段是"动手干"的核心。

---

## 十一、代码速查

| 概念 | 位置 |
|---|---|
| `executeToolCalls` | `D:/Project/WeKnora/internal/agent/act.go:217` |
| `executeToolCallsParallel` | `D:/Project/WeKnora/internal/agent/act.go:242` |
| `executeSingleToolCall` | `D:/Project/WeKnora/internal/agent/act.go:310` |
| `runToolCall` | `D:/Project/WeKnora/internal/agent/act.go:356` |
| `formatToolHint` | `D:/Project/WeKnora/internal/agent/act.go:194` |
| `toolDisplayNames` | `D:/Project/WeKnora/internal/agent/act.go:164` |
| `toolHintSensitiveArgs` | `D:/Project/WeKnora/internal/agent/act.go:188` |
| `buildToolSpanInput` | `D:/Project/WeKnora/internal/agent/act.go:69` |
| `finishToolSpan` | `D:/Project/WeKnora/internal/agent/act.go:105` |
| `truncateForLangfuse` | `D:/Project/WeKnora/internal/agent/act.go:32` |
| `toolExecutionTimeout` | `D:/Project/WeKnora/internal/agent/const.go:49` |
| `defaultToolExecTimeout = 60s` | `D:/Project/WeKnora/internal/agent/const.go:26` |
| `shellExecToolTimeout = 10m5s` | `D:/Project/WeKnora/internal/agent/const.go:30` |
| `DefaultMaxToolOutput = 24000` | `D:/Project/WeKnora/internal/agent/tools/truncate.go:12` |
| `NormalizeToolCallID` | `D:/Project/WeKnora/internal/agent/tools/normalize_id.go` |
| `RepairJSON` | `D:/Project/WeKnora/internal/agent/tools/json_repair.go` |
| `ToolExecContext` | `D:/Project/WeKnora/internal/agent/tools/exec_context.go` |
| `toolRegistry.ExecuteTool` | `D:/Project/WeKnora/internal/agent/tools/registry.go` |

---

## 十二、接续信息

本篇把 Act 阶段讲透,留的接续方向:

1. **Approval** 深挖:MCP 工具的人审闸(Redis pub/sub 跨实例审批、OAuth 授权、pending 等待)
   - `runToolCall` Step 9 提到的 `ApprovalCtx` 不带工具超时的设计,这里展开
2. **Analyze 阶段**深挖:`analyzeResponse` 的 content_filter 处理 / natural stop 判断 / emptyContent 重试 nudge
3. **Observe 阶段**深挖:`appendToolResults` 按 OpenAI 格式配对(assistant 带 tool_calls + tool 带 ToolCallID)
4. **工具注册表**深挖:`toolRegistry.ExecuteTool` 内部怎么分发到具体工具(KnowledgeSearch / DatabaseQuery / ShellExec 等)
5. **modelContext** 深挖:`DecodeToolCalls` 解析临时句柄(cN/dN/bN/wN/iN/res://)的协议补丁

不主动继续,等用户提问。