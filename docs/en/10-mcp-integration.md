# MCP Integration in Claude Code: A Deep Technical Analysis

Claude Code's MCP (Model Context Protocol) implementation is one of the most complete client-side MCP integrations I have seen in a production tool. It supports every transport the protocol defines, handles OAuth flows end-to-end, manages server lifecycles with automatic reconnection, and wires discovered tools directly into the model's tool-use loop. This document traces the architecture from transport negotiation through tool invocation, with references to the actual source.

---

## 1. MCP Client Architecture

At its core, Claude Code acts as an MCP **client**. It uses the official `@modelcontextprotocol/sdk` package to instantiate `Client` objects that connect to external MCP servers. The central hub is `src/services/mcp/client.ts` -- a large module (~3300 lines) that handles connection, tool discovery, tool invocation, and error recovery.

Each MCP server connection is represented as a discriminated union type defined in `src/services/mcp/types.ts:221-227`:

```typescript
export type MCPServerConnection =
  | ConnectedMCPServer
  | FailedMCPServer
  | NeedsAuthMCPServer
  | PendingMCPServer
  | DisabledMCPServer
```

A `ConnectedMCPServer` holds a live `Client` instance, the negotiated `ServerCapabilities`, optional `instructions` from the server, and a `cleanup` function for teardown (`types.ts:180-192`). The other states carry enough context (config, error messages, reconnect attempt counters) to drive UI and retry logic.

Connection results are memoized. The `connectToServer` function at `client.ts:595` is wrapped with lodash `memoize`, keyed on a combination of server name and serialized config. This means repeated calls for the same server return the cached connection rather than opening a new one. When a connection drops, the cache entry is explicitly deleted in the `onclose` handler (`client.ts:1384-1396`) so the next call triggers a fresh connection.

The client identifies itself to servers as:

```typescript
const client = new Client(
  {
    name: 'claude-code',
    title: 'Claude Code',
    version: MACRO.VERSION ?? 'unknown',
    description: "Anthropic's agentic coding tool",
    websiteUrl: PRODUCT_URL,
  },
  {
    capabilities: {
      roots: {},
      elicitation: {},
    },
  },
)
```

This is from `client.ts:985-1001`. The client declares support for `roots` (so servers can ask for the workspace root) and `elicitation` (so servers can prompt the user for input mid-operation). The `ListRootsRequestSchema` handler returns the original working directory as a `file://` URI (`client.ts:1009-1018`).

---

## 2. Transport Layers

Claude Code supports an impressive range of transports. The `TransportSchema` in `types.ts:23-25` enumerates them:

```typescript
export const TransportSchema = lazySchema(() =>
  z.enum(['stdio', 'sse', 'sse-ide', 'http', 'ws', 'sdk']),
)
```

In practice, the `connectToServer` function in `client.ts` branches on `serverRef.type` to create the appropriate transport.

### stdio

The most common transport for local servers. Claude Code spawns the server as a child process using `StdioClientTransport` from the MCP SDK (`client.ts:944-958`). It respects `CLAUDE_CODE_SHELL_PREFIX` for wrapping the command (useful in sandboxed environments), passes through the subprocess environment via `subprocessEnv()`, merges any user-specified `env` vars from the config, and pipes stderr to prevent server error output from polluting the UI. Stderr is accumulated (capped at 64MB) for debugging failed connections.

### SSE (Server-Sent Events)

For remote servers using the older SSE transport (`client.ts:619-677`). The `SSEClientTransport` from the SDK is configured with an `authProvider` (a `ClaudeAuthProvider` instance), custom headers from `getMcpServerHeaders`, and a fetch wrapper that applies per-request timeouts. Critically, the `eventSourceInit` uses a separate fetch function **without** the timeout wrapper -- the SSE connection is long-lived and must not be killed after 60 seconds.

### Streamable HTTP

