# What's Coming Next for Claude Code: Unreleased Features Hiding in the Source

*Everything the source reveals about upcoming models, autonomous agents, and hidden tools in Claude Code v2.1.88*

## 1. The Next Model: Numbat

The clearest signal about what's coming next:

```typescript
// src/constants/prompts.ts:402
// @[MODEL LAUNCH]: Remove this section when we launch numbat.
```

**Numbat** (袋食蚁兽, the banded anteater) is the codename for an upcoming model. The comment suggests the current output efficiency section will be revised when Numbat ships, which implies it may have better native output control than current models — meaning fewer prompt-level workarounds to manage verbosity.

### Upcoming Version Numbers

The undercover mode instructions accidentally reveal future version numbers:

```typescript
// src/utils/undercover.ts:49
- Unreleased model version numbers (e.g., opus-4-7, sonnet-4-8)
```

So **Opus 4.7** and **Sonnet 4.8** are at least planned, if not already in development.

### The Codename Lineage

Tracing the evolution through the source code:

```
Fennec (耳廓狐) → Opus 4.6 → [Numbat?]
Capybara (水豚) → Sonnet v8 → [?]
Tengu (天狗) → product/telemetry prefix (not a model per se)
```

The Fennec-to-Opus migration is well-documented in the codebase:

```typescript
// src/migrations/migrateFennecToOpus.ts:7-11
// fennec-latest → opus
// fennec-latest[1m] → opus[1m]
// fennec-fast-latest → opus[1m] + fast mode
```

### The MODEL LAUNCH Checklist

The codebase contains 20+ `@[MODEL LAUNCH]` markers scattered across the codebase — essentially a launch checklist embedded in code comments. When a new model ships, engineers need to update:

- Default model names (`FRONTIER_MODEL_NAME`)
- Model family IDs
- Knowledge cutoff dates
- Pricing tables
- Context window configurations
- Thinking mode support flags
- Display name mappings
- Migration scripts

This is solid engineering practice. Having the checklist live in the code itself means it's version-controlled and hard to miss.

## 2. KAIROS: The Autonomous Agent Mode

This is the biggest unreleased feature in the codebase, and it is ambitious. KAIROS transforms Claude Code from a tool you interact with into an agent that operates independently.

### How It Thinks About Itself

From the system prompt (`src/constants/prompts.ts:860-913`):

```
You are running autonomously.
You will receive <tick> prompts that keep you alive between turns.
If you have nothing useful to do, call SleepTool.
Bias toward action — read files, make changes, commit without asking.

## Terminal focus
- Unfocused: The user is away. Lean heavily into autonomous action.
- Focused: The user is watching. Be more collaborative.
```

The terminal focus detection is clever — it adjusts how aggressively autonomous the agent is based on whether you're actually watching. Walk away from your terminal and it becomes more independent. Come back and it shifts to collaborative mode.

### The Tools KAIROS Gets

| Tool | Feature Flag | What It Does |
|------|-------------|-------------|
| SleepTool | KAIROS / PROACTIVE | Paces itself between autonomous actions |
| SendUserFileTool | KAIROS | Proactively sends files to users |
| PushNotificationTool | KAIROS / KAIROS_PUSH_NOTIFICATION | Sends push notifications to your devices |
| SubscribePRTool | KAIROS_GITHUB_WEBHOOKS | Monitors GitHub PRs via webhooks |
| BriefTool | KAIROS_BRIEF | Sends proactive status updates |

The picture this paints is an agent that can watch your GitHub PRs, make changes, commit and push independently, and ping you on your phone when something needs attention. That's a significant leap from "coding assistant."

## 3. Voice Mode

Push-to-talk voice input is fully implemented and gated behind `VOICE_MODE`:

```typescript
// src/voice/voiceModeEnabled.ts
// Connects to Anthropic's voice_stream WebSocket endpoint
// Uses conversation_engine backed models for speech-to-text
// Hold-to-talk: hold keybinding to record, release to submit
```

Key constraints:
- OAuth only — no API key, Bedrock, or Vertex support
- Uses mTLS for WebSocket connections (solid security choice)
- Has its own killswitch: `tengu_amber_quartz_disabled`

The OAuth-only restriction suggests this might be tied to Anthropic's consumer product strategy rather than the developer-focused API offering.

## 4. Unreleased Tools Sitting in the Source

The source contains a whole collection of tools that are built but not enabled for external users:

| Tool | Feature Flag | What It Does |
|------|-------------|-------------|
| **WebBrowserTool** | `WEB_BROWSER_TOOL` | Built-in browser automation (internal codename: "bagel") |
| **TerminalCaptureTool** | `TERMINAL_PANEL` | Captures and monitors terminal panels |
| **WorkflowTool** | `WORKFLOW_SCRIPTS` | Executes predefined workflow scripts |
| **MonitorTool** | `MONITOR_TOOL` | System and process monitoring |
| **SnipTool** | `HISTORY_SNIP` | Trims conversation history to manage context |
| **ListPeersTool** | `UDS_INBOX` | Discovers peer agents via Unix domain sockets |
| **RemoteTriggerTool** | `AGENT_TRIGGERS_REMOTE` | Triggers remote agents |
| **TungstenTool** | ant-only | Internal performance monitoring |
| **VerifyPlanExecutionTool** | VERIFY_PLAN env | Verifies plan execution correctness |
| **OverflowTestTool** | `OVERFLOW_TEST_TOOL` | Tests context window overflow handling |
| **SubscribePRTool** | `KAIROS_GITHUB_WEBHOOKS` | GitHub PR webhook subscriptions |

The WebBrowserTool (codename "bagel") is interesting because it suggests Claude Code is moving toward being able to interact with web applications directly — not just code. Combined with KAIROS, you could imagine an agent that writes code, tests it in a browser, and iterates without human involvement.

## 5. Coordinator Mode

Multi-agent coordination is in progress:

```typescript
// src/coordinator/coordinatorMode.ts
// Feature flag: COORDINATOR_MODE
```

This enables multiple Claude Code agents to work together on coordinated tasks with shared state and inter-agent messaging. Combined with the ListPeersTool and Unix domain socket discovery, this points toward a future where you spin up a team of agents that divide work among themselves.

## 6. The Buddy System (Virtual Pets)

This one's just fun. There's a complete virtual pet companion system implemented but not shipped:

- **18 species**: duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk
- **5 rarity tiers**: Common (60%), Uncommon (25%), Rare (10%), Epic (4%), Legendary (1%)
- **7 hats**: crown, tophat, propeller, halo, wizard, beanie, tinyduck
- **5 stats**: DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK
- **1% shiny chance**: Sparkle variant of any species
- **Deterministic generation**: Your buddy is derived from a hash of your user ID

Source: `src/buddy/`

It is a gacha system for developer tools. The SNARK stat on a coding companion is a nice touch.

## 7. Dream Task (Background Memory)

```
// src/tasks/DreamTask/
// Auto-dreaming feature that works in the background
// Controlled by 'tengu_onyx_plover' feature flag
```

This is a background subagent that autonomously processes and consolidates the AI's memories during idle time. Think of it as the AI reflecting on what it's learned from your codebase while you're away.
