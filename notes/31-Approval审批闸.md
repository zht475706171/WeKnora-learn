# 笔记 31 · Approval 审批闸(MCP 工具的人审闸)

> 接笔记 30(Act 阶段)。笔记 30 讲 `runToolCall` Step 9 提到 `ApprovalCtx` 不带工具超时为 MCP 人审闸留后门,
> 本篇展开这个后门:WeKnora 对"危险 MCP 工具"加了一道人审闸(human-in-the-loop,人在环路里),
> 模型要调这种工具时,先暂停等用户点"同意/拒绝",用户决定了才继续。
>
> 风格延续:专业术语 + 白话解释双轨。

---

## 一句话总结

**Approval 闸 = `Gate` 结构体**(approval/gate.go,审批闸门,管等待和决议的核心),
**`NeedsApproval` 判断要不要审**(查配置,这个工具是否需要人审),
**`RequestAndWait` 发事件 + 阻塞等决议**(发 `EventToolApprovalRequired` 给前端,select 等用户决定 / 超时 / ctx 取消),
**`Resolve` 交付决议**(用户点同意/拒绝后,前端调这个 API,Gate 把决议送给等待中的工具),
**跨实例靠 Redis Pub/Sub**(多个后端副本时,决议可能要跨实例传递)。
**OAuth 授权闸**(`RequestOAuthAndWait`)是变体:MCP 服务要 OAuth 授权时,让用户 midway 授权再继续。

---

## 一、为什么需要 Approval 闸

### 1.1 背景:危险工具要人审

MCP(Model Context Protocol,模型上下文协议)是 Anthropic 搞的标准,
让模型能调外部服务(GitHub / Slack / 数据库 / 文件系统等)。
有些 MCP 工具是**危险的**——比如 `delete_repo`(删仓库)、`send_email`(发邮件)、`drop_table`(删表),
模型自己决定调这些工具风险太高(模型可能误判或被 prompt injection 攻击),
所以要加**人审闸**(human-in-the-loop,人在环路里):模型要调这种工具,先暂停,等用户点"同意"才真执行。

### 1.2 设计挑战

1. **跨轮长时间等待**:人审批可能要几分钟(用户去喝咖啡了),不能占用工具的 60s 执行超时
2. **跨实例**:后端可能多副本部署(K8s 里 N 个 pod),用户的 HTTP 请求可能打到副本 A,但等待中的工具在副本 B,决议要跨实例传递
3. **超时和取消**:用户不点怎么办?超时。用户按 stop 怎么办?取消。这两种都要处理
4. **决议篡改**:用户可以"同意但改参数"(比如模型要删 100 条,用户改成删 10 条)
5. **幂等**:用户重复点同意(网络抖动重试),不能执行两次
6. **OAuth**:MCP 服务本身要 OAuth 授权(用户没登过 GitHub),要 midway 弹授权窗

---

## 二、Gate 结构体核心(approval/gate.go:135)

### 2.1 结构

```go
type Gate struct {
    mu        sync.Mutex
    pending   map[string]*waiter  ← 等待中的审批,按 pendingID 索引
    checker   Checker             ← 判断要不要审的接口
    timeout   time.Duration       ← 默认 10 分钟
    rdb       *redis.Client       ← 可选,跨实例用;nil 单实例模式
    failClose bool                ← true=查不了配置默认要审(保守),false=默认放行
}
```

**白话**:`Gate`(闸门)管所有等待中的审批。`pending` 是个 map,key 是 pendingID(每次审批的唯一 ID),value 是 `waiter`(等待者,含一个 channel 和身份信息)。

### 2.2 waiter 结构

```go
type waiter struct {
    ch       chan Decision   ← 决议 channel,带 1 缓冲
    tenantID uint64          ← 租户 ID,用于校验
    userID   string          ← 用户 ID,用于校验
    once     sync.Once       ← 保证只交付一次
    resolved atomic.Bool     ← 原子标志,是否已决议
}
```

