# 笔记 29 · Think 阶段(callLLMWithRetry / streamThinkingToEventBus / streamLLMToEventBus)

> 接笔记 28(ReAct 循环骨架)。笔记 28 讲的是循环骨架,留的接续方向之一就是 Think 阶段深挖。
> 本篇讲 `runReActIteration` 的 Step 1(Think 阶段)是怎么实现的,涉及三个核心函数:
> `callLLMWithRetry`(带重试的 LLM 调用入口)、`streamThinkingToEventBus`(带 thinking 分流的流式调用)、`streamLLMToEventBus`(最底层流式调用)。
>
> 风格延续笔记 28:专业术语 + 白话解释双轨。

---

## 一句话总结

**Think 阶段 = `callLLMWithRetry` 顶门立户**(外层入口,负责重试退避和优雅降级),
**`streamThinkingToEventBus` 干实事**(中坚层,装配参数 + 分流 thinking 和 answer 两路输出),
**`streamLLMToEventBus` 最底层搬砖**(只管流式调模型 + 贴 prompt cache 指纹 + 拼多字节字符)。
三层调用链: `callLLMWithRetry` → `streamThinkingToEventBus` → `streamLLMToEventBus`。

---

## 一、三个函数的定位(三层调用链)

| 函数 | 定位 | 干什么 | 谁调它 |
|---|---|---|---|
| `callLLMWithRetry` | 外层入口(think.go:378) | 带重试的 LLM 调用,失败退避,重试耗尽走优雅降级 | `runReActIteration` Step 1 |
| `streamThinkingToEventBus` | 中坚层(think.go:161) | 装配 ChatOptions,用 ThinkStreamSplitter 把 chunk 分流成 thinking 和 answer 两路 | `callLLMWithRetry` |
| `streamLLMToEventBus` | 最底层(think.go:29) | 跟 ChatModel.ChatStream 对接,管超时和指纹贴标签 | `streamThinkingToEventBus` |

**白话三层比喻**:
- `callLLMWithRetry` 像店长(出问题能重试,实在不行换菜单)
- `streamThinkingToEventBus` 像厨师长(配菜配料,分清哪部分是"思考"哪部分是"答案")
- `streamLLMToEventBus` 像厨工(只管开火煮,把生米煮成熟饭)

---

## 二、最底层 `streamLLMToEventBus`(think.go:29-158)

### 2.1 干什么的

最底层的流式 LLM 调用函数,所有跟模型流式交互的细节都在这里。
`callLLMWithRetry` 和 `streamThinkingToEventBus` 不直接调模型,都通过它。

### 2.2 四步结构

```
streamLLMToEventBus(ctx, engine, state, messages, opts):
  Step 1: 装请求(构造 ChatRequest,设 120s 超时)
  Step 2: 贴指纹(PromptPrefixFingerprint 给 messages 贴 prompt cache 标签)
  Step 3: 开流(chatModel.ChatStream 拿 stream channel)
  Step 4: 消费流(for chunk := range stream 累加 Content,StreamError 单独存,Flush 拼尾部)
```

### 2.3 关键细节

**120s 超时**(`defaultLLMCallTimeout`,const.go):
- 不是整个调用 120s,是"第一个 chunk 来了之后,任意两个 chunk 之间不能超过 120s"
- 防模型卡死(模型还在想,但不出字)
- 超时返回 `StreamError`,上层 `callLLMWithRetry` 判断是不是 transient error(临时错误)决定要不要重试

**贴 prompt cache 指纹**(`PromptPrefixFingerprint`,笔记 27 讲过的基础设施):
- 每次调 LLM 前,给 messages 的"稳定前导"(system + tools schema 那一段)贴一个指纹标签
- Anthropic API 看到相同指纹,直接命中缓存,省算省 token
- **只覆盖稳定前导,不覆盖对话历史**——对话历史每轮变,贴了也命中不了

