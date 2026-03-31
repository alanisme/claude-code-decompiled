# Bridge系统与远程会话

## 1. 开篇 -- Bridge系统是什么，为什么它值得深入研究

Bridge子系统是整个Claude Code代码库中最引人注目的分布式系统工程之一。大多数人把Claude Code当作"一个CLI工具"，但Bridge系统的复杂度远超这个定位。它是Remote Control功能背后的核心引擎 -- 让你在本地启动Claude Code会话，然后通过claude.ai的Web界面远程操控它。

听起来简单对吧？但实际实现涉及工作队列、JWT生命周期管理、WebSocket/SSE混合传输、多会话容量调度、git worktree隔离，以及一个仍在进行中的双代协议迁移。

Bridge要解决的核心协调问题是：一个运行在你笔记本上的无头CLI进程，如何从浏览器接收指令、以完全的本地文件系统访问权限执行这些指令、并将结果流式回传 -- 同时还要处理认证、重连、容量限制和优雅降级？答案涉及`src/bridge/`目录下大约30个源文件，以及经历了至少两个大版本演进的协议。

分析发现Bridge的内部代号是"tengu"（在所有GrowthBook特性标志和分析事件中都能看到`tengu_`前缀）。用户层面的名称是"Remote Control"，通过`claude remote-control`或`/remote-control`斜杠命令调用。

## 2. 架构概览 -- 两种部署形态

Bridge有两种截然不同的部署形态，理解这个二元性是理解整个代码库的关键：

**独立Bridge**（`bridgeMain.ts`）：通过`claude remote-control`启动。这是一个长期运行的进程，负责向服务器注册"environment"、轮询工作、为每个会话生成子进程并管理它们的生命周期。本质上就是一个微型编排器。

**REPL Bridge**（`replBridge.ts` / `initReplBridge.ts`）：嵌入在运行中的交互式Claude Code会话内。当你从REPL内部开启Remote Control时，它直接复用现有会话 -- 注册environment、在服务器创建session、将REPL的消息流接入Bridge传输层。不需要生成子进程。

两种形态共享同一套API客户端（`bridgeApi.ts`）、认证层（`bridgeConfig.ts`、`trustedDevice.ts`）、工作密钥解码（`workSecret.ts`）、JWT刷新调度（`jwtUtils.ts`）和轮询配置（`pollConfig.ts`）。

关键文件及职责一览：

| 文件 | 职责 |
|------|------|
| `bridgeMain.ts` | 独立Bridge编排器（约1900行） |
| `bridgeApi.ts` | Environments API的HTTP客户端 |
| `bridgeConfig.ts` | 认证/URL解析，含开发环境覆盖 |
| `sessionRunner.ts` | 子进程生成器和NDJSON解析器 |
| `jwtUtils.ts` | 主动JWT刷新调度器 |
| `workSecret.ts` | 工作密钥解码、SDK URL构建、worker注册 |
| `capacityWake.ts` | 容量变化时提前唤醒轮询循环的信号原语 |
| `replBridge.ts` | REPL内嵌Bridge（约2400行） |
| `remoteBridgeCore.ts` | 无Environment的v2 Bridge |
| `trustedDevice.ts` | 设备注册和令牌管理 |

## 3. 会话生命周期 -- 创建、心跳、重连、清理

以独立Bridge路径为例来追踪一个完整的会话生命周期：

**注册阶段。** Bridge生成UUID（`bridgeId`）并调用`POST /v1/environments/bridge`，提交机器名、目录、分支、git仓库URL、最大会话数等信息。服务器返回`environment_id`和`environment_secret`。这是幂等操作 -- 传入`reuseEnvironmentId`可以重新挂载到已有环境：

```typescript
// bridgeApi.ts:142-197
async registerBridgeEnvironment(config: BridgeConfig): Promise<{
  environment_id: string; environment_secret: string
}> {
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
    ),
    'Registration',
  )
}
```