The newer HTTP transport (`client.ts:784-865`) uses `StreamableHTTPClientTransport`. It follows the MCP Streamable HTTP specification, which requires the `Accept: application/json, text/event-stream` header on every POST. The `wrapFetchWithTimeout` function at `client.ts:492-550` ensures this header is present and applies a 60-second timeout to POST requests while exempting GET requests (which are long-lived SSE streams).

### WebSocket

Two WebSocket variants exist. `ws-ide` (`client.ts:708-734`) is for IDE extension connections (VS Code, JetBrains) and supports TLS options and auth tokens. Plain `ws` (`client.ts:735-783`) is for general-purpose WebSocket servers. Both use a `WebSocketTransport` wrapper from `src/utils/mcpWebSocketTransport.ts` and handle Bun vs Node.js WebSocket API differences.

### SDK Control Transport

For servers running in the SDK process itself (`src/services/mcp/SdkControlTransport.ts`). This is a bridge transport that routes MCP JSON-RPC messages through stdout/stdin control messages between the CLI and SDK processes. The architecture is documented in the file header: CLI MCP Client calls a tool, the transport wraps it as a control request with `server_name` and `request_id`, sends it via stdout to the SDK, and the SDK returns the response through the same channel.

### In-Process Transport

For servers that run inside the Claude Code process itself (`src/services/mcp/InProcessTransport.ts`). The `createLinkedTransportPair()` function creates two linked `InProcessTransport` instances where `send()` on one delivers to `onmessage` on the other. This is used for the Chrome MCP server and the Computer Use MCP server to avoid spawning expensive subprocesses (~325MB each). Messages are delivered via `queueMicrotask` to avoid stack depth issues with synchronous request/response cycles.

### claude.ai Proxy

A special transport for claude.ai connector servers (`client.ts:868-904`). These use `StreamableHTTPClientTransport` pointed at a proxy URL that routes through Anthropic's infrastructure. The `createClaudeAiProxyFetch` function (`client.ts:372-421`) attaches OAuth bearer tokens and retries once on 401 by force-refreshing the token.

---

## 3. Server Lifecycle

### Discovery

Server configurations come from multiple scopes, merged in `getClaudeCodeMcpConfigs` (`config.ts:1071-1251`). The precedence order (lowest to highest) is:

1. **Plugin servers** -- from installed plugins, namespaced as `plugin:name:server`
2. **User servers** -- from `~/.claude/settings.json`
3. **Project servers** -- from `.mcp.json` in the project root (require explicit approval)
4. **Local servers** -- from `.claude/settings.local.json`

Enterprise configurations (`managed-mcp.json`) have exclusive control: when present, all other scopes are ignored (`config.ts:1082-1096`). Claude.ai connector servers are fetched asynchronously and deduped against manually configured servers using content-based signatures (`config.ts:281-310`).

Environment variable expansion happens at config load time via `expandEnvVarsInString` (`envExpansion.ts`), supporting `${VAR}` and `${VAR:-default}` syntax.

### Initialization

The `useManageMCPConnections` hook (`useManageMCPConnections.ts:143`) orchestrates the connection lifecycle. It loads configs, batches connection attempts (local servers in groups of 3, remote servers in groups of 20 -- `client.ts:552-560`), and updates application state as connections succeed or fail.

State updates are batched via a 16ms flush timer (`useManageMCPConnections.ts:207`) to coalesce rapid updates from multiple servers connecting simultaneously. Each update touches clients, tools, commands, and resources atomically.

### Health Monitoring

The connection monitoring logic at `client.ts:1216-1400` is thorough. The system wraps the SDK client's `onerror` and `onclose` handlers with enhanced versions that:

- Log detailed diagnostics for each error type (ECONNRESET, ETIMEDOUT, EPIPE, EHOSTUNREACH, ECONNREFUSED)
- Track consecutive terminal errors -- after 3 consecutive failures, force-close the transport (`client.ts:1350-1359`)
- Detect session expiry on HTTP transports by checking for HTTP 404 + JSON-RPC error code -32001 (`client.ts:193-206`)
- Detect the SDK's "Maximum reconnection attempts" error and trigger reconnection

### Reconnection