**StreamDecoder(流式解码器)**:
- 流式来的 chunk 是一块一块的字节,UTF-8 多字节字符(比如中文,一个字 3 字节)可能被劈成两个 chunk
- StreamDecoder 把跨 chunk 的不完整字节缓存住,等下一 chunk 来了拼完整再吐出来
- 没有 StreamDecoder,直接拼字符串会出乱码(半个中文字)

**StreamError 单独存,不混进 Content**:
- 流式过程中模型可能返回错误(`StreamError` 字段),这个错误不塞进 `Content`(正常输出文本),单独存
- 上层看 `StreamError` 判断这轮是不是出错了,不会把错误信息当正常答案给用户看

**Flush(冲洗)**:
- 流结束后,StreamDecoder 里可能还有缓存的不完整字节(理论上不该有,但兜底)
- Flush 把残留拼到 Content 末尾,保证不丢字节

---

## 三、中坚层 `streamThinkingToEventBus`(think.go:161-372)

### 3.1 干什么的

装配这次 LLM 调用的参数(Temperature / Tools / Thinking / ParallelToolCalls),
用 `ThinkStreamSplitter` 把流式来的 chunk 分流成 thinking(思考过程)和 answer(最终答案)两路,
通过 3 个回调(emitThought / closeThinking / emitAnswer)把两路内容推到事件总线给前端。

### 3.2 三段结构

```
streamThinkingToEventBus(ctx, engine, state, messages, opts):
  段 1: 装配 ChatOptions(配菜)
  段 2: 建 ThinkStreamSplitter + 3 个回调(切流)
  段 3: 调 streamLLMToEventBus(执行)
```

### 3.3 段 1:装配 ChatOptions

`ChatOptions`(给模型的参数)包含:

| 参数 | 含义 | 白话 |
|---|---|---|
| Temperature | 温度,控制随机性 | 0.7,答案有点变化但不离谱 |
| Tools | 工具列表 | 这次允许模型调哪些工具 |
| Thinking | 思考开关 | 开了模型会先"思考"再答(thinking 模式) |
| ParallelToolCalls | 并行工具调用 | 允许一轮里同时调多个工具 |

**ParallelToolCalls** 的含义:模型一轮里可以同时发起多个 tool_call(比如同时查 3 个知识库),
而不是一轮一个工具串行调。WeKnora 默认开,提速。

### 3.4 段 2:ThinkStreamSplitter + 3 个回调

**ThinkStreamSplitter(思考流分流器)**:
- 流式来的 chunk 不是纯文本,可能混着:
  - `thinking` 段(模型内部思考,不展示给用户的"草稿")
  - `answer` 段(最终答案,要展示给用户)
  - `tool_call` 段(要调工具,不是文本输出)
  - `reasoning_content` 段(类似 thinking,某些模型的推理过程)
- ThinkStreamSplitter 把混在一起的 chunk 分流成这 4 类

**3 个回调(把分流后的内容推给前端)**:

| 回调 | 触发时机 | 干什么 |
|---|---|---|
| `emitThought` | 收到 thinking / reasoning_content | 把思考片段推到事件总线,前端展示成"AI 正在思考..." |
| `closeThinking` | thinking 段结束,answer 段开始 | 关闭 thinking 区块,告诉前端"思考完了,开始答" |
| `emitAnswer` | 收到 answer 段 | 把答案片段推到事件总线,前端展示成最终答案 |

### 3.5 chunk 分流 4 类

| chunk 类型 | 处理 | 白话 |
|---|---|---|
| `tool_call` pending | 累积,不立刻推 | 模型说要调工具,先攒着,等工具 ID 和参数都齐了再处理 |
| `thinking_tool`(思考工具) | emitThought | 模型思考用的工具(比如检索思路),算思考过程 |
| `reasoning_content` | emitThought | 某些模型(o1/Claude)的推理字段,跟 thinking 一样处理 |
| `plain content` | emitAnswer | 最终答案文本,推给前端 |

### 3.6 answer 乐观渲染 + 撤回机制