**白话**:`waiter` 是"等待中的审批记录",`ch` 是个 channel(带 1 缓冲,发决议时不阻塞),`once` 保证只交付一次(防重复),`resolved` 是原子标志(读不用加锁,写要加锁)。

### 2.3 Decision 结构

```go
type Decision struct {
    Approved         bool            ← 是否同意
    ModifiedArgs     json.RawMessage ← 可选,同意但改参数
    Reason           string          ← 拒绝原因
    TimedOut         bool            ← 是否超时
    ContextCanceled  bool           ← 是否 ctx 取消
}
```

**白话**:`Decision`(决议)有 4 种状态:同意 / 拒绝(带原因)/ 超时 / ctx 取消。`ModifiedArgs` 是"同意但改参数"——用户可以改模型的调用参数(比如模型要删 100 条,用户改成删 10 条)。

---

## 三、四个核心函数

| 函数 | 定位 | 干什么 |
|---|---|---|
| `NeedsApproval` | 判断要不要审(gate.go:289) | 查 checker,看这个工具是否需要人审 |
| `RequestAndWait` | 等待决议(gate.go:309) | 发事件 + 阻塞等用户决定 / 超时 / ctx 取消 |
| `Resolve` | 交付决议(gate.go:539) | 用户点同意/拒绝后,前端调这个 API,Gate 把决议送给等待者 |
| `RequestOAuthAndWait` | OAuth 授权闸(gate.go:428) | 变体,MCP 服务要 OAuth 授权时,等用户授权 |

---

## 四、`NeedsApproval`:判断要不要审(gate.go:289)

```go
func (g *Gate) NeedsApproval(ctx, tenantID, serviceID, toolName) bool {
    if g == nil || g.checker == nil || tenantID == 0 || serviceID == "" || toolName == "" {
        return false  ← 任何条件不满足,直接放行
    }
    ok, err := g.checker.IsRequired(ctx, tenantID, serviceID, toolName)
    if err != nil {
        if g.failClose {
            return true  ← 查失败,保守要审
        }
        return false  ← 查失败,放行
    }
    return ok
}
```

**白话**:`NeedsApproval` 调 `checker.IsRequired`(查询配置,这个工具是否需要审)。

**关键设计:fail-close(保守失败)**:
- 查配置失败(DB 抖动)时,默认**要审**(`failClose=true`),不放行
- 因为审批是安全功能,查不了宁可多审一次,不能漏放
- 可以通过环境变量 `WEKNORA_AGENT_TOOL_APPROVAL_FAIL_OPEN=true` 改成放行模式( legacy 行为)

**白话**:查配置失败时,默认要审(保守,防危险工具漏放),可以通过环境变量改成放行(适合测试环境)。

---

## 五、`RequestAndWait`:发事件 + 阻塞等待(gate.go:309)

这是审批闸的核心,工具要调危险 MCP 工具时,先调这个函数阻塞等用户决定。

### 5.1 七步流程

```
RequestAndWait(ctx, req PendingRequest) (Decision, error):
  Step 1: 生成 pendingID(uuid)
  Step 2: 建 waiter,放进 g.pending map(注册等待者)
  Step 3: defer 从 map 删 pendingID(防泄漏)
  Step 4: 发 EventToolApprovalRequired 事件(告诉前端"要审批了")
  Step 5: 起 timer(10 分钟超时)
  Step 6: select 三路等待(用户决议 / 超时 / ctx 取消)
  Step 7: 发 EventToolApprovalResolved 事件(告诉前端"审批结束了")
```

### 5.2 Step 4:发事件给前端

```go
evtData := event.ToolApprovalRequiredData{
    PendingID:          pendingID,
    TenantID:           req.TenantID,
    SessionID:          req.SessionID,
    ServiceName:        req.ServiceName,
    MCPToolName:        req.MCPToolName,
    Description:        req.Description,
    Args:               argsObj,
    TimeoutSeconds:     timeoutSec,
    RequestedAtUnix:    time.Now().Unix(),
    ToolCallID:         req.ToolCallID,
    ...
}
req.EventBus.Emit(ctx, event.Event{
    Type: event.EventToolApprovalRequired,
    Data: evtData,
})
```

