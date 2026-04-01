# Multi-Agent and Swarm System: Technical Analysis

## Overview

Claude Code implements a sophisticated multi-agent collaboration system internally referred to as the "swarm" architecture. This system enables a single Claude Code session to spawn, coordinate, and manage multiple parallel agent instances -- referred to as "teammates" -- that collaborate on complex tasks under the supervision of a designated team leader. The architecture supports three distinct execution backends (tmux pane-based, iTerm2 pane-based, and in-process), a file-based mailbox communication protocol, a centralized permission delegation model, and structured lifecycle management including reconnection and crash recovery.

This document provides a thorough technical analysis of the swarm subsystem based on primary source code located primarily under `src/utils/swarm/`, `src/tasks/`, and `src/tools/AgentTool/`.

---

## 1. Swarm Architecture

### Core Design Principles

The swarm system follows a **leader-worker topology**. One Claude Code instance operates as the "team lead" (identified by the constant `TEAM_LEAD_NAME = 'team-lead'` in `src/utils/swarm/constants.ts:1`), while subordinate instances operate as "teammates." Each team is identified by a sanitized team name and tracked through a persistent JSON configuration file stored at `~/.claude/teams/{team-name}/config.json`.

The team file (`TeamFile` type defined in `src/utils/swarm/teamHelpers.ts:65-90`) serves as the system's source of truth for team composition:

```typescript
// src/utils/swarm/teamHelpers.ts:65-90
export type TeamFile = {
  name: string
  description?: string
  createdAt: number
  leadAgentId: string
  leadSessionId?: string
  hiddenPaneIds?: string[]
  teamAllowedPaths?: TeamAllowedPath[]
  members: Array<{
    agentId: string
    name: string
    agentType?: string
    model?: string
    prompt?: string
    color?: string
    planModeRequired?: boolean
    joinedAt: number
    tmuxPaneId: string
    cwd: string
    worktreePath?: string
    sessionId?: string
    subscriptions: string[]
    backendType?: BackendType
    isActive?: boolean
    mode?: PermissionMode
  }>
}
```

Each member entry captures its backend type (`'tmux' | 'iterm2' | 'in-process'`), its working directory, optional git worktree path for filesystem isolation, color assignment for visual differentiation, and current permission mode. The `isActive` field tracks whether a teammate is currently processing or idle, enabling the leader to monitor workload distribution.

### Backend Selection and Detection

The system employs a priority-based backend detection flow implemented in `src/utils/swarm/backends/registry.ts:136-254`. The detection logic follows this hierarchy:

1. **Inside tmux** -- always uses the `TmuxBackend`, even if running in iTerm2
2. **In iTerm2 with `it2` CLI available** -- uses the `ITermBackend` for native pane splitting
3. **In iTerm2 without `it2`** -- falls back to tmux if available, or signals that `it2` setup is needed
4. **tmux available but not inside tmux** -- uses `TmuxBackend` in external session mode with a PID-scoped socket (`claude-swarm-${process.pid}`)
5. **Non-interactive sessions** -- forces in-process mode since terminal panes serve no purpose
6. **No terminal multiplexer available** -- falls back to in-process execution

The `isInProcessEnabled()` function in `src/utils/swarm/backends/registry.ts:351-389` resolves the `auto` mode setting by checking both the explicit `--teammate-mode` CLI flag and the runtime environment:

```typescript
// src/utils/swarm/backends/registry.ts:351-389
export function isInProcessEnabled(): boolean {
  if (getIsNonInteractiveSession()) {
    return true
  }
  const mode = getTeammateMode()
  let enabled: boolean
  if (mode === 'in-process') {
    enabled = true
  } else if (mode === 'tmux') {
    enabled = false
  } else {
    if (inProcessFallbackActive) {
      return true
    }
    const insideTmux = isInsideTmuxSync()
    const inITerm2 = isInITerm2()
    enabled = !insideTmux && !inITerm2
  }
  return enabled
}
```

Backend detection results are cached for the process lifetime, and a `markInProcessFallback()` mechanism ensures that if a pane backend fails during the first spawn attempt, all subsequent spawns automatically route to in-process mode without re-attempting pane creation.

