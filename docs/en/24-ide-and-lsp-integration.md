# IDE and LSP Integration System

## Overview

Claude Code implements a dual integration strategy for connecting with development environments. The first axis is the **Language Server Protocol (LSP) subsystem**, where Claude Code acts as an LSP client that spawns and manages language server processes to obtain type information, diagnostics, and navigation capabilities. The second axis is the **IDE extension bridge**, where Claude Code discovers running IDE instances (VS Code, Cursor, JetBrains family, etc.) and communicates with their extensions over MCP via SSE or WebSocket transports. These two subsystems operate independently but complement each other: the LSP layer provides language-aware intelligence directly, while the IDE bridge enables file synchronization, diff rendering, and editor-side actions.

---

## 1. LSP Client Architecture

### Closure-Based Design

The LSP client is implemented as a factory function returning a typed interface rather than a class. This pattern, found throughout Claude Code's codebase, uses closures for state encapsulation:

```typescript
// src/services/lsp/LSPClient.ts:51-54
export function createLSPClient(
  serverName: string,
  onCrash?: (error: Error) => void,
): LSPClient {
  let process: ChildProcess | undefined
  let connection: MessageConnection | undefined
  let capabilities: ServerCapabilities | undefined
  let isInitialized = false
```

The client wraps the `vscode-jsonrpc` library to establish JSON-RPC message connections over stdio pipes. The `createMessageConnection` call at line 183 creates a bidirectional channel using `StreamMessageReader` and `StreamMessageWriter` connected to the spawned server process's stdout and stdin respectively.

### Lazy Handler Registration

A notable architectural decision is the pending handler queue. Notification and request handlers can be registered before the connection exists:

```typescript
// src/services/lsp/LSPClient.ts:64-71
const pendingHandlers: Array<{
  method: string
  handler: (params: unknown) => void
}> = []
const pendingRequestHandlers: Array<{
  method: string
  handler: (params: unknown) => unknown | Promise<unknown>
}> = []
```

When `onNotification()` or `onRequest()` is called before the connection is ready, handlers are queued and applied later during `start()` (lines 228-244). This supports lazy initialization patterns where diagnostic handlers must be registered before the server is actually started.

### Crash Detection and State Propagation

The `onCrash` callback (line 52) is central to the resilience model. When the server process exits with a non-zero code during operation (not during intentional shutdown), the callback fires, allowing the owning `LSPServerInstance` to transition to an error state. The `isStopping` boolean flag (line 62) prevents spurious error logging during intentional shutdown sequences.

### Protocol Compliance

The `stop()` method (lines 373-445) implements the LSP shutdown protocol faithfully: it sends a `shutdown` request followed by an `exit` notification, then disposes the connection and kills the process. Cleanup always occurs in the `finally` block regardless of whether shutdown succeeded, preventing resource leaks.

---

## 2. Server Lifecycle Management

### Three-Layer Architecture

Server lifecycle management spans three layers:

1. **`LSPClient`** (`LSPClient.ts`) -- raw JSON-RPC transport over stdio
2. **`LSPServerInstance`** (`LSPServerInstance.ts`) -- state machine, health monitoring, retry logic for a single server
3. **`LSPServerManager`** (`LSPServerManager.ts`) -- multi-server orchestration, file-extension routing, document synchronization

A fourth layer exists in **`manager.ts`** which manages the global singleton lifecycle.

### State Machine

Each `LSPServerInstance` implements a state machine with the following transitions, documented in its source:

```
stopped -> starting -> running
running -> stopping -> stopped
any -> error (on failure)
error -> starting (on retry)
```

The states are tracked via a closure variable (`let state: LspServerState = 'stopped'`) and exposed as a read-only property via a getter.

### Lazy Loading of vscode-jsonrpc

The `LSPServerInstance` uses a deliberate `require()` call instead of a static import for the LSP client:

```typescript
// src/services/lsp/LSPServerInstance.ts:109-112
const { createLSPClient } = require('./LSPClient.js') as {
  createLSPClient: typeof createLSPClientType
}
```

The inline comment explains the rationale: `vscode-jsonrpc` weighs approximately 129KB and should only be loaded when an LSP server is actually instantiated, not during module initialization. This keeps Claude Code's startup time fast when no LSP servers are configured.

### Server Initialization Parameters

When starting, the instance sends comprehensive `InitializeParams` (lines 167-237 of `LSPServerInstance.ts`) including:

- **Workspace folders** using the modern LSP 3.16+ format (`workspaceFolders`) and deprecated fields (`rootPath`, `rootUri`) for backward compatibility
- **Client capabilities** declaring support for `didSave`, `publishDiagnostics` (with related information, tag support, and code description), hover (markdown + plaintext), definition with link support, references, hierarchical document symbols, and call hierarchy
- **Position encoding** set to UTF-16 for compatibility
- **Initialization options** passed through from plugin configuration, required by servers like vue-language-server

### Retry Logic for Transient Errors

Request handling includes exponential backoff retry for LSP error code `-32801` ("content modified"):

```typescript
// src/services/lsp/LSPServerInstance.ts:17-28
const LSP_ERROR_CONTENT_MODIFIED = -32801
const MAX_RETRIES_FOR_TRANSIENT_ERRORS = 3
const RETRY_BASE_DELAY_MS = 500
```

This error commonly occurs with servers like rust-analyzer during initial project indexing. The retry delays are 500ms, 1000ms, and 2000ms (exponential backoff with base 500ms). Duck typing is used for error code detection rather than `instanceof` checks due to potential multiple versions of `vscode-jsonrpc` in the dependency tree.

### Crash Recovery and Restart Caps

The crash callback from `LSPClient` increments a `crashRecoveryCount`, and `start()` checks this against `maxRestarts` (default: 3) to prevent unbounded process spawning:

```typescript
// src/services/lsp/LSPServerInstance.ts:142-150
const maxRestarts = config.maxRestarts ?? 3
if (state === 'error' && crashRecoveryCount > maxRestarts) {
  const error = new Error(
    `LSP server '${name}' exceeded max crash recovery attempts (${maxRestarts})`,
  )
```

This cap protects against a persistently crashing server consuming system resources on every incoming request.

### Global Singleton Management

The `manager.ts` module manages the global LSP manager singleton with a generation counter pattern to handle concurrent initialization:

```typescript
// src/services/lsp/manager.ts:33-35
let initializationGeneration = 0
```

Each call to `initializeLspServerManager()` increments the generation. When the async initialization promise resolves, it checks whether `currentGeneration === initializationGeneration` before updating state. This prevents stale initialization promises from a previous attempt from overwriting the current state -- a pattern necessary because `reinitializeLspServerManager()` can be called while a prior initialization is still in flight.

The manager is skipped entirely in bare mode (`isBareMode()`), since scripted `-p` calls have no use for editor integration features.

---

## 3. Diagnostic Registry

### Asynchronous Delivery Pattern

The `LSPDiagnosticRegistry` (`LSPDiagnosticRegistry.ts`) follows the same pattern as `AsyncHookRegistry` for consistent async attachment delivery. The pipeline is:

1. LSP server sends `textDocument/publishDiagnostics` notification
2. `registerPendingLSPDiagnostic()` stores the diagnostic with a UUID key and timestamp
3. `checkForLSPDiagnostics()` retrieves, deduplicates, and volume-limits pending diagnostics
4. `getLSPDiagnosticAttachments()` in `attachments.ts` converts them to `Attachment[]`
5. The attachment system delivers them into the next conversation turn

### Cross-Turn Deduplication

The registry implements two levels of deduplication. **Within-batch deduplication** prevents the same diagnostic from appearing multiple times in a single delivery. **Cross-turn deduplication** uses an LRU cache to track previously delivered diagnostics:

```typescript
// src/services/lsp/LSPDiagnosticRegistry.ts:54-56
const deliveredDiagnostics = new LRUCache<string, Set<string>>({
  max: MAX_DELIVERED_FILES,  // 500
})
```

Each diagnostic is keyed by a JSON serialization of its message, severity, range, source, and code. The LRU cache caps at 500 files to prevent unbounded memory growth in long sessions. When a file is edited, `clearDeliveredDiagnosticsForFile()` resets that file's tracking so fresh diagnostics for the edited file will be delivered even if they match previously seen ones.

### Volume Limiting

To prevent diagnostic floods from overwhelming the conversation context, the registry enforces hard caps:

```typescript
// src/services/lsp/LSPDiagnosticRegistry.ts:42-43
const MAX_DIAGNOSTICS_PER_FILE = 10
const MAX_TOTAL_DIAGNOSTICS = 30
```

Before limiting, diagnostics are sorted by severity (Error > Warning > Info > Hint) so the most critical issues survive truncation. The sorting uses a numeric mapping where Error=1 and Hint=4, ensuring errors are always preserved.