Automatic reconnection is implemented with exponential backoff in `useManageMCPConnections.ts:370-400`. The constants are:

```
MAX_RECONNECT_ATTEMPTS = 5
INITIAL_BACKOFF_MS = 1000
MAX_BACKOFF_MS = 30000
```

Reconnection only applies to remote transports (SSE, HTTP, WebSocket) -- stdio servers do not reconnect because the subprocess is gone. During reconnection, the server is put into `pending` state with the current attempt number and max attempts, which the UI renders as a progress indicator. If the server is disabled during a reconnection backoff window, the retry stops.

Tool call retries also handle session expiry. The `call` function at `client.ts:1833` wraps tool invocation in a loop with `MAX_SESSION_RETRIES = 1`, catching `McpSessionExpiredError` to transparently reconnect and retry.

---

## 4. Tool Discovery

When a server connects, `fetchToolsForClient` (`client.ts:1743`) sends a `tools/list` request and transforms each MCP tool into Claude Code's internal `Tool` type. The transformation is detailed:

**Naming**: Tool names are prefixed with `mcp__<serverName>__` via `buildMcpToolName` to namespace them. Server names are normalized to `[a-zA-Z0-9_-]` by `normalizeNameForMCP` (`normalization.ts:17-23`). For SDK servers with the `CLAUDE_AGENT_SDK_MCP_NO_PREFIX` env var, the prefix is skipped so MCP tools can override builtins.

**Schema passthrough**: The tool's `inputSchema` from the server is passed through directly (`client.ts:1813`) as `inputJSONSchema`. Claude Code uses a `z.object({}).passthrough()` schema for its own validation, allowing any input structure.

**Annotations**: MCP tool annotations drive behavior flags:
- `readOnlyHint` controls `isConcurrencySafe()` and `isReadOnly()` (`client.ts:1796-1799`)
- `destructiveHint` controls `isDestructive()` (`client.ts:1804-1806`)
- `openWorldHint` controls `isOpenWorld()` (`client.ts:1807-1809`)
- `anthropic/searchHint` provides search metadata for tool deferred loading (`client.ts:1780-1784`)
- `anthropic/alwaysLoad` forces the tool to always be loaded (`client.ts:1785`)

**Description capping**: Tool descriptions are capped at 2048 characters (`MAX_MCP_DESCRIPTION_LENGTH` at `client.ts:218`). This is a practical defense against OpenAPI-generated MCP servers that dump 15-60KB of endpoint documentation into tool descriptions.

**Tool list changes**: The SDK's `ToolListChangedNotificationSchema` is handled in `useManageMCPConnections.ts` to dynamically refresh tools when servers add or remove them mid-session. Similarly, `ResourceListChangedNotificationSchema` and `PromptListChangedNotificationSchema` trigger re-fetches.

The `MCPTool` base template at `src/tools/MCPTool/MCPTool.ts:27-77` provides the skeleton. Its `call`, `name`, `description`, and `prompt` fields are all overridden at `fetchToolsForClient` time -- the base exists as a structural contract with the tool system.

---

## 5. OAuth Integration

The OAuth implementation in `src/services/mcp/auth.ts` is substantial -- it handles the full RFC 6749 authorization code flow with PKCE, RFC 9728 protected resource metadata discovery, and several vendor-specific workarounds.

### ClaudeAuthProvider

The `ClaudeAuthProvider` class implements the MCP SDK's `OAuthClientProvider` interface. It is instantiated per-server (`client.ts:621`) and manages:

- **Token storage**: Tokens are stored in the system's secure storage (macOS Keychain via `getSecureStorage()`), keyed by a hash of the server name and config to prevent cross-server credential reuse (`auth.ts:325-341`)
- **Metadata discovery**: Uses RFC 9728 (protected resource metadata) with fallback to RFC 8414 (authorization server metadata). A configured `authServerMetadataUrl` in the server config can override automatic discovery (`auth.ts:256-311`)
- **PKCE flow**: Generates code verifiers and challenges for the authorization code exchange
- **Token refresh**: Handled with retry logic for transient errors. Non-standard error codes from vendors like Slack (`invalid_refresh_token`, `expired_refresh_token`, `token_expired`) are normalized to `invalid_grant` so the standard error handling path fires (`auth.ts:147-191`)

