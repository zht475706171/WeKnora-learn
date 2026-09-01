# WeKnora-learn

我跟 AI 一起把 `D:/Project/WeKnora`(腾讯开源的 WeKnora 知识管理框架)从头到尾吃透的学习笔记。

## 目录结构

```
WeKnora-learn/
├── README.md            # 本文件,说明进度和怎么用这套笔记
├── notes/               # 按层级整理的学习笔记
└── sessions/            # 每次对话的原始记录(防止 AI 失忆)
```

## 阅读顺序

### 阶段 1:架构与文档解析

1. `notes/01-最外层架构.md` —— 用大白话讲项目是干嘛的、有哪些大模块、之间怎么配合
2. `notes/02-docreader文档解析.md` —— 文档怎么被转成 Markdown

### 阶段 2:切块(切分策略)

3. `notes/03-Go后端切块.md` —— Go 后端切块入口
4. `notes/04-Tier1和Tier2的正则切法.md` —— Tier 1 / Tier 2 正则总览
5. `notes/05-Tier1大白话详解.md` —— Tier 1 按标题切详解
6. `notes/06-coalesceTinyChunks详解.md` —— Tier 1 小 chunk 合并机制
7. `notes/07-Tier1用户描述纠错.md` —— 用户讲 Tier 1 时的 7 个错
8. `notes/08-Tier2启发式切详解.md` —— Tier 2 启发式 8 正则切
9. `notes/09-Tier2碎chunk处理.md` —— Tier 2 防 3 道防线
10. `notes/10-Tier3递归切详解.md` —— Tier 3 递归切(兜底)
11. `notes/11-Tier3四问递归切代码逐行+大小标准+overlap机制.md` —— Tier 3 四问 + overlap
12. `notes/12-Tier1到Tier3全过程串联.md` —— Tier 1 → Tier 3 串起来
13. `notes/13-体检ProfileDocument详解.md` —— 体检机制(11 个信号)
14. `notes/14-Tier3递归切与合并机制详解.md` —— mergeUnits 合并(不会一句一 chunk)
15. `notes/15-两阶段切分思路与overlap机制.md` —— 两阶段切分 + overlap 80 字 3 级优先级
16. `notes/16-保护区间机制详解.md` —— 7 正则保护区间 + span 字节偏移
17. `notes/17-全流程纠错与三层数字差异.md` —— 全流程纠错 + 256/128/无差异
18. `notes/18-保护机制7500硬切详解.md` —— 7500 硬切(切成什么、是不是单独 chunk)
19. `notes/19-父子分块SplitTextParentChild详解.md` —— 父子分块(跟 Tier1-3 区别、为什么要两层、全流程 8 步、切点与大小分离)
20. `notes/20-父子分块优化分析与检索双路径.md` —— 父块强塞噪声诊断、自动/Agent 双检索路径、为什么不推荐 LLM 摘要、3 个真正优化方向

### 阶段 3:向量化与入库

21. `notes/21-向量化与入库全流程.md` —— processChunks 12 步(幂等清理→建DB Chunk→写chunks表→3层拼接IndexInfo→BatchIndex批量embed→异步后处理→三态状态机)、隐藏的二次 BatchIndex(问题生成)
22. `notes/22-知识图谱后处理与Neo4j存储原理.md` —— 图谱抽实体+关系存Neo4j、图库vs关系库、定长记录偏移寻址+关系双向链表、index-free adjacency、apoc.merge去重+union累加chunks+Label按KB隔离
23. `notes/23-Wiki后处理与存储及并发控制.md` —— wiki存PG不进向量库、Map-Reduce流水线LLM生成内容、slug/SourceRefs/链接、图谱vs wiki并发根本区别(累加vs重写)、三重防护(claiming+per-slug悲观锁+乐观锁)、为什么悲观+乐观互补、冲突不能跳过LLM

### 阶段 4:检索与对话

