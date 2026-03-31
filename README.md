# Claude Code Research Reports

[![Version](https://img.shields.io/badge/Claude_Code-v2.1.88-blueviolet?style=for-the-badge)](https://www.npmjs.com/package/@anthropic-ai/claude-code)
[![Reports](https://img.shields.io/badge/Research_Reports-12%2B-red?style=for-the-badge)](docs/)

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

## Research Reports

### Core Architecture Analysis

| # | Report | What You'll Learn | Link |
|---|--------|-------------------|------|
| 06 | **Agent Loop Deep Dive** | The 785KB `query.ts` dissected — message flow, streaming tool execution, auto-compaction, error recovery, sub-agent spawning | [Read →](docs/en/06-agent-loop-deep-dive.md) |
| 07 | **Tool System Architecture** | How 40+ tools are registered, validated, permission-checked, and executed in parallel. The `buildTool` factory pattern. | [Read →](docs/en/07-tool-system-architecture.md) |
| 08 | **Permission & Security Model** | Allowlists, blocklists, auto-approve rules, YOLO mode, sandbox integration, and the permission decision tree | [Read →](docs/en/08-permission-security-model.md) |
| 09 | **System Prompt Engineering** | How the 15,000+ token system prompt is assembled from 20+ parts — context injection, tool descriptions, memory, and dynamic rules | [Read →](docs/en/09-system-prompt-engineering.md) |
| 10 | **MCP Integration & Plugin System** | Model Context Protocol client implementation — server lifecycle, tool discovery, OAuth, and transport layers | [Read →](docs/en/10-mcp-integration.md) |
| 11 | **Context Window Management** | Auto-compaction, conversation compression, token counting, and how Claude Code fights the context limit | [Read →](docs/en/11-context-window-management.md) |
| 12 | **State Management & Persistence** | Session state, conversation history, memory system, file persistence, and cross-session data flow | [Read →](docs/en/12-state-management.md) |

### Discovery & Investigation Reports

| # | Report | What You'll Learn | Link |
|---|--------|-------------------|------|
| 01 | **Telemetry & Data Collection** | Dual analytics pipeline, environment fingerprinting, what gets collected and how | [Read →](docs/en/01-telemetry-and-privacy.md) |
| 02 | **Hidden Features & Codenames** | Animal codenames, feature flags, and internal vs external build differences | [Read →](docs/en/02-hidden-features-and-codenames.md) |
| 03 | **Undercover Mode** | How Anthropic employees can hide AI authorship signals in public repos | [Read →](docs/en/03-undercover-mode.md) |
| 04 | **Remote Control & Killswitches** | Server-side settings, blocking dialogs, GrowthBook flags, and emergency controls | [Read →](docs/en/04-remote-control-and-killswitches.md) |
| 05 | **Future Roadmap** | KAIROS, Numbat, future model hints, unreleased tools, and roadmap clues | [Read →](docs/en/05-future-roadmap.md) |

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

## Legal Note

This project constitutes **research, commentary, and educational analysis** of a publicly distributed software package (the npm package `@anthropic-ai/claude-code`).

The original reports in `docs/` are the repository maintainer's own commentary and analysis. If you believe any content here infringes your rights, please open an issue for prompt review.