---

## 2. Teammate Spawning

### In-Process Teammates

In-process teammates run within the same Node.js process as the leader. Context isolation is achieved through Node.js `AsyncLocalStorage`, allowing each teammate to maintain its own identity, team affiliation, and configuration without interfering with the leader or other teammates.

The spawning flow is initiated by `spawnInProcessTeammate()` in `src/utils/swarm/spawnInProcess.ts:104-216`:

```typescript
// src/utils/swarm/spawnInProcess.ts:104-112
export async function spawnInProcessTeammate(
  config: InProcessSpawnConfig,
  context: SpawnContext,
): Promise<InProcessSpawnOutput> {
  const { name, teamName, prompt, color, planModeRequired, model } = config
  const { setAppState } = context
  const agentId = formatAgentId(name, teamName)
  const taskId = generateTaskId('in_process_teammate')
```

Key design decisions for in-process teammates:

- **Independent AbortController**: Each teammate receives its own `AbortController`, decoupled from the leader's query lifecycle. This prevents leader query interruptions from cascading to teammates (`src/utils/swarm/spawnInProcess.ts:122`).
- **Cleanup registration**: A cleanup handler is registered through `registerCleanup()` for graceful shutdown scenarios (`src/utils/swarm/spawnInProcess.ts:183-188`).
- **Perfetto tracing integration**: When performance tracing is enabled, agents are registered in the Perfetto trace hierarchy for parent-child visualization (`src/utils/swarm/spawnInProcess.ts:149-151`).
- **Memory-conscious message capping**: The `TEAMMATE_MESSAGES_UI_CAP` constant (set to 50) limits the UI mirror of teammate messages in AppState. Source comments in `src/tasks/InProcessTeammateTask/types.ts:97-100` reference a real-world incident where 292 agents spawned in 2 minutes consumed 36.8GB of RAM due to message duplication.

After `spawnInProcessTeammate()` creates the task state and registers it in AppState, the `InProcessBackend.spawn()` method in `src/utils/swarm/backends/InProcessBackend.ts:72-143` calls `startInProcessTeammate()` to begin the agent execution loop in the background:

```typescript
// src/utils/swarm/backends/InProcessBackend.ts:107-129
startInProcessTeammate({
  identity: { agentId: result.agentId, agentName: config.name, ... },
  taskId: result.taskId,
  prompt: config.prompt,
  teammateContext: result.teammateContext,
  toolUseContext: { ...this.context, messages: [] },
  abortController: result.abortController,
  model: config.model,
  ...
})
```

The parent's conversation messages are explicitly stripped (`messages: []`) to prevent pinning the leader's full context in the teammate's memory for its entire lifetime.

### Pane-Based (Forked) Teammates

Pane-based teammates run as separate OS processes in terminal panes managed by either tmux or iTerm2. The `PaneBackendExecutor` class in `src/utils/swarm/backends/PaneBackendExecutor.ts` adapts the `PaneBackend` interface to the unified `TeammateExecutor` abstraction. It creates a pane via the backend, constructs a CLI command string with all necessary identity flags and environment variables, and sends the command to execute in the new pane.

The `buildInheritedCliFlags()` function in `src/utils/swarm/spawnUtils.ts:38-89` ensures teammates inherit critical settings from the leader:

```typescript
// src/utils/swarm/spawnUtils.ts:38-89
export function buildInheritedCliFlags(options?: {
  planModeRequired?: boolean
  permissionMode?: PermissionMode
}): string {
  const flags: string[] = []
  // Propagate permission mode (but NOT bypass when plan mode is required)
  if (planModeRequired) {
    // Safety: don't inherit bypass permissions with plan mode
  } else if (permissionMode === 'bypassPermissions' || ...) {
    flags.push('--dangerously-skip-permissions')
  }
  // Also propagates: --model, --settings, --plugin-dir, --teammate-mode, --chrome
```

Environment variable forwarding (`buildInheritedEnvVars()` in `src/utils/swarm/spawnUtils.ts:135-146`) explicitly propagates API provider selection (`CLAUDE_CODE_USE_BEDROCK`, `CLAUDE_CODE_USE_VERTEX`), proxy configuration, certificate paths, and the Claude Code Remote marker. The comments note that without explicit forwarding, tmux may start a new login shell that does not inherit the parent's environment.

