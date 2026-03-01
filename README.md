# OpenCode

**AI 驱动的代码助手** - 提供 CLI、TUI 和 HTTP API，支持多模型 AI 编码、文件操作、命令执行和会话管理。

## 核心功能

- **代码生成** - 通过 AI 代理生成、编辑代码
- **文件操作** - 读取、编辑、创建、删除文件
- **命令执行** - 在沙盒中执行 Shell 命令
- **代码搜索** - 支持 grep、codesearch、LSP
- **会话管理** - 持久化对话历史，支持 fork/resume
- **多模型支持** - 20+ AI 提供商，统一接口
- **MCP 集成** - Model Context Protocol 外部工具
- **权限控制** - 基于规则的工具执行权限

## 技术栈

- **运行时**: Bun Runtime + TypeScript + ES Modules
- **UI 层**: CLI (yargs) + TUI (@opentui/solid) + Web (Hono)
- **AI 层**: Vercel AI SDK + 20+ @ai-sdk/* 提供商
- **数据层**: SQLite (Drizzle ORM) + 文件系统 + 配置系统
- **协议层**: MCP (Model Context Protocol) + ACP (Agent Protocol)

## 快速开始

```bash
# 安装依赖
bun install

# 启动 CLI
bun run cli

# 启动 TUI
bun run tui

# 启动 HTTP 服务器
bun run serve
```

## 项目结构

```
opencode-design-study/
├── OpenCode 架构文档.md    # 详细架构设计文档
├── opencode-design-doc.md  # 完整设计文档
└── README.md              # 项目说明
```

## 文档

- [OpenCode 架构文档](./OpenCode%20架构文档.md)
- [设计文档](./opencode-design-doc.md)

## 许可证

MIT