**乐观渲染**(optimistic rendering):
- 收到 answer chunk 就立刻推给前端展示,不等全部收完
- 好处:用户看到字一个一个蹦出来,体验好

**撤回机制**:
- 乐观渲染可能出错——比如模型先答了一段,后来发现要调工具,answer 段其实不算数
- 这时候撤回:前端把已展示的 answer 收回去,改成 thinking 或 tool_call
- 实现:`AnswerStreamed` 标志(是否已经流式推过 answer)+ `AnswerEventID`(推 answer 的事件 ID,撤回时用这个 ID 告诉前端"这段作废")

### 3.7 AnswerStreamed / AnswerEventID 透传

这两个字段在 state 里,`streamThinkingToEventBus` 读完流后把它们写回 state,
上层 `runReActIteration` 看 `AnswerStreamed` 判断"这轮是不是已经把答案推给前端了",
避免重复推。

---

## 四、外层 `callLLMWithRetry`(think.go:378-501)

### 4.1 干什么的

Think 阶段的入口函数,`runReActIteration` Step 1 直接调它。
它不直接调模型,而是调 `streamThinkingToEventBus`,但包了一层"重试 + 退避 + 降级"。

### 4.2 四段结构

```
callLLMWithRetry(ctx, engine, state):
  段 1: 日志只详记最后 4 条(防日志爆炸)
  段 2: SanitizeMessages 消毒(修连续同角色 + 孤儿 tool_result,下节细讲)
  段 3: 调 streamThinkingToEventBus
  段 4: 失败判断 isTransientError + 重试 maxLLMRetries=2 次线性退避
  段 5: 重试耗尽走优雅降级
```

### 4.3 段 1:日志只详记最后 4 条

messages 可能很长(几十条),全打印日志会爆炸。
这里只把最后 4 条消息详细打印(前几条只记个数)。
**白话**:日志只看最后几条,不看历史,省日志空间。

### 4.4 段 2:SanitizeMessages 消毒

这是本篇的重点之一,单独一节细讲(见第五节)。
**一句话**:把发给 LLM 的 messages 修一遍,防 OpenAI 协议报错。

### 4.5 段 4:重试退避

**isTransientError(临时错误判断)**(const.go):
- 判断返回的错误是不是"临时性"的(网络抖动 / 429 限流 / 500 服务端临时错)
- 临时错误才重试,非临时错误(比如参数错)直接返回不重试

**maxLLMRetries = 2**(const.go):最多重试 2 次,加上首次调用,总共最多 3 次尝试。

**线性退避**(linear backoff):
- 第 1 次重试等 1 秒
- 第 2 次重试等 2 秒
- 不是指数退避(1, 2, 4, 8...)那种,是线性的,温和点
- **白话**:第一次失败缓一缓(1 秒),再失败缓久点(2 秒),还不行就算了

### 4.6 段 5:优雅降级(graceful degradation)

重试 2 次还失败,不直接报错给用户,走"优雅降级":

**判断有没有历史 tool 结果**(state.Messages 里有没有 tool 角色的消息):
- **有历史 tool 结果**:调 `streamFinalAnswerToEventBus`(笔记 28 讲过,finalize.go),
  把所有 tool 结果作为 user 消息 + final prompt,独立 LLM 调用合成答案。
  返回 `nil, nil`(不报错,假装成功,但 state 里标记是降级合成的)。
  **白话**:模型卡了,但前面查到了资料,用一个简单 prompt 让模型把资料总结成答案给用户。
- **无历史 tool 结果**:返回 error(真没辙了,告诉用户这次不行)。
  **白话**:模型卡了,也没查到任何资料,只能报错。

**为什么有历史 tool 结果要降级合成**:
- 笔记 28 讲过,ReAct 循环跑满 20 轮没 natural stop 时也调 `streamFinalAnswerToEventBus`
- 这里是另一种触发路径:重试耗尽(不是跑满 20 轮)
- 共同思路:已经查到的资料不能浪费,想办法给用户一个答案

---

## 五、SanitizeMessages 消毒机制(细讲)