### Step-Up Authentication

The `wrapFetchWithStepUpDetection` function wraps fetch to detect HTTP 403 responses that indicate the server needs re-authentication with elevated privileges. When detected, the auth provider's state is reset so the next request triggers a fresh OAuth flow.

### Cross-App Access (XAA)

A separate authentication path (`src/services/mcp/xaa.ts`, `src/services/mcp/xaaIdpLogin.ts`) supports cross-application access via an Identity Provider. When `oauth.xaa` is set to `true` on a server config, the system performs a token exchange: it acquires an ID token from the configured IdP, then exchanges it for an access token at the MCP server's authorization server. This enables enterprise SSO scenarios where a single login grants access to multiple MCP servers.

### Auth Caching

A file-based cache at `~/.claude/mcp-needs-auth-cache.json` tracks which servers need authentication (`client.ts:257-316`). Entries have a 15-minute TTL. This prevents repeated connection attempts to servers that are known to require auth, avoiding wasted time during startup when many servers need credentials.

---

## 6. Security Model

### Permission System

Every MCP tool call goes through Claude Code's permission system. The `checkPermissions` method on MCP tools returns `{ behavior: 'passthrough' }` (`client.ts:1814-1831`), meaning the permission decision is delegated to the broader permission framework. The tool suggests an `addRules` action so users can create persistent allow rules for specific MCP tools.

### Enterprise Policy

Enterprise administrators can control MCP server access via allowlists and denylists in managed settings (`config.ts:364-508`). The policy system supports three matching strategies:

- **Name-based**: Match by server name string
- **Command-based**: Match by the exact command array (for stdio servers)
- **URL-based**: Match by URL pattern with wildcard support (for remote servers)

Denylists take absolute precedence over allowlists (`config.ts:422`). An empty allowlist blocks all servers. The `shouldAllowManagedMcpServersOnly` setting further restricts the allowlist to only check managed (enterprise) settings.

### Project Server Approval

Project-scoped servers from `.mcp.json` require explicit user approval before connecting (`config.ts:1164-1170`). The `getProjectMcpServerStatus` function checks whether the user has approved the server, preventing untrusted repositories from automatically connecting to arbitrary MCP servers.

### Headers Helper Security

The `headersHelper` feature (`headersHelper.ts`) allows running external scripts to generate authentication headers. For project-scoped servers, this requires workspace trust to be established first (`headersHelper.ts:40-57`). This prevents a malicious `.mcp.json` from executing arbitrary scripts before the user has reviewed and trusted the project.

### Tool Name Sanitization

Server and tool names are sanitized through `normalizeNameForMCP` (`normalization.ts:17-23`), which replaces any character outside `[a-zA-Z0-9_-]` with underscores. Claude.ai server names get additional treatment: consecutive underscores are collapsed and leading/trailing underscores are stripped to avoid interference with the `__` delimiter used in MCP tool names.

### Channel Permission Relay

For channel-based interactions (Telegram, Discord), `src/services/mcp/channelPermissions.ts` implements a structured permission system. Permission prompts are sent through active channels and require the server to parse the user's reply and emit a specific `notifications/claude/channel/permission` event with `{request_id, behavior}`. As the file's comments note, this design means "a compromised channel server CAN fabricate 'yes <id>' without the human seeing the prompt" -- an accepted risk documented in the PR discussion, since a compromised channel already has unlimited conversation-injection capability.

---

## 7. Configuration

### Settings Files

MCP servers can be configured at multiple levels:

**Project level** (`.mcp.json` in project root):
```json
{
  "mcpServers": {
    "my-server": {
      "command": "npx",
      "args": ["my-mcp-server"],
      "env": { "API_KEY": "${MY_API_KEY}" }
    }
  }
}
```

