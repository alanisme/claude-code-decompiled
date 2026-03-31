# Bridge System and Remote Sessions

## 1. Opening -- What the Bridge System Is and Why It Exists

The bridge subsystem in Claude Code is one of the most interesting pieces of distributed systems engineering hiding inside what most people think of as "just a CLI tool." The bridge system is the machinery that powers Remote Control -- the feature that lets you start a Claude Code session on your local machine and drive it from the claude.ai web interface. That sounds simple, but the implementation touches on work queues, JWT lifecycle management, WebSocket/SSE hybrid transports, multi-session capacity scheduling, git worktree isolation, and a dual-generation protocol migration that is still in flight.

At its core, the bridge solves a coordination problem: how does a headless CLI process running on your laptop receive instructions from a web browser, execute them with full local filesystem access, and stream results back -- all while handling authentication, reconnection, capacity limits, and graceful degradation? The answer involves roughly 30 source files in `src/bridge/`, and a protocol that has evolved through at least two major versions.

The bridge's internal codename appears to be "tengu" (you will see `tengu_` prefixed on every GrowthBook feature flag and analytics event). The user-facing name is "Remote Control," invoked via `claude remote-control` or the `/remote-control` slash command.

## 2. Bridge Architecture -- The Components and How They Fit Together

The bridge has two distinct deployment shapes, and understanding this duality is key to understanding the codebase:

**Standalone bridge** (`bridgeMain.ts`): Launched by `claude remote-control`. This is a long-running process that registers an "environment" with the server, polls for work, spawns child Claude Code processes for each session, and manages their lifecycle. Think of it as a mini orchestrator.

**REPL bridge** (`replBridge.ts` / `initReplBridge.ts`): Embedded inside a running interactive Claude Code session. When you toggle Remote Control on from within the REPL, this variant piggybacks on the existing session -- it registers an environment, creates a session on the server, and wires the REPL's message stream into the bridge transport. No child process spawning required.

Both shapes share the same API client (`bridgeApi.ts`), authentication layer (`bridgeConfig.ts`, `trustedDevice.ts`), work secret decoding (`workSecret.ts`), JWT refresh scheduling (`jwtUtils.ts`), and poll configuration (`pollConfig.ts`).

The key files and their responsibilities:

| File | Role |
|------|------|
| `bridgeMain.ts` | Standalone bridge orchestrator (~1900 lines) |
| `bridgeApi.ts` | HTTP client for the Environments API |
| `bridgeConfig.ts` | Auth/URL resolution with dev overrides |
| `sessionRunner.ts` | Child process spawner and NDJSON parser |
| `jwtUtils.ts` | Proactive JWT refresh scheduler |
| `workSecret.ts` | Work secret decoding, SDK URL construction, worker registration |
| `capacityWake.ts` | Signal primitive for early poll-loop wakeup |
| `replBridge.ts` | REPL-embedded bridge (~2400 lines) |
| `remoteBridgeCore.ts` | "Env-less" v2 bridge (direct session-ingress, no Environments API) |
| `types.ts` | Shared type definitions and protocol types |
| `trustedDevice.ts` | Device enrollment and token management |
| `bridgeEnabled.ts` | Feature gate checks and entitlement validation |
| `pollConfig.ts` | Server-driven poll interval tuning via GrowthBook |
| `sessionIdCompat.ts` | Session ID tag translation between `cse_*` and `session_*` |

## 3. Session Lifecycle -- Creation, Heartbeat, Reconnection, Cleanup

A bridge session's life follows a clear arc. The standalone bridge path is traced here since it is the most complete:

**Registration.** The bridge generates a UUID (`bridgeId`) and calls `POST /v1/environments/bridge` with machine name, directory, branch, git repo URL, max session count, and worker type. The server returns an `environment_id` and `environment_secret`. This is an idempotent operation -- passing a `reuseEnvironmentId` re-attaches to an existing environment:

```typescript
// bridgeApi.ts:142-197
async registerBridgeEnvironment(config: BridgeConfig): Promise<{
  environment_id: string; environment_secret: string
}> {
  // ...
  const response = await withOAuthRetry(
    (token: string) => axios.post(
      `${deps.baseUrl}/v1/environments/bridge`,
      {
        machine_name: config.machineName,
        directory: config.dir,
        branch: config.branch,
        max_sessions: config.maxSessions,
        metadata: { worker_type: config.workerType },
        ...(config.reuseEnvironmentId && {
          environment_id: config.reuseEnvironmentId,
        }),
      },
      // ...
    ),
    'Registration',
  )
  // ...
}
```

**Polling.** The bridge enters a `while (!loopSignal.aborted)` loop in `runBridgeLoop` (`bridgeMain.ts:600`), calling `GET /v1/environments/{id}/work/poll` on each iteration. The poll includes a `reclaim_older_than_ms` parameter (default 5000ms) that lets the bridge pick up stale-pending work items whose previous handler died.

**Acknowledgement.** When work arrives, the bridge explicitly acks after committing to handle it -- not before. This is a deliberate design choice documented in the code (`bridgeMain.ts:832-849`). If the bridge is at capacity, the work goes unacked so the server can re-deliver it to another bridge.

**Heartbeat.** Active sessions send periodic heartbeats via `POST /v1/environments/{id}/work/{workId}/heartbeat`. These use the session ingress JWT (not the environment secret) specifically because it avoids a database hit on the server side. The heartbeat returns `{ lease_extended: boolean, state: string }`, and a failed heartbeat triggers `reconnectSession` to re-queue the work.

**Session done.** When a child process exits, `onSessionDone` fires (`bridgeMain.ts:442-591`). It:
1. Cleans up all session tracking maps
2. Wakes the capacity signal so the poll loop can accept new work immediately
3. Calls `stopWork` (with retry) to notify the server
4. Removes any git worktree that was created
5. In single-session mode, aborts the poll loop entirely
6. In multi-session mode, archives the session and returns to idle

**Shutdown.** The shutdown sequence (`bridgeMain.ts:1403-1580`) is impressively thorough. It sends SIGTERM to all active sessions, waits up to 30 seconds, escalates to SIGKILL for stuck processes, stops all work items, awaits pending cleanups, archives sessions, deregisters the environment, and clears the crash-recovery pointer. In single-session mode with `--continue` support, it intentionally leaves the environment alive for resume.

## 4. Work Polling and Dispatch

The poll loop is where most of the complexity lives. It is not a simple fetch-and-dispatch -- it is a state machine managing capacity, backoff, heartbeat, and multiple session types.

The work response has a `data.type` discriminant:

```typescript
// types.ts:18-21
export type WorkData = {
  type: 'session' | 'healthcheck'
  id: string
}
```

Healthchecks are trivially acked. Sessions are where it gets interesting. When a `session` work item arrives:

1. The bridge checks if this session already exists (token refresh scenario)
2. If so, it delivers the fresh JWT to the child process via stdin and re-schedules the refresh timer
3. If at capacity, it breaks without acking so the server re-delivers
4. Otherwise, it decodes the work secret, determines v1 vs v2 transport, spawns the child

The poll interval is not hardcoded -- it is server-driven via GrowthBook (`pollConfig.ts`). The configuration has separate intervals for different capacity states:

```typescript
// pollConfigDefaults.ts:55-82
export const DEFAULT_POLL_CONFIG: PollIntervalConfig = {
  poll_interval_ms_not_at_capacity: 2000,           // 2s - fast pickup
  poll_interval_ms_at_capacity: 600_000,             // 10min - liveness only
  non_exclusive_heartbeat_interval_ms: 0,            // disabled by default
  multisession_poll_interval_ms_not_at_capacity: 2000,
  multisession_poll_interval_ms_partial_capacity: 2000,
  multisession_poll_interval_ms_at_capacity: 600_000,
  reclaim_older_than_ms: 5000,
  session_keepalive_interval_v2_ms: 120_000,
}
```

The at-capacity behavior is particularly well-designed. When all session slots are full, the bridge enters a heartbeat-only loop that still extends work leases without polling the database-backed work queue. The heartbeat loop and at-capacity poll are composable -- both can run simultaneously, with the heartbeat periodically yielding to a poll cycle at a configurable deadline. This is the kind of operational knob that matters at scale.

The capacity wake mechanism (`capacityWake.ts`) is a clean abstraction that lets the poll loop sleep while at capacity but wake immediately when a session ends:

```typescript
// capacityWake.ts:28-56
export function createCapacityWake(outerSignal: AbortSignal): CapacityWake {
  let wakeController = new AbortController()
  function wake(): void {
    wakeController.abort()
    wakeController = new AbortController()
  }
  function signal(): CapacitySignal {
    const merged = new AbortController()
    const abort = (): void => merged.abort()
    outerSignal.addEventListener('abort', abort, { once: true })
    const capSig = wakeController.signal
    capSig.addEventListener('abort', abort, { once: true })
    return { signal: merged.signal, cleanup: /* ... */ }
  }
  return { signal, wake }
}
```

It merges two abort signals -- the outer loop signal (shutdown) and a capacity-change signal -- into one. When a session completes, `capacityWake.wake()` fires, the merged signal aborts, and the sleep resolves immediately. Simple, elegant, no race conditions.

## 5. Authentication -- JWT Refresh, Work Secrets, Trusted Devices

Authentication in the bridge system operates at three distinct layers, and understanding how they interact is critical to understanding the system's reliability model.

**Layer 1: OAuth tokens.** The bridge authenticates to the Environments API using the user's claude.ai OAuth token. This is the "outer" credential -- it proves the user is who they say they are. The `bridgeConfig.ts` module resolves tokens with a dev-override fallback:

```typescript
// bridgeConfig.ts:38-40
export function getBridgeAccessToken(): string | undefined {
  return getBridgeTokenOverride() ?? getClaudeAIOAuthTokens()?.accessToken
}
```

On 401, `withOAuthRetry` in `bridgeApi.ts:106-139` attempts a single token refresh before giving up. This mirrors the retry pattern used for inference calls.

**Layer 2: Work secrets.** Each work item from the poll response carries a base64url-encoded `secret` containing the session ingress JWT, API base URL, source info, and auth tokens. The `decodeWorkSecret` function (`workSecret.ts:6-32`) validates version 1 format and extracts the JWT:

```typescript
// workSecret.ts:6-32
export function decodeWorkSecret(secret: string): WorkSecret {
  const json = Buffer.from(secret, 'base64url').toString('utf-8')
  const parsed: unknown = jsonParse(json)
  if (!parsed || typeof parsed !== 'object' ||
      !('version' in parsed) || parsed.version !== 1) {
    throw new Error(`Unsupported work secret version: ...`)
  }
  // ... validates session_ingress_token and api_base_url
  return parsed as WorkSecret
}
```

The work secret is the server's way of saying "here are the credentials this specific session needs." It decouples the per-session auth from the per-environment auth.

**Layer 3: Trusted device tokens.** Bridge sessions run at `SecurityTier=ELEVATED` on the server. The `trustedDevice.ts` module manages device enrollment and token persistence. During `/login`, the CLI calls `POST /api/auth/trusted_devices` to enroll, and the resulting token is stored in the OS keychain via secure storage. Every bridge API call includes this as `X-Trusted-Device-Token`:

```typescript
// bridgeApi.ts:84-86
const deviceToken = deps.getTrustedDeviceToken?.()
if (deviceToken) {
  headers['X-Trusted-Device-Token'] = deviceToken
}
```

The two-flag rollout strategy is smart -- CLI-side flag controls whether the header is sent, server-side flag controls whether it is enforced. This lets ops roll out gradually without coordinated deploys.

**JWT refresh scheduling** is handled by `createTokenRefreshScheduler` in `jwtUtils.ts`. It decodes the JWT's `exp` claim (without signature verification -- purely for timing), schedules a timer for 5 minutes before expiry, and fires an `onRefresh` callback. The scheduler has a generation counter to prevent stale async callbacks from setting orphaned timers -- a subtle but critical correctness detail:

```typescript
// jwtUtils.ts:165-230
async function doRefresh(sessionId: string, gen: number): Promise<void> {
  // ...
  // If the session was cancelled or rescheduled while we were awaiting,
  // the generation will have changed -- bail out to avoid orphaned timers.
  if (generations.get(sessionId) !== gen) {
    logForDebugging(`... stale (gen ${gen} vs ${generations.get(sessionId)}), skipping`)
    return
  }
  // ...
}
```