### 5.1 为什么需要消毒

OpenAI 的 chat completion API 对 messages 数组有严格格式要求:
1. **不能有连续同角色消息**(比如两条 `user` 连着,两条 `assistant` 连着)
   - 例外:`tool` 角色可以连续(多个工具结果连着返回是正常的)
2. **tool 角色消息必须能找到对应的 assistant tool_call**
   - 每个 `tool` 消息带一个 `ToolCallID`,前面必须有某个 assistant 消息的 `ToolCalls` 里包含这个 ID
   - 找不到就是"孤儿 tool result"(没爹的工具结果),API 会报错
3. **非 system 消息不能空内容**(content 空且没 tool_calls,API 拒收)

WeKnora 的 messages 来源复杂:
- 从 DB 读的历史(可能被压缩过,压缩可能搞出连续同角色)
- 4 道闸处理过(闸 ① trim 用占位符可能产生空消息,闸 ③ Consolidator 合并可能产生连续 system)
- 各种边角情况

SanitizeMessages 在调 LLM 前跑一遍,把这些问题修掉,防 API 报错。

### 5.2 三种消毒动作

源码:`D:/Project/WeKnora/internal/agent/tools/sanitize_messages.go`

**动作 1:跳过空消息**(line 22-25)
```
if msg.Content == "" && msg.Role != "system" && msg.Role != "tool" && len(msg.ToolCalls) == 0 {
    continue
}
```
- 跳过空内容的非 system 非 tool 消息
- system 空消息留着(有些模型要求 system 段存在,即使是空的)
- tool 空消息留着(tool 结果可能就是空的,比如某个工具没产出)

**动作 2:合并连续同角色**(line 28-35)
```
if len(result) > 0 && msg.Role != "tool" {
    prev := result[len(result)-1]
    if prev.Role == msg.Role && prev.Role != "tool" {
        result[len(result)-1].Content += "\n\n" + msg.Content
        continue
    }
}
```
- 当前消息跟上一条同角色(且不是 tool):合并到上一条
- 合并方式:上一条 Content 后面加 `\n\n` 拼当前 Content
- tool 角色不合并(多个 tool 结果连着是正常的,API 允许)

**动作 3:孤儿 tool result 转成 system**(line 38-46)
```
if msg.Role == "tool" && msg.ToolCallID != "" {
    if !hasMatchingToolCall(messages[:i], msg.ToolCallID) {
        msg.Role = "system"
        msg.Content = "[Tool result for " + msg.Name + "]: " + msg.Content
        msg.ToolCallID = ""
        msg.Name = ""
    }
}
```
- 当前是 tool 消息,带 ToolCallID
- 往前找所有 assistant 消息,看有没有哪个的 ToolCalls 里包含这个 ID
- 找不到:转成 system 消息,Content 加前缀 `[Tool result for XXX]:`,清掉 ToolCallID 和 Name
- **白话**:孤儿工具结果没爹,改成 system 消息塞进去,至少不报错,信息也不丢

### 5.3 举例

**例子 1:连续同角色合并**

消毒前:
```
[
  {role: system, content: "你是助手"},
  {role: user, content: "查 A"},
  {role: user, content: "再查 B"},      ← 跟上一条 user 连着
  {role: assistant, content: "好的"}
]
```

消毒后:
```
[
  {role: system, content: "你是助手"},
  {role: user, content: "查 A\n\n再查 B"},  ← 两条 user 合并,用 \n\n 分隔
  {role: assistant, content: "好的"}
]
```

**例子 2:孤儿 tool result 转成 system**

消毒前(假设因为某个 bug,tool 消息对应的 tool_call 被丢了):
```
[
  {role: user, content: "查知识库"},
  {role: assistant, content: "我查查", tool_calls: []},  ← tool_calls 是空数组
  {role: tool, tool_call_id: "call_123", name: "search_kb", content: "找到结果 X"}  ← 找不到对应的 tool_call
]
```

