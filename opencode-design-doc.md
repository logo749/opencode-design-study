# OpenCode 设计文档

**分析版本**: Commit `f5eade1d2b95562c7fb58e3041e662a8b2b611b6`  
**包版本**: 1.2.15  
**生成日期**: 2026-03-01

---

## 目录

1. [项目概述](#1-项目概述)
2. [系统架构](#2-系统架构)
3. [核心组件](#3-核心组件)
4. [AI 集成架构](#4-ai-集成架构)
5. [数据流设计](#5-数据流设计)
6. [存储与持久化](#6-存储与持久化)
7. [权限系统](#7-权限系统)
8. [插件与扩展](#8-插件与扩展)
9. [关键技术决策](#9-关键技术决策)
10. [设计模式](#10-设计模式)

---

## 1. 项目概述

### 1.1 项目定位

OpenCode 是一个 **AI 驱动的代码助手**，提供：
- CLI 命令行界面
- TUI 终端用户界面  
- HTTP API 服务器
- 与 AI 编码助手（如 Zed 编辑器）的 ACP 协议集成

### 1.2 核心功能

| 功能 | 描述 |
|------|------|
| **代码生成** | 通过 AI 代理生成、编辑代码 |
| **文件操作** | 读取、编辑、创建、删除文件 |
| **命令执行** | 在沙盒中执行 Shell 命令 |
| **代码搜索** | 支持 grep、codesearch、LSP |
| **会话管理** | 持久化对话历史，支持 fork/resume |
| **多模型支持** | 20+ AI 提供商，统一接口 |
| **MCP 集成** | Model Context Protocol 外部工具 |
| **权限控制** | 基于规则的工具执行权限 |

### 1.3 技术栈总览

```
┌─────────────────────────────────────────────────────────┐
│                    运行时与语言                          │
│  Bun Runtime + TypeScript + ES Modules                  │
├─────────────────────────────────────────────────────────┤
│                    UI 层                                 │
│  CLI (yargs) + TUI (@opentui/solid) + Web (Hono)        │
├─────────────────────────────────────────────────────────┤
│                    AI 层                                 │
│  Vercel AI SDK + 20+ @ai-sdk/* 提供商                    │
├─────────────────────────────────────────────────────────┤
│                    数据层                                │
│  SQLite (Drizzle ORM) + 文件系统 + 配置系统              │
├─────────────────────────────────────────────────────────┤
│                    协议层                                │
│  MCP (Model Context Protocol) + ACP (Agent Protocol)    │
└─────────────────────────────────────────────────────────┘
```

---

## 2. 系统架构

### 2.1 分层架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        接口层 (Interface)                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │   CLI/TUI    │  │  HTTP Server │  │   ACP Server (Zed)   │  │
│  │  (yargs)     │  │    (Hono)    │  │   (Agent Protocol)   │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                        命令层 (Commands)                         │
│  run | generate | agent | serve | session | mcp | auth | ...    │
├─────────────────────────────────────────────────────────────────┤
│                      业务逻辑层 (Business)                       │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │ Session │ │  Agent  │ │  Tool   │ │  File   │ │  MCP    │   │
│  │ Manager │ │ Engine  │ │ Executor│ │ Watcher │ │ Server  │   │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘   │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │ Config  │ │Provider │ │Permission│ │ Snapshot│ │  Auth   │   │
│  │  Loader │ │ Manager │ │  Engine │ │ Tracker │ │  OAuth  │   │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘   │
├─────────────────────────────────────────────────────────────────┤
│                        事件总线 (Event Bus)                      │
│         Bus.publish() / Bus.subscribe() 解耦通信                 │
├─────────────────────────────────────────────────────────────────┤
│                        数据持久层 (Persistence)                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────┐  │
│  │  SQLite 数据库   │  │  JSON 配置文件   │  │   文件系统      │  │
│  │  - Sessions     │  │  - opencode.json│  │   - 工作目录    │  │
│  │  - Messages     │  │  - agents/      │  │   - 快照        │  │
│  │  - Parts/Todos  │  │  - commands/    │  │   - 日志        │  │
│  └─────────────────┘  └─────────────────┘  └────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 目录结构

```
packages/opencode/
├── bin/                          # CLI 入口脚本
│   └── opencode                  # 跨平台二进制包装器
├── src/
│   ├── index.ts                  # 主入口 (CLI 初始化)
│   ├── acp/                      # Agent Communication Protocol
│   ├── agent/                    # AI 代理定义与执行
│   ├── auth/                     # OAuth/API Key 认证
│   ├── cli/
│   │   ├── cmd/                  # CLI 命令实现
│   │   │   ├── run.ts            # 主执行命令
│   │   │   ├── agent.ts          # 代理管理
│   │   │   ├── serve.ts          # 服务器模式
│   │   │   ├── mcp.ts            # MCP 管理
│   │   │   ├── github.ts         # GitHub 集成
│   │   │   └── ...               # 20+ 命令
│   │   ├── ui.ts                 # TUI 组件
│   │   └── bootstrap.ts          # 启动引导
│   ├── config/                   # 配置加载与解析
│   ├── mcp/                      # Model Context Protocol
│   │   ├── index.ts              # MCP 服务器管理
│   │   └── auth.ts               # MCP OAuth
│   ├── provider/                 # AI 提供商抽象
│   │   ├── provider.ts           # 提供商注册表
│   │   ├── models.ts             # 模型定义
│   │   └── transform.ts          # 消息转换
│   ├── session/                  # 会话管理
│   │   ├── index.ts              # 会话处理器
│   │   └── llm.ts                # LLM 流式处理
│   ├── tool/                     # 工具实现
│   │   ├── registry.ts           # 工具注册表
│   │   ├── bash.ts               # Shell 执行
│   │   ├── read.ts               # 文件读取
│   │   ├── edit.ts               # 文件编辑
│   │   └── ...                   # 15+ 工具
│   ├── permission/               # 权限系统
│   │   └── next.ts               # 规则引擎
│   ├── server/                   # HTTP 服务器
│   │   ├── server.ts             # Hono 服务器
│   │   ├── routes/               # API 路由
│   │   └── mdns.ts               # 服务发现
│   ├── storage/                  # 数据库层
│   │   ├── db.ts                 # SQLite 连接
│   │   └── schema.sql.ts         # 数据库模式
│   ├── file/                     # 文件操作
│   │   └── watcher.ts            # 文件监听
│   ├── snapshot/                 # 快照系统
│   ├── project/                  # 项目管理
│   │   └── instance.ts           # 实例隔离
│   ├── installation/             # 版本与升级
│   ├── global/                   # 全局路径 (XDG)
│   ├── bus/                      # 事件总线
│   └── util/                     # 工具函数
├── migration/                    # 数据库迁移
├── script/                       # 构建脚本
└── test/                         # 测试文件
```

---

## 3. 核心组件

### 3.1 CLI 入口 (`src/index.ts`)

**职责**: CLI 初始化、命令注册、全局错误处理

```typescript
// 核心流程
1. 设置 unhandledRejection/uncaughtException 处理器
2. 配置 yargs CLI 框架
3. 中间件初始化日志和数据库
4. 注册 20+ CLI 命令
5. 解析参数并执行

// 关键代码位置
process.on("unhandledRejection", ...)  // 行 34
process.on("uncaughtException", ...)   // 行 40
middleware: async (opts) => {          // 行 58 - 初始化中间件
  await Log.init({...})
  // 数据库迁移检查 (行 74-98)
}
```

### 3.2 会话管理器 (`src/session/index.ts`)

**职责**: 会话生命周期管理、消息处理、LLM 流式响应

```typescript
核心功能:
├── create()      - 创建新会话
├── get()         - 获取现有会话
├── process()     - 处理用户输入并执行工具
├── stream()      - 流式响应处理
└── compact()     - 上下文压缩

数据流:
用户输入 → 创建 Message → Agent 选择 → LLM 调用 → 工具执行 → 结果存储
```

### 3.3 代理引擎 (`src/agent/agent.ts`)

**职责**: 代理定义、模型选择、提示词构建

**内置代理**:

| 代理 | 模式 | 描述 |
|------|------|------|
| `build` | primary | 默认代理，执行工具 |
| `plan` | primary | 计划模式 (禁止编辑) |
| `general` | subagent | 通用子代理 |
| `explore` | subagent | 代码探索 |
| `compaction` | primary | 上下文压缩 (隐藏) |
| `title` | primary | 会话标题生成 (隐藏) |

```typescript
// 代理定义结构
export const Info = z.object({
  name: z.string(),
  mode: z.enum(["subagent", "primary", "all"]),
  permission: PermissionNext.Ruleset,
  model: z.object({...}).optional(),
  prompt: z.string().optional(),
  tools: z.array(z.string()).optional(),
})
```

### 3.4 工具执行器 (`src/tool/`)

**职责**: 定义和执行 AI 可调用的工具

**内置工具**:

| 工具 | 文件 | 描述 |
|------|------|------|
| `bash` | bash.ts | 执行 Shell 命令 |
| `read` | read.ts | 读取文件内容 |
| `edit` | edit.ts | 编辑文件 (diff) |
| `write` | write.ts | 写入文件 |
| `glob` | glob.ts | 文件模式匹配 |
| `grep` | grep.ts | 代码搜索 |
| `codesearch` | codesearch.ts | 语义代码搜索 |
| `lsp` | lsp.ts | LSP 集成 |
| `webfetch` | webfetch.ts | 获取网页内容 |
| `websearch` | websearch.ts | 网络搜索 |
| `task` | task.ts | 子代理委托 |
| `question` | question.ts | 向用户提问 |
| `todo` | todo.ts | 任务列表管理 |
| `apply_patch` | apply_patch.ts | 应用补丁 |
| `batch` | batch.ts | 批量操作 |
| `multiedit` | multiedit.ts | 多重编辑 |
| `plan` | plan.ts | 计划制定 |

**工具定义模式**:

```typescript
export const BashTool = Tool.define("bash", async () => ({
  description: "Execute shell commands",
  parameters: z.object({
    command: z.string(),
    timeout: z.number().optional(),
    workdir: z.string().optional(),
    description: z.string(),
  }),
  async execute(params, ctx) {
    // 1. 解析命令进行权限分析
    const tree = parser().then(p => p.parse(params.command))
    
    // 2. 请求权限
    await ctx.ask({ permission: "bash", patterns: [...] })
    
    // 3. 执行命令
    const proc = spawn(params.command, { shell, cwd, ... })
    
    // 4. 返回结果
    return { title: params.description, output, metadata }
  }
}))
```

### 3.5 配置系统 (`src/config/config.ts`)

**职责**: 多层配置加载与合并

**配置优先级** (低 → 高):

```
1. 远程 .well-known/opencode (组织默认)
2. 全局配置 (~/.config/opencode/opencode.json)
3. 自定义配置 (OPENCODE_CONFIG 环境变量)
4. 项目配置 (opencode.json 项目根目录)
5. .opencode/ 目录 (agents/, commands/, plugins/)
6. 内联配置 (OPENCODE_CONFIG_CONTENT 环境变量)
7. 托管配置 (/etc/opencode 或企业路径)
```

**配置结构**:

```typescript
export const Info = z.object({
  $schema: z.string().optional(),
  model: ModelId.optional(),                    // 默认模型
  agent: z.object({...}).catchall(Agent),       // 代理定义
  provider: z.record(z.string(), Provider),     // 提供商配置
  permission: Permission.optional(),            // 权限规则
  mcp: z.record(z.string(), Mcp),               // MCP 服务器
  plugin: z.array(z.string()).optional(),       // 插件列表
  skill: z.record(z.string(), Skill).optional(),// 技能定义
})
```

### 3.6 服务器 (`src/server/server.ts`)

**职责**: HTTP API、SSE 事件流、反向代理

**API 路由**:

| 路由 | 方法 | 描述 |
|------|------|------|
| `/session` | GET/POST | 会话管理 |
| `/provider` | GET/POST | 提供商配置 |
| `/permission` | POST | 权限请求 |
| `/event` | GET | SSE 事件流 |
| `/mcp` | * | MCP 代理 |
| `/global` | GET | 全局状态 |
| `/project` | GET | 项目信息 |

**SSE 事件流**:

```typescript
app.get("/event", async (c) => {
  return streamSSE(c, async (stream) => {
    const unsub = Bus.subscribeAll(async (event) => {
      await stream.writeSSE({ data: JSON.stringify(event) })
    })
  })
})
```

---

## 4. AI 集成架构

### 4.1 提供商注册表 (`src/provider/provider.ts`)

**支持的 AI 提供商** (20+):

```typescript
const BUNDLED_PROVIDERS: Record<string, (options: any) => SDK> = {
  "@ai-sdk/amazon-bedrock": createAmazonBedrock,
  "@ai-sdk/anthropic": createAnthropic,
  "@ai-sdk/azure": createAzure,
  "@ai-sdk/google": createGoogleGenerativeAI,
  "@ai-sdk/google-vertex": createVertex,
  "@ai-sdk/openai": createOpenAI,
  "@ai-sdk/openai-compatible": createOpenAICompatible,
  "@openrouter/ai-sdk-provider": createOpenRouter,
  "@ai-sdk/xai": createXai,
  "@ai-sdk/mistral": createMistral,
  "@ai-sdk/groq": createGroq,
  "@ai-sdk/deepinfra": createDeepInfra,
  "@ai-sdk/cerebras": createCerebras,
  "@ai-sdk/cohere": createCohere,
  "@ai-sdk/gateway": createGateway,
  "@ai-sdk/togetherai": createTogetherAI,
  "@ai-sdk/perplexity": createPerplexity,
  "@ai-sdk/vercel": createVercel,
  "@gitlab/gitlab-ai-provider": createGitLab,
}
```

### 4.2 模型调用流程

```
┌─────────────────────────────────────────────────────────────┐
│                      模型调用流程                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  用户请求                                                    │
│       │                                                     │
│       ▼                                                     │
│  ┌─────────────────┐                                       │
│  │ Agent.getModel()│                                       │
│  └────────┬────────┘                                       │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                       │
│  │ Provider.getSDK()│ ← 从注册表获取提供商                   │
│  └────────┬────────┘                                       │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                       │
│  │Provider.getLang()│ ← 获取语言模型实例                    │
│  └────────┬────────┘                                       │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                       │
│  │  Transform      │ ← 提供商特定消息转换                   │
│  │  (transform.ts) │                                       │
│  └────────┬────────┘                                       │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                       │
│  │ Vercel AI SDK   │                                       │
│  │ generateText()  │                                       │
│  │ streamText()    │                                       │
│  └────────┬────────┘                                       │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                       │
│  │ Provider SDK    │                                       │
│  │ (@ai-sdk/*)     │                                       │
│  └─────────────────┘                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 消息转换 (`src/provider/transform.ts`)

针对不同提供商的特殊处理:

| 提供商 | 转换规则 |
|--------|----------|
| **Anthropic** | 过滤空消息，对系统/最终消息应用缓存 |
| **Mistral** | 工具调用 ID 标准化为 9 位字母数字 |
| **Google** | JSON Schema 枚举整数转字符串 |
| **OpenAI 兼容** | 通过 `reasoning_content` 处理推理内容 |
| **缓存** | 对系统和最终消息应用临时缓存控制 |

### 4.4 推理模型变体

```typescript
// OpenAI 推理努力级别
{
  none: { reasoningEffort: "none" },
  minimal: { reasoningEffort: "minimal" },
  low: { reasoningEffort: "low" },
  medium: { reasoningEffort: "medium" },
  high: { reasoningEffort: "high" },
  xhigh: { reasoningEffort: "xhigh" }
}

// Anthropic 自适应思考
{
  low: { thinking: { type: "adaptive" }, effort: "low" },
  medium: { thinking: { type: "adaptive" }, effort: "medium" },
  high: { thinking: { type: "enabled", budgetTokens: 16000 } },
  max: { thinking: { type: "enabled", budgetTokens: 31999 } }
}
```

---

## 5. 数据流设计

### 5.1 用户请求 → AI 响应流程

```
┌─────────────────────────────────────────────────────────────────┐
│                     完整数据流                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  用户输入 (CLI/TUI/Web)                                         │
│         │                                                       │
│         ▼                                                       │
│  ┌───────────────────┐                                         │
│  │  Command Handler  │  src/cli/cmd/*.ts                       │
│  └─────────┬─────────┘                                         │
│            │                                                    │
│            ▼                                                    │
│  ┌───────────────────┐                                         │
│  │  Session Manager  │  src/session/index.ts                   │
│  │  - 创建 message    │                                         │
│  │  - 跟踪 tokens     │                                         │
│  └─────────┬─────────┘                                         │
│            │                                                    │
│            ▼                                                    │
│  ┌───────────────────┐                                         │
│  │  Agent Engine     │  src/agent/agent.ts                     │
│  │  - 选择模型        │                                         │
│  │  - 构建提示词      │                                         │
│  │  - 应用权限        │                                         │
│  └─────────┬─────────┘                                         │
│            │                                                    │
│            ▼                                                    │
│  ┌───────────────────┐                                         │
│  │  LLM Stream       │  src/session/llm.ts                     │
│  │  - Vercel AI SDK  │                                         │
│  │  - 流式 tokens     │                                         │
│  └─────────┬─────────┘                                         │
│            │                                                    │
│            ▼                                                    │
│  ┌───────────────────┐                                         │
│  │  Tool Executor    │  src/tool/*.ts                          │
│  │  - 读/写文件       │                                         │
│  │  - Bash 命令       │                                         │
│  │  - MCP 工具        │                                         │
│  └─────────┬─────────┘                                         │
│            │                                                    │
│            ▼                                                    │
│  ┌───────────────────┐                                         │
│  │  File Watcher     │  src/file/watcher.ts                    │
│  │  - 检测变更        │                                         │
│  │  - 发布事件        │                                         │
│  └─────────┬─────────┘                                         │
│            │                                                    │
│            ▼                                                    │
│  ┌───────────────────┐                                         │
│  │  Snapshot System  │  src/snapshot/*.ts                      │
│  │  - 跟踪补丁        │                                         │
│  │  - 支持回滚        │                                         │
│  └─────────┬─────────┘                                         │
│            │                                                    │
│            ▼                                                    │
│  ┌───────────────────┐                                         │
│  │  Database (SQLite)│  src/storage/db.ts                      │
│  │  - 持久化状态      │                                         │
│  │  - 会话历史        │                                         │
│  └───────────────────┘                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 事件总线通信

```
┌─────────────────────────────────────────────────────────────┐
│                      事件总线架构                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────┐    publish()    ┌──────────────────┐          │
│  │  Tool    │────────────────►│      Bus         │          │
│  │ Executor │                 │  (EventEmitter)  │          │
│  └──────────┘                 └────────┬─────────┘          │
│                                        │                     │
│                    ┌───────────────────┼───────────────────┐ │
│                    │                   │                   │ │
│                    ▼                   ▼                   ▼ │
│           ┌──────────────┐   ┌──────────────┐   ┌──────────┐│
│           │  TUI Client  │   │  HTTP (SSE)  │   │ Database ││
│           │  (订阅事件)   │   │  (推送更新)   │   │ (记录)   ││
│           └──────────────┘   └──────────────┘   └──────────┘│
│                                                             │
│  关键事件类型:                                               │
│  - permission.asked      - 权限请求                         │
│  - permission.replied    - 权限响应                         │
│  - message.updated       - 消息更新                         │
│  - message.part.updated  - 消息部分更新                     │
│  - tool.state.updated    - 工具状态更新                     │
│  - file.watcher.updated  - 文件变更                         │
│  - session.error         - 会话错误                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 权限请求流程

```
工具执行请求权限
        │
        ▼
┌─────────────────┐
│ Permission.ask()│ 评估规则集
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌───────┐ ┌─────────┐
│allow/ │ │  action │
│ deny  │ │  = ask  │
└───────┘ └────┬────┘
               │
               ▼
        ┌──────────────┐
        │ 创建待处理请求 │
        │ Bus.publish  │
        └───────┬──────┘
                │
                ▼
        ┌──────────────┐
        │ UI/Server    │
        │ 显示权限请求  │
        └───────┬──────┘
                │
                ▼
        ┌──────────────┐
        │Permission.reply│
        │ once/always/  │
        │ reject        │
        └──────────────┘
```

---

## 6. 存储与持久化

### 6.1 SQLite 数据库模式

```
┌─────────────────────────────────────────────────────────────┐
│              ~/.local/share/opencode/opencode.db            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │   project   │    │   session   │    │   message   │     │
│  ├─────────────┤    ├─────────────┤    ├─────────────┤     │
│  │ id (PK)     │◄───│ project_id  │    │ id (PK)     │     │
│  │ directory   │    │ id (PK)     │◄───│ session_id  │     │
│  │ workspace_id│    │ parent_id   │    │ data (JSON) │     │
│  │ ...         │    │ slug        │    │ created_at  │     │
│  └─────────────┘    │ title       │    │ updated_at  │     │
│                     │ version     │    └─────────────┘     │
│                     │ summary_*   │           │             │
│                     │ permission  │           ▼             │
│                     │ revert      │    ┌─────────────┐     │
│                     └─────────────┘    │    part     │     │
│                            │           ├─────────────┤     │
│                            │           │ id (PK)     │     │
│                            ▼           │ message_id  │     │
│                     ┌─────────────┐    │ session_id  │     │
│                     │    todo     │    │ data (JSON) │     │
│                     ├─────────────┤    │ type        │     │
│                     │ session_id  │    │ (text/tool/ │     │
│                     │ content     │    │  reasoning) │     │
│                     │ status      │    └─────────────┘     │
│                     │ priority    │                        │
│                     │ position    │    ┌─────────────┐     │
│                     └─────────────┘    │  permission │     │
│                                        ├─────────────┤     │
│                     ┌─────────────┐    │ project_id  │     │
│                     │   workspace │    │ data (JSON) │     │
│                     ├─────────────┤    │ (ruleset)   │     │
│                     │ id (PK)     │    └─────────────┘     │
│                     │ name        │                        │
│                     │ directory   │                        │
│                     └─────────────┘                        │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 文件系统存储

| 路径 | 用途 |
|------|------|
| `~/.local/share/opencode/` | 数据库、会话、快照 |
| `~/.config/opencode/` | 全局配置 |
| `./.opencode/` | 项目特定配置、代理、命令 |
| `~/.local/state/opencode/` | 运行时状态、日志 |

### 6.3 迁移系统

- **Drizzle Kit** 管理 Schema 迁移
- 迁移文件位于 `migration/` 目录
- 支持 JSON → SQLite 迁移 (遗留数据)
- WAL 模式支持并发访问

---

## 7. 权限系统

### 7.1 权限规则结构

```typescript
export const Rule = z.object({
  permission: z.string(),     // 权限类型 (read, edit, bash, ...)
  pattern: z.string(),        // 文件/命令模式
  action: z.enum(["allow", "deny", "ask"]),  // 动作
})

export type Ruleset = Rule[]
```

### 7.2 权限类型

| 权限 | 描述 | 模式示例 |
|------|------|----------|
| `read` | 读取文件 | `**/*.ts` |
| `edit` | 编辑文件 | `src/**/*` |
| `bash` | 执行命令 | `npm run *` |
| `glob` | 文件匹配 | `**/*.json` |
| `grep` | 代码搜索 | `*` |
| `external_directory` | 外部目录访问 | `/tmp/*` |
| `question` | 向用户提问 | `*` |

### 7.3 权限评估引擎

```typescript
// src/permission/next.ts

// 评估函数
export function evaluate(
  permission: string,
  pattern: string,
  ...rulesets: Ruleset[]
): Rule {
  // 查找最后匹配的规则
  // 支持多条规则集合并
}

// 请求权限
export const ask = fn(Request.partial({...}), async (input) => {
  // 1. 评估规则集
  const rule = evaluate(input.permission, input.pattern, ...rulesets)
  
  // 2. 如果 action="ask"，创建待处理请求
  if (rule.action === "ask") {
    const pending = createPendingRequest(input)
    Bus.publish("permission.asked", { request: pending })
    await waitForReply(pending.id)
  }
  
  // 3. 返回允许/拒绝
  return rule.action === "allow"
})

// 回复权限请求
export const reply = fn({...}, async (input) => {
  // 支持: "once", "always", "reject"
  // "always" 会更新规则集
})
```

---

## 8. 插件与扩展

### 8.1 插件系统

**插件加载位置**:
- `.opencode/plugins/` 目录
- npm 包 (`@opencode-ai/plugin-*`)

**插件 SDK**: `@opencode-ai/plugin`

**插件钩子**:
- `experimental.chat.system.transform` - 系统提示词转换
- `experimental.text.complete` - 文本补全
- 自定义工具注册

### 8.2 MCP (Model Context Protocol)

**连接类型**:
- **远程 HTTP** (`StreamableHTTPClientTransport`)
- **远程 SSE** (`SSEClientTransport`)
- **本地 stdio** (`StdioClientTransport`)

**MCP 功能**:
- 外部工具调用
- 提示词模板
- 资源读取

**OAuth 支持**:
```typescript
// src/mcp/auth.ts
export async function startAuth(mcpName: string) {
  // 1. 创建 OAuth Provider
  // 2. 触发 OAuth 流程
  // 3. 存储 tokens
}
```

### 8.3 ACP (Agent Communication Protocol)

**协议版本**: v1

**功能**:
- 会话加载/恢复
- 会话 fork
- MCP 能力协商
- 认证方法

**集成目标**: Zed 编辑器等 AI 编码助手

---

## 9. 关键技术决策

### 9.1 运行时选择：Bun

| 优势 | 说明 |
|------|------|
| **启动速度** | 比 Node.js 快 3-5 倍 |
| **原生 SQLite** | `bun:sqlite` 无需额外依赖 |
| **内置测试** | `bun test` 替代 Jest/Vitest |
| **单文件分发** | 简化部署 |

### 9.2 UI 框架：SolidJS + @opentui

| 选择 | 原因 |
|------|------|
| **SolidJS** | 细粒度响应式，无虚拟 DOM |
| **@opentui** | 专为终端 UI 设计 |
| **@clack/prompts** | 交互式 CLI 提示 |

### 9.3 AI SDK：Vercel AI SDK

| 优势 | 说明 |
|------|------|
| **统一接口** | 所有提供商使用相同 API |
| **流式支持** | `streamText()` 原生支持 |
| **工具调用** | 自动处理 tool calls |
| **类型安全** | 完整的 TypeScript 支持 |

### 9.4 数据库：SQLite + Drizzle

| 选择 | 原因 |
|------|------|
| **SQLite** | 零配置、嵌入式、可靠 |
| **Drizzle ORM** | 类型安全、迁移支持 |
| **WAL 模式** | 并发读写支持 |

### 9.5 文件监听：@parcel/watcher

| 优势 | 说明 |
|------|------|
| **原生后端** | Windows API / FSEvents / inotify |
| **高性能** | 比 chokidar 更快 |
| **忽略支持** | 内置 .gitignore 解析 |

---

## 10. 设计模式

### 10.1 上下文/作用域模式

```typescript
// src/project/instance.ts
export const Instance = {
  async provide<R>(input: { 
    directory: string
    init?: () => Promise<any>
    fn: () => R 
  }): Promise<R> {
    // 为目录创建/获取缓存实例
    const { project, sandbox } = await Project.fromDirectory(input.directory)
    await context.provide(ctx, async () => { 
      await input.init?.() 
    })
  },
  get directory() { return context.use().directory },
  get worktree() { return context.use().worktree },
  get project() { return context.use().project },
}
```

### 10.2 状态模式

```typescript
// src/agent/agent.ts
const state = Instance.state(async () => {
  // 延迟初始化状态
  const result: Record<string, Info> = {
    build: { name: "build", ... },
    plan: { name: "plan", ... },
    // ...
  }
  return result
})
```

### 10.3 事件总线 (Pub/Sub)

```typescript
// src/bus/index.ts
export const Bus = {
  publish(event: string, data: any) {
    // 发布事件到所有订阅者
  },
  subscribe(event: string, handler: (data) => void) {
    // 订阅事件
    return () => { /* 取消订阅 */ }
  },
  subscribeAll(handler: (event) => void) {
    // 订阅所有事件
  }
}
```

### 10.4 提供者抽象

```typescript
// src/provider/provider.ts
interface Provider {
  getSDK(model: Model): Promise<SDK>
  getLanguage(model: Model): Promise<LanguageModelV2>
  transform(messages: Message[]): Message[]
}
```

### 10.5 工具定义模式

```typescript
// src/tool/tool.ts
export function define<Parameters, Result>(
  id: string,
  init: Info<Parameters, Result>["init"]
): Info<Parameters, Result> {
  // 包装 execute 进行验证和截断
  return {
    id,
    init: async () => ({
      ...await init(),
      execute: async (args, ctx) => {
        // 参数验证
        // 权限检查
        // 执行
        // 结果截断
      }
    })
  }
}
```

---

## 附录

### A. 依赖总览

**核心依赖**:
- `ai` (Vercel AI SDK) - AI 抽象层
- `@ai-sdk/*` - 各 AI 提供商 SDK
- `hono` - HTTP 服务器
- `drizzle-orm` - ORM
- `yargs` - CLI 框架
- `@opentui/*` - TUI
- `solid-js` - 响应式框架
- `@modelcontextprotocol/sdk` - MCP
- `@parcel/watcher` - 文件监听

### B. 命令列表

| 命令 | 描述 |
|------|------|
| `run` | 执行 AI 任务 |
| `generate` | 生成代码 |
| `agent` | 代理管理 |
| `serve` | 启动 HTTP 服务器 |
| `session` | 会话管理 |
| `mcp` | MCP 服务器管理 |
| `auth` | 认证管理 |
| `github` | GitHub 集成 |
| `export/import` | 会话导出导入 |
| `upgrade` | 版本升级 |

### C. 配置文件示例

```json
{
  "$schema": "https://opencode.ai/schema.json",
  "model": "anthropic/claude-sonnet-4-20250514",
  "agent": {
    "build": {
      "model": "anthropic/claude-sonnet-4-20250514",
      "permission": {
        "read": "allow",
        "edit": "ask",
        "bash": "ask"
      }
    }
  },
  "provider": {
    "anthropic": {
      "apiKey": "sk-ant-..."
    }
  },
  "mcp": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem"]
    }
  }
}
```

---

*文档生成完成*
