# State Management in Claude Code

## 1. Session State Architecture

Claude Code's state management is built around two distinct layers: a global bootstrap singleton and a reactive application store. The split exists for practical reasons -- some state must be available before React mounts, while other state needs to trigger UI re-renders.

### The Bootstrap Singleton

The first layer lives in `src/bootstrap/state.ts`. It is a plain module-scoped object with getter/setter functions, initialized once at startup. The file carries a stern warning at line 31:

```typescript
// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE
```

Despite this, the file contains well over 100 fields. The `State` type tracks everything that needs to exist before the UI is ready: the working directory, cost accumulators, model usage counters, session identity, telemetry providers, and a grab bag of session-scoped flags.

```typescript
// src/bootstrap/state.ts:46-67
type State = {
  originalCwd: string
  projectRoot: string
  totalCostUSD: number
  totalAPIDuration: number
  totalAPIDurationWithoutRetries: number
  totalToolDuration: number
  // ... dozens more fields
  cwd: string
  modelUsage: { [modelName: string]: ModelUsage }
  mainLoopModelOverride: ModelSetting | undefined
  sessionId: SessionId
  parentSessionId: SessionId | undefined
}
```

Initialization resolves symlinks and normalizes the cwd immediately (`src/bootstrap/state.ts:261-279`), because a mismatched cwd at startup causes session files to be written to one directory but looked up in another. That exact bug is documented in the code comments, and the fix is to call `realpathSync` before anything else touches the path.

### The Reactive Store

The second layer is a custom reactive store in `src/state/store.ts`. It is a minimal implementation -- 34 lines total:

```typescript
// src/state/store.ts:10-34
export function createStore<T>(
  initialState: T,
  onChange?: OnChange<T>,
): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,
    setState: (updater: (prev: T) => T) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },
    subscribe: (listener: Listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

No Redux. No Zustand. No middleware chains. Just a closure over a value, a set of listeners, and an `Object.is` check to prevent unnecessary notifications. The `onChange` callback is the key integration point -- it is where side effects happen when state transitions occur.

The store is exposed to React via a context provider in `src/state/AppState.tsx`, which wraps the tree in `AppStoreContext` and uses `useSyncExternalStore` for subscription. Components read the store through the context, and the provider prevents nesting:

```typescript
// src/state/AppState.tsx:44-47
const hasAppStateContext = useContext(HasAppStateContext);
if (hasAppStateContext) {
  throw new Error("AppStateProvider can not be nested within another AppStateProvider");
}
```

### The AppState Type

`AppState` (`src/state/AppStateStore.ts:89-452`) is a massive union type that wraps most fields in `DeepImmutable<>` but carves out exceptions for types containing functions (like `TaskState`) or mutable collections (like `Map`). It covers permissions, MCP server connections, plugin state, speculation state, team/agent context, bridge connections, and more. The `getDefaultAppState()` function (`src/state/AppStateStore.ts:456-569`) returns the initial values for all of these.

### The Side-Effect Bridge

When AppState changes, `onChangeAppState` in `src/state/onChangeAppState.ts` fires synchronously. This is the choke point for syncing internal state changes with external systems. The most critical case is permission mode synchronization:

```typescript
// src/state/onChangeAppState.ts:65-92
const prevMode = oldState.toolPermissionContext.mode
const newMode = newState.toolPermissionContext.mode
if (prevMode !== newMode) {
  const prevExternal = toExternalPermissionMode(prevMode)
  const newExternal = toExternalPermissionMode(newMode)
  if (prevExternal !== newExternal) {
    notifySessionMetadataChanged({
      permission_mode: newExternal,
      is_ultraplan_mode: isUltraplan,
    })
  }
  notifyPermissionModeChanged(newMode)
}
```

The comment above this block explains that prior to this centralized handler, permission mode changes were relayed by only 2 of 8+ mutation paths, leaving the web UI perpetually out of sync. Other side effects in this handler include persisting model overrides to settings, saving verbose/expanded-view preferences to the global config, and clearing auth caches when settings change.

## 2. Conversation History

Conversation history is the most complex persistence subsystem. Messages are stored as JSONL files on disk, linked by a UUID-based parent chain that enables branching and sidechains.

### The Transcript File

Each session writes to a JSONL file at `~/.claude/projects/<sanitized-cwd>/<sessionId>.jsonl`. The path is computed by `getTranscriptPath()`:

```typescript
// src/utils/sessionStorage.ts:202-205
export function getTranscriptPath(): string {
  const projectDir = getSessionProjectDir() ?? getProjectDir(getOriginalCwd())
  return join(projectDir, `${getSessionId()}.jsonl`)
}
```

The `getProjectDir` function sanitizes the working directory into a path-safe key:

```typescript
// src/utils/sessionStorage.ts:436-438
export const getProjectDir = memoize((projectDir: string): string => {
  return join(getProjectsDir(), sanitizePath(projectDir))
})
```

### Message Chain Architecture

Messages are not stored as a flat array. Each message carries a `uuid` and a `parentUuid`, forming a linked list. This design enables branching: when you rewind a conversation and take a different path, the old branch stays in the file as a "sidechain."

The `insertMessageChain` method (`src/utils/sessionStorage.ts:993-1069`) writes messages with chain metadata:

```typescript
// src/utils/sessionStorage.ts:1039-1064
const transcriptMessage: TranscriptMessage = {
  parentUuid: isCompactBoundary ? null : effectiveParentUuid,
  logicalParentUuid: isCompactBoundary ? parentUuid : undefined,
  isSidechain,
  teamName: teamInfo?.teamName,
  agentName: teamInfo?.agentName,
  promptId: message.type === 'user' ? (getPromptId() ?? undefined) : undefined,
  agentId,
  ...message,
  userType: getUserType(),
  entrypoint: getEntrypoint(),
  cwd: getCwd(),
  sessionId,
  version: VERSION,
  gitBranch,
  slug,
}
```

Every message is stamped with the session ID, working directory, git branch, CLI version, and entrypoint. The `sessionId` field is explicitly re-stamped after the spread (the comment at lines 1049-1056 explains that on `--fork-session`, messages carry the source session's ID and must be overwritten to match the new session's content-replacement entries).

### Loading a Conversation

Loading a transcript means reading the JSONL, building a UUID map, finding the most recent non-sidechain leaf, and walking the parent chain backward to reconstruct the conversation. This happens in `loadTranscriptFile` and `buildConversationChain` (exported from `src/utils/sessionStorage.ts`).

Progress messages were previously included in the chain but caused chain forks that orphaned real messages (referenced as bugs #14373, #23537). They are now explicitly excluded:

```typescript
// src/utils/sessionStorage.ts:139-146
export function isTranscriptMessage(entry: Entry): entry is TranscriptMessage {
  return (
    entry.type === 'user' ||
    entry.type === 'assistant' ||
    entry.type === 'attachment' ||
    entry.type === 'system'
  )
}
```

### The Session File Materialization Pattern

An interesting design choice: the session JSONL file is not created until the first real user or assistant message appears. Metadata and hook outputs are buffered in `pendingEntries` (the `Project` class at `src/utils/sessionStorage.ts:532-552`):

```typescript
// src/utils/sessionStorage.ts:532-552
class Project {
  currentSessionTag: string | undefined
  currentSessionTitle: string | undefined
  // ...
  sessionFile: string | null = null
  private pendingEntries: Entry[] = []
```

This prevents ghost session files from piling up every time Claude Code starts and the user immediately quits without typing anything.

## 3. The Memory System

Claude Code has a layered memory system. There are two distinct mechanisms: the file-based "CLAUDE.md" instructions (which the user manages) and the automatic "session memory" system (which the tool manages itself).

### CLAUDE.md Files

The `claudemd.ts` module (`src/utils/claudemd.ts`) loads instruction files in a specific priority order, documented at the top of the file:

1. **Managed memory** (`/etc/claude-code/CLAUDE.md`) -- Global instructions for all users on a machine
2. **User memory** (`~/.claude/CLAUDE.md`) -- Private global instructions for all projects
3. **Project memory** (`CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md`) -- Committed to the repo
4. **Local memory** (`CLAUDE.local.md`) -- Private project-specific instructions

Files are loaded in reverse priority order: later files get more model attention. The system traverses from the current directory upward to the git root (or filesystem root), checking each directory for memory files.

Memory files support an `@include` directive for pulling in other files. The syntax supports `@path`, `@./relative/path`, `@~/home/path`, and `@/absolute/path`. Circular references are tracked and prevented. Only text file extensions are allowed to prevent binary file injection.

There is a hard limit:

```typescript
// src/utils/claudemd.ts:92-93
export const MAX_MEMORY_CHARACTER_COUNT = 40000
```

### Auto-Memory (memdir)

The auto-memory system lives in `src/memdir/`. It provides persistent memory across sessions via a `MEMORY.md` file stored in a project-specific directory under `~/.claude/projects/<sanitized-cwd>/memory/`.

The path resolution (`src/memdir/paths.ts:223-235`) uses the canonical git root so that all worktrees of the same repo share one memory directory:

```typescript
// src/memdir/paths.ts:223-235
export const getAutoMemPath = memoize(
  (): string => {
    const override = getAutoMemPathOverride() ?? getAutoMemPathSetting()
    if (override) {
      return override
    }
    const projectsDir = join(getMemoryBaseDir(), 'projects')
    return (
      join(projectsDir, sanitizePath(getAutoMemBase()), AUTO_MEM_DIRNAME) + sep
    ).normalize('NFC')
  },
  () => getProjectRoot(),
)
```

The entrypoint file (`MEMORY.md`) has hard limits to prevent context window bloat:

```typescript
// src/memdir/memdir.ts:35-37
export const MAX_ENTRYPOINT_LINES = 200
export const MAX_ENTRYPOINT_BYTES = 25_000
```

### Session Memory

The session memory system (`src/services/SessionMemory/sessionMemory.ts`) is a background agent that automatically maintains notes about the current conversation. It is gated behind a GrowthBook feature flag (`tengu_session_memory`) and only runs when auto-compact is enabled.

The mechanism is elegant: a post-sampling hook fires after each model response. It checks two thresholds before triggering:

```typescript
// src/services/SessionMemory/sessionMemory.ts:134-181
export function shouldExtractMemory(messages: Message[]): boolean {
  const currentTokenCount = tokenCountWithEstimation(messages)
  if (!isSessionMemoryInitialized()) {
    if (!hasMetInitializationThreshold(currentTokenCount)) {
      return false
    }
    markSessionMemoryInitialized()
  }
  // Both token AND tool-call thresholds must be met
  const hasMetTokenThreshold = hasMetUpdateThreshold(currentTokenCount)
  const toolCallsSinceLastUpdate = countToolCallsSince(messages, lastMemoryMessageUuid)
  const hasMetToolCallThreshold =
    toolCallsSinceLastUpdate >= getToolCallsBetweenUpdates()
  // ...
}
```

When triggered, it spawns a forked subagent with access to only the `FileEdit` tool, restricted to editing the session memory file. This isolation prevents the memory agent from accidentally modifying project files.

## 4. File-Based Persistence

### The ~/.claude/ Directory

The global config directory (defaulting to `~/.claude/`, overridable via `CLAUDE_CONFIG_DIR`) stores all persistent state. The config system uses a read-through cache with a filesystem watcher for cross-process consistency.

The `GlobalConfig` type (`src/utils/config.ts:183+`) contains over 100 fields spanning:

- Authentication state (`primaryApiKey`, `oauthAccount`)
- UI preferences (`theme`, `editorMode`, `verbose`, `autoCompactEnabled`)
- Per-project configs in a nested `projects` record (allowed tools, MCP servers, trust settings)
- Onboarding flags, feature hints, and usage counters
- Cost tracking fields per project (`lastCost`, `lastModelUsage`, `lastSessionId`)
- IDE integration state, terminal setup flags, notification preferences

The config is stored as a single JSON file, read with a lock (`saveConfigWithLock`), and uses a background freshness watcher so that changes from other Claude Code instances are eventually visible.

### The Projects Directory

Under `~/.claude/projects/`, each project gets a sanitized directory. Inside that directory:

- `<sessionId>.jsonl` -- Conversation transcripts
- `memory/MEMORY.md` -- Auto-managed memory
- `memory/logs/YYYY/MM/YYYY-MM-DD.md` -- Daily log files (for assistant mode)
- Project-level config (allowed tools, trust status)

The `sanitizePath` function converts absolute paths to safe directory names. For example, `/Users/alan/my-project` becomes something like `Users-alan-my-project`.

### The Input History

The `GlobalConfig` has a `HistoryEntry` type for command history:

```typescript
// src/utils/config.ts:64-73
export interface SerializedStructuredHistoryEntry {
  display: string
  pastedContents?: Record<number, PastedContent>
  pastedText?: string
}
export interface HistoryEntry {
  display: string
  pastedContents: Record<number, PastedContent>
}
```

History entries preserve pasted content (including images with their media types and dimensions) alongside the display string, enabling rich history recall.

## 5. Project-Level State

### .claude/ Directory

Within a project, the `.claude/` directory can contain:

- `CLAUDE.md` -- Project instructions (checked into version control)
- `settings.json` -- Project-level settings (MCP servers, environment variables)
- `settings.local.json` -- Local project settings (gitignored)
- `rules/*.md` -- Additional instruction files
- `scheduled_tasks.json` -- Cron-scheduled tasks

The `CLAUDE.local.md` file at the project root provides private instructions that are not committed.

### Trust Model

Project configs track trust separately. The `hasTrustDialogAccepted` field in `ProjectConfig` (`src/utils/config.ts:111`) determines whether the user has accepted the trust dialog for a given project directory. There is also a session-only trust mechanism (`sessionTrustAccepted` in the bootstrap state) for home-directory usage where persisting trust to disk is undesirable.

## 6. Session Recovery

Session recovery is one of the most carefully engineered subsystems. The `sessionRestore.ts` module handles the full lifecycle of resuming an interrupted or past conversation.

### Loading the Conversation

`loadConversationForResume` (`src/utils/conversationRecovery.ts:456-597`) is the main entry point. It handles three source types:

- `undefined` -- Load the most recent session (for `--continue`), skipping any live background/daemon sessions
- `string` -- Load a specific session by ID (for `--resume <id>`)
- `LogOption` -- An already-loaded conversation object

### Detecting Interruptions

The deserialization pipeline detects mid-turn interruptions (`src/utils/conversationRecovery.ts:272-333`). It classifies the session's end state:

- **interrupted_prompt** -- The user typed something but the model never responded
- **interrupted_turn** -- The model was mid-tool-execution when the session died
- **none** -- The session ended cleanly

For interrupted turns, a synthetic "Continue from where you left off." message is injected. For interrupted prompts, the original user message is preserved for re-submission.

### The Full Restore Pipeline

`processResumedConversation` (`src/utils/sessionRestore.ts:409-551`) orchestrates the full restore:

1. Matches coordinator/normal mode to the resumed session
2. Switches to the resumed session ID (unless forking)
3. Restores cost state from the project config
4. Restores session metadata (agent name/color, custom title, tags)
5. Restores worktree state (cd back into the worktree if the session was inside one)
6. Restores context-collapse commit log
7. Restores agent settings and model overrides
8. Refreshes agent definitions if the mode switched
9. Computes the initial `AppState` with all restored values

The worktree restore (`src/utils/sessionRestore.ts:332-366`) is particularly careful: it uses `process.chdir` as a TOCTOU-safe existence check, and if the directory no longer exists, it overrides the stale cache to prevent re-persisting a dead path.

### Skill Preservation

Skills loaded during a session are tracked in `STATE.invokedSkills` (bootstrap state). On resume, `restoreSkillStateFromMessages` (`src/utils/conversationRecovery.ts:382-403`) walks the transcript looking for `invoked_skills` attachment messages and re-registers them. Without this, a compaction after resume would lose the skills.

## 7. Cross-Session Data Flow

What actually persists between conversations?

**Always persists:**
- `GlobalConfig` -- All user preferences, allowed tools, project trust, auth tokens
- Conversation transcripts -- JSONL files survive indefinitely
- Auto-memory MEMORY.md -- Maintained across sessions by the memory agent
- CLAUDE.md files -- User-managed, project-level instructions

**Persists per-project:**
- Last session cost/duration/token usage (in `ProjectConfig`)
- Allowed tools and MCP server approvals
- Trust dialog acceptance

**Does NOT persist:**
- The reactive `AppState` -- Rebuilt from defaults + restore data each launch
- Bootstrap state accumulators (cost counters, line counts) -- Restored from project config only on `--resume`/`--continue`
- MCP connections, plugin state, speculation state -- All session-scoped
- File history snapshots -- Restored from transcript on resume only
- Permission mode -- Saved to transcript metadata, restored on resume

The boundary is clean: disk state is the source of truth, in-memory state is derived. On resume, the system reads the transcript, extracts metadata, and rebuilds the in-memory state to match.

## 8. Cost & Usage Tracking

Cost tracking is split between the bootstrap singleton (live accumulators) and the project config (persisted snapshots).

### Live Tracking

The bootstrap state accumulates costs in real time:

```typescript
// src/bootstrap/state.ts:280-283
totalCostUSD: 0,
totalAPIDuration: 0,
totalAPIDurationWithoutRetries: 0,
totalToolDuration: 0,
```

Model usage is tracked per-model with full token breakdowns:

```typescript
// src/bootstrap/state.ts:67
modelUsage: { [modelName: string]: ModelUsage }
```

### Persistence

On session exit (or periodically), `saveCurrentSessionCosts` (`src/cost-tracker.ts:143-175`) writes the accumulated values to the project config:

```typescript
// src/cost-tracker.ts:143-175
export function saveCurrentSessionCosts(fpsMetrics?: FpsMetrics): void {
  saveCurrentProjectConfig(current => ({
    ...current,
    lastCost: getTotalCostUSD(),
    lastAPIDuration: getTotalAPIDuration(),
    lastModelUsage: Object.fromEntries(
      Object.entries(getModelUsage()).map(([model, usage]) => [
        model,
        {
          inputTokens: usage.inputTokens,
          outputTokens: usage.outputTokens,
          cacheReadInputTokens: usage.cacheReadInputTokens,
          cacheCreationInputTokens: usage.cacheCreationInputTokens,
          webSearchRequests: usage.webSearchRequests,
          costUSD: usage.costUSD,
        },
      ]),
    ),
    lastSessionId: getSessionId(),
  }))
}
```

### Restore on Resume

When resuming a session, `restoreCostStateForSession` (`src/cost-tracker.ts:130-137`) reads the saved costs back, but only if the session ID matches. This prevents applying one session's costs to a different session:

```typescript
// src/cost-tracker.ts:87-95
export function getStoredSessionCosts(sessionId: string): StoredCostState | undefined {
  const projectConfig = getCurrentProjectConfig()
  if (projectConfig.lastSessionId !== sessionId) {
    return undefined
  }
  // ...
}
```

### Statistics

The `stats.ts` module (`src/utils/stats.ts`) provides aggregate statistics across all sessions: total sessions, messages, daily activity, streaks, and per-model token usage. It scans transcript JSONL files under the projects directory and uses a cache (`statsCache.ts`) to avoid re-parsing unchanged files.

## 9. My Take

This is a system that grew organically under real pressure, and it shows -- in both good and bad ways.

**What works well:**

The custom store is a masterclass in restraint. Thirty-four lines of code, no dependencies, and it does exactly what is needed. The temptation to reach for Redux or Zustand in a project this size must have been significant, and resisting it was the right call. The `Object.is` check for preventing unnecessary renders, the immutable update pattern, and the `onChange` callback for side effects -- it is textbook reactive state management without the ceremony.

The UUID-based message chain is clever. By linking messages via parent UUIDs rather than array indices, the system naturally supports branching, sidechains, and forking without complex merge logic. The JSONL format means conversations are append-only on the write path (no read-modify-write cycles), which is both fast and crash-safe.

The session recovery pipeline is impressively thorough. The fact that it can detect whether a session was interrupted mid-prompt, mid-tool-execution, or ended cleanly, and then synthesize the appropriate continuation message -- that is the kind of detail that separates a tool people tolerate from one they trust.

**What is less great:**

The bootstrap singleton is a code smell. The file says "DO NOT ADD MORE STATE HERE" and then defines 100+ fields. The split between bootstrap state and AppState feels arbitrary in places. Cost tracking lives in bootstrap, but model selection lives in both. Session identity is in bootstrap, but session hooks state is in AppState. The boundary is "what existed before React" rather than "what logically belongs together."

The `Project` class in `sessionStorage.ts` is doing too much. It manages the JSONL file handle, buffers pending writes, caches session metadata, handles agent transcripts, and coordinates with the ingress service. It is the kind of god object that accumulates responsibilities because it is the most convenient place to put them.

The config system stores everything in a single JSON file with a growing collection of ad-hoc fields. `memoryUsageCount`, `voiceNoticeSeenCount`, `effortCalloutV2Dismissed` -- these are all top-level fields on a single object. There is no schema versioning or migration system; backward compatibility is handled by optional fields and default values.

**What can we learn:**

The most important lesson is the value of a clear persistence boundary. In-memory state is ephemeral and derived; disk state is the source of truth. When something needs to survive a restart, it gets written to disk. When it does not, it stays in memory. The system does not try to be clever about caching or lazy loading -- it reads what it needs at startup and writes what it needs at shutdown. That simplicity makes the system debuggable: if a session resume fails, you can read the JSONL file and understand exactly what happened.

The second lesson is that file-based persistence at this scale (hundreds of thousands of sessions across millions of users) works surprisingly well. JSONL is append-only, human-readable, and naturally handles concurrent writes (each line is atomic on most filesystems). No database, no migration scripts, no schema evolution headaches. The tradeoff is that queries are expensive (scanning all files for statistics), but for the access patterns Claude Code has (write often, read rarely, scan occasionally), it is the right choice.

The third lesson: sometimes the boring solution is the right one. A 34-line store with no middleware. JSONL files instead of SQLite. A singleton module instead of dependency injection. These choices make the system easy to understand, easy to debug, and hard to break. That matters more than architectural purity when you are building a tool that developers depend on every day.