The refresh chain also has a fallback: even after a successful refresh, it schedules a follow-up 30 minutes later. This ensures long-running sessions (the kind that run for hours) do not silently die when the initial one-shot timer is the only thing keeping them alive.

## 6. Transport Layer -- WebSocket, SSE, HTTP Hybrid Approach

The bridge supports two transport generations, and both can be active simultaneously depending on server-side feature flags:

**v1: HybridTransport** -- WebSocket reads from Session-Ingress, HTTP POST writes. The SDK URL is constructed as a WebSocket URL (`wss://.../v1/session_ingress/ws/{sessionId}`). There is a subtle localhost vs production distinction:

```typescript
// workSecret.ts:41-48
export function buildSdkUrl(apiBaseUrl: string, sessionId: string): string {
  const isLocalhost = apiBaseUrl.includes('localhost') || apiBaseUrl.includes('127.0.0.1')
  const protocol = isLocalhost ? 'ws' : 'wss'
  const version = isLocalhost ? 'v2' : 'v1'  // Envoy rewrites /v1/ -> /v2/ in prod
  const host = apiBaseUrl.replace(/^https?:\/\//, '').replace(/\/+$/, '')
  return `${protocol}://${host}/${version}/session_ingress/ws/${sessionId}`
}
```

**v2: SSETransport + CCRClient** -- Server-Sent Events for reads, CCR (Claude Code Runtime) worker endpoints for writes. The SDK URL is an HTTP URL pointing at `/v1/code/sessions/{id}`. The child process derives both the SSE stream path and worker endpoints from this base.

The v2 transport introduces a `worker_epoch` concept: each `registerWorker` call bumps the epoch, and the child process includes it in every request. This gives the server a monotonic ordering to detect stale workers:

```typescript
// workSecret.ts:97-127
export async function registerWorker(
  sessionUrl: string, accessToken: string
): Promise<number> {
  const response = await axios.post(`${sessionUrl}/worker/register`, {}, {
    headers: { Authorization: `Bearer ${accessToken}`, /* ... */ },
    timeout: 10_000,
  })
  const raw = response.data?.worker_epoch
  const epoch = typeof raw === 'string' ? Number(raw) : raw
  // ... validates epoch is a safe integer
  return epoch
}
```

The `replBridgeTransport.ts` module defines a `ReplBridgeTransport` abstraction that papers over the v1/v2 difference for the REPL bridge. The abstraction includes v2-specific operations like `reportState`, `reportMetadata`, and `reportDelivery` that are no-ops on v1 -- a clean way to handle the protocol mismatch.

The v1 adapter is a thin wrapper:

```typescript
// replBridgeTransport.ts:78-100
export function createV1ReplTransport(hybrid: HybridTransport): ReplBridgeTransport {
  return {
    write: msg => hybrid.write(msg),
    writeBatch: msgs => hybrid.writeBatch(msgs),
    close: () => hybrid.close(),
    // v1 Session-Ingress WS doesn't use SSE sequence numbers
    getLastSequenceNum: () => 0,
    reportState: () => {},     // no-op on v1
    reportMetadata: () => {},  // no-op on v1
    reportDelivery: () => {},  // no-op on v1
    // ...
  }
}
```

**The env-less bridge** (`remoteBridgeCore.ts`) represents a third evolution: it skips the Environments API entirely. Instead of register-poll-ack-stop, it does:
1. `POST /v1/code/sessions` (create session with OAuth)
2. `POST /v1/code/sessions/{id}/bridge` (get worker JWT + epoch)
3. SSE transport with proactive `/bridge` re-calls for token refresh

As the file's header comment notes: "No register/poll/ack/stop/heartbeat/deregister environment lifecycle." This is gated by `tengu_bridge_repl_v2` and currently REPL-only. The comment explains the architectural motivation: the Environments API historically existed because CCR's worker endpoints required a session_id+role=worker JWT that only the work-dispatch layer could mint. Server PR #292605 adds the `/bridge` endpoint as a direct OAuth-to-worker-JWT exchange, making the environment layer optional.

## 7. Multi-Session Management -- Spawn Modes

The standalone bridge supports three spawn modes, gated by `tengu_ccr_bridge_multi_session`:

```typescript
// types.ts:64-69
export type SpawnMode = 'single-session' | 'worktree' | 'same-dir'
```

**single-session**: One session in cwd, bridge tears down when it ends. This is the original model and the default.

**same-dir**: Persistent server, all sessions share the working directory. Simple but sessions can stomp each other's file changes.

**worktree**: Persistent server, each session gets an isolated git worktree. The initial session (pre-created by `--create-session-in-dir`) runs in `config.dir` for UX continuity; subsequent on-demand sessions get worktrees named `bridge-{sessionId}`. Worktree creation and cleanup are tracked per-session (`sessionWorktrees` map) and cleaned up both on session completion and during shutdown.

The capacity model is straightforward: `config.maxSessions` (default 32 in multi-session, 1 in single-session) gates how many sessions can run concurrently. The poll loop checks `activeSessions.size >= config.maxSessions` before spawning and sleeps with the capacity wake signal when full.

Session spawning (`sessionRunner.ts:248-548`) uses `child_process.spawn` with piped stdio. The child gets `--print --sdk-url ... --session-id ... --input-format stream-json --output-format stream-json --replay-user-messages`. The spawner parses NDJSON from stdout, tracks activities in a ring buffer, detects `control_request` messages for permission forwarding, and extracts the first user message for title derivation.

The child process environment is carefully sanitized:

```typescript
// sessionRunner.ts:306-323
const env: NodeJS.ProcessEnv = {
  ...deps.env,
  CLAUDE_CODE_OAUTH_TOKEN: undefined,  // Strip bridge's OAuth token
  CLAUDE_CODE_ENVIRONMENT_KIND: 'bridge',
  ...(deps.sandbox && { CLAUDE_CODE_FORCE_SANDBOX: '1' }),
  CLAUDE_CODE_SESSION_ACCESS_TOKEN: opts.accessToken,
  CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2: '1',
  ...(opts.useCcrV2 && {
    CLAUDE_CODE_USE_CCR_V2: '1',
    CLAUDE_CODE_WORKER_EPOCH: String(opts.workerEpoch),
  }),
}
```

Note that the parent's OAuth token is explicitly stripped (`CLAUDE_CODE_OAUTH_TOKEN: undefined`) so the child uses the session-scoped JWT for inference. This is a security-conscious design -- the child process should only have the credentials it needs for its specific session.

Token updates to running child processes are delivered via stdin as structured JSON messages:

```typescript
// sessionRunner.ts:533-542
updateAccessToken(token: string): void {
  handle.accessToken = token
  handle.writeStdin(
    jsonStringify({
      type: 'update_environment_variables',
      variables: { CLAUDE_CODE_SESSION_ACCESS_TOKEN: token },
    }) + '\n',
  )
}
```

The child's StructuredIO handler picks up `update_environment_variables` messages and sets `process.env` directly, so the next transport reconnect uses the fresh token.

## 8. Permission Callbacks -- How Permissions Work in Remote Sessions

When a bridge-spawned child process needs permission to use a tool, it emits a `control_request` on stdout. The session runner detects these and forwards them to the server:

```typescript
// sessionRunner.ts:32-43
export type PermissionRequest = {
  type: 'control_request'
  request_id: string
  request: {
    subtype: 'can_use_tool'
    tool_name: string
    input: Record<string, unknown>
    tool_use_id: string
  }
}
```

The permission response flows back through the bridge API (`sendPermissionResponseEvent` in `bridgeApi.ts:419-449`) as a `control_response` event posted to `/v1/sessions/{id}/events`. The server relays this to the web UI, the user approves or denies, and the response comes back down.

The `bridgePermissionCallbacks.ts` module defines the callback interface with `sendRequest`, `sendResponse`, `cancelRequest`, and `onResponse`. There is a type guard for validating responses:

```typescript
// bridgePermissionCallbacks.ts:32-41
function isBridgePermissionResponse(value: unknown): value is BridgePermissionResponse {
  if (!value || typeof value !== 'object') return false
  return 'behavior' in value && (value.behavior === 'allow' || value.behavior === 'deny')
}
```

The response includes not just allow/deny but also `updatedInput` (modified tool parameters) and `updatedPermissions` (permission rule changes). This allows the web UI to both approve specific actions and grant broader permissions in a single interaction.

The v2 transport adds `reportState` which can push `requires_action` to the backend so claude.ai shows a "waiting for input" indicator. This is important for the web experience -- without it, users see the session as idle when it is actually waiting for their approval.

Activity tracking in `sessionRunner.ts` provides human-readable summaries of what each session is doing. Tool names are mapped to verbs:

```typescript
// sessionRunner.ts:70-89
const TOOL_VERBS: Record<string, string> = {
  Read: 'Reading',
  Write: 'Writing',
  Edit: 'Editing',
  Bash: 'Running',
  Glob: 'Searching',
  Grep: 'Searching',
  WebFetch: 'Fetching',
  Task: 'Running task',
  // ...
}
```

These summaries bubble up through the logger and appear in both the CLI status display and (via the transport) the web UI.

## 9. Error Handling and Resilience

The bridge has one of the most thorough error handling strategies in any Node.js application of comparable scope. It manages two independent error tracks with exponential backoff:

**Connection errors** (ECONNREFUSED, ECONNRESET, ETIMEDOUT, ENETUNREACH, EHOSTUNREACH, 5xx): Start at 2s, cap at 2 minutes, give up after 10 minutes.

**General errors** (everything else): Start at 500ms, cap at 30s, give up after 10 minutes.

```typescript
// bridgeMain.ts:72-79
const DEFAULT_BACKOFF: BackoffConfig = {
  connInitialMs: 2_000,
  connCapMs: 120_000,        // 2 minutes
  connGiveUpMs: 600_000,     // 10 minutes
  generalInitialMs: 500,
  generalCapMs: 30_000,
  generalGiveUpMs: 600_000,  // 10 minutes
}
```

Each track resets the other when a new error type occurs, preventing cross-contamination. Jitter is applied via `addJitter` (plus or minus 25%) to prevent thundering herds.

**Sleep detection** is particularly clever. If the gap between poll errors exceeds 2x the connection backoff cap, the bridge assumes the machine slept (laptop lid closed) and resets the entire error budget:

```typescript
// bridgeMain.ts:1276-1290
if (lastPollErrorTime !== null &&
    now - lastPollErrorTime > pollSleepDetectionThresholdMs(backoffConfig)) {
  logForDebugging(`Detected system sleep (${...}s gap), resetting error budget`)
  connErrorStart = null
  connBackoff = 0
  generalErrorStart = null
  generalBackoff = 0
}
```

Without this, opening your laptop after a night would immediately hit the give-up threshold because the elapsed time since `connErrorStart` would exceed 10 minutes.

**Fatal errors** (`BridgeFatalError`) are non-retryable: 401 (auth failed), 403 (access denied), 404 (not found), 410 (environment expired). The 410 case gets a human-friendly message: "Remote Control session has expired. Please restart." There is also an `isExpiredErrorType` helper that checks for both "expired" and "lifetime" substrings in the error type -- a pragmatic approach to handling evolving server error taxonomy.

**Suppressible 403s** are another notable nuance. Some 403 errors (like missing `external_poll_sessions` scope or `environments:manage` permission) are cosmetic -- they do not affect core functionality and should not be shown to users. `isSuppressible403` checks for these patterns and silently logs them.

**stopWork retry** uses a 3-attempt exponential backoff with base delay of 1 second (`bridgeMain.ts:1627-1676`). Auth errors short-circuit. This prevents server-side zombie work items.

**Token refresh** has its own resilience: up to 3 consecutive failures before giving up, with 60-second retry delays. The generation counter ensures stale refresh callbacks do not interfere.

**Completed work deduplication** is handled by a `completedWorkIds` Set. The server may re-deliver work before processing the stop request; the bridge skips these silently. When at capacity and receiving duplicate work, it still respects the capacity throttle to avoid tight-looping.

**Path traversal protection** is enforced at the API layer: every server-provided ID interpolated into a URL is validated against `SAFE_ID_PATTERN = /^[a-zA-Z0-9_-]+$/`:

```typescript
// bridgeApi.ts:48-53
export function validateBridgeId(id: string, label: string): string {
  if (!id || !SAFE_ID_PATTERN.test(id)) {
    throw new Error(`Invalid ${label}: contains unsafe characters`)
  }
  return id
}
```