### Fork Subagents

A separate but related mechanism is the "fork subagent" feature gated behind the `FORK_SUBAGENT` feature flag (`src/tools/AgentTool/forkSubagent.ts:32-39`). Unlike team-based teammates that receive independent prompts, fork subagents inherit the parent's full conversation context and rendered system prompt bytes for exact prompt-cache parity. Fork agents are blocked from recursive forking by detecting the `FORK_BOILERPLATE_TAG` in conversation history.

---

## 3. Permission Synchronization

### File-Based Permission Request Protocol

The swarm permission system is documented extensively in `src/utils/swarm/permissionSync.ts:1-19`, which describes the complete flow:

1. A worker agent encounters a tool use requiring permission
2. The worker writes a `SwarmPermissionRequest` to the team's `permissions/pending/` directory
3. The leader polls the pending directory and surfaces the request to the user
4. The user approves or denies via the leader's UI
5. The leader writes the resolution to `permissions/resolved/` and removes the pending file
6. The worker polls `permissions/resolved/` for the response

All file operations are protected by directory-level lock files (`src/utils/swarm/permissionSync.ts:227-229`):

```typescript
// src/utils/swarm/permissionSync.ts:227-229
const lockFilePath = join(lockDir, '.lock')
await writeFile(lockFilePath, '', 'utf-8')
let release = await lockfile.lock(lockFilePath)
```

### Mailbox-Based Permission Routing

A newer mailbox-based approach coexists with the file system. The `sendPermissionRequestViaMailbox()` function (`src/utils/swarm/permissionSync.ts:676-722`) routes permission requests through the teammate mailbox system, using structured `createPermissionRequestMessage` and `createPermissionResponseMessage` helpers. This approach routes messages to either in-process or file-based mailboxes depending on the recipient's backend type.

### Sandbox Permission Requests

A specialized variant handles sandbox runtime network access requests. The `sendSandboxPermissionRequestViaMailbox()` function (`src/utils/swarm/permissionSync.ts:805-869`) handles cases where a sandboxed teammate needs approval to access a specific network host. The leader can approve or deny individual host-level access through `sendSandboxPermissionResponseViaMailbox()`.

### Team-Wide Allowed Paths

The `TeamFile` structure supports `teamAllowedPaths` -- paths that all teammates can edit without individual approval. During teammate initialization (`src/utils/swarm/teammateInit.ts:45-79`), these paths are applied as session-scoped permission rules:

```typescript
// src/utils/swarm/teammateInit.ts:50-78
for (const allowedPath of teamFile.teamAllowedPaths) {
  const ruleContent = allowedPath.path.startsWith('/')
    ? `/${allowedPath.path}/**`
    : `${allowedPath.path}/**`
  setAppState(prev => ({
    ...prev,
    toolPermissionContext: applyPermissionUpdate(
      prev.toolPermissionContext,
      { type: 'addRules', rules: [{ toolName: allowedPath.toolName, ruleContent }],
        behavior: 'allow', destination: 'session' },
    ),
  }))
}
```

---

## 4. Leader-Worker Model

### Leader Permission Bridge

The leader permission bridge (`src/utils/swarm/leaderPermissionBridge.ts`) provides a module-level registry that allows in-process teammates to use the leader's native ToolUseConfirm dialog rather than the degraded mailbox-based permission prompt. Two functions are registered by the REPL:

```typescript
// src/utils/swarm/leaderPermissionBridge.ts:28-54
export function registerLeaderToolUseConfirmQueue(
  setter: SetToolUseConfirmQueueFn,
): void { registeredSetter = setter }

export function registerLeaderSetToolPermissionContext(
  setter: SetToolPermissionContextFn,
): void { registeredPermissionContextSetter = setter }
```

When an in-process teammate encounters an `ask` permission result, the `createInProcessCanUseTool()` function in `src/utils/swarm/inProcessRunner.ts:128-449` first checks for the bridge:

- **Bridge available**: The teammate's permission request is added to the leader's `ToolUseConfirmQueue` with a `workerBadge` displaying the teammate's name and color. The user sees the standard tool-specific permission UI (e.g., `BashPermissionRequest`, `FileEditToolDiff`) with an additional badge identifying which teammate is requesting.
- **Bridge unavailable**: Falls back to the mailbox system, sending a serialized permission request to the leader's inbox and polling the teammate's own mailbox for the response at 500ms intervals (`PERMISSION_POLL_INTERVAL_MS`).

A critical detail in the bridge path: when the user approves "always allow" rules, permission updates are written back to the leader's shared context with `preserveMode: true` to prevent the worker's transformed `acceptEdits` context from leaking into the coordinator's permission state (`src/utils/swarm/inProcessRunner.ts:276-279`).

### Bash Classifier Integration

For bash commands specifically, teammates attempt classifier auto-approval before escalating to the leader dialog. Unlike the main agent (which races the classifier against user interaction), teammates await the classifier result synchronously (`src/utils/swarm/inProcessRunner.ts:159-175`):

```typescript
// src/utils/swarm/inProcessRunner.ts:159-175
if (feature('BASH_CLASSIFIER') && tool.name === BASH_TOOL_NAME
    && result.pendingClassifierCheck) {
  const classifierDecision = await awaitClassifierAutoApproval(
    result.pendingClassifierCheck,
    abortController.signal,
    toolUseContext.options.isNonInteractiveSession,
  )
  if (classifierDecision) {
    return { behavior: 'allow', updatedInput: input, decisionReason: classifierDecision }
  }
}
```

### Idle Notification

When a teammate completes its work, the Stop hook registered during initialization (`src/utils/swarm/teammateInit.ts:98-128`) sends an idle notification to the leader's mailbox and marks the member as inactive in the team file:

```typescript
// src/utils/swarm/teammateInit.ts:103-128
addFunctionHook(setAppState, sessionId, 'Stop', '',
  async (messages, _signal) => {
    void setMemberActive(teamName, agentName, false)
    const notification = createIdleNotification(agentName, {
      idleReason: 'available',
      summary: getLastPeerDmSummary(messages),
    })
    await writeToMailbox(leadAgentName, {
      from: agentName,
      text: jsonStringify(notification),
      timestamp: new Date().toISOString(),
      color: getTeammateColor(),
    })
    return true // Don't block the Stop
  },
  'Failed to send idle notification to team leader',
  { timeout: 10000 },
)
```

---

## 5. Agent Communication

### Teammate Mailbox System

All inter-agent communication flows through a file-based mailbox system. Each agent has a named mailbox (keyed by agent name within a team), and messages are written via `writeToMailbox()` and read via `readMailbox()` from the `teammateMailbox` module. Messages carry structured payloads including sender identification, timestamps, color metadata, and typed content.

The mailbox supports several message types:
- **Direct messages**: Text communication between agents
- **Permission requests/responses**: Structured tool-use authorization flows
- **Sandbox permission requests/responses**: Network access authorization
- **Shutdown requests**: Graceful termination signals
- **Idle notifications**: Completion signals from worker to leader

### System Prompt Addendum

Teammates receive a mandatory system prompt addendum (`src/utils/swarm/teammatePromptAddendum.ts:8-18`) that explains communication constraints:

```typescript
// src/utils/swarm/teammatePromptAddendum.ts:8-18
export const TEAMMATE_SYSTEM_PROMPT_ADDENDUM = `
# Agent Teammate Communication
IMPORTANT: You are running as an agent in a team. To communicate with anyone:
- Use the SendMessage tool with \`to: "<name>"\` for specific teammates
- Use the SendMessage tool with \`to: "*"\` sparingly for team-wide broadcasts
Just writing a response in text is not visible to others on your team -
you MUST use the SendMessage tool.
`
```

This design enforces that all inter-agent communication is explicit and tool-mediated, preventing the assumption that text output is visible to other agents.

### Mode Synchronization

Teammates synchronize their permission mode back to the team file via `syncTeammateMode()` (`src/utils/swarm/teamHelpers.ts:397-407`), allowing the leader to observe and modify individual teammate permission modes. Batch mode updates use `setMultipleMemberModes()` (`src/utils/swarm/teamHelpers.ts:415-445`) to avoid race conditions when updating multiple teammates simultaneously.

---

## 6. Task Types

The swarm system integrates with Claude Code's task framework through three primary task types, as enumerated in `src/tasks/types.ts:12-19`:

### InProcessTeammateTask

Defined in `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx`, this task type manages in-process teammate lifecycle. The state type (`InProcessTeammateTaskState` in `src/tasks/InProcessTeammateTask/types.ts:22-76`) carries:

- **Identity**: `agentId` (format `name@team`), `agentName`, `teamName`, `color`, `planModeRequired`, `parentSessionId`
- **Execution state**: `prompt`, `model`, `abortController`, `currentWorkAbortController` (aborts current turn without killing the agent), `awaitingPlanApproval`
- **Communication**: `pendingUserMessages` queue for injecting user messages, `messages` array (capped at 50 for memory)
- **Lifecycle callbacks**: `onIdleCallbacks` for efficient leader waiting without polling, `shutdownRequested` flag

The task exposes three key operations:
- `kill()` -- immediate termination via `killInProcessTeammate()`
- `requestTeammateShutdown()` -- graceful shutdown by setting the `shutdownRequested` flag
- `injectUserMessageToTeammate()` -- direct user-to-teammate communication when viewing the teammate's transcript

### LocalAgentTask

Defined in `src/tasks/LocalAgentTask/LocalAgentTask.tsx`, this handles traditional background agents spawned via the Agent tool. These agents run in-process but without team awareness. They track progress through `AgentProgress` including `toolUseCount`, `tokenCount`, and `recentActivities`. The task supports git worktree isolation, output file tracking, and structured notification XML.

### RemoteAgentTask

Defined in `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx`, this manages agents executing on remote infrastructure (Claude Code Remote). The state includes `sessionId` for API calls, `todoList` for progress tracking, `log` of SDK messages, and remote-specific features like `ultraplanPhase` for multi-phase planning workflows. Remote tasks support long-running mode, review progress tracking, and poll-based completion checking.

### Task State Union

All task types are unified through the `TaskState` union type (`src/tasks/types.ts:12-19`), which also includes `LocalShellTask`, `LocalWorkflowTask`, `MonitorMcpTask`, and `DreamTask`. The `isBackgroundTask()` predicate determines visibility in the background tasks indicator based on `status` and `isBackgrounded` flags.

---

## 7. Reconnection and Recovery

### Session Reconnection

The reconnection module (`src/utils/swarm/reconnection.ts`) handles two scenarios:

**Fresh spawn**: The `computeInitialTeamContext()` function (`src/utils/swarm/reconnection.ts:23-66`) reads the dynamic team context set from CLI arguments in `main.tsx` and constructs the initial `teamContext` for AppState *before the first render*, eliminating race conditions:

```typescript
// src/utils/swarm/reconnection.ts:23-66
export function computeInitialTeamContext(): AppState['teamContext'] | undefined {
  const context = getDynamicTeamContext()
  if (!context?.teamName || !context?.agentName) return undefined
  const teamFile = readTeamFile(teamName)
  if (!teamFile) { logError(...); return undefined }
  return {
    teamName, teamFilePath, leadAgentId: teamFile.leadAgentId,
    selfAgentId: agentId, selfAgentName: agentName,
    isLeader: !agentId, teammates: {},
  }
}
```

**Resumed session**: `initializeTeammateContextFromSession()` (`src/utils/swarm/reconnection.ts:75-119`) reconstructs team context from transcript-stored `teamName` and `agentName`, looking up the `agentId` from the team file's member list.

### Crash Recovery and Cleanup

The system tracks teams created during a session via `registerTeamForSessionCleanup()` / `unregisterTeamForSessionCleanup()` (`src/utils/swarm/teamHelpers.ts:560-570`). On ungraceful exit (SIGINT/SIGTERM), the `cleanupSessionTeams()` function (`src/utils/swarm/teamHelpers.ts:576-590`) performs cleanup:

1. Kills orphaned teammate panes by reading the team file, filtering for pane-backed members, and calling `backend.killPane()` for each
2. Destroys git worktrees using `git worktree remove --force`, with a fallback to `rm -rf`
3. Removes team directories (`~/.claude/teams/{team-name}/`)
4. Removes task directories (`~/.claude/tasks/{taskListId}/`)

For in-process teammates, the `killInProcessTeammate()` function (`src/utils/swarm/spawnInProcess.ts:227-328`) handles cleanup:

```typescript
// src/utils/swarm/spawnInProcess.ts:237-299
setAppState((prev: AppState) => {
  const task = prev.tasks[taskId]
  if (!task || task.type !== 'in_process_teammate') return prev
  const teammateTask = task as InProcessTeammateTaskState
  // Abort the controller to stop execution
  teammateTask.abortController?.abort()
  teammateTask.unregisterCleanup?.()
  // Call pending idle callbacks to unblock waiters
  teammateTask.onIdleCallbacks?.forEach(cb => cb())
  // Remove from teamContext.teammates
  // ... state update with status: 'killed', cleared references
})
// Remove from team file (outside state updater)
if (teamName && agentId) removeMemberByAgentId(teamName, agentId)
```

The cleanup carefully nullifies all runtime references (`abortController`, `unregisterCleanup`, `currentWorkAbortController`) to prevent memory leaks, and caps remaining messages to the last entry only.

---

## 8. Layout and UI Coordination

### Pane Layout Management

The `teammateLayoutManager` module (`src/utils/swarm/teammateLayoutManager.ts`) coordinates the visual presentation of multi-agent output. It provides:

- **Color assignment**: `assignTeammateColor()` assigns colors from the `AGENT_COLORS` palette in round-robin order, with assignments persisted per session to maintain consistency
- **Pane creation**: `createTeammatePaneInSwarmView()` delegates to the detected backend, with the tmux backend implementing a leader-on-left (30%), teammates-on-right (70%) layout with serialized pane creation via a lock mechanism to prevent race conditions
- **Pane border status**: `enablePaneBorderStatus()` activates pane title display for teammate identification

The TmuxBackend (`src/utils/swarm/backends/TmuxBackend.ts`) implements a locking mechanism to serialize pane creation:

```typescript
// src/utils/swarm/backends/TmuxBackend.ts:43-53
function acquirePaneCreationLock(): Promise<() => void> {
  let release: () => void
  const newLock = new Promise<void>(resolve => { release = resolve })
  const previousLock = paneCreationLock
  paneCreationLock = newLock
  return previousLock.then(() => release!)
}
```

A 200ms delay (`PANE_SHELL_INIT_DELAY_MS`) follows each pane creation to allow shell initialization (rc files, prompt rendering) to complete before sending commands.

### Hidden Pane Management

The team file tracks `hiddenPaneIds` for panes that are temporarily removed from the visible layout. The `addHiddenPaneId()` and `removeHiddenPaneId()` functions (`src/utils/swarm/teamHelpers.ts:235-276`) manage this list, while the PaneBackend's `hidePane()` and `showPane()` methods handle the actual tmux/iTerm2 operations of breaking panes out to hidden windows and joining them back.

### In-Process UI Integration

In-process teammates integrate with the leader's UI through several mechanisms:

- **Task pills**: Each in-process teammate appears as a task pill in the background indicator, with random spinner verbs for visual variety
- **Zoomed transcript view**: Users can view a teammate's conversation by zooming into its task, which renders the capped `messages` array from `InProcessTeammateTaskState`
- **Direct message injection**: `injectUserMessageToTeammate()` (`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:68-84`) allows users viewing a teammate's transcript to type messages directly to the teammate
- **Worker badge on permission prompts**: When an in-process teammate surfaces a permission request via the leader bridge, the prompt displays a colored badge (`workerBadge: { name, color }`) identifying the requesting teammate

### Model Selection

Teammate model selection defaults to the latest Opus model through `getHardcodedTeammateModelFallback()` (`src/utils/swarm/teammateModel.ts:8-9`), which returns the provider-aware model ID for the current API provider (Anthropic, Bedrock, Vertex, or Foundry). This can be overridden per-teammate via the `model` field in spawn configuration, or globally via the `teammateDefaultModel` configuration setting.
