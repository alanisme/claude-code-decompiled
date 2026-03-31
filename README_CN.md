# Claude Code 研究报告集

[![Version](https://img.shields.io/badge/Claude_Code-v2.1.88-blueviolet?style=for-the-badge)](https://www.npmjs.com/package/@anthropic-ai/claude-code)
[![Reports](https://img.shields.io/badge/研究报告-20-red?style=for-the-badge)](docs/)

围绕 **Claude Code v2.1.88** 的独立研究报告，重点分析其架构、代理循环、权限模型、系统提示词、MCP 集成与上下文管理。

本仓库刻意保持为**纯文档仓库**。这里只收录原创分析与评论，不以可运行源码分发或 CLI 构建为目标。

**语言**: [English](README.md) | **中文** | [日本語](README_JA.md) | [한국어](README_KO.md) | [Español](README_ES.md)

---

## 仓库目的

Claude Code 是目前最复杂的生产级 AI 编程代理之一。这些报告试图把它的内部机制拆开说明：

- **代理开发者** 可以研究工具编排、上下文管理和错误恢复模式
- **安全研究者** 可以观察遥测、远程控制机制和权限边界
- **AI 研究者** 可以分析系统提示词组装、模型路由与代理循环设计
- **工程师** 可以把这些报告当作构建 CLI 代理系统的参考材料

这些内容基于对**公开发布 npm 包**的技术分析，定位是研究与评论。

---

## 这个仓库主要回答什么问题

这个仓库的内容，主要围绕下面这些问题展开：

- Claude Code 的内部架构到底是什么样？
- 它的主代理循环是怎么跑起来的？
- 工具系统如何注册、校验、做权限检查并执行？
- 系统提示词是如何由静态段和动态段拼装出来的？
- 遥测与隐私相关的数据路径有哪些？
- 权限模型在实际执行时是怎么决策的？
- Claude Code 是怎么接入 MCP 服务器和外部工具的？
- 它如何处理上下文压力、自动压缩和会话持久化？
- 源码里有哪些隐藏功能、内部代号和远程控制机制？

如果有人是因为搜索这些问题来到这里，例如“Claude Code 架构”“Claude Code 源代码分析”“Claude Code 系统提示词”“Claude Code 遥测”“Claude Code MCP”“Claude Code 权限模型”，这个仓库就是为这些入口准备的。

---

## 研究报告

### 核心架构分析

| # | 报告 | 核心内容 | 链接 |
|---|------|----------|------|
| 13 | **整体架构总览** | 从系统层面串起 Claude Code 的提示词装配、主循环、工具运行时、权限、压缩、持久化与 MCP | [阅读 →](docs/zh/13-Claude-Code-整体架构总览.md) |
| 14 | **BashTool 安全与执行模型** | 系统如何校验 shell 输入、识别只读命令、决定沙箱路径，并把 Bash 变成可控工具表面 | [阅读 →](docs/zh/14-BashTool-安全与执行模型.md) |
| 15 | **记忆与指令系统** | CLAUDE.md、MEMORY.md、session memory 与多层指令如何共同影响长会话行为与提示词连续性 | [阅读 →](docs/zh/15-记忆与指令系统.md) |
| 16 | **会话存储、压缩与恢复机制** | transcript 链、sidechain、compact boundary、resume 与会话连续性到底是怎么实现的 | [阅读 →](docs/zh/16-会话存储-压缩与恢复机制.md) |
| 06 | **代理循环深度剖析** | 785KB 的 query.ts 全面解析 — 消息流、流式工具执行、自动压缩、错误恢复、子代理生成 | [阅读 →](docs/zh/06-代理循环深度剖析.md) |
| 07 | **工具系统架构** | 40+ 工具如何注册、验证、权限检查和并行执行。buildTool 工厂模式详解 | [阅读 →](docs/zh/07-工具系统架构.md) |
| 08 | **权限与安全模型** | 白名单、黑名单、自动批准规则、YOLO 模式、沙箱集成和权限决策树 | [阅读 →](docs/zh/08-权限与安全模型.md) |
| 09 | **系统提示词工程** | 15,000+ token 的系统提示词如何从 20+ 个部分组装 — 上下文注入、工具描述、记忆和动态规则 | [阅读 →](docs/zh/09-系统提示词工程.md) |
| 10 | **MCP 集成与插件系统** | Model Context Protocol 客户端实现 — 服务器生命周期、工具发现、OAuth、传输层 | [阅读 →](docs/zh/10-MCP-集成与插件系统.md) |
| 11 | **上下文窗口管理** | 自动压缩、会话压缩、token 计数，以及 Claude Code 如何对抗上下文限制 | [阅读 →](docs/zh/11-上下文窗口管理.md) |
| 12 | **状态管理与持久化** | 会话状态、会话历史、记忆系统、文件持久化和跨会话数据流 | [阅读 →](docs/zh/12-状态管理与持久化.md) |
| 17 | **Bridge 系统与远程会话** | Claude Desktop 与 CLI 的通信机制 — 会话生命周期、JWT 认证、工作密钥、多会话协调 | [阅读 →](docs/zh/17-Bridge系统与远程会话.md) |
| 18 | **React/Ink 终端 UI 系统** | 用 React 渲染终端界面 — Yoga 布局引擎、自定义组件、事件系统、流式输出 | [阅读 →](docs/zh/18-React-Ink终端UI系统.md) |
| 19 | **流式传输与 Transport 层** | WebSocket/SSE/HTTP 混合传输、批量上传、背压控制、NDJSON 协议、重连逻辑 | [阅读 →](docs/zh/19-流式传输与Transport层.md) |
| 20 | **斜杠命令与费用追踪** | 80+ 命令注册表、模糊搜索、Feature-Gated 命令、token 转 USD 费用追踪 | [阅读 →](docs/zh/20-斜杠命令与费用追踪.md) |

### 发现与调查报告

| # | 报告 | 核心内容 | 链接 |
|---|------|----------|------|
| 01 | **遥测与数据收集** | 双分析管道、环境指纹、收集内容与方式 | [阅读 →](docs/zh/01-遥测与隐私分析.md) |
| 02 | **隐藏功能与代号** | 动物代号体系、Feature Flag 与内外部构建差异 | [阅读 →](docs/zh/02-隐藏功能与模型代号.md) |
| 03 | **卧底模式** | 公开仓库中的自动隐身机制及其影响 | [阅读 →](docs/zh/03-卧底模式分析.md) |
| 04 | **远程控制与紧急开关** | 服务端设置、阻断对话框、GrowthBook 标志和紧急控制面 | [阅读 →](docs/zh/04-远程控制与紧急开关.md) |
| 05 | **未来路线图** | KAIROS、Numbat、未来模型线索与未发布工具 | [阅读 →](docs/zh/05-未来路线图.md) |

---

## 阅读路线

### 想先建立整体认识

- [docs/zh/13-Claude-Code-整体架构总览.md](docs/zh/13-Claude-Code-整体架构总览.md)
- [docs/zh/06-代理循环深度剖析.md](docs/zh/06-代理循环深度剖析.md)
- [docs/zh/07-工具系统架构.md](docs/zh/07-工具系统架构.md)
- [docs/zh/08-权限与安全模型.md](docs/zh/08-权限与安全模型.md)
- [docs/zh/14-BashTool-安全与执行模型.md](docs/zh/14-BashTool-安全与执行模型.md)

### 想看提示词、记忆和上下文管理

- [docs/zh/09-系统提示词工程.md](docs/zh/09-系统提示词工程.md)
- [docs/zh/15-记忆与指令系统.md](docs/zh/15-记忆与指令系统.md)
- [docs/zh/11-上下文窗口管理.md](docs/zh/11-上下文窗口管理.md)
- [docs/zh/12-状态管理与持久化.md](docs/zh/12-状态管理与持久化.md)
- [docs/zh/16-会话存储-压缩与恢复机制.md](docs/zh/16-会话存储-压缩与恢复机制.md)

### 想看安全、隐私与控制面

- [docs/zh/01-遥测与隐私分析.md](docs/zh/01-遥测与隐私分析.md)
- [docs/zh/04-远程控制与紧急开关.md](docs/zh/04-远程控制与紧急开关.md)
- [docs/zh/08-权限与安全模型.md](docs/zh/08-权限与安全模型.md)

### 想看更“出圈”的隐藏内容

- [docs/zh/02-隐藏功能与模型代号.md](docs/zh/02-隐藏功能与模型代号.md)
- [docs/zh/03-卧底模式分析.md](docs/zh/03-卧底模式分析.md)
- [docs/zh/05-未来路线图.md](docs/zh/05-未来路线图.md)

按主题整理的中文导航页见 [docs/zh/README.md](docs/zh/README.md)。

---

## 被分析代码库概况

下面这些数字指的是**被分析的 Claude Code 代码快照**，不是当前这个纯文档仓库本身的文件数量。

| 指标 | 数值 |
|------|------|
| TypeScript 源文件 | **1,884** |
| 总代码行数 | **512,664** |
| 最大单文件 | `query.ts` — **785KB** |
| 内置工具 | **40+** |
| 斜杠命令 | **80+** |
| npm 依赖 | **192 个包** |
| Feature-Gated 模块 | **108 个** |
| 运行时模型 | Bun 构建、Node.js 目标包 |

---

## 架构概览

```
                          ┌─────────────────┐
                          │   用户输入       │
                          │ (CLI / SDK / IDE)│
                          └────────┬────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │        入口层                │
                    │                              │
                    │  cli.tsx → main.tsx → REPL   │
                    │              └→ QueryEngine   │
                    └──────────────┬──────────────┘
                                   │
              ┌────────────────────▼────────────────────┐
              │           查询引擎核心                    │
              │                                         │
              │  ┌─────────────────────────────────┐    │
              │  │  系统提示词组装                    │    │
              │  │  (15K+ tokens, 20+ 个部分)       │    │
              │  └─────────────┬───────────────────┘    │
              │                │                         │
              │  ┌─────────────▼───────────────────┐    │
              │  │  代理循环 (query.ts — 785KB)     │    │
              │  │                                  │    │
              │  │  用户消息 → Claude API → 响应     │    │
              │  │       ↑                    │     │    │
              │  │       │    tool_use? ──→ 是      │    │
              │  │       │         │               │    │
              │  │       │    执行工具（并行）       │    │
              │  │       │         │               │    │
              │  │       └─── tool_result ◄────┘   │    │
              │  └─────────────────────────────────┘    │
              │                                         │
              │  ┌─────────────────────────────────┐    │
              │  │  生产级线束层                     │    │
              │  │  • 权限检查                       │    │
              │  │  • 流式传输与并发                  │    │
              │  │  • 自动压缩                       │    │
              │  │  • 子代理管理                     │    │
              │  │  • 成本追踪                       │    │
              │  │  • 错误恢复                       │    │
              │  │  • MCP 编排                       │    │
              │  │  • 遥测与日志                     │    │
              │  └─────────────────────────────────┘    │
              └────────────────────┬────────────────────┘
                                   │
         ┌─────────────────────────▼─────────────────────────┐
         │                  工具层（40+ 工具）                │
         │                                                     │
         │   Read / Write / Edit / Bash / Glob / Grep / Agent │
         │   MCP 工具 / task 工具 / notebook 工具 / 更多      │
         └─────────────────────────────────────────────────────┘
```

---

## 被分析源码结构

下面这棵树描述的是**报告里分析的源码结构**：

```
src/
├── main.tsx
├── QueryEngine.ts
├── query.ts
├── Tool.ts
├── Task.ts
├── tools.ts
├── commands.ts
├── context.ts
├── cost-tracker.ts
├── setup.ts
├── bridge/
├── cli/
├── commands/
├── components/
├── entrypoints/
├── hooks/
├── services/
├── state/
├── tasks/
├── tools/
├── types/
├── utils/
└── vendor/
```

当前仓库自身仍然保持为纯文档结构：

```
docs/
├── en/
└── zh/

README.md
README_CN.md
README_JA.md
README_KO.md
README_ES.md
QUICKSTART.md
```

---

## 使用方式

- 直接阅读 `docs/` 下的报告
- 按单篇文档进行引用或分享
- 将本仓库视为文档归档，而不是软件发行包

最简阅读说明见 [QUICKSTART.md](QUICKSTART.md)。

---

## 关键词

Claude Code、Anthropic、Claude Code 源代码分析、Claude Code 架构、Claude Code 代理循环、Claude Code 系统提示词、Claude Code 遥测、Claude Code 权限模型、Claude Code MCP、Claude Code 隐藏功能、Claude Code 逆向工程、AI 编程代理架构、Model Context Protocol、提示词工程、上下文管理、持久化、安全研究

---

## 法律说明

本项目构成对公开发布软件包（npm 上的 `@anthropic-ai/claude-code`）的**研究、评论和教育分析**。

`docs/` 目录中的报告为维护者撰写的原创分析与评论。如有权利疑虑，请提交 issue 以便及时复核。