**白话**:发一个 `EventToolApprovalRequired`(工具审批需要)事件给前端,前端弹个框"AI 要调 `delete_repo` 工具,参数是 `{"repo": "xxx"}`,同意/拒绝?"。

### 5.3 Step 5-6:select 三路等待

```go
timer := time.NewTimer(g.timeout)  ← 10 分钟
defer timer.Stop()

var d Decision
select {
case d = <-w.ch:        ← 路径 A:用户决议(Resolve 送到 ch)
    emitResolved(d)
    return d, nil
case <-timer.C:         ← 路径 B:10 分钟超时
    d = Decision{Approved: false, Reason: "approval timeout", TimedOut: true}
    _ = w.deliver(d)    ← 自己给自己发决议(占住 once)
    d = <-w.ch          ← 等待 deliver 完成
    emitResolved(d)
    return d, nil
case <-ctx.Done():      ← 路径 C:ctx 取消(用户按 stop)
    d = Decision{Approved: false, Reason: "request canceled", ContextCanceled: true}
    _ = w.deliver(d)
    d = <-w.ch
    emitResolved(d)
    return d, nil
}
```

**三路 select**:
- **路径 A:用户决议**——`Resolve` 调 `w.deliver(d)`,决议写入 `w.ch`,这里读出来
- **路径 B:超时**——10 分钟到,自己构造一个"拒绝+超时"决议,自己 deliver(占住 once 防后续 Resolve 再投)
- **路径 C:ctx 取消**——用户按 stop,ctx.Done() 触发,自己构造"拒绝+取消"决议

**为什么超时和取消要 `w.deliver(d)` 再 `d = <-w.ch`**:
- `deliver` 用 `sync.Once` 保证只发一次。如果超时先发了,后续用户点同意也不会再进 ch
- 但 `deliver` 后还要 `<-w.ch` 读出来,因为 `deliver` 是写到 ch(带缓冲),要消费掉才能 return
- 这样保证 return 的就是那个超时/取消的决议,不会卡在 ch

### 5.4 `emitResolved` 用 `context.WithoutCancel(ctx)`

```go
emitResolved := func(d Decision) {
    _ = req.EventBus.Emit(context.WithoutCancel(ctx), event.Event{
        Type: event.EventToolApprovalResolved,
        ...
    })
}
```

**白话**:`WithoutCancel`(Go 1.21 新 API)创建一个不会因 ctx 取消而取消的 context。
即使 ctx 已经取消了(用户按 stop),也要发"审批结束"事件给前端(前端要清理弹框),
用 `WithoutCancel` 保证事件能发出去。

---

## 六、`Resolve`:交付决议(gate.go:539)

用户点同意/拒绝后,前端调 HTTP API,后端调 `Gate.Resolve`。

### 6.1 三路分支

```go
func (g *Gate) Resolve(tenantID, userID, pendingID, d Decision) error {
    switch err := g.deliverLocal(tenantID, userID, pendingID, d); {
    case err == nil:
        return nil  ← 本实例有等待者,交付成功
    case errors.Is(err, ErrTenantMismatch),
         errors.Is(err, ErrUserMismatch),
         errors.Is(err, ErrAlreadyResolved):
        return err  ← 错误:租户/用户不匹配,或已决议
    case errors.Is(err, ErrPendingNotFound):
        if g.rdb == nil {
            return err  ← 本地找不到,且没 Redis,真找不到
        }
        return g.resolveCrossInstance(tenantID, userID, pendingID, d)  ← 跨实例找
    }
}
```

**白话**:`Resolve` 先试本地交付(`deliverLocal`),找不到再跨实例找(`resolveCrossInstance`)。

### 6.2 `deliverLocal`:本实例交付(gate.go:638)