消毒后:
```
[
  {role: user, content: "查知识库"},
  {role: assistant, content: "我查查", tool_calls: []},
  {role: system, content: "[Tool result for search_kb]: 找到结果 X"}  ← 转成 system,前缀标注来源
]
```

**例子 3:空消息跳过**

消毒前:
```
[
  {role: system, content: "你是助手"},
  {role: user, content: ""},              ← 空 user 消息
  {role: user, content: "查 A"},
  {role: assistant, content: ""}
]
```

消毒后:
```
[
  {role: system, content: "你是助手"},
  {role: user, content: "查 A"},          ← 空 user 被跳过
  {role: assistant, content: ""}           ← 这条 assistant 空内容,但因为后面没有非空 user 跟着,可能被下轮处理
]
```

注意:`assistant` 空消息如果后面还有内容,可能在下一轮被 trim 或合并处理,
SanitizeMessages 自己只跳过"非 system 非 tool 且 content 空 且 tool_calls 空"的消息,
`assistant` 带 tool_calls 的不算空消息(即使 content 空),不会被跳。

### 5.4 一个完整复杂例子

消毒前(4 道闸处理后的产物,可能有各种问题):
```
[
  {role: system, content: "你是助手"},
  {role: system, content: "知识库说明"},       ← 跟上一条 system 连着
  {role: user, content: "查 A"},
  {role: assistant, content: "", tool_calls: [{id: "call_1", ...}]},
  {role: tool, tool_call_id: "call_1", name: "search", content: "结果 A"},  ← 有爹
  {role: tool, tool_call_id: "call_999", name: "orphan", content: "孤儿"},   ← 没爹(call_999 不存在)
  {role: user, content: ""},                    ← 空 user
  {role: user, content: "查 B"},                ← 跟上一条 user 连着(如果空 user 没被跳)
  {role: assistant, content: "好的"}
]
```

消毒后:
```
[
  {role: system, content: "你是助手\n\n知识库说明"},      ← 两条 system 合并
  {role: user, content: "查 A"},
  {role: assistant, content: "", tool_calls: [{id: "call_1", ...}]},
  {role: tool, tool_call_id: "call_1", name: "search", content: "结果 A"},
  {role: system, content: "[Tool result for orphan]: 孤儿"},  ← 孤儿 tool 转 system
  {role: user, content: "查 B"},                    ← 空 user 被跳,这条独立了
  {role: assistant, content: "好的"}
]
```

---

## 六、关键设计要点

### 6.1 三层分工的好处

- **`streamLLMToEventBus` 只管流式细节**(超时/指纹/解码),不关心 thinking 还是 answer
- **`streamThinkingToEventBus` 只管分流**(thinking/answer/tool_call),不关心重试
- **`callLLMWithRetry` 只管重试和降级**,不关心流式细节

每一层职责单一,改一层不影响另一层。
比如换模型(从 Anthropic 换 OpenAI),只改 `streamLLMToEventBus` 底层对接,上层不动。

### 6.2 重试和降级的分层

- `callLLMWithRetry` 重试的是 transient error(临时错误),2 次线性退避
- 重试耗尽走优雅降级,不直接报错给用户
- 降级路径复用 `streamFinalAnswerToEventBus`(跟笔记 28 的 handleMaxIterations 共用)
- **白话**:小错自己扛(重试),大错想办法(降级合成),实在不行才告诉用户

### 6.3 消毒的位置

SanitizeMessages 在 `callLLMWithRetry` 段 2,每次调 LLM 前跑。
不是在 4 道闸里跑,是在 4 道闸之后、真正调 LLM 之前兜底。

- 4 道闸是"主动管理"(trim/redact/Consolidator/Compress,笔记 27)
- SanitizeMessages 是"被动兜底"(4 道闸可能没考虑到所有边角,它来补)

### 6.4 流式输出的三个层次

