# 笔记 34 · MCP 和 Skill 怎么接入

> 接笔记 33(工具案例细讲)。笔记 32 讲 `ToolRegistry.ExecuteTool` 7 步分发的总机制,
> 笔记 33 讲 3 个内置工具案例,本篇讲两套**外部接入机制**(MCP / Skill)怎么进 ToolRegistry。
>
> 风格延续:专业术语 + 白话解释双轨。

---

## 一句话总结

**MCP 是"接外部能力的口子"**(动态连外部服务,运行时拉工具);
**Skill 是"内置能力的分册说明书"**(本地文件 + 沙箱,渐进加载指令)。
两套都通过 ToolRegistry 注册进同一个工具分发管道(笔记 32 的 ExecuteTool 7 步),
但**发现机制、执行路径、安全模型完全不同**。

---

## 一、MCP 接入:外部服务动态发现

**接入路径**:DB 配置 → client 建连 → ListTools 拉工具 → 注册进 registry → 运行时调用

### 1. 配置来源(DB)

MCP 服务存数据库,租户隔离。`agent_service.go:286` 拉服务列表:
- `config.MCPSelectionMode` 三种模式:`all`(全部启用)/ `selected`(按 ID 选)/ `none`(禁用)
- `selected` 模式按 `config.MCPServices` ID 列表查;`all` 模式 `ListMCPServices(tenantID)` 全拉
- 只留 `svc.Enabled == true` 的

### 2. 注册入口 `RegisterMCPTools`(mcp_tool.go:411)

每个启用的服务跑一遍:
```
for service := range enabledServices:
    1. getOrCreateMCPClientWithOAuthRetry  // 建连,带 OAuth 重试
    2. client.ListTools(30s 超时)          // 从 MCP server 拉工具清单
    3. 连接断了 → Disconnect + 重连 + 重试一次(非 stdio 才重试)
    4. for mcpTool := range mcpTools:
         tool := NewMCPTool(service, mcpTool, mcpManager, gate, authWait)
         registry.RegisterTool(tool)        // 注册进 ToolRegistry
```

**关键**:MCP 工具是**运行时动态发现**的——工具清单不在代码里,而是每次启 agent 时连 MCP server 拉 `ListTools` 才知道有哪些工具。

### 3. MCPTool 适配器(mcp_tool.go:20)

`MCPTool` 是个 wrapper,把 MCP 协议的工具包装成 `types.Tool` 接口:
- **Name**:`mcp_{serviceName}_{toolName}`(防冲突,带服务名前缀,超 64 字符截断)
- **Description**:带 `[MCP Service: xxx (external)]` 前缀,标明外部来源防 prompt injection
- **Parameters**:直接用 MCP server 给的 `InputSchema`
- **Execute**:走 `MCPManager` 调远程 MCP server 的 `CallTool`,接 Approval 闸(笔记 31)

### 4. 运行时调用

模型选了 `mcp_xxx_yyy` 工具 → `ExecuteTool` 7 步(笔记 32)→ `MCPTool.Execute` → Approval 闸(危险工具要人审)→ `mcpManager.CallTool` 调远程 → 结果回传

---

## 二、Skill 接入:文件系统静态发现 + Progressive Disclosure

**接入路径**:文件系统扫描 → 解析 SKILL.md → 元数据注入 system prompt → 模型按需 read_skill / execute_skill_script

### 1. 存储位置(文件系统)

Skill 是**文件夹**,每个 skill 一个目录,核心文件 `SKILL.md`(带 YAML frontmatter):
```
skills/
└── my-skill/
    ├── SKILL.md          # frontmatter(name+description)+ 正文 instructions
    ├── scripts/          # 可执行脚本
    │   └── analyze.py
    └── refs/             # 参考文件
        └── FORMS.md
```

也支持租户级 source(`TenantSkillSource`,从 DB 拉 skill 元数据)。

### 2. 发现机制 `Manager.Initialize`(manager.go:156)

```
1. DiscoverSkills() 扫所有 SkillDirs
2. 每个 SKILL.md 只读 frontmatter(YAML 头)→ SkillMetadata{name, description, basePath}
3. 缓存 metadata(Level 1,轻量)
```

**关键**:Skill 用 **Progressive Disclosure 三级加载**(skill.go:32):
- **Level 1 元数据**(name + description):启动时全加载,注入 system prompt
- **Level 2 指令**(SKILL.md 正文):模型 `read_skill` 时才按需加载
- **Level 3 资源**(目录里其他文件):模型 `read_skill(file_path=...)` 时才按需加载

### 3. 注入 system prompt(prompts.go:232)

`formatSkillsMetadata` 把所有 skill 的 Level 1 元数据拼成一段话塞进 system prompt,包含:
- "Skill Matching Protocol" 强制要求模型每个请求都扫一遍 skill 描述
- 匹配上就调 `read_skill` 加载 Level 2 指令
- 列出所有可用 skill 的 name + description

### 4. 注册三个 skill 相关工具(agent_service.go:414)

跟 MCP 不一样,**skill 工具是内置的静态工具**,在 `Initialize` 时就注册进 registry:
- `read_skill`(skill_read.go):模型按需加载 SKILL.md 正文或目录里其他文件
- `execute_skill_script`(skill_execute.go):在沙箱里跑 skill 自带的脚本(py/sh/js 等)
- `list_sandbox_files` / `read_sandbox_file`:沙箱文件浏览(也是 skill 体系的一部分)

**门控条件**:
- `config.SkillsEnabled == true` 才注册 `read_skill`
- `SkillsEnabled + sandbox 非禁用` 才注册 `execute_skill_script`(脚本要在沙箱里跑)
- `SkillsEnabled + SessionFileStore 能力` 才注册 `list/read_sandbox_files`

