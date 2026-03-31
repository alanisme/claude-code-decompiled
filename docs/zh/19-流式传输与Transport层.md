# 流式传输与Transport层

## 1. 开篇 -- 为什么传输层对AI编码Agent至关重要

当你构建一个逐token流式响应、写文件、运行shell命令、在危险操作前请求权限的编码Agent时，传输层不是事后才考虑的东西 -- 它是整个系统的神经系统。从生成token到用户在终端看到它之间的每一毫秒延迟，都在侵蚀"与智能体对话"的体验感。网络故障期间的每一条丢失消息，都可能意味着半写入的文件或永远不会到达的权限提示。

Claude Code运行在多种部署模式下：本地CLI对stdout输出、嵌入VS Code的SDK、以及运行在Cloudflare代理后面的远程会话（你的笔记本可能在对话中途休眠）。每种环境有截然不同的可靠性特征。本地管道永远不会丢消息。经过企业代理的WebSocket可能在空闲5分钟后被RST。来自地球另一端服务器的SSE流可能停滞30秒然后倾泻积压事件。

Claude Code传输层是一个精心分层的系统。它有一个清晰的`Transport`接口、三个具体实现（WebSocket、Hybrid、SSE）、一个带背压的序列化批量上传器、SDK模式的结构化I/O协议，以及围绕睡眠/唤醒检测、NDJSON安全性和优雅关闭的周到边界情况处理。

## 2. Transport抽象 -- 接口与三种实现

传输层组织在`src/cli/transports/`中。基础是一个`Transport`接口，定义了每个传输必须满足的契约：`connect()`、`write(message)`、`setOnData(callback)`、`setOnClose(callback)`、`isConnectedStatus()`、`close()`。

`transportUtils.ts`中的工厂函数揭示了选择逻辑：

```typescript
// src/cli/transports/transportUtils.ts:16-45
export function getTransportForUrl(
  url: URL,
  headers: Record<string, string> = {},
  sessionId?: string,
  refreshHeaders?: () => Record<string, string>,
): Transport {
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_CCR_V2)) {
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

三种传输，三个部署时代。纯WebSocket是最初的。Hybrid是过渡步骤 -- 保留WebSocket读取但将写入切换到HTTP POST。SSE是最新的，专为CCR v2架构构建。通过环境变量在同一接口背后切换传输实现 -- 这正是在不破坏生产的情况下演进基础设施的方式。

## 3. Hybrid Transport -- WebSocket读 + HTTP POST写

`HybridTransport`（`src/cli/transports/HybridTransport.ts`）是三者中架构上最有趣的。它继承`WebSocketTransport`用于读取，但完全覆盖写入路径使用HTTP POST。类注释解释了为什么：

```typescript
// src/cli/transports/HybridTransport.ts:49-53
// Why serialize? Bridge mode fires writes via `void transport.write()`
// (fire-and-forget). Without this, concurrent POSTs -> concurrent Firestore
// writes to the same document -> collisions -> retry storms -> pages oncall.
```

这是从生产中学到的教训：对共享文档存储的并发写入导致写冲突。解决方案是通过单一队列序列化所有写入。

URL转换也很巧妙 -- 通过交换协议和路径段将WebSocket URL转换为HTTP端点：

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

`postOnce()`方法区分永久失败（非429的4xx）和可重试失败（429、5xx、网络错误）。永久失败静默丢弃，可重试失败抛出异常让上传器重新排队并退避。这正是正确的行为。

关闭时的grace period也是经过生产锤炼的细节：

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

给排队的写入最多3秒来排空，然后硬停。`close()`本身保持同步（立即返回）-- grace period在后台运行。这是一个务实的妥协：尽力投递但不阻塞关闭。

## 4. SSE Transport -- 带重连的Server-Sent Events

`SSETransport`（`src/cli/transports/SSETransport.ts`）是最新的传输，为CCR v2架构构建。SSE读取 + HTTP POST写入，完全不用WebSocket。

SSE帧解析器是手写的，而不是依赖`EventSource` API。这是刻意选择：原生`EventSource` API不支持自定义header（认证需要），第三方polyfill增加不必要的依赖：

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
    // ... 解析event:、id:、data:字段
  }
  return { frames, remaining: buffer.slice(pos) }
}
```

`remaining`返回值是关键 -- SSE帧可能跨TCP段到达，解析器必须在缓冲区中积累部分帧，只处理完整的（以`\n\n`分隔的）帧。

序列号追踪启用断连后的恢复。传输追踪高水位标记（`lastSequenceNum`），在重连时作为查询参数和`Last-Event-ID` header发送。还有一个使用`seenSequenceNums` Set的去重机制，超过1000条时修剪以防止无限内存增长。

活性检测简单有效：服务器每15秒发送keepalive注释，如果45秒内没有帧到达，连接被视为死亡。超时回调作为类属性提升以避免每帧的闭包分配 -- 一个在每个SSE帧上重置定时器时很重要的微优化。

## 5. 批量事件上传 -- SerialBatchEventUploader

`SerialBatchEventUploader`（`src/cli/transports/SerialBatchEventUploader.ts`）是一个通用的可复用原语，解决"多生产者，一个慢消费者"问题。

背压通过使`enqueue()`异步实现 -- 当`pending.length + items.length > maxQueueSize`时阻塞：

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

