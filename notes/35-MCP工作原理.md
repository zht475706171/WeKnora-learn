# 笔记 35 · MCP 工作原理(host / client / server 三角色)

> 接笔记 34(MCP 和 Skill 接入)。本篇深挖 MCP 协议本身——三个角色各自干啥、
> 一次完整调用怎么跑、用一个实际案例串起来。
>
> 风格延续:专业术语 + 白话解释双轨。

---

## 一句话总结

**MCP(Model Context Protocol)是 LLM 跟外部工具通信的标准协议**,
把"Host 调外部能力"这件事标准化成 `Host ↔ Client ↔ Server` 三层架构。
**Host** 跑 LLM 是总管,**Client** 是 Host 雇的外交官(一个 Server 配一个),
**Server** 是独立干活的外部程序。三者通过 JSON-RPC 2.0 通信,
Host 不用知道 Server 内部咋实现,只管调 `ListTools` / `CallTool`。

**类比 USB 接口**:Host = 电脑(只认 USB 协议)/ Client = USB 控制器(管协议握手)/ Server = USB 设备(键盘/鼠标/摄像头,内部完全不同但都走 USB)。

---

## 一、三个角色是啥

### 1. Host(宿主)—— 跑 LLM 的那个程序

**白话**:Host 就是"装着大脑的那个应用"。

具体到 WeKnora,Host 就是 **WeKnora 后端服务本身**——它跑着 Agent 引擎,管着 LLM 调用、对话历史、工具注册表。Host 是**整个系统的总管**,决定啥时候调 LLM、啥时候用工具、结果怎么拼回去给用户。

### 2. Client(客户端)—— Host 里的"外交官"

**白话**:Client 是 Host 雇的翻译官,专门负责跟外部 Server 打交道。

WeKnora 里对应的就是 `mcp.MCPClient`(client.go:22 接口,`mcpGoClient` 结构体实现)。它干三件事:
- **Connect**:跟 Server 建立连接(stdio 管道 / SSE / HTTP 三种传输方式)
- **Initialize**:握手,确认双方协议版本、能力
- **ListTools / CallTool**:拉工具清单 / 调具体工具

**一个 Host 可以管多个 Client**——每接一个 MCP Server 就建一个 Client。WeKnora 里用 `MCPManager` 管这一堆 Client。

### 3. Server(服务端)—— 真正干活的那个程序

**白话**:Server 是"会干某件事的外部程序",独立于 Host 跑着,暴露自己的能力。

比如:
- GitHub MCP Server(会建仓库、提 PR、查 issue)
- Slack MCP Server(会发消息、拉频道)
- 数据库 MCP Server(会跑 SQL)
- 文件系统 MCP Server(会读写本地文件)

Server 是**别人写的、独立运行**的程序,Host 完全不知道它内部咋实现,只知道它暴露了啥工具。

---

## 二、三者关系(一张图)

```
┌─────────────────────────────────────────┐
│  Host(WeKnora 后端,跑 LLM 的总管)      │
│                                          │
│  ┌──────────────┐   ┌──────────────┐    │
│  │  Agent 引擎  │   │ ToolRegistry │    │
│  │  (调 LLM)    │   │ (工具注册表) │    │
│  └──────┬───────┘   └──────┬───────┘    │
│         │                  │            │
│         │            ┌─────┴─────┐      │
│         │            │ MCPManager│      │
│         │            └─────┬─────┘      │
│         │           ┌──────┼──────┐    │
│         │      Client A  Client B  Client C   ← 每个Server一个Client
│         │      (stdio)   (SSE)    (HTTP)
└─────────┼──────────┼───────┼────────┼─────┘
          │          │       │        │
      ┌───┴───┐  ┌───┴───┐ ┌───┴───┐ ┌───┴───┐
      │Server │  │Server │ │Server │ │Server │
      │  A    │  │  B    │ │  C    │ │  D    │
      │GitHub │  │Slack  │ │ DB    │ │ FS    │
      └───────┘  └───────┘ └───────┘ └───────┘
```

**核心关系**:
- Host 管着多个 Client,每个 Client 对接一个 Server
- Client 和 Server 之间走 MCP 协议(JSON-RPC 2.0)
- Host 不直接跟 Server 说话,永远通过 Client 转发
- Server 之间互相不知道,各自独立

---

## 三、MCP 协议核心(JSON-RPC 2.0)

### 传输层三种方式

| 传输 | 白话 | 适用场景 |
|---|---|---|
| **stdio** | 子进程,通过 stdin/stdout 管道通信 | 本地 MCP server(跟 Host 同机器) |
| **SSE** | Server-Sent Events,HTTP 长连接 | 远程 MCP server,单向流 |
| **HTTP** | 普通 HTTP 请求响应 | 远程 MCP server,无状态 |

WeKnora 用 `mark3labs/mcp-go` 库实现,`mcpGoClient`(client.go:67)包装这个库。