| 层次 | 函数 | 前端看到 |
|---|---|---|
| 字节流 | `streamLLMToEventBus` | 没看到(底层) |
| 分流 | `streamThinkingToEventBus` | "AI 正在思考..." + 答案字一个一个蹦 |
| 重试 | `callLLMWithRetry` | 没看到(失败重试,前端只看到最终结果) |

---

## 七、跟笔记 28(ReAct 循环骨架)的衔接

- 笔记 28 讲 `runReActIteration` 一轮 4 步,Step 1 Think 就是调 `callLLMWithRetry`
- 本篇展开 `callLLMWithRetry` 内部三层调用链 + 重试 + 降级
- 笔记 28 讲的"stuck loop 检测"和"ctx 取消保 partial"是在 `runReActIteration` 层,不在 Think 内部
- 笔记 28 讲的"empty retry"(LLM 答空白重试 2 次)是在 `analyzeResponse` 层(Step 2),不在 Think 内部

**Think 阶段的边界**:从 `callLLMWithRetry` 入口到返回,中间不含 stuck loop / ctx 取消 / empty retry,
这些是 `runReActIteration` 在 Think 返回后处理的。

---

## 八、跟 KnowledgeQA 的对比

| 维度 | KnowledgeQA | Agent Think 阶段 |
|---|---|---|
| 重试 | 无 | `callLLMWithRetry` 2 次线性退避 |
| 降级 | 无 | 有历史 tool 结果走 `streamFinalAnswerToEventBus` |
| 流式 | 单路(只有 answer) | 双路(thinking + answer,ThinkStreamSplitter 分流) |
| 消毒 | 不需要(单轮单次,messages 简单) | 需要(多轮 ReAct,messages 复杂) |
| 超时 | 无明确超时 | 120s chunk 间超时 |
| prompt cache | 不贴指纹 | 贴 PromptPrefixFingerprint 指纹 |

**白话**:KnowledgeQA 简单粗暴(一次调用,不重试,不降级,不消毒),
Agent Think 阶段复杂(重试 + 降级 + 双路流式 + 消毒 + 指纹),因为 Agent 多轮多工具,边角情况多。

---

## 九、代码速查

| 概念 | 位置 |
|---|---|
| `callLLMWithRetry` | `D:/Project/WeKnora/internal/agent/think.go:378` |
| `streamThinkingToEventBus` | `D:/Project/WeKnora/internal/agent/think.go:161` |
| `streamLLMToEventBus` | `D:/Project/WeKnora/internal/agent/think.go:29` |
| `SanitizeMessages` | `D:/Project/WeKnora/internal/agent/tools/sanitize_messages.go:14` |
| `hasMatchingToolCall` | `D:/Project/WeKnora/internal/agent/tools/sanitize_messages.go:55` |
| `isTransientError` | `D:/Project/WeKnora/internal/agent/const.go` |
| `maxLLMRetries = 2` | `D:/Project/WeKnora/internal/agent/const.go` |
| `defaultLLMCallTimeout = 120s` | `D:/Project/WeKnora/internal/agent/const.go` |
| `PromptPrefixFingerprint` | `D:/Project/WeKnora/internal/llm/prompt_cache.go`(笔记 27) |
| `streamFinalAnswerToEventBus`(降级路径) | `D:/Project/WeKnora/internal/agent/finalize.go:27`(笔记 28) |

---

## 十、接续信息

本篇把 Think 阶段讲透,留的接续方向:

1. **Act 阶段**深挖:`executeToolCalls` 串并行分发(ParallelToolCalls 配置)、
   `executeSingleToolCall` 单工具执行、tool span 追踪、参数脱敏、`formatToolHint` 提示词注入
2. **Approval** 深挖:MCP 工具的人审闸(Redis pub/sub 跨实例审批、OAuth 授权、pending 等待)
3. **Analyze 阶段**深挖:`analyzeResponse` 的 content_filter 处理 / natural stop 判断 / emptyContent 重试 nudge
4. Agent 路径其他:`image_requirement.go`(图片要求)、skills(Progressive Disclosure)、modelContext(协议补丁)

不主动继续,等用户提问。