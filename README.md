# OpenCode 设计研究

研究 OpenCode 源码实现原理和架构的分析文档仓库。

## 项目目的

本仓库用于存放 OpenCode 项目的架构分析文档和设计研究成果，主要包含：

- OpenCode 源码实现原理分析
- 系统架构设计文档
- 核心组件解析
- 技术栈与关键技术决策
- 可用于团队分享的技术文档

## 文档内容

| 文档 | 描述 |
|------|------|
| [OpenCode 架构文档](./OpenCode%20架构文档.md) | OpenCode 系统架构详细分析 |
| [设计文档](./opencode-design-doc.md) | 完整的设计研究成果，包含核心组件、数据流、权限系统等 |

## 研究范围

- **架构分析**: 分层架构、组件设计、事件总线
- **AI 集成**: 多模型支持、Provider 管理、流式处理
- **协议实现**: MCP (Model Context Protocol)、ACP (Agent Protocol)
- **数据持久化**: SQLite + Drizzle ORM、会话管理
- **权限系统**: 基于规则的工具执行控制

## 版本信息

- **分析版本**: Commit `f5eade1d2b95562c7fb58e3041e662a8b2b611b6`
- **包版本**: 1.2.15
- **更新日期**: 2026-03-01

## 关于 OpenCode

OpenCode 是一个 AI 驱动的代码助手，提供 CLI、TUI 和 HTTP API，支持：
- 代码生成与文件操作
- 命令执行与代码搜索
- 会话管理与多模型支持
- MCP 外部工具集成

## 许可证

MIT
