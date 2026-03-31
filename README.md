# Claude Code Research Reports

[![Version](https://img.shields.io/badge/Claude_Code-v2.1.88-blueviolet?style=for-the-badge)](https://www.npmjs.com/package/@anthropic-ai/claude-code)
[![Reports](https://img.shields.io/badge/Research_Reports-20-red?style=for-the-badge)](docs/)

Independent research reports on **Claude Code v2.1.88**, focused on architecture, agent behavior, permission design, prompt assembly, MCP integration, and context management.

This repository is intentionally **documentation-only**. It collects original analysis and commentary about a publicly distributed software package. It is not presented as a runnable source release or supported CLI build.

**Language**: **English** | [中文](README_CN.md) | [日本語](README_JA.md) | [한국어](README_KO.md) | [Español](README_ES.md)

---

## Why This Repo Exists

Claude Code is one of the most sophisticated production AI coding agents available today. These reports aim to make its architecture and behavior easier to study:

- **Agent developers** can study production patterns for tool orchestration, context management, and recovery logic.
- **Security researchers** can inspect telemetry, remote control mechanisms, and permission boundaries.
- **AI researchers** can examine prompt assembly, model routing, and agent loop design.
- **Engineers** can use the reports as references when building CLI-based agent systems.

These reports are based on technical analysis of a **publicly distributed npm package** and are presented here as research commentary.

---

## What This Repository Covers

This repository is designed to answer a fairly specific set of questions about Claude Code:

- How is Claude Code architected internally?
- How does the main agent loop actually work?
- How are tools registered, validated, permission-checked, and executed?
- How is the system prompt assembled from static and dynamic parts?
- What telemetry and privacy-relevant data paths exist?
- How does the permission model work in practice?
- How does Claude Code integrate with MCP servers and external tools?
- How does it manage context pressure, compaction, and session persistence?
- What hidden features, internal codenames, and remote control mechanisms exist in the codebase?

If someone lands here searching for Claude Code internals, Claude Code architecture, Claude Code source analysis, Claude Code system prompts, Claude Code telemetry, or Claude Code MCP integration, this repository is meant to be a useful entry point.

---

## Research Reports

### Core Architecture Analysis

| # | Report | What You'll Learn | Link |
|---|--------|-------------------|------|
| 13 | **Architecture Overview** | A system-level synthesis of how Claude Code fits together across prompt assembly, agent loop, tools, permissions, compaction, persistence, and MCP | [Read →](docs/en/13-claude-code-architecture-overview.md) |
| 14 | **BashTool Security & Execution** | How Claude Code validates shell input, classifies read-only commands, routes sandboxed execution, and turns Bash into a controllable tool surface | [Read →](docs/en/14-bash-tool-security-and-execution.md) |
| 15 | **Memory & Instruction System** | How CLAUDE.md, MEMORY.md, session memory, and instruction layering shape long-session behavior and prompt continuity | [Read →](docs/en/15-memory-and-instruction-system.md) |
| 16 | **Transcripts, Compaction & Resume** | How transcript chains, sidechains, compact boundaries, resume, and session continuity work under the hood | [Read →](docs/en/16-transcripts-compaction-and-session-resume.md) |
| 06 | **Agent Loop Deep Dive** | The 785KB `query.ts` dissected — message flow, streaming tool execution, auto-compaction, error recovery, sub-agent spawning | [Read →](docs/en/06-agent-loop-deep-dive.md) |
| 07 | **Tool System Architecture** | How 40+ tools are registered, validated, permission-checked, and executed in parallel. The `buildTool` factory pattern. | [Read →](docs/en/07-tool-system-architecture.md) |
| 08 | **Permission & Security Model** | Allowlists, blocklists, auto-approve rules, YOLO mode, sandbox integration, and the permission decision tree | [Read →](docs/en/08-permission-security-model.md) |
| 09 | **System Prompt Engineering** | How the 15,000+ token system prompt is assembled from 20+ parts — context injection, tool descriptions, memory, and dynamic rules | [Read →](docs/en/09-system-prompt-engineering.md) |
| 10 | **MCP Integration & Plugin System** | Model Context Protocol client implementation — server lifecycle, tool discovery, OAuth, and transport layers | [Read →](docs/en/10-mcp-integration.md) |
| 11 | **Context Window Management** | Auto-compaction, conversation compression, token counting, and how Claude Code fights the context limit | [Read →](docs/en/11-context-window-management.md) |
| 12 | **State Management & Persistence** | Session state, conversation history, memory system, file persistence, and cross-session data flow | [Read →](docs/en/12-state-management.md) |
| 17 | **Bridge System & Remote Sessions** | How Claude Desktop talks to the CLI — session lifecycle, JWT auth, work secrets, multi-session coordination, capacity management | [Read →](docs/en/17-bridge-system-and-remote-sessions.md) |
| 18 | **React/Ink Terminal UI** | Using React to render a TUI — Yoga layout, custom components, event system, streaming output, performance optimization | [Read →](docs/en/18-react-ink-terminal-ui.md) |
| 19 | **Streaming & Transport Layers** | WebSocket/SSE/HTTP hybrid transport, batch uploading, backpressure, NDJSON protocol, reconnection logic | [Read →](docs/en/19-streaming-and-transport-layers.md) |
| 20 | **Slash Commands & Cost Tracking** | 80+ command registry, fuzzy search, feature-gated commands, token-to-USD cost tracking, billing integration | [Read →](docs/en/20-slash-commands-and-cost-tracking.md) |

### Discovery & Investigation Reports

| # | Report | What You'll Learn | Link |
|---|--------|-------------------|------|
| 01 | **Telemetry & Data Collection** | Dual analytics pipeline, environment fingerprinting, what gets collected and how | [Read →](docs/en/01-telemetry-and-privacy.md) |
| 02 | **Hidden Features & Codenames** | Animal codenames, feature flags, and internal vs external build differences | [Read →](docs/en/02-hidden-features-and-codenames.md) |
| 03 | **Undercover Mode** | How Anthropic employees can hide AI authorship signals in public repos | [Read →](docs/en/03-undercover-mode.md) |
| 04 | **Remote Control & Killswitches** | Server-side settings, blocking dialogs, GrowthBook flags, and emergency controls | [Read →](docs/en/04-remote-control-and-killswitches.md) |
| 05 | **Future Roadmap** | KAIROS, Numbat, future model hints, unreleased tools, and roadmap clues | [Read →](docs/en/05-future-roadmap.md) |

---

## Reading Guide

### Start here if you want the big picture

- [Architecture Overview](docs/en/13-claude-code-architecture-overview.md)
- [Agent Loop Deep Dive](docs/en/06-agent-loop-deep-dive.md)
- [Tool System Architecture](docs/en/07-tool-system-architecture.md)
- [Permission & Security Model](docs/en/08-permission-security-model.md)
- [BashTool Security](docs/en/14-bash-tool-security-and-execution.md)

### Read these if you care about prompts, memory, and context

- [System Prompt Engineering](docs/en/09-system-prompt-engineering.md)
- [Memory & Instruction System](docs/en/15-memory-and-instruction-system.md)
- [Context Window Management](docs/en/11-context-window-management.md)
- [State Management](docs/en/12-state-management.md)
- [Transcripts, Compaction & Resume](docs/en/16-transcripts-compaction-and-session-resume.md)

### Read these if you care about infrastructure and networking

- [Bridge System & Remote Sessions](docs/en/17-bridge-system-and-remote-sessions.md)
- [React/Ink Terminal UI](docs/en/18-react-ink-terminal-ui.md)
- [Streaming & Transport Layers](docs/en/19-streaming-and-transport-layers.md)
- [Slash Commands & Cost Tracking](docs/en/20-slash-commands-and-cost-tracking.md)
- [MCP Integration](docs/en/10-mcp-integration.md)

### Read these if you care about security, privacy, or control surfaces

- [Telemetry & Privacy](docs/en/01-telemetry-and-privacy.md)
- [Remote Control & Killswitches](docs/en/04-remote-control-and-killswitches.md)
- [Permission & Security Model](docs/en/08-permission-security-model.md)

### Read these if you care about unusual or hidden behavior

- [Hidden Features & Codenames](docs/en/02-hidden-features-and-codenames.md)
- [Undercover Mode](docs/en/03-undercover-mode.md)
- [Future Roadmap](docs/en/05-future-roadmap.md)

For a topic-based report index, see [docs/en/README.md](docs/en/README.md).

---

## Analyzed Codebase Snapshot

These figures refer to the **analyzed Claude Code codebase snapshot**, not to the contents of this documentation repository.

| Metric | Value |
|--------|-------|
| TypeScript Source Files | **1,884** |
| Total Lines of Code | **512,664** |
| Largest Single File | `query.ts` — **785KB** |
| Built-in Tools | **40+** |
| Slash Commands | **80+** |
| npm Dependencies | **192 packages** |
| Feature-Gated Modules | **108** |
| Runtime Model | Bun-built package targeting Node.js |

---

## Architecture Overview

```
                          ┌─────────────────┐
                          │   User Input     │
                          │ (CLI / SDK / IDE)│
                          └────────┬────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │        Entry Layer           │
                    │                              │
                    │  cli.tsx → main.tsx → REPL   │
                    │              └→ QueryEngine   │
                    └──────────────┬──────────────┘
                                   │
              ┌────────────────────▼────────────────────┐
              │           Query Engine Core              │
              │                                         │
              │  ┌─────────────────────────────────┐    │
              │  │  System Prompt Assembly          │    │
              │  │  (15K+ tokens, 20+ parts)       │    │
              │  └─────────────┬───────────────────┘    │
              │                │                         │
              │  ┌─────────────▼───────────────────┐    │
              │  │  Agent Loop (query.ts — 785KB)  │    │
              │  │                                  │    │
              │  │  User msg → Claude API → Response│    │
              │  │       ↑                    │     │    │
              │  │       │    tool_use? ──→ YES     │    │
              │  │       │         │               │    │
              │  │       │    Execute tools        │    │
              │  │       │    (parallel)           │    │
              │  │       │         │               │    │
              │  │       └─── tool_result ◄────┘   │    │
              │  └─────────────────────────────────┘    │
              │                                         │
              │  ┌─────────────────────────────────┐    │
              │  │  Harness Layer                   │    │
              │  │  • Permission checks             │    │
              │  │  • Streaming & concurrency       │    │
              │  │  • Auto-compaction               │    │
              │  │  • Sub-agent management          │    │
              │  │  • Cost tracking                 │    │
              │  │  • Error recovery                │    │
              │  │  • MCP orchestration             │    │
              │  │  • Telemetry & logging           │    │
              │  └─────────────────────────────────┘    │
              └────────────────────┬────────────────────┘
                                   │
         ┌─────────────────────────▼─────────────────────────┐
         │              Tool Layer (40+ tools)               │
         │                                                     │
         │  Read / Write / Edit / Bash / Glob / Grep / Agent  │
         │  MCP tools / task tools / notebook tools / more    │
         └─────────────────────────────────────────────────────┘
```

---

## Observed Module Layout

The following tree describes the **analyzed source layout** discussed throughout the reports:

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

This repository itself remains documentation-only:

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

## Usage

- Read reports directly from `docs/`
- Share links to individual reports
- Treat this repository as a documentation archive, not a software distribution

See [QUICKSTART.md](QUICKSTART.md) for a minimal reading guide.

---

## Keywords

Claude Code, Anthropic, Claude Code source analysis, Claude Code architecture, Claude Code agent loop, Claude Code system prompts, Claude Code telemetry, Claude Code permission model, Claude Code MCP, Claude Code hidden features, Claude Code reverse engineering, AI coding agent architecture, Model Context Protocol, prompt engineering, context management, persistence, security research

---

## Legal Note

This project constitutes **research, commentary, and educational analysis** of a publicly distributed software package (the npm package `@anthropic-ai/claude-code`).

The original reports in `docs/` are the repository maintainer's own commentary and analysis. If you believe any content here infringes your rights, please open an issue for prompt review.