### Passive Feedback Registration

The `passiveFeedback.ts` module wires up the diagnostic pipeline by registering `textDocument/publishDiagnostics` handlers on all configured servers:

```typescript
// src/services/lsp/passiveFeedback.ts:161-163
serverInstance.onNotification(
  'textDocument/publishDiagnostics',
  (params: unknown) => { ... }
)
```

The handler validates params structure, converts LSP diagnostic format to Claude's internal `DiagnosticFile[]` format (mapping LSP severity numbers 1-4 to string labels), and registers them via `registerPendingLSPDiagnostic()`. Consecutive failure tracking warns after 3+ failures per server, providing visibility into problematic language servers without disrupting other servers.

---

## 4. IDE Detection

### Supported IDE Catalog

Claude Code maintains an extensive catalog of supported IDEs in `src/utils/ide.ts`, organized into two families:

- **VS Code family** (`ideKind: 'vscode'`): VS Code, Cursor, Windsurf
- **JetBrains family** (`ideKind: 'jetbrains'`): IntelliJ IDEA, PyCharm, WebStorm, PhpStorm, RubyMine, CLion, GoLand, Rider, DataGrip, AppCode, DataSpell, Aqua, Gateway, Fleet, Android Studio

Each entry specifies platform-specific process detection keywords:

```typescript
// src/utils/ide.ts:131-137
cursor: {
  ideKind: 'vscode',
  displayName: 'Cursor',
  processKeywordsMac: ['Cursor Helper', 'Cursor.app'],
  processKeywordsWindows: ['cursor.exe'],
  processKeywordsLinux: ['cursor'],
},
```

Some JetBrains entries deliberately leave certain platform keywords empty -- for example, Aqua, Gateway, and Fleet have empty `processKeywordsMac` arrays because their process names are too generic for reliable auto-detection.

### Lockfile-Based Discovery

The primary detection mechanism uses lockfiles written by IDE extensions to `~/.claude/ide/`. Each lockfile is named `{port}.lock` and contains JSON with workspace folders, process ID, IDE name, transport type (SSE vs WebSocket), authentication token, and platform information:

```typescript
// src/utils/ide.ts:73-80
type LockfileJsonContent = {
  workspaceFolders?: string[]
  pid?: number
  ideName?: string
  transport?: 'ws' | 'sse'
  runningInWindows?: boolean
  authToken?: string
}
```

The `detectIDEs()` function reads all lockfiles in parallel (previously serial I/O was showing up as ~500ms in CPU profiles), sorts by modification time, and validates each against the current working directory. Workspace folder matching uses NFC-normalized Unicode comparison to handle macOS's NFD path representation.

### Process Ancestry Verification

When running inside a supported IDE's built-in terminal, Claude Code performs PID ancestry verification to disambiguate multiple IDE windows with overlapping workspace folders:

```typescript
// src/utils/ide.ts:690
const needsAncestryCheck = getPlatform() !== 'wsl' && isSupportedTerminal()
```

The ancestor PID lookup is lazily computed and cached within a single `detectIDEs()` call. It walks up to 10 levels of the process tree. This check runs **after** workspace folder matching to avoid unnecessary process tree traversal for non-matching lockfiles -- a key optimization since the previous implementation shelled out once per lockfile and dominated CPU profiles during the polling loop.

### Terminal Detection

The `isSupportedTerminal()` function (line 279) returns true if Claude Code is running inside a VS Code, Cursor, Windsurf, or JetBrains integrated terminal. This is detected from the `TERM_PROGRAM` environment variable (for VS Code family) or from `envDynamic.terminal` (for JetBrains). Terminal multiplexers like tmux or screen overwrite `TERM_PROGRAM`, but the `CLAUDE_CODE_SSE_PORT` environment variable is inherited, allowing auto-connect to still work.

### Stale Lockfile Cleanup

Before searching for IDEs, `findAvailableIDE()` calls `cleanupStaleIdeLockfiles()` which removes lockfiles for processes that are no longer running and ports that are not responding. The port liveness check uses a raw TCP socket connection with a 500ms timeout.

### WSL Support

When running under WSL, the detection system handles additional complexity: it resolves Windows `USERPROFILE` paths (falling back to `powershell.exe` invocation if the environment variable is unavailable), converts Windows paths to WSL paths using `WindowsToWSLConverter`, validates WSL distro matches, and detects the correct host IP for cross-environment communication by checking the default gateway route via `ip route show`.

