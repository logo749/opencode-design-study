# OpenCode 架构与代码原理文档

## 目录

1. [概述](#概述)
2. [技术栈](#技术栈)
3. [系统架构](#系统架构)
4. [核心模块详解](#核心模块详解)
5. [Agent 系统](#agent-系统)
6. [Tool 系统](#tool-系统)
7. [Session 管理](#session-管理)
8. [MCP 集成](#mcp-集成)
9. [Provider 系统](#provider-系统)
10. [配置系统](#配置系统)
11. [权限系统](#权限系统)
12. [插件系统](#插件系统)
13. [数据流](#数据流)
14. [设计原则](#设计原则)

---

## 概述

OpenCode 是一个基于 **Bun 运行时** 的 AI 编程助手框架，采用模块化架构设计，支持多 LLM Provider、多 Agent 协作、可扩展 Tool 系统和 MCP（Model Context Protocol）协议。

**核心特性**：
- 🤖 多 Agent 系统（Build、Plan、Explore、General）
- 🔧 20+ 内置工具 + MCP 工具扩展
- 💬 Session 管理与上下文压缩
- 🔌 插件系统与技能扩展
- 🌐 75+ LLM Provider 支持
- 📱 CLI + TUI + Web + Desktop 多端支持

---

## 技术栈

| 层次 | 技术 |
|------|------|
| **运行时** | Bun |
| **后端框架** | Hono（HTTP 服务器） |
| **前端框架** | SolidJS |
| **桌面应用** | Tauri |
| **数据库** | SQLite + Drizzle ORM |
| **LLM 集成** | AI SDK（Vercel） |
| **协议支持** | MCP、ACP、LSP |
| **CLI** | yargs、@clack/prompts |
| **构建工具** | TypeScript |

---

## 系统架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        OpenCode 系统                            │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │   CLI/TUI   │  │   Web UI    │  │  Desktop    │            │
│  │  (SolidJS)  │  │  (Astro)    │  │  (Tauri)    │            │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘            │
│         │                │                │                     │
│         └────────────────┴────────────────┘                     │
│                          │                                      │
│         ┌────────────────▼────────────────┐                     │
│         │      @opencode-ai/sdk           │                     │
│         │    (HTTP + SSE 通信层)            │                     │
│         └────────────────┬────────────────┘                     │
│                          │                                      │
│  ┌───────────────────────▼────────────────────────────────┐    │
│  │                  Core Backend                          │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐            │    │
│  │  │  Agent   │  │  Session │  │   Tool   │            │    │
│  │  │  System  │  │ Manager  │  │ Registry │            │    │
│  │  └──────────┘  └──────────┘  └──────────┘            │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐            │    │
│  │  │ Provider │  │   MCP    │  │ Permission│            │    │
│  │  │ Manager  │  │ Gateway  │  │  System  │            │    │
│  │  └──────────┘  └──────────┘  └──────────┘            │    │
│  └────────────────────────────────────────────────────────┘    │
│                          │                                      │
│         ┌────────────────▼────────────────┐                     │
│         │        SQLite Database          │                     │
│         │      (Drizzle ORM + Migrations) │                     │
│         └─────────────────────────────────┘                     │
└─────────────────────────────────────────────────────────────────┘
```

### 目录结构

```
packages/opencode/
├── bin/                          # CLI 入口可执行文件
├── migration/                    # 数据库迁移（Drizzle）
├── script/                       # 构建脚本
├── src/                          # 主源代码目录
│   ├── agent/                    # Agent 系统
│   │   ├── agent.ts              # 核心 Agent 实现
│   │   ├── generate.txt          # 生成提示词
│   │   └── prompt/               # Agent 提示词模板
│   ├── tool/                     # Tool 系统
│   │   ├── edit.ts               # 文件编辑工具（20KB）
│   │   ├── read.ts               # 文件读取工具
│   │   ├── bash.ts               # Shell 命令执行
│   │   ├── grep.ts / glob.ts     # 文件搜索
│   │   ├── apply_patch.ts        # Patch 应用
│   │   ├── batch.ts              # 批量操作
│   │   ├── codesearch.ts         # 网络代码搜索
│   │   ├── lsp.ts                # LSP 集成
│   │   ├── task.ts               # 任务委托
│   │   ├── plan.ts               # 规划工具
│   │   ├── multiedit.ts          # 多文件编辑
│   │   └── registry.ts           # Tool 注册表
│   ├── session/                  # Session 管理
│   │   ├── index.ts              # Session 编排器（27KB）
│   │   ├── llm.ts                # LLM 交互层
│   │   ├── message.ts            # 消息处理
│   │   ├── message-v2.ts         # 消息 V2 格式
│   │   ├── processor.ts          # 消息处理器
│   │   ├── prompt.ts             # 提示词工程（66KB）
│   │   ├── compaction.ts         # 上下文压缩
│   │   ├── revert.ts             # Session 回滚
│   │   ├── summary.ts            # Session 摘要
│   │   └── session.sql.ts        # 数据库 Schema
│   ├── cli/                      # CLI 命令
│   │   ├── cmd/                  # 20+ CLI 命令
│   │   │   ├── run.ts            # 主 Agent 执行
│   │   │   ├── generate.ts       # 代码生成
│   │   │   ├── auth.ts           # 认证
│   │   │   ├── serve.ts          # Web 服务器
│   │   │   ├── mcp.ts            # MCP 管理
│   │   │   └── ...
│   │   ├── ui.ts                 # CLI UI 组件
│   │   └── error.ts              # 错误格式化
│   ├── provider/                 # LLM Provider
│   │   ├── provider.ts           # Provider 管理
│   │   ├── models.ts             # 模型定义
│   │   └── transform.ts          # Provider 转换
│   ├── mcp/                      # MCP 集成
│   │   ├── index.ts              # MCP 核心（30KB）
│   │   ├── auth.ts               # MCP 认证
│   │   ├── oauth-provider.ts     # OAuth Provider
│   │   └── oauth-callback.ts     # OAuth 回调
│   ├── config/                   # 配置管理
│   │   ├── config.ts             # 配置加载与解析
│   │   ├── paths.ts              # 配置路径
│   │   └── markdown.ts           # Markdown 配置解析
│   ├── storage/                  # 存储层
│   │   ├── db.ts                 # 数据库客户端
│   │   └── storage.ts            # 文件存储
│   ├── server/                   # 服务器组件
│   ├── auth/                     # 认证系统
│   ├── project/                  # 项目管理
│   ├── permission/               # 权限系统
│   ├── plugin/                   # 插件系统
│   ├── skill/                    # 技能系统
│   ├── snapshot/                 # 快照管理
│   ├── patch/                    # Patch 管理
│   ├── shell/                    # Shell 执行
│   ├── ide/                      # IDE 集成
│   ├── question/                 # 问题处理
│   ├── scheduler/                # 任务调度
│   ├── flag/                     # 特性开关
│   └── util/                     # 工具函数
├── test/                         # 测试文件
├── package.json                  # 依赖与脚本
├── tsconfig.json                 # TypeScript 配置
├── drizzle.config.ts             # 数据库配置
├── bunfig.toml                   # Bun 运行时配置
└── Dockerfile                    # 容器配置
```

---

## 核心模块详解

### 1. 入口点 (`src/index.ts`)

**功能**：CLI 应用主入口，使用 yargs 进行命令解析

**核心流程**：
```typescript
import yargs from "yargs"
import { RunCommand } from "./cli/cmd/run"
import { GenerateCommand } from "./cli/cmd/generate"
// ... 20+ 命令

let cli = yargs(hideBin(process.argv))
  .scriptName("opencode")
  .command(RunCommand)
  .command(GenerateCommand)
  // ... 注册所有命令
  .middleware(async (opts) => {
    // 初始化日志系统
    await Log.init({ print: process.argv.includes("--print-logs") })
    
    // 数据库迁移
    await JsonMigration.run(Database.Client().$client, { progress: ... })
  })

await cli.parse()
```

**特点**：
- 支持 20+ CLI 命令
- 中间件处理日志初始化和数据库迁移
- 统一的错误处理和格式化输出

---

### 2. Agent 系统 (`src/agent/`)

**核心文件**：`src/agent/agent.ts`

**Agent 类型定义**：
```typescript
export const Info = z.object({
  name: z.string(),
  description: z.string().optional(),
  mode: z.enum(["subagent", "primary", "all"]),
  native: z.boolean().optional(),
  hidden: z.boolean().optional(),
  topP: z.number().optional(),
  temperature: z.number().optional(),
  color: z.string().optional(),
  permission: PermissionNext.Ruleset,
  model: z.object({
    modelID: z.string(),
    providerID: z.string(),
  }).optional(),
  prompt: z.string().optional(),
  options: z.record(z.string(), z.any()),
  steps: z.number().int().positive().optional(),
})
```

**内置 Agent**：

| Agent | Mode | 描述 | 权限特点 |
|-------|------|------|----------|
| **build** | primary | 默认 Agent，执行配置的工具 | 允许 question、plan_enter |
| **plan** | primary | 规划模式，禁止编辑工具 | 仅允许 plan_exit |
| **general** | subagent | 通用 Agent，处理复杂问题 | 禁止 todoread/todowrite |
| **explore** | subagent | 快速探索代码库 | 只读工具（grep、glob、read） |
| **compaction** | primary | 上下文压缩（隐藏） | 禁止所有工具 |
| **title** | primary | 生成会话标题（隐藏） | 禁止所有工具 |
| **summary** | primary | 生成会话摘要（隐藏） | 禁止所有工具 |

**Agent 状态管理**：
```typescript
const state = Instance.state(async () => {
  const cfg = await Config.get()
  
  // 默认权限规则
  const defaults = PermissionNext.fromConfig({
    "*": "allow",
    doom_loop: "ask",
    external_directory: { "*": "ask" },
    question: "deny",
    plan_enter: "deny",
    plan_exit: "deny",
    read: { "*": "allow", "*.env": "ask", "*.env.*": "ask" },
  })
  
  // 合并用户配置
  const user = PermissionNext.fromConfig(cfg.permission ?? {})
  
  const result: Record<string, Info> = {
    build: { /* ... */ },
    plan: { /* ... */ },
    general: { /* ... */ },
    explore: { /* ... */ },
  }
  
  // 合并自定义 Agent
  for (const [key, value] of Object.entries(cfg.agent ?? {})) {
    // ...
  }
  
  return result
})
```

**Agent 生成（动态创建）**：
```typescript
export async function generate(input: {
  description: string
  model?: { providerID: string; modelID: string }
}) {
  const defaultModel = input.model ?? (await Provider.defaultModel())
  const model = await Provider.getModel(defaultModel.providerID, defaultModel.modelID)
  
  const result = await generateObject({
    messages: [
      { role: "system", content: PROMPT_GENERATE },
      { role: "user", content: `Create agent: "${input.description}"` }
    ],
    model,
    schema: z.object({
      identifier: z.string(),
      whenToUse: z.string(),
      systemPrompt: z.string(),
    }),
  })
  
  return result.object
}
```

---

### 3. Tool 系统 (`src/tool/`)

**核心文件**：`src/tool/registry.ts`

**Tool 注册表**：
```typescript
export const state = Instance.state(async () => {
  const custom = [] as Tool.Info[]
  
  // 加载自定义工具（从 {tool,tools}/*.{js,ts}）
  const matches = await Glob.scanSync("{tool,tools}/*.{js,ts}", { cwd: dir })
  for (const match of matches) {
    const mod = await import(pathToFileURL(match).href)
    for (const [id, def] of Object.entries<ToolDefinition>(mod)) {
      custom.push(fromPlugin(id, def))
    }
  }
  
  // 加载插件工具
  const plugins = await Plugin.list()
  for (const plugin of plugins) {
    for (const [id, def] of Object.entries(plugin.tool ?? {})) {
      custom.push(fromPlugin(id, def))
    }
  }
  
  return { custom }
})
```

**内置工具列表**：
```typescript
async function all(): Promise<Tool.Info[]> {
  return [
    InvalidTool,
    QuestionTool,           // 向用户提问
    BashTool,               // Shell 命令执行
    ReadTool,               // 文件读取
    GlobTool,               // 文件模式匹配
    GrepTool,               // 内容搜索
    EditTool,               // 文件编辑
    WriteTool,              // 文件写入
    TaskTool,               // 子任务委托
    WebFetchTool,           // 网页抓取
    TodoWriteTool,          // TODO 管理
    WebSearchTool,          // 网络搜索
    CodeSearchTool,         // 代码搜索
    SkillTool,              // 技能调用
    ApplyPatchTool,         // Patch 应用
    LspTool,                // LSP 工具（实验性）
    BatchTool,              // 批量操作（实验性）
    PlanExitTool,           // 退出规划模式
    ...custom,              // 自定义工具
  ]
}
```

**Tool 状态生命周期**：
```typescript
export type ToolStatePending = {
  status: "pending"
  input: Record<string, unknown>
  raw: string
}

export type ToolStateRunning = {
  status: "running"
  input: Record<string, unknown>
  title?: string
  metadata?: Record<string, unknown>
  time: { start: number }
}

export type ToolStateCompleted = {
  status: "completed"
  input: Record<string, unknown>
  output: string
  title: string
  metadata: Record<string, unknown>
  time: { start: number; end: number }
  attachments?: Array<FilePart>
}

export type ToolStateError = {
  status: "error"
  input: Record<string, unknown>
  error: string
  time: { start: number; end: number }
}
```

**Tool 执行流程**：
```typescript
// 1. 工具调用
const tool = await t.init({ agent })

// 2. 执行前钩子
await Plugin.trigger("tool.execute.before", {
  tool: t.id,
  sessionID,
  callID,
}, { args })

// 3. 执行工具
const result = await tool.execute(args, ctx)

// 4. 执行后钩子
await Plugin.trigger("tool.execute.after", {
  tool: t.id,
  sessionID,
  callID,
  args,
}, { title: result.title, output: result.output })

// 5. 输出截断（防止 token 溢出）
const out = await Truncate.output(result, {}, agent)
```

---

### 4. Session 管理 (`src/session/`)

**核心文件**：`src/session/index.ts`（27KB）

**Session Schema**：
```typescript
export const Info = z.object({
  id: Identifier.schema("session"),
  slug: z.string(),
  projectID: z.string(),
  workspaceID: z.string().optional(),
  directory: z.string(),
  parentID: Identifier.schema("session").optional(),
  summary: z.object({
    additions: z.number(),
    deletions: z.number(),
    files: z.number(),
    diffs: Snapshot.FileDiff.array().optional(),
  }).optional(),
  share: z.object({ url: z.string() }).optional(),
  title: z.string(),
  version: z.string(),
  time: z.object({
    created: z.number(),
    updated: z.number(),
    compacting: z.number().optional(),
    archived: z.number().optional(),
  }),
  permission: PermissionNext.Ruleset.optional(),
  revert: z.object({
    messageID: z.string(),
    partID: z.string().optional(),
    snapshot: z.string().optional(),
    diff: z.string().optional(),
  }).optional(),
})
```

**Session 创建**：
```typescript
export async function createNext(input: {
  id?: string
  title?: string
  parentID?: string
  directory: string
  permission?: PermissionNext.Ruleset
}) {
  const result: Info = {
    id: Identifier.descending("session", input.id),
    slug: Slug.create(),
    version: Installation.VERSION,
    projectID: Instance.project.id,
    directory: input.directory,
    workspaceID: WorkspaceContext.workspaceID,
    parentID: input.parentID,
    title: input.title ?? createDefaultTitle(!!input.parentID),
    permission: input.permission,
    time: { created: Date.now(), updated: Date.now() },
  }
  
  Database.use((db) => {
    db.insert(SessionTable).values(toRow(result)).run()
    Bus.publish(Event.Created, { info: result })
  })
  
  // 自动分享（如果配置）
  if (Flag.OPENCODE_AUTO_SHARE || cfg.share === "auto") {
    share(result.id).catch(() => {})
  }
  
  return result
}
```

**Session Fork（分支）**：
```typescript
export const fork = fn(
  z.object({
    sessionID: Identifier.schema("session"),
    messageID: Identifier.schema("message").optional(),
  }),
  async (input) => {
    const original = await get(input.sessionID)
    const title = getForkedTitle(original.title)
    
    // 创建新 Session
    const session = await createNext({ directory: Instance.directory, title })
    
    // 复制消息历史
    const msgs = await messages({ sessionID: input.sessionID })
    const idMap = new Map<string, string>()
    
    for (const msg of msgs) {
      if (input.messageID && msg.info.id >= input.messageID) break
      const newID = Identifier.ascending("message")
      idMap.set(msg.info.id, newID)
      
      const cloned = await updateMessage({
        ...msg.info,
        sessionID: session.id,
        id: newID,
      })
      
      for (const part of msg.parts) {
        await updatePart({
          ...part,
          id: Identifier.ascending("part"),
          messageID: cloned.id,
          sessionID: session.id,
        })
      }
    }
    
    return session
  }
)
```

**消息管理**：
```typescript
// 获取消息（带分页）
export const messages = fn(
  z.object({
    sessionID: Identifier.schema("session"),
    limit: z.number().optional(),
  }),
  async (input) => {
    const result = [] as MessageV2.WithParts[]
    for await (const msg of MessageV2.stream(input.sessionID)) {
      if (input.limit && result.length >= input.limit) break
      result.push(msg)
    }
    result.reverse()
    return result
  }
)

// 更新消息
export const updateMessage = fn(MessageV2.Info, async (msg) => {
  Database.use((db) => {
    db.insert(MessageTable)
      .values({ id: msg.id, session_id: msg.sessionID, time_created: msg.time.created, data: msg.data })
      .onConflictDoUpdate({ target: MessageTable.id, set: { data: msg.data } })
      .run()
    Bus.publish(MessageV2.Event.Updated, { info: msg })
  })
})

// 删除消息
export const removeMessage = fn(
  z.object({
    sessionID: Identifier.schema("session"),
    messageID: Identifier.schema("message"),
  }),
  async (input) => {
    Database.use((db) => {
      db.delete(MessageTable)
        .where(and(eq(MessageTable.id, input.messageID), eq(MessageTable.session_id, input.sessionID)))
        .run()
      Bus.publish(MessageV2.Event.Removed, { sessionID: input.sessionID, messageID: input.messageID })
    })
  }
)
```

**Token 使用计算**：
```typescript
export const getUsage = fn(
  z.object({
    model: z.custom<Provider.Model>(),
    usage: z.custom<LanguageModelV2Usage>(),
    metadata: z.custom<ProviderMetadata>().optional(),
  }),
  (input) => {
    const inputTokens = safe(input.usage.inputTokens ?? 0)
    const outputTokens = safe(input.usage.outputTokens ?? 0)
    const reasoningTokens = safe(input.usage.reasoningTokens ?? 0)
    const cacheReadInputTokens = safe(input.usage.cachedInputTokens ?? 0)
    const cacheWriteInputTokens = safe(input.metadata?.["anthropic"]?.["cacheCreationInputTokens"] ?? 0)
    
    // Anthropic 的 inputTokens 不包含缓存 tokens
    const excludesCachedTokens = !!(input.metadata?.["anthropic"] || input.metadata?.["bedrock"])
    const adjustedInputTokens = safe(
      excludesCachedTokens ? inputTokens : inputTokens - cacheReadInputTokens - cacheWriteInputTokens
    )
    
    const tokens = {
      total: adjustedInputTokens + outputTokens + cacheReadInputTokens + cacheWriteInputTokens,
      input: adjustedInputTokens,
      output: outputTokens,
      reasoning: reasoningTokens,
      cache: { write: cacheWriteInputTokens, read: cacheReadInputTokens },
    }
    
    // 计算成本
    const cost = new Decimal(0)
      .add(new Decimal(tokens.input).mul(costInfo?.input ?? 0).div(1_000_000))
      .add(new Decimal(tokens.output).mul(costInfo?.output ?? 0).div(1_000_000))
      .add(new Decimal(tokens.cache.read).mul(costInfo?.cache?.read ?? 0).div(1_000_000))
      .add(new Decimal(tokens.cache.write).mul(costInfo?.cache?.write ?? 0).div(1_000_000))
      .toNumber()
    
    return { cost, tokens }
  }
)
```

---

## MCP 集成 (`src/mcp/`)

**核心文件**：`src/mcp/index.ts`（30KB）

**MCP 配置类型**：
```typescript
// 本地 MCP（启动进程）
export const McpLocal = z.object({
  type: z.literal("local"),
  command: z.string().array(),  // 命令和参数
  environment: z.record(z.string(), z.string()).optional(),
  enabled: z.boolean().optional(),
  timeout: z.number().int().positive().optional(),
})

// 远程 MCP（HTTP 连接）
export const McpRemote = z.object({
  type: z.literal("remote"),
  url: z.string(),
  enabled: z.boolean().optional(),
  headers: z.record(z.string(), z.string()).optional(),
  oauth: z.union([McpOAuth, z.literal(false)]).optional(),
  timeout: z.number().int().positive().optional(),
})
```

**MCP 状态**：
```typescript
export const Status = z.discriminatedUnion("status", [
  z.object({ status: z.literal("connected") }),
  z.object({ status: z.literal("disabled") }),
  z.object({ status: z.literal("failed"), error: z.string() }),
  z.object({ status: z.literal("needs_auth") }),
  z.object({ status: z.literal("needs_client_registration"), error: z.string() }),
])
```

**MCP 客户端创建**：
```typescript
async function create(key: string, mcp: Config.Mcp) {
  if (mcp.type === "remote") {
    // OAuth 认证（默认启用）
    const oauthDisabled = mcp.oauth === false
    let authProvider: McpOAuthProvider | undefined
    
    if (!oauthDisabled) {
      authProvider = new McpOAuthProvider(key, mcp.url, {
        clientId: mcp.oauth?.clientId,
        clientSecret: mcp.oauth?.clientSecret,
        scope: mcp.oauth?.scope,
      }, {
        onRedirect: async (url) => {
          // 存储授权 URL
        },
      })
    }
    
    // 尝试多种传输协议
    const transports = [
      { name: "StreamableHTTP", transport: new StreamableHTTPClientTransport(...) },
      { name: "SSE", transport: new SSEClientTransport(...) },
    ]
    
    for (const { name, transport } of transports) {
      try {
        const client = new Client({ name: "opencode", version: Installation.VERSION })
        await withTimeout(client.connect(transport), mcp.timeout ?? DEFAULT_TIMEOUT)
        registerNotificationHandlers(client, key)
        mcpClient = client
        status = { status: "connected" }
        break
      } catch (error) {
        if (error instanceof UnauthorizedError) {
          status = { status: "needs_auth" }
          break
        }
      }
    }
  }
  
  if (mcp.type === "local") {
    const [cmd, ...args] = mcp.command
    const transport = new StdioClientTransport({
      command: cmd,
      args,
      cwd: Instance.directory,
      env: { ...process.env, ...mcp.environment },
    })
    
    const client = new Client({ name: "opencode", version: Installation.VERSION })
    await withTimeout(client.connect(transport), mcp.timeout ?? DEFAULT_TIMEOUT)
    mcpClient = client
    status = { status: "connected" }
  }
  
  return { mcpClient, status }
}
```

**MCP 工具转换**：
```typescript
async function convertMcpTool(mcpTool: MCPToolDef, client: MCPClient, timeout?: number): Promise<Tool> {
  const schema: JSONSchema7 = {
    ...(mcpTool.inputSchema as JSONSchema7),
    type: "object",
    properties: (mcpTool.inputSchema.properties ?? {}) as JSONSchema7["properties"],
    additionalProperties: false,
  }
  
  return dynamicTool({
    description: mcpTool.description ?? "",
    inputSchema: jsonSchema(schema),
    execute: async (args: unknown) => {
      return client.callTool(
        { name: mcpTool.name, arguments: args as Record<string, unknown> },
        CallToolResultSchema,
        { resetTimeoutOnProgress: true, timeout }
      )
    },
  })
}
```

**MCP OAuth 流程**：
```typescript
export async function authenticate(mcpName: string): Promise<Status> {
  const { authorizationUrl } = await startAuth(mcpName)
  
  // 生成并存储 state 参数（CSRF 防护）
  const oauthState = Array.from(crypto.getRandomValues(new Uint8Array(32)))
    .map((b) => b.toString(16).padStart(2, "0"))
    .join("")
  await McpAuth.updateOAuthState(mcpName, oauthState)
  
  // 打开浏览器
  await open(authorizationUrl)
  
  // 等待回调
  const code = await McpOAuthCallback.waitForCallback(oauthState)
  
  // 验证 state
  const storedState = await McpAuth.getOAuthState(mcpName)
  if (storedState !== oauthState) {
    await McpAuth.clearOAuthState(mcpName)
    throw new Error("OAuth state mismatch - potential CSRF attack")
  }
  
  // 完成认证
  return finishAuth(mcpName, code)
}
```

---

## Provider 系统 (`src/provider/`)

**核心文件**：`src/provider/provider.ts`

**支持的 Provider**：
- OpenAI、Anthropic、Google、Azure、Amazon Bedrock
- Cohere、Groq、Mistral、Perplexity、XAI
- GitLab AI、Vercel AI、OpenRouter
- GitHub Copilot、Cloudflare AI Gateway
- 等 75+ Provider

**Provider 配置**：
```typescript
export const Info = z.object({
  id: z.string(),
  name: z.string(),
  source: z.enum(["env", "config", "custom", "api"]),
  env: z.string().array(),
  key: z.string().optional(),
  options: z.record(z.string(), z.any()),
  models: z.record(z.string(), Model),
})

export const Model = z.object({
  id: z.string(),
  providerID: z.string(),
  api: z.object({
    id: z.string(),
    url: z.string(),
    npm: z.string(),
  }),
  name: z.string(),
  capabilities: z.object({
    temperature: z.boolean(),
    reasoning: z.boolean(),
    attachment: z.boolean(),
    toolcall: z.boolean(),
    input: z.object({ text: z.boolean(), audio: z.boolean(), image: z.boolean(), video: z.boolean(), pdf: z.boolean() }),
    output: z.object({ text: z.boolean(), audio: z.boolean(), image: z.boolean(), video: z.boolean(), pdf: z.boolean() }),
    interleaved: z.union([z.boolean(), z.object({ field: z.enum(["reasoning_content", "reasoning_details"]) })]),
  }),
  cost: z.object({
    input: z.number(),
    output: z.number(),
    cache: z.object({ read: z.number(), write: z.number() }),
    experimentalOver200K: z.object({ input: z.number(), output: z.number(), cache: z.object({ read: z.number(), write: z.number() }) }).optional(),
  }),
  limit: z.object({ context: z.number(), input: z.number().optional(), output: z.number() }),
  status: z.enum(["alpha", "beta", "deprecated", "active"]),
  variants: z.record(z.string(), z.record(z.string(), z.any())).optional(),
})
```

**Provider 加载流程**：
```typescript
const state = Instance.state(async () => {
  const config = await Config.get()
  const modelsDev = await ModelsDev.get()
  
  // 1. 从 models.dev 加载 Provider 数据库
  const database = mapValues(modelsDev, fromModelsDevProvider)
  
  // 2. 从配置扩展 Provider
  for (const [providerID, provider] of Object.entries(config.provider ?? {})) {
    const existing = database[providerID]
    const parsed: Info = {
      id: providerID,
      name: provider.name ?? existing?.name ?? providerID,
      env: provider.env ?? existing?.env ?? [],
      options: mergeDeep(existing?.options ?? {}, provider.options ?? {}),
      source: "config",
      models: existing?.models ?? {},
    }
    
    // 合并模型配置
    for (const [modelID, model] of Object.entries(provider.models ?? {})) {
      const parsedModel: Model = { /* ... */ }
      parsed.models[modelID] = parsedModel
    }
    
    database[providerID] = parsed
  }
  
  // 3. 从环境变量加载 API Key
  const env = Env.all()
  for (const [providerID, provider] of Object.entries(database)) {
    const apiKey = provider.env.map((item) => env[item]).find(Boolean)
    if (!apiKey) continue
    mergeProvider(providerID, { source: "env", key: apiKey })
  }
  
  // 4. 从认证存储加载 API Key
  for (const [providerID, provider] of Object.entries(await Auth.all())) {
    if (provider.type === "api") {
      mergeProvider(providerID, { source: "api", key: provider.key })
    }
  }
  
  // 5. 从插件加载自定义 Provider
  for (const plugin of await Plugin.list()) {
    if (!plugin.auth) continue
    const providerID = plugin.auth.provider
    const auth = await Auth.get(providerID)
    if (!auth) continue
    
    const options = await plugin.auth.loader(() => Auth.get(providerID), database[providerID])
    mergeProvider(providerID, { source: "custom", options })
  }
  
  // 6. 自定义加载器（特殊 Provider）
  for (const [providerID, fn] of Object.entries(CUSTOM_LOADERS)) {
    const data = database[providerID]
    if (!data) continue
    
    const result = await fn(data)
    if (result && (result.autoload || providers[providerID])) {
      if (result.getModel) modelLoaders[providerID] = result.getModel
      mergeProvider(providerID, { source: "custom", options: result.options ?? {} })
    }
  }
  
  return { providers, models: languages, sdk, modelLoaders }
})
```

**自定义 Provider 加载器**：
```typescript
const CUSTOM_LOADERS: Record<string, CustomLoader> = {
  async anthropic() {
    return {
      autoload: false,
      options: {
        headers: {
          "anthropic-beta": "claude-code-20250219,interleaved-thinking-2025-05-14,fine-grained-tool-streaming-2025-05-14",
        },
      },
    }
  },
  
  async "amazon-bedrock"() {
    const config = await Config.get()
    const providerConfig = config.provider?.["amazon-bedrock"]
    
    // 区域解析逻辑
    const configRegion = providerConfig?.options?.region
    const envRegion = Env.get("AWS_REGION")
    const defaultRegion = configRegion ?? envRegion ?? "us-east-1"
    
    // Profile 支持
    const configProfile = providerConfig?.options?.profile
    const envProfile = Env.get("AWS_PROFILE")
    const profile = configProfile ?? envProfile
    
    // Bearer Token 支持
    const awsBearerToken = process.env.AWS_BEARER_TOKEN_BEDROCK
    
    if (!profile && !awsAccessKeyId && !awsBearerToken) return { autoload: false }
    
    return {
      autoload: true,
      options: { region: defaultRegion, credentialProvider: fromNodeProviderChain(profile ? { profile } : {}) },
      async getModel(sdk: any, modelID: string, options?: Record<string, any>) {
        // 跨区域推理前缀处理
        const crossRegionPrefixes = ["global.", "us.", "eu.", "jp.", "apac.", "au."]
        if (crossRegionPrefixes.some((prefix) => modelID.startsWith(prefix))) {
          return sdk.languageModel(modelID)
        }
        
        // 根据区域添加前缀
        const region = options?.region ?? defaultRegion
        let regionPrefix = region.split("-")[0]
        
        if (regionPrefix === "us" && modelRequiresPrefix) {
          modelID = `${regionPrefix}.${modelID}`
        } else if (regionPrefix === "eu" && regionRequiresPrefix && modelRequiresPrefix) {
          modelID = `${regionPrefix}.${modelID}`
        }
        
        return sdk.languageModel(modelID)
      },
    }
  },
  
  async "google-vertex"(provider) {
    const project = provider.options?.project ?? Env.get("GOOGLE_CLOUD_PROJECT")
    const location = provider.options?.location ?? Env.get("VERTEX_LOCATION") ?? "us-central1"
    
    const autoload = Boolean(project)
    if (!autoload) return { autoload: false }
    
    return {
      autoload: true,
      options: { project, location },
      async getModel(sdk: any, modelID: string) {
        return sdk.languageModel(String(modelID).trim())
      },
    }
  },
}
```

---

## 配置系统 (`src/config/`)

**核心文件**：`src/config/config.ts`

**配置加载优先级**（从低到高）：
1. 远程配置（`.well-known/opencode`）
2. 全局配置（`~/.config/opencode/opencode.json`）
3. 自定义路径（`OPENCODE_CONFIG` 环境变量）
4. 项目配置（`opencode.json`）
5. `.opencode/` 目录配置
6. 内联配置（`OPENCODE_CONFIG_CONTENT` 环境变量）
7. 托管配置（`/etc/opencode` - 企业版）

**配置加载流程**：
```typescript
export const state = Instance.state(async () => {
  const auth = await Auth.all()
  
  let result: Info = {}
  
  // 1. 远程配置
  for (const [key, value] of Object.entries(auth)) {
    if (value.type === "wellknown") {
      const response = await fetch(`${key}/.well-known/opencode`)
      const wellknown = await response.json()
      const remoteConfig = wellknown.config ?? {}
      result = mergeConfigConcatArrays(result, await load(JSON.stringify(remoteConfig)))
    }
  }
  
  // 2. 全局配置
  result = mergeConfigConcatArrays(result, await global())
  
  // 3. 自定义路径
  if (Flag.OPENCODE_CONFIG) {
    result = mergeConfigConcatArrays(result, await loadFile(Flag.OPENCODE_CONFIG))
  }
  
  // 4. 项目配置
  if (!Flag.OPENCODE_DISABLE_PROJECT_CONFIG) {
    for (const file of await ConfigPaths.projectFiles("opencode", Instance.directory, Instance.worktree)) {
      result = mergeConfigConcatArrays(result, await loadFile(file))
    }
  }
  
  // 5. .opencode 目录
  const directories = await ConfigPaths.directories(Instance.directory, Instance.worktree)
  for (const dir of unique(directories)) {
    if (dir.endsWith(".opencode") || dir === Flag.OPENCODE_CONFIG_DIR) {
      for (const file of ["opencode.jsonc", "opencode.json"]) {
        result = mergeConfigConcatArrays(result, await loadFile(path.join(dir, file)))
      }
    }
    
    // 加载命令、Agent、模式、插件
    result.command = mergeDeep(result.command ?? {}, await loadCommand(dir))
    result.agent = mergeDeep(result.agent, await loadAgent(dir))
    result.agent = mergeDeep(result.agent, await loadMode(dir))
    result.plugin.push(...(await loadPlugin(dir)))
  }
  
  // 6. 内联配置
  if (process.env.OPENCODE_CONFIG_CONTENT) {
    result = mergeConfigConcatArrays(result, await load(process.env.OPENCODE_CONFIG_CONTENT))
  }
  
  // 7. 托管配置（企业版）
  if (existsSync(managedDir)) {
    for (const file of ["opencode.jsonc", "opencode.json"]) {
      result = mergeConfigConcatArrays(result, await loadFile(path.join(managedDir, file)))
    }
  }
  
  return { config: result, directories, deps }
})
```

**配置 Schema**：
```typescript
export const Config = z.object({
  // Agent 配置
  agent: z.record(z.string(), AgentConfig).optional(),
  
  // MCP 配置
  mcp: z.record(z.string(), z.union([McpLocal, McpRemote])).optional(),
  
  // Provider 配置
  provider: z.record(z.string(), ProviderConfig).optional(),
  
  // 权限配置
  permission: Permission.optional(),
  
  // 插件配置
  plugin: z.array(z.string()).optional(),
  
  // 技能配置
  skills: z.object({
    paths: z.array(z.string()).optional(),
    urls: z.array(z.string()).optional(),
  }).optional(),
  
  // 分享配置
  share: z.enum(["auto", "manual", "disabled"]).optional(),
  
  // 压缩配置
  compaction: z.object({
    auto: z.boolean().optional(),
    prune: z.boolean().optional(),
  }).optional(),
  
  // 实验性特性
  experimental: z.object({
    batch_tool: z.boolean().optional(),
    openTelemetry: z.boolean().optional(),
    mcp_timeout: z.number().optional(),
  }).optional(),
  
  // 快捷键配置
  keybinds: Keybinds.optional(),
  
  // 服务器配置
  server: z.object({
    port: z.number().int().positive().optional(),
    hostname: z.string().optional(),
    mdns: z.boolean().optional(),
    cors: z.array(z.string()).optional(),
  }).optional(),
})
```

---

## 权限系统 (`src/permission/`)

**权限动作**：
```typescript
export type PermissionAction = "allow" | "deny" | "ask"
```

**权限规则**：
```typescript
export type PermissionRule = {
  permission: string      // 权限类型（read、edit、bash 等）
  pattern: string         // 文件模式（*.ts、src/**/* 等）
  action: PermissionAction
}

export type PermissionRuleset = Array<PermissionRule>
```

**权限配置**：
```typescript
export const Permission = z
  .preprocess(permissionPreprocess, z.object({
    read: PermissionRule.optional(),
    edit: PermissionRule.optional(),
    glob: PermissionRule.optional(),
    grep: PermissionRule.optional(),
    list: PermissionRule.optional(),
    bash: PermissionRule.optional(),
    task: PermissionRule.optional(),
    external_directory: PermissionRule.optional(),
    todowrite: PermissionAction.optional(),
    todoread: PermissionAction.optional(),
    question: PermissionAction.optional(),
    webfetch: PermissionAction.optional(),
    websearch: PermissionAction.optional(),
    codesearch: PermissionAction.optional(),
    lsp: PermissionRule.optional(),
    doom_loop: PermissionAction.optional(),
    skill: PermissionRule.optional(),
  }).catchall(PermissionRule).or(PermissionAction))
  .transform(permissionTransform)
```

**权限请求流程**：
```typescript
export type PermissionRequest = {
  id: string
  sessionID: string
  permission: string
  patterns: Array<string>
  metadata: Record<string, unknown>
  always: Array<string>  // 永久授权选项
  tool?: { messageID: string; callID: string }
}

// 权限事件
export type EventPermissionAsked = {
  type: "permission.asked"
  properties: PermissionRequest
}

export type EventPermissionReplied = {
  type: "permission.replied"
  properties: {
    sessionID: string
    requestID: string
    reply: "once" | "always" | "reject"
  }
}
```

---

## 插件系统 (`src/plugin/`)

**插件 Hook 类型**：
```typescript
export interface Hooks {
  // 聊天消息钩子
  "chat.message"?: (input: {
    sessionID: string
    agent?: string
    model?: { providerID: string; modelID: string }
    messageID?: string
    variant?: string
  }, output: { message: UserMessage; parts: Part[] }) => Promise<void>
  
  // 聊天参数钩子
  "chat.params"?: (input: {
    sessionID: string
    agent: string
    model: Model
    provider: ProviderContext
    message: UserMessage
  }, output: {
    messages: ModelMessage[]
    mode: string
    system: string[]
    tools: Tool[]
  }) => Promise<void>
  
  // 工具执行前钩子
  "tool.execute.before"?: (input: {
    tool: string
    sessionID: string
    callID: string
  }, output: { args: any }) => Promise<void>
  
  // 工具执行后钩子
  "tool.execute.after"?: (input: {
    tool: string
    sessionID: string
    callID: string
    args: any
  }, output: { title: string; output: string; metadata: any }) => Promise<void>
  
  // 系统提示词钩子
  "experimental.chat.system.transform"?: (input: {
    model: Model
  }, output: { system: string[] }) => Promise<void>
  
  // 工具定义钩子
  "tool.definition"?: (input: { toolID: string }, output: {
    description: string
    parameters: JSONSchema7
  }) => Promise<void>
}
```

**插件注册**：
```typescript
export type PluginDefinition = {
  name: string
  version: string
  description?: string
  auth?: {
    provider: string
    loader: (getAuth: () => Promise<Auth>, provider: Provider.Info) => Promise<Record<string, any>>
  }
  tool?: Record<string, ToolDefinition>
  skill?: string[]
  command?: Record<string, CommandDefinition>
}
```

---

## 数据流

### 1. 用户请求流程

```
用户输入
  │
  ▼
┌─────────────────┐
│   CLI/TUI/Web   │
│   (SolidJS)     │
└────────┬────────┘
         │ HTTP POST /session/{id}/message
         ▼
┌─────────────────┐
│   SDK Server    │
│   (Hono)        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Session Prompt │
│  (src/session/  │
│   prompt.ts)    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Agent Select  │
│  (build/plan/   │
│   explore)      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Tool Registry  │
│  (src/tool/     │
│   registry.ts)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Tool Execute   │
│  (bash/read/    │
│   edit/...)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Permission     │
│  Check          │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  LLM Call       │
│  (Provider)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Response       │
│  Streaming      │
│  (SSE)          │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   UI Update     │
│  (SolidJS)      │
└─────────────────┘
```

### 2. 事件驱动架构

**核心事件类型**：
```typescript
// Session 事件
EventSessionCreated
EventSessionUpdated
EventSessionDeleted
EventSessionStatus      // idle/busy/retry
EventSessionIdle
EventSessionCompacted
EventSessionDiff
EventSessionError

// Message 事件
EventMessageUpdated
EventMessagePartUpdated
EventMessagePartRemoved
EventMessagePartDelta

// Permission 事件
EventPermissionAsked
EventPermissionReplied

// MCP 事件
EventMcpToolsChanged
EventMcpBrowserOpenFailed
```

**事件发布流程**：
```typescript
// 数据库操作触发事件
Database.use((db) => {
  db.insert(SessionTable).values(row).run()
  
  // 发布事件（通过 SSE 推送给客户端）
  Database.effect(() =>
    Bus.publish(Event.Created, { info: result })
  )
})

// 事件通过 SSE 流式传输到客户端
// 客户端通过 EventSource 订阅
```

---

## 设计原则

### 1. 模块化设计

- **职责分离**：Agent、Tool、Session、Provider、MCP 各自独立模块
- **插件化扩展**：通过 Hook 系统扩展核心功能
- **配置驱动**：所有行为可通过配置调整

### 2. 事件驱动

- **发布/订阅模式**：通过 Bus 系统解耦组件
- **SSE 实时推送**：所有状态变更实时推送到客户端
- **事件溯源**：所有操作通过事件记录

### 3. 权限优先

- **最小权限原则**：默认拒绝，显式授权
- **模式匹配**：文件路径支持 glob 模式
- **三级授权**：allow（自动）、ask（询问）、deny（拒绝）

### 4. 上下文管理

- **Session 隔离**：每个会话独立上下文
- **自动压缩**：超出 token 限制自动压缩历史
- **分支支持**：通过 parentID 实现会话分支

### 5. 多 Provider 支持

- **Provider 抽象**：统一接口，支持 75+ Provider
- **动态加载**：按需加载 Provider，避免冗余
- **成本计算**：统一成本计算逻辑

### 6. 可扩展性

- **自定义工具**：支持 .ts/.js 文件动态加载
- **自定义 Agent**：通过 Markdown 或 JSON 定义
- **自定义命令**：通过 Markdown 模板定义
- **MCP 集成**：支持本地和远程 MCP 服务器

### 7. 开发者体验

- **类型安全**：全栈 TypeScript + Zod 验证
- **热重载**：配置变更无需重启
- **调试友好**：详细日志 + 结构化错误

---

## 总结

OpenCode 是一个设计精良的 AI 编程助手框架，其核心优势在于：

1. **模块化架构**：清晰的职责分离，便于维护和扩展
2. **事件驱动**：实时状态同步，支持多客户端
3. **权限系统**：细粒度权限控制，保障安全
4. **多 Provider**：支持 75+ LLM Provider，避免厂商锁定
5. **插件系统**：通过 Hook 系统实现无限扩展
6. **MCP 支持**：遵循 Model Context Protocol，生态兼容

该架构既适合个人开发者使用，也适合企业级部署（通过托管配置和权限控制）。

---

## 参考资源

- **官方文档**：https://open-code.ai/en/docs
- **GitHub 仓库**：https://github.com/anomalyco/opencode
- **DeepWiki 架构**：https://deepwiki.com/anomalyco/opencode
- **models.dev**：https://models.dev（Provider 模型定义）