### 5. 运行时调用

- 模型看 system prompt 知道有啥 skill(Level 1 已注入)
- 匹配上 → 调 `read_skill(skill_name="xxx")` 加载 Level 2 指令
- 指令要跑脚本 → 调 `execute_skill_script(skill_name, script_path, args, input)` → `Manager.ExecuteScript`(manager.go:273)在沙箱里跑

---

## 三、两套机制对比(一张表)

| 维度 | MCP | Skill |
|---|---|---|
| **来源** | 外部 MCP server(网络/stdio) | 本地文件系统(或租户 DB source) |
| **工具发现** | 运行时 `ListTools` 动态拉 | 启动时 `DiscoverSkills` 静态扫 |
| **注册时机** | 每次启 agent 都重新连+注册 | 启动时注册 3 个固定内置工具 |
| **工具数量** | 不固定,取决于 server 暴露多少 | 固定 3 个(read/execute/list_sandbox) |
| **工具 schema** | server 给的 `InputSchema` | `BaseTool` 静态定义 |
| **能力暴露给模型** | 工具列表(让模型看到所有工具) | system prompt 注入 Level 1 元数据 + 模型按需 read_skill |
| **执行** | `MCPManager.CallTool` 调远程 | `Manager.ExecuteScript` 本地沙箱跑 |
| **人审闸** | 有(Approval gate,危险工具要审) | 无(沙箱本身做隔离) |
| **OAuth** | 有(`getOrCreateMCPClientWithOAuthRetry`) | 无 |
| **隔离** | 靠 MCP server 自己保证 | 靠 sandbox(网络禁、文件限 skill 目录) |
| **命名** | `mcp_{service}_{tool}` 动态拼 | `read_skill` / `execute_skill_script` 固定名 |
| **first-wins 防劫持** | 有(MCPTool 之间查冲突,笔记 32) | 有(ToolRegistry first-wins,但 skill 工具固定不会冲突) |
| **卸载** | stdio 连接用完就 Disconnect | 不需要(本地文件) |

---

## 四、本质区别

**MCP 是"接外部能力的口子"**:WeKnora 不可能预知第三方工具长啥样,所以做成动态发现——你配一个 MCP server,它暴露啥工具我就注册啥,运行时远程调用。口子开了就得加防线:Approval 闸(危险工具人审)+ OAuth(授权)+ first-wins(防冒名)+ `[external]` 前缀(防 prompt injection)。

**Skill 是"内置能力的分册说明书"**:能力是 WeKnora 自己的(读文件 + 沙箱跑脚本),但说明书分三级渐进加载——启动时只把目录(name+description)塞进 system prompt 省 token,模型用得上才 read_skill 读正文,要跑脚本才 execute_skill_script。本质是 **prompt engineering 的工程化**:把"什么时候该干啥"的指令拆成分册,按需喂给模型,避免一次性把所有指令塞进 system prompt 撑爆 context。

**一句话总结**:MCP 给模型装"外挂手"(动态连外部服务),Skill 给模型配"分册说明书"(渐进加载本地指令)。两套都通过 ToolRegistry 注册进同一个工具分发管道(笔记 32 的 ExecuteTool 7 步),但发现机制、执行路径、安全模型完全不同。

---

## 五、代码速查

| 文件 | 行号 | 作用 |
|---|---|---|
| `internal/agent/tools/mcp_tool.go` | 20 | `MCPTool` 结构体,wrapper 适配 MCP 工具 |
| `internal/agent/tools/mcp_tool.go` | 411 | `RegisterMCPTools` 注册入口 |
| `internal/application/service/agent_service.go` | 262 | `registerMCPTools` 拉 DB 服务列表 |
| `internal/application/service/agent_service.go` | 414 | skill 工具注册门控 |
| `internal/agent/skills/skill.go` | 32 | `Skill` 结构体 + Progressive Disclosure 三级 |
| `internal/agent/skills/manager.go` | 156 | `Manager.Initialize` 发现 skill |
| `internal/agent/skills/manager.go` | 273 | `Manager.ExecuteScript` 沙箱跑脚本 |
| `internal/agent/prompts.go` | 232 | `formatSkillsMetadata` 注入 system prompt |
| `internal/agent/tools/skill_read.go` | 17 | `read_skill` 工具定义 |
| `internal/agent/tools/skill_execute.go` | 17 | `execute_skill_script` 工具定义 |
| `internal/mcp/client.go` | 22 | `MCPClient` 接口 |

---

## 六、接续信息

- 笔记 32 讲 `ExecuteTool` 7 步分发的总机制
- 笔记 33 讲 3 个内置工具案例(knowledge_search / thinking / todo_write)
- 本篇讲 MCP 和 Skill 两套接入机制
- 下一篇笔记 35 深挖 MCP 工作原理(host / client / server 三角色 + 完整调用链 + 实际案例)

---

## 七、下次接续方向

1. **MCP 工作原理**深挖:host / client / server 三角色 + JSON-RPC 协议 + 三种传输(stdio/SSE/HTTP)→ 笔记 35
2. **MCP OAuth 流程**深挖:`getOrCreateMCPClientWithOAuthRetry` 完整重试机制
3. **Skill 沙箱执行**深挖:`Manager.ExecuteScript` 沙箱内部 + `WEKNORA_SKILL_OUTPUT_DIR` 产物收集
4. **Analyze 阶段**深挖:`analyzeResponse` 的 content_filter / natural stop / emptyContent 重试 nudge
5. **Observe 阶段**深挖:`appendToolResults` 按 OpenAI 格式配对

不主动继续,等用户提问。