### 协议握手流程

```
Client                          Server
  │                               │
  │── Initialize(协议版本,能力) ──→│
  │                               │
  │←── InitializeResult(能力) ────│
  │                               │
  │── Initialized(确认) ─────────→│
  │                               │
  │── ListTools() ───────────────→│
  │                               │
  │←── [tool1, tool2, ...] ───────│
  │                               │
  │── CallTool(name, args) ──────→│
  │                               │
  │←── CallToolResult ────────────│
```

握手完成后,Client 可以反复调 `ListTools` / `CallTool`,不用重新握手。

---

## 四、一次完整调用是怎么跑的

**前提**:WeKnora 配了一个 GitHub MCP Server,里面有个 `create_repo` 工具。

### 阶段 1:启动时注册(笔记 32 讲过)

1. WeKnora 启动 agent → `registerMCPTools`(agent_service.go:262)拉启用的 MCP 服务
2. `MCPManager` 给 GitHub Server 建一个 `Client`(stdio 或 SSE 连接)
3. `Client.Initialize()` 握手:确认协议版本、交换能力清单
4. `Client.ListTools()` 拉 GitHub Server 暴露的工具列表 → `[create_repo, list_repos, create_issue, ...]`
5. 每个工具包成 `MCPTool` 注册进 `ToolRegistry`(笔记 32 的 first-wins)

### 阶段 2:对话中调用

6. 用户说"帮我建个叫 demo 的仓库"
7. LLM 思考后决定调 `mcp_github_create_repo` 工具,参数 `{name: "demo"}`
8. `ExecuteTool` 7 步(笔记 32)→ `MCPTool.Execute`(mcp_tool.go:103)
9. **Approval 闸**(笔记 31):GitHub `create_repo` 是危险操作,弹框问用户
10. 用户点"同意"
11. `MCPTool.Execute` → `Client.CallTool("create_repo", {name: "demo"})`
12. Client 通过 MCP 协议把请求发给 GitHub Server
13. GitHub Server 内部调 GitHub API 真建仓库
14. Server 返回结果给 Client → Client 返回给 `MCPTool` → 返回给 `ToolRegistry` → 返回给 Agent
15. Agent 把结果喂给 LLM,LLM 生成"仓库 demo 建好了"回给用户

---

## 五、实际案例:接 GitHub MCP Server 帮用户建仓库

### 配置阶段(管理员干一次)

管理员在 WeKnora 后台配一个 MCP 服务:
- 名字:`github`
- 传输:`stdio`(本地跑)或 `sse`(远程连)
- 命令:`npx -y @modelcontextprotocol/server-github`
- 环境变量:`GITHUB_TOKEN=ghp_xxx`
- 启用 + 存 DB

### 启动阶段(每次启 agent)

```
agent_service.registerMCPTools
  → ListMCPServices(tenantID) 拉到 [github]
  → RegisterMCPTools(mcp_tool.go:411)
       → getOrCreateMCPClientWithOAuthRetry  // 建 Client
            → stdio 连接:启动 npx 进程,通过 stdin/stdout 通信
       → Client.Initialize()                  // 握手
       → Client.ListTools()                   // 拉 [create_repo, ...]
       → 每个 mcpTool 包成 MCPTool 注册进 registry
```

### 调用阶段(用户对话)

```
用户:帮我建个叫 demo 的仓库
  ↓
LLM:我要调 mcp_github_create_repo(name="demo")
  ↓
ToolRegistry.ExecuteTool 7 步
  ↓
MCPTool.Execute(mcp_tool.go:103)
  ↓
Approval 闸:NeedsApproval=true → 弹框(笔记 31)
  ↓
用户点"同意"
  ↓
Client.CallTool("create_repo", {name: "demo"})
  ↓ (MCP 协议,JSON-RPC over stdio)
GitHub MCP Server 收到请求
  ↓
Server 内部调 GitHub API: POST /user/repos
  ↓
GitHub 真建了仓库
  ↓
Server 返回 {repo_url: "https://github.com/user/demo"}
  ↓
Client 收到 → 返回 MCPTool → 返回 ToolRegistry → 返回 Agent
  ↓
Agent 喂给 LLM:"工具返回了 repo_url=..."
  ↓
LLM:仓库建好了,地址是 https://github.com/user/demo
  ↓
用户看到回答
```

---

## 六、为啥要搞三个角色,不直接 Host 调 Server?

**核心原因:解耦 + 标准化**。

如果 Host 直接调每个 Server,每接一个新服务就得写一套适配代码——GitHub 一套、Slack 一套、数据库一套,接口全不一样,Host 会膨胀成怪物。

**MCP 协议把这层标准化了**:
- Server 只要按 MCP 协议暴露 `ListTools` / `CallTool`,内部咋实现随便
- Host 只要会调 `Client.ListTools` / `Client.CallTool`,不用关心 Server 内部
- Client 是协议层,翻译 Host 的请求成 MCP 格式,翻译 Server 的响应回 Host 格式