**轮询阶段。** Bridge进入`runBridgeLoop`中的`while (!loopSignal.aborted)`循环，每次迭代调用`GET /v1/environments/{id}/work/poll`。轮询包含`reclaim_older_than_ms`参数（默认5000ms），允许Bridge回收前任处理器已死的过期工作项。

**确认阶段。** 这里有一个值得关注的设计决策：Bridge在承诺处理工作之后才发送确认，而不是之前。如果Bridge已满负荷，工作项保持未确认状态，服务器可以重新分发给其他Bridge。

**心跳阶段。** 活跃会话通过`POST /v1/environments/{id}/work/{workId}/heartbeat`定期发送心跳。值得注意的是，心跳使用session ingress JWT而不是environment secret，因为这样可以避免服务端的数据库查询。

**关闭阶段。** 关闭序列（`bridgeMain.ts:1403-1580`）非常严谨：向所有活跃会话发送SIGTERM，等待最多30秒，对卡住的进程升级为SIGKILL，停止所有工作项，等待清理完成，归档会话，注销environment，清除崩溃恢复指针。

## 4. 工作轮询与分发

轮询循环是复杂度最集中的地方。它不是简单的获取-分发，而是一个管理容量、退避、心跳和多种会话类型的状态机。

轮询间隔不是硬编码的，而是通过GrowthBook由服务端驱动：

```typescript
// pollConfigDefaults.ts:55-82
export const DEFAULT_POLL_CONFIG: PollIntervalConfig = {
  poll_interval_ms_not_at_capacity: 2000,           // 2s - 快速接单
  poll_interval_ms_at_capacity: 600_000,             // 10min - 仅保活
  multisession_poll_interval_ms_not_at_capacity: 2000,
  multisession_poll_interval_ms_at_capacity: 600_000,
  reclaim_older_than_ms: 5000,
  session_keepalive_interval_v2_ms: 120_000,
}
```

满负荷时的行为设计得特别精妙。当所有会话槽位都满了，Bridge进入心跳模式，只维持已有工作的租约而不轮询数据库支撑的工作队列。

容量唤醒机制（`capacityWake.ts`）是一个极其优雅的抽象：

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

它把两个AbortSignal合并为一个 -- 外层循环信号（关闭）和容量变化信号。会话结束时`capacityWake.wake()`触发，合并信号中止，sleep立即解析。简单、优雅、没有竞态条件。

## 5. 认证 -- 三层防线

Bridge的认证在三个独立层面运作，理解它们的交互方式对于理解系统的可靠性模型至关重要。

**第一层：OAuth令牌。** Bridge使用用户的claude.ai OAuth令牌向Environments API认证。遇到401时，`withOAuthRetry`尝试一次令牌刷新。

**第二层：工作密钥。** 每个轮询返回的工作项携带base64url编码的`secret`，包含session ingress JWT、API基地址和认证令牌。`decodeWorkSecret`函数验证版本1格式并提取JWT：

```typescript
// workSecret.ts:6-32
export function decodeWorkSecret(secret: string): WorkSecret {
  const json = Buffer.from(secret, 'base64url').toString('utf-8')
  const parsed: unknown = jsonParse(json)
  if (!parsed || typeof parsed !== 'object' ||
      !('version' in parsed) || parsed.version !== 1) {
    throw new Error(`Unsupported work secret version: ...`)
  }
  return parsed as WorkSecret
}
```

工作密钥是服务器对特定会话进行凭证隔离的方式 -- 每个会话只获得它需要的凭证。

**第三层：可信设备令牌。** Bridge会话在服务端以`SecurityTier=ELEVATED`运行。`trustedDevice.ts`管理设备注册和令牌持久化，令牌存储在OS钥匙串中。

JWT刷新调度由`createTokenRefreshScheduler`处理。调度器有一个generation计数器来防止过期的异步回调设置孤儿定时器：