```go
func (g *Gate) deliverLocal(tenantID, userID, pendingID, d Decision) error {
    g.mu.Lock()
    w, ok := g.pending[pendingID]
    if !ok {
        return ErrPendingNotFound  ← 本实例没这个 pending
    }
    if w.tenantID != tenantID {
        return ErrTenantMismatch  ← 租户不匹配
    }
    if w.userID != "" && w.userID != userID {
        return ErrUserMismatch  ← 用户不匹配(防其他用户决议别人的审批)
    }
    g.mu.Unlock()
    
    if w.resolved.Load() {
        return ErrAlreadyResolved  ← 已决议(超时/取消先到)
    }
    if !w.deliver(d) {
        return ErrAlreadyResolved  ← deliver 失败(Once 已用)
    }
    return nil
}
```

**三层校验**:
1. **pendingID 存在**:本实例有这个等待者
2. **租户匹配**:发起审批的租户跟决议的租户一致(防跨租户越权)
3. **用户匹配**:发起审批的用户跟决议的用户一致(防 A 用户决议 B 用户的审批)

**白话**:本地交付要查三层(pendingID 在不在 / 租户对不对 / 用户对不对),都过了才发决议到 channel。

### 6.3 `resolveCrossInstance`:跨实例交付(gate.go:565)

多副本部署时,等待者可能在另一个实例。本实例找不到,通过 Redis Pub/Sub 跨实例找。

```go
func (g *Gate) resolveCrossInstance(tenantID, userID, pendingID, d Decision) error {
    nonce := uuid.New().String()  ← 防并发 Resolve 串 ack
    replyChannel := pubsubChannel() + ":reply:" + pendingID  ← 专用回复频道
    sub := g.rdb.Subscribe(context.Background(), replyChannel)
    defer sub.Close()
    
    // 先订阅回复频道,再发布决议
    sub.Receive(subCtx)  ← 等订阅活跃
    
    payload, _ := json.Marshal(resolveMessage{
        TenantID:     tenantID,
        PendingID:    pendingID,
        Approved:     d.Approved,
        ReplyChannel: replyChannel,  ← 告诉拥有者往哪个频道回 ack
        OriginID:     instanceID,    ← 标记发起者(防自己处理自己的消息)
        RequestNonce: nonce,         ← 防并发串 ack
    })
    g.rdb.Publish(pubCtx, pubsubChannel(), payload)  ← 发布到主频道
    
    // 等拥有者回 ack
    for {
        msg, err := sub.ReceiveMessage(ackCtx)  ← 3 秒超时
        if err != nil {
            return ErrPendingNotFound  ← 超时没回,真找不到
        }
        var ack resolveAck
        json.Unmarshal(msg.Payload, &ack)
        if ack.RequestNonce != nonce {
            continue  ← 不是我的 ack(并发 Resolve 串了),忽略
        }
        switch ack.Status {
        case "ok":           return nil
        case "tenant_mismatch": return ErrTenantMismatch
        case "user_mismatch":    return ErrUserMismatch
        case "already_resolved": return ErrAlreadyResolved
        case "not_found":        return ErrPendingNotFound
        }
    }
}
```

**白话**:跨实例流程:
1. 副本 A 收到 Resolve,本地找不到
2. 副本 A 订阅一个专用回复频道 `reply:pendingID`
3. 副本 A 往主频道 `weknora:mcp_approval:resolve` 发决议,带上"往回复频道回 ack"的请求
4. 副本 B(等待者所在实例)订阅主频道,收到决议,本地交付,往回复频道发 ack(带状态码)
5. 副本 A 从回复频道收 ack,根据状态码返回 HTTP 响应(200/404/409)

### 6.4 三个关键设计

**① `instanceID` 防自处理**:
```go
if m.OriginID == instanceID {
    continue  ← 跳过自己发的消息
}
```
**白话**:每个实例有个唯一 ID(`instanceID`,进程启动时生成),收到自己发的消息跳过(防本地噪音)。