24. `notes/24-检索全链路.md` —— 9阶段pipeline(query改写→双路并行检索→RRF融合→rerank两阶段粗排精排→merge8步增厚→filter_topk→喂LLM)、cross-encoder rerank、复合分+MMR、无token budget裁剪
25. `notes/25-对话历史记忆召回改写意图与数据库选型.md` —— 对话历史(取最近N轮+三层压缩+丢弃检索结果原文+Agent KB结果涂黑)、记忆召回(跨会话5kind+常驻无条件+情境RRF混合+显式+蒸馏写入)、query改写意图(一次LLM三任务+9意图路由+@mention)、数据库选型(pgvector+ParadeDB同库+Redis基础设施+Lite)
26. `notes/26-载入历史对话细讲与优化点.md` —— KnowledgeQA路径载入历史5步(取20行→request_id配对→丢弃RenderedContent+剥think+补图片附件→丢不完整轮→截5轮)+10机制+4优化点(最该做:KnowledgeQA复用Agent压缩闸)
27. `notes/27-Agent路径四道上下文管理闸.md` —— Agent路径每次LLM调用前跑manageContextWindow 4道闸:①trimCurrentTurnToolResults当前轮工具结果按预算裁(占位符+倒序补全+头尾预览1:3+配对不拆,3个盲区:单工具无限制/渐进式塞满/头尾预览丢中间)②redactHistoryKBResults历史轮KB工具结果涂黑(默认开,两层处理:落盘CompactToolOutputForHistory+装配redact,8个KB工具,占位符精确语义,跟KnowledgeQA丢RenderedContent对比)③Consolidator旧历史LLM摘要成system消息(token>0.5触发,findKeepBoundary分组不拆,rawArchive兜底,prompt翻译)④CompressContext硬砍最旧group(token>0.8兜底,groupToolMessages分组,worked example)。4道全视图层不动DB原版。★stable prefix机制(prompt_cache.go):PromptPrefixFingerprint只哈希前导system+tools schema,Agent每round贴指纹观测cache, wiki用BuildPromptCacheKey+awaitWikiPromptWarmup主动协调。★更正:Agent用了prompt cache不是不用,但只覆盖stable prefix不覆盖对话历史。★两层面拆分:触发压缩(层面A生存问题)和保cache(层面B效率问题)是两个不同层面,触发压缩时cache失效是必然物理后果不是隐患。★冻结决策真正价值是保住压缩成果不让历史膨胀,不是保cache命中(WeKnora靠DB-centered已保证字节稳定)。★claude-code对比:历史非KB工具结果claude-code压缩+冻结保小体积/WeKnora不压全量保留→压缩触发WeKnora早claude-code晚,历史段命中率两者差不多,差在总token体积。Q1/Q2/Q3收敛到Consolidator摘要落盘+复用+CompressContext优先保留summary(主价值省LLM调用,顺带cache命中率提升)。4道闸全讲透+stable prefix讲透+两层面拆分讲透
28. `notes/28-ReAct循环骨架.md` —— AgentEngine.Execute入口装配state/systemPrompt/messages/tools 4样,executeLoop主循环for CurrentRound<MaxIterations(默认20),3种iterOutcome(next进下一轮/continue重跑本轮/break退出)switch控制,defer emitCompletion用WithoutCancel保证exactly-one完成事件。runReActIteration一轮4步:Step 0 manageContextWindow(笔记27 4道闸,token估算用lastUsage+delta优化)→Step 1 Think callLLMWithRetry→Step 1.5 stuck loop检测(连续2轮相同内容无tool_call强停,防unhandled finish reason)→Step 1.6 ctx取消保partial step(用户按stop保留半截思考,不设IsComplete)→Step 2 Analyze analyzeResponse(content_filter/natural stop无tool_call→done,emptyContent重试2次nudge再失败fallback)→Step 3 Act executeToolCalls→Step 4 Observe appendToolResults(OpenAI格式assistant带tool_calls+tool带ToolCallID配对不拆)。4类异常路径(ctx取消salvage/stuck loop/empty retry/content_filter)各自兜底。循环结束handleMaxIterations跑满20轮没natural stop时streamFinalAnswerToEventBus合成(tool结果作为user消息不是tool消息,独立LLM调用),ctx已取消时不跑(防fallback文本泄露UI)。emitCompletionEvent写state.RoundSteps进EventAgentComplete持久化到Message.AgentSteps。AgentEngine跨轮无状态(DB-centered),单次Execute内有状态。跟KnowledgeQA对比(单轮单次vs多轮ReAct)。★跟claude-code对比(DB-centered vs in-memory state设计哲学):WeKnora每次提问翻账本(DB是主角,4道闸是视图层不动DB原版,不需要冻结决策,DB不变+算法确定保字节稳定);claude-code活账本上直接划(内存state.messages是主角,REPL.tsx:1182 useState维护,整个session活着不是每次重建,压缩破坏性改state.messages microcompact把旧tool_result换成"[Old tool result cleared]",需要冻结决策ContentReplacementState seenIds/replacements Map保字节一致保prompt cache命中)。最根本差异:WeKnora牺牲性能换简单和完整,claude-code牺牲复杂度换性能和省token。claude-code代码位置:state.messages REPL.tsx:1182 / appendEntry sessionStorage.ts:1128 / ContentReplacementState toolResultStorage.ts:390 / microCompact.ts:36
29. `notes/29-Think阶段.md` —— Think阶段三层调用链:callLLMWithRetry(外层入口,think.go:378)→streamThinkingToEventBus(中坚层,think.go:161)→streamLLMToEventBus(底层,think.go:29)。★streamLLMToEventBus 4步:装请求(120s chunk间超时defaultLLMCallTimeout)→贴prompt cache指纹(PromptPrefixFingerprint只覆盖稳定前导不覆盖对话历史)→开流(chatModel.ChatStream)→消费流(StreamDecoder处理UTF-8多字节跨chunk拼接,StreamError单独存不混进Content,Flush拼尾部)。★streamThinkingToEventBus 3段:装配ChatOptions(Temperature 0.7/Tools/Thinking/ParallelToolCalls默认开提速)+建ThinkStreamSplitter+3回调(emitThought推思考/closeThinking关思考区块/emitAnswer推答案)+调streamLLMToEventBus。chunk分流4类(tool_call pending累积不立刻推/thinking_tool emitThought/reasoning_content emitThought/plain content emitAnswer),answer乐观渲染+撤回机制(模型改主意要调工具时撤回answer,AnswerStreamed/AnswerEventID透传控制)。★callLLMWithRetry 5段:日志只详记最后4条(防日志爆炸)+SanitizeMessages消毒+调streamThinkingToEventBus+isTransientError重试maxLLMRetries=2次线性退避(1s/2s不是指数)+重试耗尽走优雅降级(有历史tool结果调streamFinalAnswerToEventBus合成答案返回nil,nil;无历史tool结果返回error)。★SanitizeMessages消毒机制(sanitize_messages.go:14)3动作:①跳过空消息(非system非tool,content空且tool_calls空)②合并连续同角色(当前跟上一条同角色且不是tool,合并Content用\n\n拼,tool角色不合并因多个tool结果连着正常)③孤儿tool result转system(hasMatchingToolCall往前找assistant的ToolCalls里有没有匹配ID,找不到转成system,Content加前缀[Tool result for XXX]:,清ToolCallID和Name,信息不丢只改角色)。4个举例(连续同角色合并/孤儿tool转system/空消息跳过/完整复杂例子)。跟KnowledgeQA对比(无重试vs2次退避/无降级vs有历史tool结果降级合成/单路流式vs双路thinking+answer/不需要消毒vs需要/无超时vs120s/不贴指纹vs贴PromptPrefixFingerprint)
30. `notes/30-Act阶段.md` —— Act阶段4函数:executeToolCalls(外层入口,act.go:217)看ParallelToolCalls开关+工具数≥2决定走并行还是串行;executeToolCallsParallel(act.go:242)用errgroup并发,3关键设计(i,tc:=i,tc捕获循环变量防串号/results[i]=toolCall按原序写入不按完成顺序/return nil不取消兄弟任务因工具独立);executeSingleToolCall(act.go:310)for循环串行跑完立刻推事件;runToolCall(act.go:356)核心13步串并行都用它。★runToolCall 13步:①NormalizeToolCallID规范化ID ②json.Unmarshal解析参数 ③失败调RepairJSON修复再解析,再失败返回错误给模型(末尾加[Analyze the error and try a different approach]提示换思路),修复成功重新走modelContext.DecodeToolCalls ④formatToolHint生成UI提示 ⑤发EventAgentToolCall事件 ⑥PipelineInfo日志 ⑦开langfuse span(buildToolSpanInput记model_arguments和resolved_arguments双层,敏感工具database_query脱敏只记key) ⑧取principal+设execTimeout ⑨装ToolExecContext(ApprovalCtx用toolCtx不带工具超时,为MCP人审闸留后门) ⑩查UnresolvedHandles(cN/dN/bN/wN/iN/res://未解析句柄)有就拒绝执行(防幻觉/过期句柄进DB或外部服务) ⑪context.WithTimeout→toolRegistry.ExecuteTool真正执行→toolCancel ⑫装结果ToolCall{ID/Name/Args/Result/Duration/ProviderMetadata} ⑬finishToolSpan+三级PipelineInfo/Warn/Error日志。★formatToolHint(act.go:194)UI提示:toolDisplayNames 19个工具内部名→中文显示名映射(web_search→搜索网页/knowledge_search→知识搜索/database_query→查询数据等),toolHintSensitiveArgs只有database_query敏感(SQL暴露实现细节)只显示名字不展示参数,参数字符串截断40字拼成"搜索网页("腾讯")"格式。★超时分流(const.go:toolExecutionTimeout):大部分工具60s(shellExecToolTimeout=10m5s给长命令留时间),按工具名分流不统一超时。★ApprovalCtx不带工具超时:用toolCtx(round级)不用toolExecCtx(带60s),因MCP人审闸可能几分钟60s会打断审批,ExecTimeout还是60s但审批等待不算在内。★跟Think对比(单次LLM调用vs多工具并行/有重试降级vs失败让模型换思路/120s chunk间vs60s工具超时/SanitizeMessages消毒vs不动messages只动参数/prompt cache指纹vs langfuse span)。设计哲学:Think跟模型对话有重试有降级模型卡了给用户答案;Act干活失败告诉模型让模型换思路不替模型兜底
31. `notes/31-Approval审批闸.md` —— Approval闸(MCP工具人审闸,approval/gate.go)。Gate结构体(gate.go:135)pending map+checker+timeout(默认10分钟)+rdb(可选跨实例)+failClose(默认true保守)。waiter(gate.go:144)ch带1缓冲+once防重复+resolved原子。Decision(gate.go:76)4状态(同意/拒绝+原因/超时/ctx取消)+ModifiedArgs同意但改参数。★4核心函数:NeedsApproval(gate.go:289)查checker.IsRequired,失败fail-close默认要审(WEKNORA_AGENT_TOOL_APPROVAL_FAIL_OPEN=true改放行);RequestAndWait(gate.go:309)7步(生成pendingID/注册waiter/发EventToolApprovalRequired前端弹框/起timer 10分钟/select三路/emitResolved),select三路(用户决议/超时/ctx取消),超时和取消自己deliver占住Once防后续Resolve再投,emitResolved用context.WithoutCancel保证ctx取消也能发事件清理前端弹框;Resolve(gate.go:539)先deliverLocal(三层校验:pendingID存在/租户匹配/用户匹配,空caller userID也算mismatch fail-close)找不到走resolveCrossInstance;RequestOAuthAndWait(gate.go:428)OAuth授权闸变体不查checker reactively由transport错误驱动,WaitTimeout可配,授权完成重试工具调用。★跨实例Redis Pub/Sub:主频道weknora:mcp_approval:resolve(带WEKNORA_REDIS_NAMESPACE隔离)+回复频道reply:<pendingID> per-pending专用防ack串;instanceID(进程uuid)防自处理;RequestNonce(per-call uuid)防并发Resolve串ack;runSubscriber每实例一goroutine订阅主频道,Redis断了指数退避重连(1s/2s/.../30s封顶);resolveCrossInstance流程:订阅reply频道→发决议带ReplyChannel→拥有者本地交付回ack(ok/tenant_mismatch/user_mismatch/already_resolved/not_found)→发起者收ack返回HTTP状态码(3秒超时)。★跟mcp_tool.go衔接(mcp_tool.go:116-183):MCPTool.Execute里gate.NeedsApproval判断要审→waitCtx=meta.ApprovalCtx(不带60s防审批被打断)→gate.RequestAndWait阻塞→同意但改参数decision.ModifiedArgs→审批后重新派生fresh ctx(从ApprovalCtx派生带ExecTimeout)给真正MCP CallTool完整超时窗口。★完整例子(delete_repo)流程:executeToolCalls→runToolCall→MCPTool.Execute→RequestAndWait发事件前端弹框→用户点同意→前端POST /api/agent/approval/resolve→gate.Resolve→deliverLocal→w.deliver→ch→RequestAndWait返回Approved→MCPTool.Execute重新派生ctx→connectAndCall真执行。异常路径:超时10分钟/用户按stop/多副本跨实例Pub/Sub。★跟Think Act对比(流式不阻塞vs工具执行60s vs人审10分钟/不涉及跨实例vs不涉及vs Redis Pub/Sub/模型输出vs工具返回vs用户点击),设计哲学:Think快/Act中等/Approval很久跨实例需要Pub/Sub
32. `notes/32-工具注册表.md` —— ToolRegistry工具分发(registry.go)。ToolRegistry结构体(registry.go:19)tools map+maxToolOutputSize(0用默认24000)。types.Tool接口4方法(Name/Description/Parameters/Execute),BaseTool(tool.go:11)公共实现具体工具嵌入+自己写Execute。★注册:RegisterTool(registry.go:55)first-wins policy防劫持,同名先到先得后到拒绝,防name collision attack恶意插件冒充内置工具偷数据(参考GHSA-67q9-58vj-32qx)。GetFunctionDefinitions(registry.go:92)sort.Strings排序保字节稳定,Go map迭代随机化不排序会乱,Qwen explicit caching字节级前缀匹配不排序就miss。★ExecuteTool 7步(registry.go:112):①PipelineInfo日志 ②GetTool找不到返回错误+toolErrorHint ③CastParams类型转换(LLM输出"true"转true/"123"转123/"[{...}]"parse数组,按JSON Schema安全转换)④ValidateParams校验(required/type/enum/minimum/maximum/minLength/maxLength)失败返回nil err+ToolResult.Success=false(不是执行错误是参数错,让runToolCall不走错误路径把错误给模型换思路)⑤算输出预算(maxOutput默认24000,工具实现outputLimitProvider.OutputLimitChars(args)可自定义只放大不缩小)⑥tool.Execute(WithOutputBudget(ctx,maxOutput),args)真正执行 ⑦TruncateToolOutput截断(utf8.RuneCountInString不是len()防中文过度截断,中文3字节算1字符,truncationMarkerReserve=200留给截断标记"... [truncated]")+三级日志(PipelineError执行错误/PipelineWarn工具返回失败/PipelineInfo成功)。★toolErrorHint常量(registry.go:16)"[Analyze the error above and try a different approach.]"三处加(工具找不到/参数校验失败/工具返回Success=false)提示模型换思路。★三类错误区分:工具找不到返回err/参数校验失败返回nil err/工具执行失败nil err或err,执行错误是panic或ctx错,参数错误和工具失败是模型问题返回nil err让模型换思路。★工具集清单(definitions.go):19个内置工具名常量(thinking/todo_write/knowledge_search/grep_chunks等+wiki_*10个),DefaultAllowedTools默认列表(web_search/search_memory/shell_exec/wiki_*不在默认需显式开,search_memory由workspace/user/agent三方开关决定),maxFunctionNameLength=64(OpenAI API限制),AvailableToolDefinitions给UI用元数据。★KB能力要求(capabilities.go):KBCapability 5种(vector/keyword/wiki/graph/faq),ToolRequirement(AnyOf至少一个/AllOf全部满足/ConsumesFiles消费文件),ToolCapabilityRequirements映射表(thinking/todo_write无依赖,knowledge_search等AnyOf[vector,keyword]+ConsumesFiles,wiki_*AllOf[wiki]),前后端同步capabilities.go跟frontend/src/utils/tool-capabilities.ts必须同步(前端灰化+后端兜底防绕过),DeriveKBFilterForAgent quick-answer模式强制vector或keyword,ToolsConsumeFiles未知工具(MCP)宽松视为消费防隐藏文件选择器。★授权层(scope_authorization.go):authorizeKnowledgeInSearchTargets/authorizeChunkInSearchTargets/validateKnowledgeBaseIDsInSearchTargets/resolveAuthorizedSourceRefs,工具内部调用前先校验后端兜底防绕过前端。★完整例子(knowledge_search)流程:runToolCall Step 11→ExecuteTool 7步→KnowledgeSearchTool.Execute内部(解析参数+授权校验+检索pipeline+主动按预算截断)→返回ToolResult→TruncateToolOutput兜底截断→三级日志。异常路径:工具幻觉Step 2失败/参数错Step 4失败/工具失败Step 6返回Success=false,都加toolErrorHint给模型换思路。跟KnowledgeQA对比(不调工具vs多工具分发/不需要校验vs CastParams+ValidateParams/不涉及预算vs 24000 rune/不涉及KB能力vs AnyOf/AllOf前后端同步/不涉及字节稳定vs GetFunctionDefinitions排序)
33. `notes/33-工具案例细讲.md` —— 3个典型工具三层结构(Definition/Content/Execute)细讲。★工具三层:定义(BaseTool{name/description/schema}静态名片给模型看)/内容(Tool结构体+字段装干活家伙)/执行(Execute方法真正干活)。★案例1 knowledge_search(外部检索型):KnowledgeSearchTool结构体装KBStore/VectorStore/Embedder/RerankModel/ChatModel/LLMClient/GraphStore/WikiStore/seenChunks。Execute 10步:解析参数+授权校验→算预算→concurrentSearchByTargets按embedding model分组并发检索→双rerank(rerankModel→chatModel fallback,rerankWithLLM 15个一批温度0.1)→applyMMR(lambda=0.7 Jaccard去冗余)→deduplicateResults(多key+内容签名)→formatOutput(XML结构化+seenChunks防重复省token+retrieval_statistics覆盖率统计)→返回。compositeScore=0.6*modelScore+0.3*baseScore+0.1*sourceWeight。★案例2 thinking(思维辅助型):SequentialThinkingTool纯内存不调外部服务,thoughtHistory+branches。Execute:解析→validate(thought非空/thoughtNumber>=1/totalThoughts>=1)→动态调整totalThoughts(模型可中途加步骤)→append thoughtHistory→处理branches(分支思考回溯)→返回display_type:thinking+incomplete_steps标志。★案例3 todo_write(状态追踪型):TodoWriteTool只有BaseTool无状态,模型每次传完整列表。Execute:解析→补默认task→generatePlanOutput(格式化计划+进度统计+Important Reminder)→返回display_type:plan。★三类工具横向对比:外部检索型(调外部服务最复杂seenChunks跨轮复用)/思维辅助型(纯内存thoughtHistory+branches帮模型理清思路)/状态追踪型(无状态只格式化真状态在模型脑子)。★工具本质:工具是模型的"手和眼",手执行动作/眼获取信息/脑外挂帮模型分步思考,工具不替模型做决策只替模型干活+提供信息,模型是大脑工具是手脚。跟笔记32衔接(ExecuteTool 7步分发总机制vs具体工具Execute内部)

## 当前进度

✅ **已完成**:
- 阶段 1:架构与文档解析(笔记 01-02)
- 阶段 2:切块(笔记 03-20,完整覆盖 Tier 1/2/3 + 体检 + overlap + 保护区间 + 7500 硬切 + 父子分块 + 父子分块优化分析)—— **阶段 2 收尾,两条切分路线(普通 SplitText / 父子分块)都讲透,含优化方向**
- 阶段 3:向量化与入库(笔记 21)—— **阶段 3 闭环,入库 12 步全流程 + 二次 BatchIndex + 三态状态机**
- 后处理:知识图谱(笔记 22,Neo4j 存储)+ Wiki(笔记 23,存储与并发控制)—— **入库后 4 大异步后处理讲透 2 个(图谱/wiki),含图谱vs wiki 并发模型根本区别**
- 阶段 4:检索与对话(笔记 24-26)—— **RAG "读"侧闭环,检索 9 阶段全链路 + 对话历史/记忆召回/改写意图/数据库选型 + KnowledgeQA 载入历史细讲**
- Agent 路径上下文管理(笔记 27)—— **Agent 路径 4 道闸全讲透:闸①trim(含 3 盲区)+ 闸②redact(两层处理:落盘 CompactToolOutputForHistory + 装配 redact,8 个 KB 工具,占位符精确语义,跟 KnowledgeQA 丢 RenderedContent 对比)+ 闸③Consolidator(摘要 prompt 翻译)+ 闸④Compress(worked example)。stable prefix 机制讲透(prompt_cache.go 基础设施 + 3 个调用点 + wiki warmup 主动协调)。两层面拆分讲透(触发压缩是生存问题/保cache是效率问题,不该混在一起)。claude-code 对比讲透(冻结决策真正价值是保压缩成果不是保cache命中,WeKnora压缩触发早claude-code晚)。Q1/Q2/Q3 收敛到 Consolidator 摘要落盘(主价值省 LLM 调用)**
- Agent 路径 ReAct 循环骨架(笔记 28)—— **主循环骨架讲透:Execute 入口装配 4 样,executeLoop for 20 轮 + 3 种 iterOutcome 控制流转 + defer emitCompletion 保证恰好一次,runReActIteration 一轮 4 步(4 道闸/Think/stuck 检测/ctx 取消保 partial/Analyze/Act/Observe),4 类异常路径(ctx 取消 salvage/stuck loop/empty retry/content_filter)各自兜底,handleMaxIterations 跑满 20 轮合成答案,ctx 取消时不跑防 fallback 泄露 UI,appendToolResults 按 OpenAI 格式配对不拆。AgentEngine 跨轮无状态(DB-centered)。跟 KnowledgeQA 对比(单轮单次 vs 多轮 ReAct)。★用户偏好更新:技术讲解要专业术语+白话双轨(记进 memory/preferences.md)**
- Agent 路径 Think 阶段(笔记 29)—— **Think 阶段三层调用链讲透:callLLMWithRetry(外层入口,5 段:日志精简+消毒+调中层+isTransientError 重试 2 次线性退避+优雅降级)→ streamThinkingToEventBus(中坚层,3 段:装配 ChatOptions+ThinkStreamSplitter 分流 4 类+3 回调+answer 乐观渲染与撤回)→ streamLLMToEventBus(底层,4 步:120s chunk 间超时+贴 prompt cache 指纹+ChatStream 开流+StreamDecoder 多字节拼接+StreamError 单独存+Flush)。SanitizeMessages 消毒机制讲透(3 动作:跳空消息/合并连续同角色/孤儿 tool result 转 system,hasMatchingToolCall 往前找匹配,信息不丢只改角色)。跟 KnowledgeQA 对比(重试/降级/双路流式/消毒/超时/指纹全有 vs 全无)**
- Agent 路径 Act 阶段(笔记 30)—— **Act 阶段 4 函数讲透:executeToolCalls(外层入口,看 ParallelToolCalls 开关+工具数 ≥ 2 决定串并行)+ executeToolCallsParallel(errgroup 并发,3 关键设计:捕获循环变量防串号/按原序写入不按完成顺序/不连坐因工具独立)+ executeSingleToolCall(for 循环串行)+ runToolCall(核心 13 步:规范 ID/解析参数+JSON 修复两层兜底/formatToolHint UI 提示/开 langfuse span 双层记录/装 ToolExecContext ApprovalCtx 不带工具超时为 MCP 人审闸留后门/查 UnresolvedHandles 严格拒绝/执行+超时/装结果+关 span+三级日志)。formatToolHint 讲透(toolDisplayNames 19 工具中文名映射,toolHintSensitiveArgs 只 database_query 敏感,截断 40 字)。超时分流讲透(60s/shell_exec 10m5s)。ApprovalCtx 设计讲透(用 toolCtx 不用 toolExecCtx,审批等待不算 60s)。跟 Think 对比(有重试降级 vs 失败让模型换思路,设计哲学差异:对话 vs 干活)**
- Agent 路径 Approval 审批闸(笔记 31)—— **MCP 工具人审闸讲透:Gate 结构体(pending map/checker/10 分钟超时/Redis 可选/failClose 保守)+ waiter(ch 带缓冲/once 防重复/resolved 原子)+ Decision(4 状态+ModifiedArgs 改参数)。4 核心函数讲透:NeedsApproval(fail-close 查失败默认要审,环境变量可改放行)+ RequestAndWait(7 步+select 三路+超时取消自己 deliver 占 Once+emitResolved 用 WithoutCancel)+ Resolve(deliverLocal 三层校验:pendingID/租户/用户,找不到走 resolveCrossInstance)+ RequestOAuthAndWait(OAuth 变体不查 checker reactively 触发)。跨实例 Redis Pub/Sub 讲透:主频道+per-pending reply 频道+instanceID 防自处理+RequestNonce 防并发串 ack+runSubscriber 指数退避重连+resolveCrossInstance 完整流程(订阅 reply→发决议带 ReplyChannel→拥有者交付回 ack→发起者收 ack 返回 HTTP)。跟 mcp_tool.go 衔接讲透:waitCtx 用 ApprovalCtx 不带 60s+审批后重新派生 fresh ctx 给真正 CallTool 完整超时窗口。完整 delete_repo 例子讲透(含超时/取消/多副本异常路径)。跟 Think/Act 对比(阻塞时长/跨实例/决议来源差异)**
- Agent 路径工具注册表(笔记 32)—— **ToolRegistry 工具分发讲透:ToolRegistry 结构体(tools map+输出预算)+ types.Tool 接口(4 方法)+ BaseTool 公共实现。注册 first-wins 防劫持讲透(参考 GHSA-67q9-58vj-32qx,防恶意插件冒充内置工具)。GetFunctionDefinitions 排序保字节稳定讲透(Go map 随机+Qwen 字节级 cache 需要排序)。ExecuteTool 7 步讲透(GetTool+CastParams 类型转换+ValidateParams 校验+算预算+tool.Execute+TruncateToolOutput 截断+三级日志)。CastParams 讲透("true"转 true/按 JSON Schema 安全转换)。ValidateParams 讲透(required/type/enum/范围/长度,失败返回 nil err 是模型问题不是执行错误)。输出预算讲透(24000 rune 不是 byte,防中文过度截断,truncationMarkerReserve=200 留给标记,outputLimitProvider 工具自定义只放大不缩小)。toolErrorHint 讲透(三处加提示模型换思路)。三类错误区分讲透(工具找不到/参数校验失败/工具执行失败,执行错误是 panic,参数错是模型问题返回 nil err)。工具集清单讲透(19 内置工具+DefaultAllowedTools+maxFunctionNameLength=64+AvailableToolDefinitions)。KB 能力要求讲透(5 种能力+AnyOf/AllOf/ConsumesFiles+前后端同步+quick-answer 强制 vector/keyword+未知工具宽松)。授权层讲透(scope_authorization 后端兜底)。完整 knowledge_search 例子讲透(含异常路径)。跟 KnowledgeQA 对比(不调工具 vs 多工具分发)**
- Agent 路径工具案例细讲(笔记 33)—— **3 个典型工具三层结构(Definition/Content/Execute)细讲:①knowledge_search(外部检索型)10 步 pipeline(并发检索按 embedding model 分组→双 rerank(rerankModel→chatModel fallback,rerankWithLLM 15 个一批温度 0.1)→applyMMR lambda=0.7 Jaccard→deduplicateResults 多 key+内容签名→formatOutput XML+seenChunks 防重复省 token+retrieval_statistics 覆盖率统计),compositeScore=0.6*modelScore+0.3*baseScore+0.1*sourceWeight ②thinking(思维辅助型)纯内存不调外部服务,thoughtHistory+branches,动态调整 totalThoughts,分支思考回溯,incomplete_steps 标志 ③todo_write(状态追踪型)无状态模型每次传完整列表,generatePlanOutput 格式化计划+进度统计+Important Reminder,display_type:plan。三类工具横向对比讲透(外部检索型调外部服务最复杂 seenChunks 跨轮复用/思维辅助型纯内存帮模型理清思路/状态追踪型无状态只格式化真状态在模型脑子)。工具本质讲透(工具是模型的"手和眼",手执行动作/眼获取信息/脑外挂帮模型分步思考,工具不替模型做决策只替模型干活+提供信息,模型是大脑工具是手脚)。跟笔记 32 衔接(ExecuteTool 7 步分发总机制 vs 具体工具 Execute 内部)**

🚧 **待落盘**:
- 评估笔记 26 优化点②(KnowledgeQA 复用 Agent 压缩闸)看完 4 道闸 + cache 基础设施 + 两层面拆分后是否还成立
- 评估笔记 27 第九节收敛的优化方向(Consolidator 摘要落盘,主价值省 LLM 调用)要不要深挖成具体方案

⏳ **还没进行(按后续讲解顺序)**:

1. **Agent 路径其他点** —— Analyze 阶段(analyzeResponse 的 content_filter / natural stop / emptyContent 重试 nudge)/ Observe 阶段(appendToolResults 按 OpenAI 格式配对)/ 其他工具内部细讲(knowledge_search 已在笔记 33 讲透 / shell_exec 沙箱 / data_analysis DuckDB 流程 / grep_chunks / database_query / MCP OAuth 流程 getOrCreateMCPClientWithOAuthRetry 完整重试机制)/ image_requirement / skills / modelContext(笔记 28 讲循环骨架,笔记 29 讲 Think,笔记 30 讲 Act,笔记 31 讲 Approval,笔记 32 讲工具注册表,笔记 33 讲 3 个工具案例,这些还没展开)
2. **端到端走一个具体场景** —— 拿真实文档从体检到检索全跑一遍
3. 剩余后处理:摘要生成 / FAQ / 多模态(图片 OCR/Caption)/ 答案生成
4. 用户消化笔记 24-28 后的针对性问题

## 切分阶段两条路线(阶段 2 总结)

```
切分阶段(由 EnableParentChild 开关二选一):

  路线 A(EnableParentChild=false):普通分块
    └─ SplitText(text, cfg) → 内部走 Tier 1/2/3
    → 产出一套 chunk(都进向量库)

  路线 B(EnableParentChild=true):父子分块
    └─ SplitTextParentChild(text, parentCfg, childCfg)
         ├─ SplitText(text, parentCfg)        // 切 parent(4096)
         └─ 对每个 parent:
              └─ SplitText(parent, childCfg)  // 切 child(384)
    → 产出 parent(不进向量库)+ child(进向量库)
```

**关键**:两条路线都用 SplitText,区别只是切一次还是切两次。父子分块不是独立算法,是"在普通切分外面套一层"。**切点不由 size 定,封口由 size 定,选刀受 size 间接影响**——ChunkSize 只管装箱封口阈值,下刀靠 Tier1/2/3 的 separators/体检/保护区间/7500。详见笔记 19。

## 我们怎么学

- 每一步都从外到内,先看懂"这是啥",再下钻"怎么实现的"
- 每个笔记都用大白话,听不懂就让我重写
- 原始对话存 `sessions/`,整理后的笔记存 `notes/`
- 不直接改源码,只读和记;有歧义先查代码再下结论,不瞎点头