**类比 USB 接口**:
- Host = 电脑(只认 USB 协议)
- Client = USB 控制器(管协议握手)
- Server = USB 设备(键盘 / 鼠标 / 摄像头,内部完全不同,但都走 USB)

电脑不用知道连的是罗技键盘还是微软鼠标,USB 协议把它们都标准化了。MCP 一样,WeKnora 不用知道连的是 GitHub 还是 Slack,MCP 协议把它们都标准化了。

---

## 七、MCP 协议的三类原语

MCP 不只 `ListTools` / `CallTool`,完整协议有三大类原语:

| 原语 | 作用 | WeKnora 用到 |
|---|---|---|
| **Tools** | 工具调用(模型主动调) | ✅ `ListTools` / `CallTool`(client.go:33/39) |
| **Resources** | 资源读取(模型读外部数据) | ✅ `ListResources` / `ReadResource`(client.go:36/42) |
| **Prompts** | 提示词模板(预定义 prompt) | ❌ WeKnora 没用 |

**Tools vs Resources 区别**:
- **Tools**:模型主动调,有副作用(建仓库 / 发消息 / 改数据)
- **Resources**:模型读,无副作用(读文件 / 查配置 / 拉列表)

类比 REST API:Tools 像 POST(有副作用),Resources 像 GET(只读)。

---

## 八、三种传输方式对比

| 传输 | 连接方式 | 生命周期 | 适用场景 | WeKnora 处理 |
|---|---|---|---|---|
| **stdio** | 子进程 stdin/stdout 管道 | 跟子进程同生命周期 | 本地 server(跟 Host 同机器) | 用完 `Disconnect`(mcp_tool.go:458)杀子进程 |
| **SSE** | HTTP 长连接(Server-Sent Events) | 持续连接 | 远程 server,实时流 | 长连接复用,不每次断 |
| **HTTP** | 普通 HTTP 请求响应 | 每次请求独立 | 远程 server,无状态 | 每次请求建连 |

**stdio 的特殊性**:每次启 agent 建子进程,用完得杀(不然僵尸进程)。mcp_tool.go:458 的 `defer client.Disconnect()` 就是干这个的。

---

## 九、跟 Skill 的本质区别(顺带提一句)

- **MCP**:外部独立程序,Host 通过网络/stdio 调,**能力在 Server 里**
- **Skill**:本地文件 + 沙箱脚本,Host 自己跑,**能力在 Host 里**(只是指令分册加载)

MCP 是"接外挂",Skill 是"翻分册说明书"。

---

## 十、代码速查

| 文件 | 行号 | 作用 |
|---|---|---|
| `internal/mcp/client.go` | 22 | `MCPClient` 接口(Connect/Initialize/ListTools/CallTool) |
| `internal/mcp/client.go` | 67 | `mcpGoClient` 包装 mark3labs/mcp-go 库 |
| `internal/agent/tools/mcp_tool.go` | 20 | `MCPTool` 适配器,实现 `types.Tool` 接口 |
| `internal/agent/tools/mcp_tool.go` | 103 | `MCPTool.Execute` 调 `Client.CallTool` |
| `internal/agent/tools/mcp_tool.go` | 411 | `RegisterMCPTools` 注册入口 |
| `internal/agent/tools/mcp_oauth.go` | - | `getOrCreateMCPClientWithOAuthRetry` OAuth 重试 |
| `internal/application/service/agent_service.go` | 262 | `registerMCPTools` 拉 DB 服务列表 |
| `internal/types/mcp.go` | - | `MCPService` / `MCPTool` 类型定义 |

---

## 十一、接续信息

- 笔记 32 讲 `ExecuteTool` 7 步分发的总机制
- 笔记 33 讲 3 个内置工具案例
- 笔记 34 讲 MCP 和 Skill 两套接入机制对比
- 本篇深挖 MCP 工作原理(host / client / server 三角色 + 完整调用链 + 实际案例)

---

## 十二、下次接续方向

1. **MCP OAuth 流程**深挖:`getOrCreateMCPClientWithOAuthRetry` 完整重试机制 + `oauthSessionFromToolExec` + token store
2. **Skill 沙箱执行**深挖:`Manager.ExecuteScript` 沙箱内部 + `WEKNORA_SKILL_OUTPUT_DIR` 产物收集 + 网络禁用 + 文件隔离
3. **Analyze 阶段**深挖:`analyzeResponse` 的 content_filter / natural stop / emptyContent 重试 nudge
4. **Observe 阶段**深挖:`appendToolResults` 按 OpenAI 格式配对
5. **其他工具内部**深挖:shell_exec 沙箱 / data_analysis DuckDB 流程

不主动继续,等用户提问。