**② `RequestNonce` 防并发串 ack**:
- 同一个 pendingID 可能有多个并发 Resolve(用户抖动重试)
- 每个 Resolve 用唯一 `nonce`,ack 带回 nonce,只有 nonce 匹配的 Resolve 才消费这个 ack
- **白话**:防 A 的 Resolve 消费 B 的 ack

**③ `ReplyChannel` per-pending**:
- 每个 pending 一个专用回复频道 `reply:pendingID`
- 不是共用一个回复频道,防不同 pending 的 ack 串

---

## 七、`runSubscriber`:订阅主频道(gate.go:212)

```go
func (g *Gate) runSubscriber() {
    ctx := context.Background()
    channel := pubsubChannel()
    backoff := time.Second
    const maxBackoff = 30 * time.Second
    for {
        sub := g.rdb.Subscribe(ctx, channel)
        ch := sub.Channel()
        backoff = time.Second  ← 订阅成功,重置 backoff
        for msg := range ch {
            var m resolveMessage
            json.Unmarshal([]byte(msg.Payload), &m)
            if m.OriginID == instanceID {
                continue  ← 跳过自己发的
            }
            err := g.deliverLocal(m.TenantID, m.UserID, m.PendingID, Decision{...})
            if m.ReplyChannel != "" {
                // 根据 err 决定 ack 状态码,回复给发起者
                ackPayload, _ := json.Marshal(resolveAck{
                    PendingID:    m.PendingID,
                    Status:       status,  ← ok / tenant_mismatch / user_mismatch / already_resolved
                    OriginID:     instanceID,
                    RequestNonce: m.RequestNonce,
                })
                g.rdb.Publish(pubCtx, m.ReplyChannel, ackPayload)
            }
        }
        _ = sub.Close()
        time.Sleep(backoff)  ← Redis 断了,指数退避重连
        backoff *= 2
        if backoff > maxBackoff {
            backoff = maxBackoff  ← 最多 30 秒
        }
    }
}
```

**白话**:每个实例启动一个 goroutine 订阅主频道,收到跨实例决议就本地交付,回复 ack。
Redis 断了用指数退避(1s/2s/4s/.../30s 封顶)重连。

---

## 八、`RequestOAuthAndWait`:OAuth 授权闸(gate.go:428)

### 8.1 跟 `RequestAndWait` 的区别

| 维度 | `RequestAndWait`(人审闸) | `RequestOAuthAndWait`(OAuth 授权闸) |
|---|---|---|
| 触发 | 模型要调危险工具,预先审 | MCP 服务返回"要 OAuth 授权",reactively 触发 |
| 判断 | `NeedsApproval` 查 checker | 不查 checker,直接由 transport 错误驱动 |
| 决议内容 | 同意/拒绝/改参数 | 授权完成/超时/取消 |
| 后续 | 同意则执行工具 | 授权完成则**重试**工具调用 |
| 事件类型 | `EventToolApprovalRequired` / `Resolved` | `EventMCPOAuthRequired` / `Resolved` |

**白话**:
- 人审闸是"模型要调危险工具,先问用户同意吗"
- OAuth 授权闸是"模型调工具,MCP 服务说'用户没授权',弹窗让用户授权,授权完重试"

### 8.2 流程

```
RequestOAuthAndWait(ctx, req OAuthPendingRequest) (Decision, error):
  Step 1: 生成 pendingID,注册 waiter
  Step 2: 发 EventMCPOAuthRequired 事件(告诉前端"要授权了")
  Step 3: select 三路等待(用户授权完成 / 超时 / ctx 取消)
  Step 4: 发 EventMCPOAuthResolved 事件
```

### 8.3 关键差异:`WaitTimeout` 可配

```go
waitTimeout := g.timeout  ← 默认 10 分钟
if req.WaitTimeout > 0 {
    waitTimeout = req.WaitTimeout  ← 可覆盖
}
```

**白话**:OAuth 授权可能比人审更慢(用户要跳去 GitHub 登录),可以让调用方自己设超时。

---

## 九、跟 mcp_tool.go 的衔接(mcp_tool.go:116-183)