```typescript
// jwtUtils.ts:165-230
async function doRefresh(sessionId: string, gen: number): Promise<void> {
  if (generations.get(sessionId) !== gen) {
    logForDebugging(`... stale (gen ${gen} vs ${generations.get(sessionId)}), skipping`)
    return
  }
}
```

刷新链还有一个后备：即使刷新成功，也会在30分钟后调度后续检查。这确保长时间运行的会话不会因为一次性定时器是唯一保活机制而悄悄死掉。

## 6. 传输层 -- 三代协议并存

Bridge支持多代传输协议，通过服务端特性标志控制激活：

**v1：HybridTransport** -- WebSocket读取，HTTP POST写入。SDK URL构建为WebSocket URL（`wss://.../v1/session_ingress/ws/{sessionId}`）。

**v2：SSETransport + CCRClient** -- SSE读取，CCR worker端点写入。引入了`worker_epoch`概念，提供单调递增的序号来检测过时worker。

**无Environment Bridge**（`remoteBridgeCore.ts`）代表第三代演进：完全跳过Environments API。正如文件注释所说："No register/poll/ack/stop/heartbeat/deregister environment lifecycle。" 这由服务端PR #292605添加的`/bridge`端点支持，实现了从OAuth到worker JWT的直接交换。

渐进式传输迁移通过特性标志实现，每层都有kill switch和兼容性垫片（`sessionIdCompat.ts`）。这是不停机迁移的正确姿势。

## 7. 多会话管理

独立Bridge支持三种生成模式：

```typescript
// types.ts:64-69
export type SpawnMode = 'single-session' | 'worktree' | 'same-dir'
```

**worktree模式**最有意思：每个会话获得隔离的git worktree。初始会话在`config.dir`中运行保持UX连贯性，后续按需会话获得名为`bridge-{sessionId}`的worktree。

子进程环境被仔细清理 -- 父进程的OAuth令牌被显式剥离（`CLAUDE_CODE_OAUTH_TOKEN: undefined`），子进程只能使用session范围的JWT：

```typescript
// sessionRunner.ts:306-323
const env: NodeJS.ProcessEnv = {
  ...deps.env,
  CLAUDE_CODE_OAUTH_TOKEN: undefined,  // 剥离Bridge的OAuth令牌
  CLAUDE_CODE_ENVIRONMENT_KIND: 'bridge',
  CLAUDE_CODE_SESSION_ACCESS_TOKEN: opts.accessToken,
  CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2: '1',
}
```

## 8. 权限回调

当Bridge生成的子进程需要工具使用权限时，它在stdout发出`control_request`。会话运行器检测到后转发给服务器，服务器中继给Web UI，用户批准或拒绝，响应再流回。

响应不仅包含allow/deny，还包含`updatedInput`（修改后的工具参数）和`updatedPermissions`（权限规则变更），允许Web UI在单次交互中既批准特定操作又授予更广泛的权限。

v2传输添加了`reportState`，可以向后端推送`requires_action`状态，让claude.ai显示"等待输入"指示器。

## 9. 错误处理与韧性

Bridge管理两个独立的错误轨道，各自有指数退避：

```typescript
// bridgeMain.ts:72-79
const DEFAULT_BACKOFF: BackoffConfig = {
  connInitialMs: 2_000,
  connCapMs: 120_000,        // 2分钟
  connGiveUpMs: 600_000,     // 10分钟
  generalInitialMs: 500,
  generalCapMs: 30_000,
  generalGiveUpMs: 600_000,  // 10分钟
}
```

**睡眠检测**特别聪明。如果轮询错误之间的间隔超过2倍的连接退避上限，Bridge假设机器进入了睡眠并重置整个错误预算。没有这个机制，早上打开笔记本会立即触发放弃阈值。

**路径遍历防护**在API层强制执行 -- 每个服务器提供的ID都要通过`SAFE_ID_PATTERN = /^[a-zA-Z0-9_-]+$/`验证。致命错误（401、403、404、410）不可重试。已完成工作去重通过`completedWorkIds` Set处理。