但代码库明确记录了一个注意事项：bridge调用者使用`void transport.write()`（fire-and-forget），所以它们从不await，从不体验背压。HybridTransport中的`maxQueueSize: 100,000`是内存限制，不是流控机制。注释说"在后续follow-up中接入真正的背压" -- 诚实的技术债务追踪。

`takeBatch()`方法支持基于计数和基于字节的批处理。第一项无论大小都会被包含（防止超大事件的队头阻塞），后续项只在符合`maxBatchBytes`时才添加。

对不可序列化项（BigInt、循环引用）的静默丢弃设计尤为精妙。没有这个，一个毒丸事件会永远阻塞队列：

```typescript
// src/cli/transports/SerialBatchEventUploader.ts:213-233
private takeBatch(): T[] {
  // ...
  try {
    itemBytes = Buffer.byteLength(jsonStringify(this.pending[count]))
  } catch {
    this.pending.splice(count, 1) // 丢弃不可序列化的项
    continue
  }
  // ...
}
```

重试延迟支持服务器提供的`Retry-After`值，经过clamp和jitter以防止惊群效应：

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

clamp很重要：它意味着行为异常的服务器不能告诉客户端等待0ms（热循环）或24小时（停滞）。

## 6. 结构化I/O协议 -- SDK模式的stdin/stdout消息协议

当Claude Code作为SDK子进程运行（嵌入VS Code时），它通过stdin/stdout上的换行分隔JSON通信。`StructuredIO`类（`src/cli/structuredIO.ts`）管理这个协议。

读取端是一个async generator，积累传入数据并按换行分割：

```typescript
// src/cli/structuredIO.ts:222-241
private async *read() {
  let content = ''
  const splitAndProcess = async function* (this: StructuredIO) {
    for (;;) {
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

`prependedLines`机制允许在迭代过程中注入合成用户消息 -- bridge用于重连时重放消息。

权限请求/响应协议是StructuredIO中最复杂的部分。当Claude Code需要工具使用权限时，它通过stdout发送`control_request`并在stdin等待`control_response`。bridge也可以通过`injectControlResponse()`注入响应，与SDK宿主的响应竞争。先到达的胜出。有一个`resolvedToolUseIds` Set（限制1000条）防止重复响应被处理。

## 7. 流事件合并 -- 100ms缓冲批处理

HybridTransport中最巧妙的优化之一是流事件合并。Claude生成token时，会产生一连串微小的`stream_event`消息。每个单独发HTTP POST是极其浪费的。

解决方案是100ms延迟缓冲：

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
  // 非流事件立即刷新缓冲区以保持顺序
  await this.uploader.enqueue([...this.takeStreamEvents(), message])
  return this.uploader.flush()
}
```

流事件积累最多100ms后作为单个批次刷新。但如果非流事件到达（如`control_request`），缓冲区立即刷新以保持消息顺序。流事件的promise立即解析 -- 调用者不等待它们 -- 而非流事件正确等待flush。

100ms窗口选择得恰到好处：足够快让用户不感知延迟，足够慢能将几十个token事件批处理到单个POST中。

## 8. 错误恢复 -- 超时、重连、优雅关闭

WebSocket传输有最成熟的重连逻辑。指数退避加jitter防止惊群：

```typescript
// src/cli/transports/WebSocketTransport.ts:510-517
const baseDelay = Math.min(
  DEFAULT_BASE_RECONNECT_DELAY * Math.pow(2, this.reconnectAttempts - 1),
  DEFAULT_MAX_RECONNECT_DELAY,
)
const delay = Math.max(
  0,
  baseDelay + baseDelay * 0.25 * (2 * Math.random() - 1),
)
```

重连预算是基于时间的（10分钟，`DEFAULT_RECONNECT_GIVE_UP_MS = 600_000`），而不是基于次数的。这是正确的选择：网络中断持续时间变化极大。

**睡眠/唤醒检测**在两个层面处理。在重连路径中：如果连续重连尝试之间的间隔超过60秒，机器可能休眠了，预算重置。在ping间隔中：如果`setInterval` tick之间的间隔超过阈值，进程被挂起了（盖上笔记本盖子），socket被假定为死亡：

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

注释解释了原因：对死socket的`ws.ping()`会静默成功（字节进入内核缓冲区），所以等待ping/pong往返在挂起后是不可靠的。

重连时的消息重放使用一个1000条消息的`CircularBuffer`。关键的是，消息在重放后不会从缓冲区中清除 -- 它们保留到服务器在下一次重连时确认收到，防止连接在重放期间再次断开时丢失消息。

## 9. NDJSON序列化 -- 防护无效JSON

`ndjsonSafeStringify`函数（`src/cli/ndjsonSafeStringify.ts`）解决了一个微妙但真实的问题。JSON中原始U+2028（行分隔符）和U+2029（段分隔符）字符在ECMA-404下是合法的，但JavaScript按ECMA-262将它们视为行终止符。NDJSON协议按行边界分割时，JSON字符串内的原始U+2028会把消息一分为二：

```typescript
// src/cli/ndjsonSafeStringify.ts:16-22
const JS_LINE_TERMINATORS = /\u2028|\u2029/g

function escapeJsLineTerminators(json: string): string {
  return json.replace(JS_LINE_TERMINATORS, c =>
    c === '\u2028' ? '\\u2028' : '\\u2029',
  )
}
```

转义形式（`\u2028`）是等价JSON -- 解析结果相同 -- 但它永远不会被误认为行终止符。单个regex用alternation是刻意的性能选择：一次遍历而不是两次。