这是审批闸真正接入工具执行的地方。

### 9.1 流程

```go
// MCPTool.Execute 里
if t.gate != nil {
    if meta, ok := ToolExecFromContext(ctx); ok && meta.EventBus != nil {
        tenantID, _ := types.TenantIDFromContext(ctx)
        if t.gate.NeedsApproval(ctx, tenantID, t.service.ID, t.mcpTool.Name) {
            // 关键:用 ApprovalCtx 不用 ctx,防 60s 工具超时打断审批
            waitCtx := ctx
            if meta.ApprovalCtx != nil {
                waitCtx = meta.ApprovalCtx
            }
            decision, waitErr := t.gate.RequestAndWait(waitCtx, approval.PendingRequest{...})
            
            if waitErr != nil || !decision.Approved {
                return &types.ToolResult{Success: false, Error: ...}, nil  ← 拒绝
            }
            
            if len(decision.ModifiedArgs) > 0 {
                args = decision.ModifiedArgs  ← 用户改了参数,用新参数
            }
            
            // 关键:审批可能耗尽 60s 预算,重新派生一个 fresh ctx
            if meta.ApprovalCtx != nil {
                freshTimeout := meta.ExecTimeout
                freshCtx, freshCancel := context.WithTimeout(meta.ApprovalCtx, freshTimeout)
                defer freshCancel()
                ctx = freshCtx
            }
        }
    }
}

// 真正调 MCP CallTool
result, err := connectAndCall(ctx, ...)
```

### 9.2 两个关键设计

**① 用 `ApprovalCtx` 等,不用 `ctx`**:
- `ctx` 带 60s 工具超时,审批可能要几分钟,会被打断
- `ApprovalCtx` 是 round 级 ctx,不带工具超时,审批可以等很久
- 用户按 stop 还是能传到 `ApprovalCtx`(它是 round ctx 的子),不会卡死

**② 审批后重新派生 fresh ctx**:
- 审批可能耗掉了 60s 工具超时的大部分,剩下的时间不够真正调 MCP
- 重新从 `ApprovalCtx` 派生一个带 `freshTimeout` 的新 ctx,给真正的 MCP CallTool 一个完整超时窗口
- **白话**:审批等待不算在 60s 内,但真正调工具还是 60s 超时

---

## 十、关键设计要点总结

### 10.1 fail-close(保守失败)

`NeedsApproval` 查配置失败时默认**要审**(不放行)。
**为什么**:审批是安全功能,查不了宁可多审一次,不能漏放危险工具。可通过环境变量改放行模式。

### 10.2 跨实例 Pub/Sub + per-pending reply channel

- 主频道 `weknora:mcp_approval:resolve`:发跨实例决议
- 回复频道 `weknora:mcp_approval:resolve:reply:<pendingID>`:per-pending 专用,防 ack 串
- `instanceID` 防自处理,`RequestNonce` 防并发串 ack

### 10.3 三路 select + sync.Once 防重复

```go
select {
case d = <-w.ch:        ← 用户决议
case <-timer.C:         ← 超时
case <-ctx.Done():      ← 取消
}
```

`waiter.deliver` 用 `sync.Once` 保证只交付一次,无论哪路先到,后续都不会再投。
超时和取消路自己 `deliver(d)` 占住 Once,防后续用户点同意再投。

### 10.4 ApprovalCtx 不带工具超时(接笔记 30)

`runToolCall` Step 9 装 `ToolExecContext` 时,`ApprovalCtx` 用 `toolCtx`(round 级)不用 `toolExecCtx`(带 60s)。
**为什么**:审批可能几分钟,60s 打断就坏了。审批不算在 60s 内,真正调工具还是 60s。

### 10.5 审批后重新派生 fresh ctx

审批可能耗掉 60s 预算,真正调 MCP 前从 `ApprovalCtx` 派生新 ctx,给完整超时窗口。

### 10.6 三层校验(租户 + 用户 + 已决议)

