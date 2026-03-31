# Streaming Infrastructure and Transport Layers

## 1. Opening -- Why Streaming Matters for an AI Coding Agent

When you are building a coding agent that streams token-by-token responses from an LLM, writes files, runs shell commands, and asks for permission before dangerous operations, the transport layer is not an afterthought. It is the nervous system. Every millisecond of latency between a generated token and the user seeing it in their terminal chips away at the feeling of "talking to something intelligent." Every dropped message during a network hiccup could mean a half-written file or a permission prompt that never arrives.

Claude Code operates in multiple deployment modes: local CLI talking to stdout, an SDK embedded in VS Code, and a remote session running behind Cloudflare proxies where your laptop might sleep mid-conversation. Each of these environments has radically different reliability characteristics. A local pipe never drops a message. A WebSocket through a corporate proxy might get RST'd after five minutes of idle. An SSE stream from a server on the other side of the planet might stall for 30 seconds and then dump a backlog of events.

The Claude Code transport layer is a carefully layered system that handles all of this. There is a clean `Transport` interface, three concrete implementations (WebSocket, Hybrid, SSE), a serialized batch uploader with backpressure, a structured I/O protocol for SDK mode, and a set of thoughtful edge-case handling around sleep/wake detection, NDJSON safety, and graceful shutdown. The following sections walk through all of it.

## 2. Transport Abstraction -- The Transport Interface and Its Implementations

The transport layer is organized in `src/cli/transports/`. At the foundation is a `Transport` interface (referenced via `import type { Transport } from './Transport.js'` throughout the codebase) that defines the contract every transport must satisfy:

- `connect()` -- establish the connection
- `write(message)` -- send a message to the server
- `setOnData(callback)` -- register a handler for incoming data
- `setOnClose(callback)` -- register a handler for connection close
- `isConnectedStatus()` / `isClosedStatus()` -- state inspection
- `close()` -- tear down the connection

The factory function in `transportUtils.ts` reveals the selection logic:

```typescript
// src/cli/transports/transportUtils.ts:16-45
export function getTransportForUrl(
  url: URL,
  headers: Record<string, string> = {},
  sessionId?: string,
  refreshHeaders?: () => Record<string, string>,
): Transport {
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_CCR_V2)) {
    // v2: SSE for reads, HTTP POST for writes
    return new SSETransport(sseUrl, headers, sessionId, refreshHeaders)
  }

  if (url.protocol === 'ws:' || url.protocol === 'wss:') {
    if (isEnvTruthy(process.env.CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2)) {
      return new HybridTransport(url, headers, sessionId, refreshHeaders)
    }
    return new WebSocketTransport(url, headers, sessionId, refreshHeaders)
  }
}
```

Three transports, three deployment eras. The pure WebSocket transport is the original. The Hybrid transport is a transitional step that keeps WebSocket for reading but switches writes to HTTP POST. The SSE transport is the newest, purpose-built for the CCR v2 architecture. This kind of incremental migration -- where you can feature-flag between transport implementations behind the same interface -- is exactly how you evolve infrastructure without breaking production.

## 3. Hybrid Transport -- WebSocket Reads + HTTP POST Writes

The `HybridTransport` in `src/cli/transports/HybridTransport.ts` is the most architecturally interesting of the three. It extends `WebSocketTransport` for reads but completely overrides the write path to use HTTP POST. The class comment explains why this exists:

```typescript
// src/cli/transports/HybridTransport.ts:49-53
// Why serialize? Bridge mode fires writes via `void transport.write()`
// (fire-and-forget). Without this, concurrent POSTs -> concurrent Firestore
// writes to the same document -> collisions -> retry storms -> pages oncall.
```

This is a lesson learned from production: concurrent writes to a shared document store cause write conflicts. The solution is to serialize all writes through a single queue. The ASCII art in the class comment is worth reproducing because it perfectly describes the data flow:

```
write(stream_event) ---+
                       | (100ms timer)
                       v
write(other) -------> uploader.enqueue()  (SerialBatchEventUploader)
                       ^    |
writeBatch() ----------+   | serial, batched, retries indefinitely,
                           | backpressure at maxQueueSize
                           v
                      postOnce()  (single HTTP POST, throws on retryable)
```

The URL conversion is clever -- it transforms a WebSocket URL into an HTTP endpoint by swapping protocols and path segments:

```typescript
// src/cli/transports/HybridTransport.ts:269-282
function convertWsUrlToPostUrl(wsUrl: URL): string {
  const protocol = wsUrl.protocol === 'wss:' ? 'https:' : 'http:'
  let pathname = wsUrl.pathname
  pathname = pathname.replace('/ws/', '/session/')
  if (!pathname.endsWith('/events')) {
    pathname = pathname.endsWith('/')
      ? pathname + 'events'
      : pathname + '/events'
  }
  return `${protocol}//${wsUrl.host}${pathname}${wsUrl.search}`
}
```

The `postOnce()` method distinguishes permanent failures (4xx except 429) from retryable ones (429, 5xx, network errors). Permanent failures are silently dropped -- the uploader moves on. Retryable failures throw, causing the `SerialBatchEventUploader` to re-queue and back off. This is exactly the right behavior: you do not want to retry a 403 forever, but a 503 or 429 is transient and should be retried.

The grace period on close is another production-hardened detail:

```typescript
// src/cli/transports/HybridTransport.ts:184-195
void Promise.race([
  uploader.flush(),
  new Promise<void>(r => {
    graceTimer = setTimeout(r, CLOSE_GRACE_MS)
  }),
]).finally(() => {
  clearTimeout(graceTimer)
  uploader.close()
})
```

The transport gives queued writes up to 3 seconds to drain on close, then hard-stops. The `close()` method itself remains synchronous (returns immediately) -- the grace period runs in the background. This is a pragmatic compromise: best-effort delivery without blocking shutdown.

## 4. SSE Transport -- Server-Sent Events with Reconnection

The `SSETransport` in `src/cli/transports/SSETransport.ts` is the most recent transport, built for the CCR v2 architecture. It uses standard SSE for reads and HTTP POST for writes -- no WebSocket at all.

The SSE frame parser is hand-written rather than relying on the `EventSource` API. This is a deliberate choice: the native `EventSource` API does not support custom headers (needed for auth), and third-party polyfills add unnecessary dependencies. The parser is clean and correct:

```typescript
// src/cli/transports/SSETransport.ts:58-116
export function parseSSEFrames(buffer: string): {
  frames: SSEFrame[]
  remaining: string
} {
  const frames: SSEFrame[] = []
  let pos = 0
  let idx: number
  while ((idx = buffer.indexOf('\n\n', pos)) !== -1) {
    const rawFrame = buffer.slice(pos, idx)
    pos = idx + 2
    // ... parse event:, id:, data: fields
  }
  return { frames, remaining: buffer.slice(pos) }
}
```

The `remaining` return value is key -- SSE frames can arrive split across TCP segments, so the parser must accumulate partial frames in a buffer and only process complete ones (delimited by `\n\n`).

Sequence number tracking enables resumption after disconnects. The transport tracks a high-water mark (`lastSequenceNum`) and sends it as both a query parameter and `Last-Event-ID` header on reconnection:

```typescript
// src/cli/transports/SSETransport.ts:245-248
const sseUrl = new URL(this.url.href)
if (this.lastSequenceNum > 0) {
  sseUrl.searchParams.set('from_sequence_num', String(this.lastSequenceNum))
}
```

There is also a deduplication mechanism using a `seenSequenceNums` Set that prunes entries beyond 1000 to prevent unbounded memory growth. This handles the case where the server replays events the client already processed during a reconnection window.

The liveness detection is simple and effective: the server sends keepalive comments every 15 seconds, and if no frame arrives within 45 seconds, the connection is treated as dead. The timeout callback is hoisted as a class property to avoid per-frame closure allocation -- a micro-optimization that matters when you are resetting the timer on every single SSE frame:

```typescript
// src/cli/transports/SSETransport.ts:542-550
private readonly onLivenessTimeout = (): void => {
  this.livenessTimer = null
  logForDebugging('SSETransport: Liveness timeout, reconnecting', {
    level: 'error',
  })
  this.abortController?.abort()
  this.handleConnectionError()
}
```

## 5. Batch Event Uploading -- SerialBatchEventUploader, Backpressure, Retry

The `SerialBatchEventUploader` in `src/cli/transports/SerialBatchEventUploader.ts` is a general-purpose primitive that both the HybridTransport and (conceptually) the SSE transport's write path rely on. It solves the fundamental problem of getting events from producer to server reliably when the producer does not want to wait.

The design is simple: a pending queue, a drain loop guarded by a `draining` flag (at most one in-flight), batch slicing, and exponential backoff with jitter on failure. But the details are what make it production-ready.

Backpressure is implemented by making `enqueue()` async -- it blocks when `pending.length + items.length > maxQueueSize`:

```typescript
// src/cli/transports/SerialBatchEventUploader.ts:107-114
while (
  this.pending.length + items.length > this.config.maxQueueSize &&
  !this.closed
) {
  await new Promise<void>(resolve => {
    this.backpressureResolvers.push(resolve)
  })
}
```

This is elegant but comes with a caveat the codebase explicitly documents: bridge callers use `void transport.write()` (fire-and-forget), so they never actually await and never experience backpressure. The `maxQueueSize` of 100,000 in HybridTransport is a memory bound, not a flow control mechanism. The comment says "Wire real backpressure in a follow-up once callers await" -- honest engineering debt tracking.

The `takeBatch()` method supports both count-based and byte-based batching. The first item always goes in regardless of size (to prevent head-of-line blocking on oversized events), and subsequent items are added only if they fit within `maxBatchBytes`:

```typescript
// src/cli/transports/SerialBatchEventUploader.ts:213-233
private takeBatch(): T[] {
  const { maxBatchSize, maxBatchBytes } = this.config
  if (maxBatchBytes === undefined) {
    return this.pending.splice(0, maxBatchSize)
  }
  let bytes = 0
  let count = 0
  while (count < this.pending.length && count < maxBatchSize) {
    let itemBytes: number
    try {
      itemBytes = Buffer.byteLength(jsonStringify(this.pending[count]))
    } catch {
      this.pending.splice(count, 1) // Drop un-serializable items
      continue
    }
    if (count > 0 && bytes + itemBytes > maxBatchBytes) break
    bytes += itemBytes
    count++
  }
  return this.pending.splice(0, count)
}
```

The silent drop of un-serializable items (BigInt, circular refs, throwing `toJSON`). Without this, a single poison-pill event would jam the queue forever.

The `RetryableError` class supports server-provided `Retry-After` values, clamped and jittered to prevent thundering herd:

```typescript
// src/cli/transports/SerialBatchEventUploader.ts:235-253
private retryDelay(failures: number, retryAfterMs?: number): number {
  const jitter = Math.random() * this.config.jitterMs
  if (retryAfterMs !== undefined) {
    const clamped = Math.max(
      this.config.baseDelayMs,
      Math.min(retryAfterMs, this.config.maxDelayMs),
    )
    return clamped + jitter
  }
  const exponential = Math.min(
    this.config.baseDelayMs * 2 ** (failures - 1),
    this.config.maxDelayMs,
  )
  return exponential + jitter
}
```

The clamping is important: it means a misbehaving server cannot tell the client to wait 0ms (hot-loop) or 24 hours (stall). The jitter on top of the server's hint prevents synchronized retry storms.

## 6. Structured I/O Protocol -- stdin/stdout Message Protocol for SDK Mode

When Claude Code runs as an SDK subprocess (embedded in VS Code, for example), it communicates via newline-delimited JSON on stdin/stdout. The `StructuredIO` class in `src/cli/structuredIO.ts` manages this protocol.

The read side is an async generator that accumulates incoming data and splits on newlines:

```typescript
// src/cli/structuredIO.ts:222-241
private async *read() {
  let content = ''
  const splitAndProcess = async function* (this: StructuredIO) {
    for (;;) {
      if (this.prependedLines.length > 0) {
        content = this.prependedLines.join('') + content
        this.prependedLines = []
      }
      const newline = content.indexOf('\n')
      if (newline === -1) break
      const line = content.slice(0, newline)
      content = content.slice(newline + 1)
      const message = await this.processLine(line)
      if (message) yield message
    }
  }.bind(this)

  yield* splitAndProcess()
  for await (const block of this.input) {
    content += block
    yield* splitAndProcess()
  }
}
```

The `prependedLines` mechanism allows injecting synthetic user messages into the stream mid-iteration -- used by the bridge to replay messages on reconnection.

The write side is equally simple -- serialize to JSON, escape NDJSON-unsafe characters (more on that in section 9), write to stdout:

```typescript
// src/cli/structuredIO.ts:465-467
async write(message: StdoutMessage): Promise<void> {
  writeToStdout(ndjsonSafeStringify(message) + '\n')
}
```

The outbound queue (`this.outbound = new Stream<StdoutMessage>()`) ensures ordering between control requests and stream events. Both `sendRequest()` and the print layer enqueue here, and a single drain loop is the only writer. This prevents a control_request (like a permission prompt) from overtaking queued stream_events that the user needs to see first.

The permission request/response protocol is the most complex part of StructuredIO. When Claude Code needs permission to use a tool, it sends a `control_request` via stdout and waits for a `control_response` on stdin. The bridge can also inject responses via `injectControlResponse()`, racing against the SDK host's response. The first one to arrive wins; the loser gets cancelled. There is a `resolvedToolUseIds` Set (bounded to 1000 entries) that prevents duplicate responses from being processed -- critical for WebSocket reconnection scenarios where the server might replay messages.

## 7. Stream Event Coalescing -- 100ms Buffer for Batching Stream Events

One of the cleverest optimizations in the HybridTransport is the stream event coalescing. When Claude is generating tokens, it produces a torrent of tiny `stream_event` messages -- one per token or content delta. Sending each as a separate HTTP POST would be absurdly wasteful.

The solution is a 100ms delay buffer:

```typescript
// src/cli/transports/HybridTransport.ts:117-129
override async write(message: StdoutMessage): Promise<void> {
  if (message.type === 'stream_event') {
    this.streamEventBuffer.push(message)
    if (!this.streamEventTimer) {
      this.streamEventTimer = setTimeout(
        () => this.flushStreamEvents(),
        BATCH_FLUSH_INTERVAL_MS,
      )
    }
    return
  }
  // Non-stream events flush the buffer immediately to preserve ordering
  await this.uploader.enqueue([...this.takeStreamEvents(), message])
  return this.uploader.flush()
}
```

Stream events accumulate for up to 100ms before being flushed as a single batch. But if a non-stream event arrives (like a control_request), the buffer is flushed immediately to preserve message ordering. The promise for stream events resolves immediately -- callers do not await them -- while non-stream events properly await the flush, ensuring that operations like permission prompts are fully persisted before proceeding.

This is the textbook approach to reducing HTTP overhead for high-frequency events while maintaining ordering guarantees for control messages. The 100ms window is well-chosen: fast enough that users do not perceive delay, slow enough to batch dozens of token events into a single POST.

## 8. Error Recovery -- Timeouts, Reconnection, Graceful Shutdown

The WebSocket transport has the most sophisticated reconnection logic, and it serves as the foundation for both itself and the HybridTransport (which inherits from it).

Exponential backoff with jitter prevents thundering herd:

```typescript
// src/cli/transports/WebSocketTransport.ts:510-517
const baseDelay = Math.min(
  DEFAULT_BASE_RECONNECT_DELAY * Math.pow(2, this.reconnectAttempts - 1),
  DEFAULT_MAX_RECONNECT_DELAY,
)
// Add +/-25% jitter to avoid thundering herd
const delay = Math.max(
  0,
  baseDelay + baseDelay * 0.25 * (2 * Math.random() - 1),
)
```

The reconnection budget is time-based (10 minutes, `DEFAULT_RECONNECT_GIVE_UP_MS = 600_000`), not attempt-based. This is the right choice: network outages vary wildly in duration, and a time budget prevents both giving up too early on slow backoff and retrying too aggressively.

Sleep/wake detection is handled at two levels. First, in the reconnection path: if the gap between consecutive reconnection attempts exceeds `SLEEP_DETECTION_THRESHOLD_MS` (60 seconds), the machine likely slept, so the budget resets:

```typescript
// src/cli/transports/WebSocketTransport.ts:476-488
if (
  this.lastReconnectAttemptTime !== null &&
  now - this.lastReconnectAttemptTime > SLEEP_DETECTION_THRESHOLD_MS
) {
  this.reconnectStartTime = now
  this.reconnectAttempts = 0
}
```

Second, in the ping interval: if the gap between `setInterval` ticks exceeds the threshold, the process was suspended (laptop lid closed), and the socket is assumed dead:

```typescript
// src/cli/transports/WebSocketTransport.ts:724-735
if (gap > SLEEP_DETECTION_THRESHOLD_MS) {
  logForDebugging(
    `WebSocketTransport: ${Math.round(gap / 1000)}s tick gap detected
     -- process was suspended, forcing reconnect`,
  )
  this.handleConnectionError()
  return
}
```

The comment explains why: `ws.ping()` on a dead socket succeeds silently (bytes go to the kernel buffer), so waiting for a ping/pong round-trip is unreliable after suspension.

Permanent close codes (1002, 4001, 4003) cause immediate shutdown with no retry, except 4003 (unauthorized) which can be retried if `refreshHeaders()` produces a new token. Message replay on reconnection uses a `CircularBuffer` of 1000 messages that persists across reconnects. The server communicates its last-received message ID via upgrade response headers, and the client replays everything after that point. Critically, messages are NOT cleared from the buffer after replay -- they stay until the server confirms receipt on the next reconnection, preventing loss if the connection drops again during replay.

Listener cleanup on disconnect prevents memory leaks under network instability:

```typescript
// src/cli/transports/WebSocketTransport.ts:389-395
if (this.ws) {
  this.removeWsListeners(this.ws)
  this.ws.close()
  this.ws = null
}
```

Without this, each reconnect would orphan the old WebSocket object and its five event handler closures until garbage collection.

## 9. NDJSON Serialization -- Guard Against Invalid JSON

The `ndjsonSafeStringify` function in `src/cli/ndjsonSafeStringify.ts` addresses a subtle but real problem. JSON is valid when it contains raw U+2028 (LINE SEPARATOR) and U+2029 (PARAGRAPH SEPARATOR) characters per ECMA-404. But JavaScript treats these as line terminators per ECMA-262 Section 11.3. When the NDJSON protocol splits on line boundaries, a raw U+2028 inside a JSON string will split the message in half.

```typescript
// src/cli/ndjsonSafeStringify.ts:16-22
const JS_LINE_TERMINATORS = /\u2028|\u2029/g

function escapeJsLineTerminators(json: string): string {
  return json.replace(JS_LINE_TERMINATORS, c =>
    c === '\u2028' ? '\\u2028' : '\\u2029',
  )
}
```

The escaped form (`\u2028`) is equivalent JSON -- it parses to the same string -- but it can never be mistaken for a line terminator. The comment references a specific issue (gh-28405) where `ProcessTransport` would crash on split lines, and notes that even with the crash fix (silently skipping non-JSON lines), the message was still lost. This escape is the proper fix.

The single regex with alternation is a deliberate performance choice: one pass through the string instead of two. For high-throughput stream events, this matters.