**User level** (`~/.claude/settings.json`):
Same structure under the `mcpServers` key in the global config.

**Local level** (`.claude/settings.local.json`):
Project-specific overrides not committed to version control.

**Enterprise level** (`managed-mcp.json`):
When present, takes exclusive control over all MCP server configuration.

### Config Schemas

Each transport type has its own Zod schema (`types.ts:28-134`):

- `McpStdioServerConfigSchema`: `command`, `args`, `env`
- `McpSSEServerConfigSchema`: `url`, `headers`, `headersHelper`, `oauth`
- `McpHTTPServerConfigSchema`: `url`, `headers`, `headersHelper`, `oauth`
- `McpWebSocketServerConfigSchema`: `url`, `headers`, `headersHelper`
- `McpSdkServerConfigSchema`: `name` only
- `McpClaudeAIProxyServerConfigSchema`: `url`, `id`

The OAuth sub-config supports `clientId`, `callbackPort`, `authServerMetadataUrl`, and an `xaa` boolean for cross-app access.

### Environment Variable Expansion

All string values in configs are expanded at load time. The `expandEnvVarsInString` function (`envExpansion.ts:10-38`) handles `${VAR}` and `${VAR:-default}` syntax. Missing variables are tracked and reported.

### Dynamic Config

The `dynamicMcpConfig` parameter in `MCPConnectionManager` (`MCPConnectionManager.tsx:33`) allows runtime injection of server configs. This is used by the SDK to add servers programmatically (via the `mcp_set_servers` control message) and by `--mcp-config` CLI flags.

### Server Enable/Disable

Individual servers can be disabled without removing their configuration. The `isMcpServerDisabled` function checks a `disabledMcpServers` list in settings, and `setMcpServerEnabled` toggles membership. Disabled servers appear in the UI but are not connected.

---

## 8. Error Handling

### Connection Timeouts

Connection attempts are wrapped in a `Promise.race` against a timeout (`client.ts:1048-1077`). The default timeout is 30 seconds, configurable via the `MCP_TIMEOUT` environment variable (`client.ts:457`). On timeout, the transport is force-closed and a `TelemetrySafeError` is thrown.

### Tool Call Timeouts

Tool calls have a separate, effectively infinite timeout of ~27.8 hours (`DEFAULT_MCP_TOOL_TIMEOUT_MS = 100_000_000` at `client.ts:211`), configurable via `MCP_TOOL_TIMEOUT`. Individual HTTP requests within a tool call are wrapped with a 60-second timeout (`MCP_REQUEST_TIMEOUT_MS` at `client.ts:463`). The `wrapFetchWithTimeout` function uses `setTimeout` + `clearTimeout` rather than `AbortSignal.timeout()` to avoid a Bun-specific memory leak where the internal timer's ~2.4KB of native memory lingers for the full duration even after the request completes.

### Session Expiry

HTTP/Streamable-HTTP connections can experience session expiry when the server returns HTTP 404 with JSON-RPC error code -32001 (`client.ts:193-206`). The `isMcpSessionExpiredError` function checks both signals to avoid false positives from generic 404s. On detection, the transport is closed, the connection cache is cleared, and the next operation triggers a fresh connection.

### Auth Errors

`McpAuthError` (`client.ts:152-159`) is thrown when tool calls fail due to expired OAuth tokens (401 responses). This is caught at the tool execution layer to transition the server's status to `needs-auth`, prompting the user to re-authenticate. The `handleRemoteAuthFailure` helper (`client.ts:340-361`) centralizes this for SSE, HTTP, and claude.ai proxy transports.

### Malformed Responses

MCP tool results are sanitized through `recursivelySanitizeUnicode` (`client.ts:1758`) to handle broken Unicode. Tool descriptions are truncated to 2048 characters. Binary content from tool results is detected and persisted to disk rather than passed through the conversation. Result sizes are checked against `maxResultSizeChars` (100,000 characters, set in `MCPTool.ts:35`).

### Stderr Accumulation