`deliverLocal` 三层校验:pendingID 存在 / 租户匹配 / 用户匹配。
防跨租户越权,防 A 决议 B 的审批,防重复决议。

### 10.7 emitResolved 用 WithoutCancel

即使 ctx 已取消(用户按 stop),也要发"审批结束"事件给前端清理弹框。
用 `context.WithoutCancel(ctx)` 保证事件能发出去。

### 10.8 OAuth 闸不查 checker

人审闸预先查 checker(配置驱动),OAuth 闸 reactively 触发(transport 错误驱动,不查 checker)。

---

## 十一、完整例子:用户调 `delete_repo` 工具

### 11.1 场景

用户问"帮我删掉 xxx 仓库",模型决定调 GitHub MCP 的 `delete_repo` 工具。
这个工具配置了"需要人审"。

### 11.2 流程

```
1. runReActIteration Step 3 → executeToolCalls → runToolCall
2. runToolCall Step 11 → toolRegistry.ExecuteTool → MCPTool.Execute
3. MCPTool.Execute 里:
   a. gate.NeedsApproval → true(配置说要审)
   b. waitCtx = meta.ApprovalCtx(不带 60s)
   c. gate.RequestAndWait(waitCtx, ...)
      - 生成 pendingID = "abc-123"
      - 发 EventToolApprovalRequired 事件
        (前端弹框:"AI 要调 delete_repo,参数 {repo: xxx},同意/拒绝?")
      - select 三路等待
4. 用户点"同意"(不改参数)
   - 前端调 HTTP API: POST /api/agent/approval/resolve
     body: {pending_id: "abc-123", approved: true}
   - 后端调 gate.Resolve(tenantID, userID, "abc-123", Decision{Approved: true})
   - deliverLocal 找到 waiter,deliver 到 ch
5. RequestAndWait 的 select 路径 A 触发,d = <-w.ch,return Decision{Approved: true}
6. MCPTool.Execute 收到 decision.Approved == true
   - 不改参数(decision.ModifiedArgs 为空)
   - 重新派生 fresh ctx(从 ApprovalCtx 派生,带 60s)
   - 调 connectAndCall(freshCtx)
7. MCP CallTool 真正执行 delete_repo
8. 返回结果给模型,模型继续 ReAct 循环
```

### 11.3 异常路径

**用户不点,10 分钟超时**:
- RequestAndWait select 路径 B 触发
- 自己 deliver 一个"拒绝+超时"决议
- MCPTool.Execute 收到 decision.Approved == false
- 返回 `ToolResult{Success: false, Error: "approval timeout"}`
- 模型看到"工具调用超时被拒",换思路

**用户按 stop**:
- ctx.Done() 触发(RequestAndWait select 路径 C)
- 自己 deliver"拒绝+取消"决议
- 同上,返回失败结果给模型

**多副本:用户 HTTP 打到副本 A,等待者在副本 B**:
- 副本 A 调 Resolve,本地找不到(ErrPendingNotFound)
- 走 resolveCrossInstance:订阅 reply 频道,发决议到主频道
- 副本 B 订阅主频道,收到决议,本地交付成功,回 ack "ok"
- 副本 A 收到 ack,返回 200 给前端

---

## 十二、跟笔记 30(Act 阶段)的衔接

- 笔记 30 讲 `runToolCall` Step 9 装 `ToolExecContext`,提到 `ApprovalCtx` 用 `toolCtx` 不带工具超时
- 本篇展开 `ApprovalCtx` 是给谁用、怎么用
- **Approval 闸的位置**:`MCPTool.Execute` 里,在 `runToolCall` 调 `toolRegistry.ExecuteTool` 之后,
  进入 `MCPTool.Execute` 之后,真正调 `client.CallTool` 之前
- 笔记 30 的 `runToolCall` 不直接调 Gate,是 `MCPTool.Execute` 里调

---

## 十三、跟 Think 阶段(笔记 29)、Act 阶段(笔记 30)的对比