---

## 5. Extension Communication

### MCP-Based Transport

IDE extensions communicate with Claude Code through the Model Context Protocol (MCP). The integration hook in `useIDEIntegration.tsx` registers the IDE as a dynamic MCP server configuration:

```typescript
// src/hooks/useIDEIntegration.tsx (from source map):
setDynamicMcpConfig(prev => ({
  ...prev,
  ide: {
    type: ide.url.startsWith('ws:') ? 'ws-ide' : 'sse-ide',
    url: ide.url,
    ideName: ide.name,
    authToken: ide.authToken,
    ideRunningInWindows: ide.ideRunningInWindows,
    scope: 'dynamic' as const,
  },
}))
```

Two transport types are supported: **SSE** (Server-Sent Events over HTTP) for the traditional VS Code extension protocol, and **WebSocket** for newer or JetBrains-based transports. The transport type is determined by the lockfile's `transport` field.

### Auto-Connect Logic

Auto-connect is governed by a multi-condition evaluation. Connection happens if any of the following are true:

- The `autoConnectIde` global config setting is enabled
- The `--ide` CLI flag was passed
- The process is running inside a supported IDE terminal
- The `CLAUDE_CODE_SSE_PORT` environment variable is set (handles tmux/screen scenarios)
- An explicit IDE extension installation was requested
- The `CLAUDE_CODE_AUTO_CONNECT_IDE` environment variable is truthy

Auto-connect is suppressed if `CLAUDE_CODE_AUTO_CONNECT_IDE` is explicitly set to a falsy value.

### IDE RPC Calls

Once connected, Claude Code can invoke IDE-side tools through `callIdeRpc()`:

```typescript
// src/services/mcp/client.ts:2116-2128
export async function callIdeRpc(
  toolName: string,
  args: Record<string, unknown>,
  client: ConnectedMCPServer,
): Promise<string | ContentBlockParam[] | undefined> {
  const result = await callMCPTool({
    client,
    tool: toolName,
    args,
    signal: createAbortController().signal,
  })
  return result.content
}
```

This function wraps the standard MCP `callTool` mechanism, providing a typed interface for invoking IDE-specific operations such as diff viewing, file opening, and editor navigation. The function checks that the IDE MCP server is in a `connected` state before making calls.

### Extension Installation

For VS Code-family IDEs, Claude Code can auto-install its extension using the CLI commands (`code --install-extension`, `cursor --install-extension`, etc.). The extension ID is `anthropic.claude-code` (or `anthropic.claude-code-internal` for Anthropic employees). Installation includes version checking -- the bundled version (currently `2.1.88` as seen in `ide.ts:929`) is compared against the installed version, and updates are applied if the installed version is older.

For JetBrains IDEs, automatic installation is not supported through CLI. Instead, the system shows a prominent notice directing users to download from the marketplace.

The `/ide` command (`src/commands/ide/ide.tsx`) provides a selection UI for manually connecting to a detected IDE, showing available instances with workspace disambiguation when multiple instances of the same IDE are running.

---

## 6. File Synchronization

### LSP Document Lifecycle

The `LSPServerManager` implements the full LSP text document synchronization protocol through four methods:

- **`openFile(filePath, content)`** -- sends `textDocument/didOpen` with language ID derived from file extension, tracks the file-to-server mapping in an `openedFiles` map
- **`changeFile(filePath, content)`** -- sends `textDocument/didChange` with full document content (not incremental); falls back to `openFile()` if the file has not been opened yet
- **`saveFile(filePath)`** -- sends `textDocument/didSave` to trigger server-side diagnostics
- **`closeFile(filePath)`** -- sends `textDocument/didClose` and removes from tracking

The `openedFiles` map (`Map<string, string>`) tracks which files are open on which servers by URI, preventing duplicate `didOpen` notifications and ensuring the LSP protocol invariant that `didOpen` must precede `didChange`.

### Integration with File Write/Edit Tools

The file synchronization is triggered from Claude Code's tool implementations. When `FileWriteTool` writes a file:

```typescript
// src/tools/FileWriteTool/FileWriteTool.ts:308-320
const lspManager = getLspServerManager()
if (lspManager) {
  clearDeliveredDiagnosticsForFile(`file://${fullFilePath}`)
  lspManager.changeFile(fullFilePath, content).catch(...)
  lspManager.saveFile(fullFilePath).catch(...)
}
```

Three operations happen in sequence: (1) previously delivered diagnostics for the file are cleared so new diagnostics will be shown, (2) `didChange` is sent with the new content, and (3) `didSave` triggers the language server to recompute diagnostics. Both LSP calls are fire-and-forget (errors are caught and logged but do not block the tool response).

### Extension-to-Server Routing

The `LSPServerManager` maintains an `extensionMap` (`Map<string, string[]>`) that maps file extensions to server names. When a file operation arrives, `getServerForFile()` extracts the file extension and looks up the responsible server. The routing is derived from the `extensionToLanguage` configuration of each plugin-provided server. If multiple servers handle the same extension, the first registered server wins (priority-based selection is noted as a future enhancement in the code).

### IDE-Side File Notifications

Beyond LSP synchronization, file changes are also propagated to the IDE extension via `notifyVscodeFileUpdated()` from the `vscodeSdkMcp` module. This dual notification ensures both the language server (for diagnostics) and the IDE (for file tree refresh, diff updates) stay synchronized.

---

## 7. Context Enhancement

### The LSP Tool

The `LSPTool` (`src/tools/LSPTool/LSPTool.ts`) exposes language server capabilities directly to the agent as a callable tool. It supports nine operations:

- `goToDefinition` -- navigate to symbol definitions
- `findReferences` -- find all references to a symbol
- `hover` -- get type information and documentation
- `documentSymbol` -- list symbols in a document
- `workspaceSymbol` -- search symbols across the workspace
- `goToImplementation` -- find interface implementations
- `prepareCallHierarchy` -- get call hierarchy item at position
- `incomingCalls` -- who calls this function
- `outgoingCalls` -- what does this function call

The tool is conditionally enabled based on `isLspConnected()`, which checks that at least one server in the manager is healthy:

```typescript
// src/services/lsp/manager.ts:100-110
export function isLspConnected(): boolean {
  if (initializationState === 'failed') return false
  const manager = getLspServerManager()
  if (!manager) return false
  const servers = manager.getAllServers()
  if (servers.size === 0) return false
  for (const server of servers.values()) {
    if (server.state !== 'error') return true
  }
  return false
}
```

### Diagnostic-Driven Feedback Loop

The diagnostic attachment system creates a feedback loop between the agent's file modifications and the language server's analysis. When the agent edits a file:

1. `FileWriteTool` or `FileEditTool` writes the content
2. LSP `didChange` and `didSave` notifications are sent
3. The language server processes the change and publishes diagnostics
4. `textDocument/publishDiagnostics` notification fires
5. `passiveFeedback.ts` handler captures and formats diagnostics
6. `LSPDiagnosticRegistry` stores them as pending
7. On the next conversation turn, `getLSPDiagnosticAttachments()` delivers them as attachments
8. The agent sees type errors, warnings, etc. and can self-correct

This loop operates entirely passively -- no explicit "check diagnostics" step is needed. The agent naturally receives feedback about the consequences of its edits.

### Plugin-Based Configuration

LSP server configuration comes exclusively from plugins, not from user or project settings:

```typescript
// src/services/lsp/config.ts:15-17
export async function getAllLspServers(): Promise<{
  servers: Record<string, ScopedLspServerConfig>
}> {
```

Plugins provide LSP configuration through either a `.lsp.json` file in the plugin directory or a `manifest.lspServers` field. The `lspPluginIntegration.ts` module validates configurations against a Zod schema, validates that binary paths stay within the plugin directory (preventing path traversal attacks), and substitutes plugin variables into command strings.

Server configurations include:
- `command` and `args` for the language server binary
- `extensionToLanguage` mapping (e.g., `{".ts": "typescript", ".tsx": "typescriptreact"}`)
- `initializationOptions` for server-specific configuration
- `startupTimeout`, `maxRestarts`, `workspaceFolder` lifecycle parameters

### Re-initialization on Plugin Refresh

A notable edge case addressed in `reinitializeLspServerManager()` (manager.ts:226-253): the plugin loader is memoized and can be called very early in startup before marketplace plugins are reconciled, caching an empty plugin list. Unlike commands, agents, hooks, and MCP servers, LSP was not re-initialized on plugin refresh until this was identified as a bug (GitHub issue #15521). The fix ensures that when plugins are refreshed, the old LSP manager is shut down and a new one is created with the updated plugin configuration.