For stdio servers, stderr output is accumulated during connection and logged on failure (`client.ts:966-983`). The buffer is capped at 64MB to prevent unbounded memory growth from chatty servers. After successful connection, the buffer is cleared.

### Consecutive Error Tracking

The enhanced error handler at `client.ts:1266` tracks consecutive terminal connection errors. After `MAX_ERRORS_BEFORE_RECONNECT` (3) consecutive terminal errors (ECONNRESET, ETIMEDOUT, EPIPE, etc.), the transport is force-closed to trigger reconnection. Non-terminal errors reset the counter.

---

## 9. My Take

Having traced through this implementation, a few things stand out.

**MCP as an extension system works.** The architecture cleanly separates protocol concerns (transports, authentication, connection lifecycle) from application concerns (tool discovery, permission checking, UI state). A new MCP server plugs into Claude Code without any code changes -- you write a config entry, and the system handles everything from spawning the process to presenting the tools to the model. That is real extension system behavior, not just a protocol adapter.

**The transport abstraction is doing heavy lifting.** Supporting stdio, SSE, streamable HTTP, WebSocket, in-process, and SDK-bridged transports from a single `connectToServer` function is not trivial. The in-process transport for Chrome and Computer Use servers is a particularly smart optimization -- avoiding a 325MB subprocess by running the server in-process via linked transport pairs. The SDK control transport for bridging to external processes via JSON-RPC over stdout is another creative solution to a real problem.

**OAuth is the hardest part, and it shows.** The `auth.ts` file handles RFC 9728 discovery with RFC 8414 fallback, PKCE, token refresh with vendor-specific error normalization (Slack returning `invalid_refresh_token` instead of `invalid_grant`), cross-app access via IdP token exchange, credential storage in system keychains, and lock-file-based concurrency control for token refresh. The `normalizeOAuthErrorBody` function that rewrites HTTP 200 responses with error bodies to 400s (because Slack returns 200 for errors) captures the kind of real-world messiness this code has to deal with. This is the code that has clearly been battle-tested.

**The permission model is well-layered but has a known trade-off.** Every MCP tool call goes through the same permission framework as built-in tools, which is correct. Enterprise policy enforcement via allowlists/denylists gives organizations real control. Project-level servers requiring approval is the right call. But the `checkPermissions` returning `passthrough` means the actual allow/deny decision happens elsewhere in the permission chain -- tracing exactly where an MCP tool call gets approved requires understanding the broader permission framework, not just the MCP layer.

**Reconnection is thoughtful but asymmetric.** Remote servers get automatic reconnection with exponential backoff. Stdio servers do not reconnect because the child process is gone. This is the correct default, but it means a stdio server crash during a long-running task silently fails. The error handling for this case (catching McpError -32000 "Connection closed") is there, but the user experience is "tool failed" rather than "server crashed, want to restart it?"

**The batched state update pattern is worth noting.** The 16ms flush timer for MCP state updates (`useManageMCPConnections.ts:207`) is a sensible optimization for the case where 20+ remote servers connect simultaneously. Without it, each connection would trigger a separate React render cycle. This is the kind of optimization you add after profiling startup with many servers.

**Deduplication is surprisingly complex.** The system deduplicates plugin servers against manual servers, claude.ai connector servers against manual servers, and plugin servers against each other -- all using content-based signatures that compare the actual command arrays or URLs rather than just names. The `unwrapCcrProxyUrl` function even handles the case where a claude.ai connector URL has been rewritten to route through a proxy, extracting the original vendor URL for comparison. This suggests real-world deployment scenarios where users end up with the same server configured multiple ways.

The overall impression is of a system designed by people who have deployed MCP integrations at scale and hit every edge case. The code is pragmatic -- it handles Slack's non-standard OAuth errors, Bun's memory leaks in `AbortSignal.timeout()`, Java MCP SDK servers that reject unknown fields in the elicitation capability declaration, and CCR proxy URL rewriting. Each of these workarounds has comments explaining the specific failure it prevents. That is the mark of production-grade code that has learned from real failures.