| 维度 | Think(笔记 29) | Act(笔记 30) | Approval(笔记 31) |
|---|---|---|---|
| 阻塞性 | 流式输出,不阻塞 | 工具执行,阻塞但有超时 | 人审,阻塞可达 10 分钟 |
| 超时 | 120s chunk 间 | 60s 工具执行 | 10 分钟审批 |
| 重试 | isTransientError 2 次 | 无 | 无(等一次决议) |
| 取消 | ctx 取消保 partial | ctx 取消返回错误 | ctx 取消返回"拒绝+取消" |
| 跨实例 | 不涉及 | 不涉及 | Redis Pub/Sub |
| 决议来源 | 模型输出 | 工具返回 | 用户点击 |

**白话**:
- Think 跟模型对话(流式,快)
- Act 干活(工具执行,中等时间)
- Approval 等人(可能很久,跨实例,需要 Pub/Sub)

---

## 十四、代码速查

| 概念 | 位置 |
|---|---|
| `Gate` 结构体 | `D:/Project/WeKnora/internal/agent/approval/gate.go:135` |
| `waiter` 结构体 | `D:/Project/WeKnora/internal/agent/approval/gate.go:144` |
| `Decision` 结构体 | `D:/Project/WeKnora/internal/agent/approval/gate.go:76` |
| `PendingRequest` 结构体 | `D:/Project/WeKnora/internal/agent/approval/gate.go:85` |
| `OAuthPendingRequest` 结构体 | `D:/Project/WeKnora/internal/agent/approval/gate.go:109` |
| `NeedsApproval` | `D:/Project/WeKnora/internal/agent/approval/gate.go:289` |
| `RequestAndWait` | `D:/Project/WeKnora/internal/agent/approval/gate.go:309` |
| `RequestOAuthAndWait` | `D:/Project/WeKnora/internal/agent/approval/gate.go:428` |
| `Resolve` | `D:/Project/WeKnora/internal/agent/approval/gate.go:539` |
| `resolveCrossInstance` | `D:/Project/WeKnora/internal/agent/approval/gate.go:565` |
| `deliverLocal` | `D:/Project/WeKnora/internal/agent/approval/gate.go:638` |
| `runSubscriber` | `D:/Project/WeKnora/internal/agent/approval/gate.go:212` |
| `waiter.deliver` | `D:/Project/WeKnora/internal/agent/approval/gate.go:157` |
| `pubsubChannelBase` | `D:/Project/WeKnora/internal/agent/approval/gate.go:26` |
| `instanceID` | `D:/Project/WeKnora/internal/agent/approval/gate.go:30` |
| `MCPTool.Execute` 接入点 | `D:/Project/WeKnora/internal/agent/tools/mcp_tool.go:116-183` |
| `EventToolApprovalRequired` | `D:/Project/WeKnora/internal/event/` |
| `WEKNORA_AGENT_TOOL_APPROVAL_FAIL_OPEN` | 环境变量,改 fail-close 为 fail-open |
| `WEKNORA_REDIS_NAMESPACE` | 环境变量,Redis 频道命名空间隔离 |

---

## 十五、接续信息

本篇把 Approval 审批闸讲透,留的接续方向:

1. **Analyze 阶段**深挖:`analyzeResponse` 的 content_filter 处理 / natural stop 判断 / emptyContent 重试 nudge
2. **Observe 阶段**深挖:`appendToolResults` 按 OpenAI 格式配对(assistant 带 tool_calls + tool 带 ToolCallID)
3. **工具注册表**深挖:`toolRegistry.ExecuteTool` 内部怎么分发到具体工具(KnowledgeSearch / DatabaseQuery / ShellExec / MCPTool 等)
4. **modelContext** 深挖:`DecodeToolCalls` 解析临时句柄(cN/dN/bN/wN/iN/res://)的协议补丁
5. **MCP OAuth 流程**深挖:`getOrCreateMCPClientWithOAuthRetry` + `oauthSessionFromToolExec` 的完整 OAuth 重试机制
6. Agent 路径其他:`image_requirement.go` / skills / modelContext

不主动继续,等用户提问。