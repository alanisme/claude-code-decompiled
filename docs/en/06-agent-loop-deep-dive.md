# The Agent Loop: A Deep Dive into the Heart of Claude Code

If you want to understand how an AI coding agent *actually* works at the systems level, forget the marketing pages. Open `src/query.ts`. At 1,729 lines, it is the single largest file in Claude Code's codebase, and it is where every user message enters, every tool call is dispatched, every error is recovered from, and every response eventually exits. This file is the beating heart of the entire system.

I spent a couple of days reading through it and the surrounding modules, and what I found is a surprisingly honest piece of production engineering. Not a clean-room research prototype, but a system that has clearly been shaped by thousands of real-world failure modes. Let me walk you through it.

## 1. What query.ts Is (and Why It's One Giant File)

The `query` function is an **async generator** that takes a user message and yields a stream of events back to the caller. The signature tells you a lot about the architecture right away:

```typescript
// src/query.ts:219-228
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  | StreamEvent
  | RequestStartEvent
  | Message
  | TombstoneMessage
  | ToolUseSummaryMessage,
  Terminal
>
```

The return type is a union of five different event types. The `Terminal` return value (as opposed to yielded values) carries the *reason* the loop exited: `'completed'`, `'aborted_streaming'`, `'aborted_tools'`, `'max_turns'`, `'prompt_too_long'`, `'model_error'`, `'image_error'`, `'hook_stopped'`, or `'stop_hook_prevented'`. Each of those exit paths was added because of a real bug in production.

The outer `query()` function is actually just a thin wrapper that delegates to `queryLoop()` and does lifecycle bookkeeping for consumed commands:

```typescript
// src/query.ts:229-239
const consumedCommandUuids: string[] = []
const terminal = yield* queryLoop(params, consumedCommandUuids)
for (const uuid of consumedCommandUuids) {
  notifyCommandLifecycle(uuid, 'completed')
}
return terminal
```

This separation exists so command lifecycle notifications only fire on successful completion -- not on throws or `.return()` calls that abort the generator.

## 2. The Core Loop: From User Message to Response

The `queryLoop` function is one giant `while (true)` loop. Each iteration represents one round-trip to the Claude API. The loop continues as long as the model keeps requesting tool calls.

### State Management

The loop carries a mutable `State` object between iterations:

```typescript
// src/query.ts:204-217
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined
}
```

The `transition` field is quietly brilliant. It records *why* the previous iteration continued -- `'next_turn'`, `'reactive_compact_retry'`, `'max_output_tokens_recovery'`, `'collapse_drain_retry'`, `'stop_hook_blocking'`, `'max_output_tokens_escalate'`, or `'token_budget_continuation'`. This means the current iteration can make different decisions based on what happened last time. For example, if a `collapse_drain_retry` already fired and the API still returned a prompt-too-long error, the loop knows to fall through to reactive compact instead of draining again.

### The Iteration Anatomy

Each iteration of the `while (true)` loop follows this sequence:

1. **Pre-processing**: Apply tool result budgets, snip compaction, microcompact, context collapse, and auto-compact to the message history. This is where the system fights context window pressure before ever hitting the API.

2. **API Call**: Stream the response from Claude via `deps.callModel()`. While streaming, tool use blocks are detected and (with streaming tool execution enabled) immediately dispatched to the `StreamingToolExecutor`.

3. **Post-streaming**: Handle withheld errors (prompt-too-long, max-output-tokens, media size errors), execute post-sampling hooks, check for user abort.

4. **Tool execution**: Either drain remaining results from the `StreamingToolExecutor` or fall back to `runTools()` from `toolOrchestration.ts`.

5. **Attachment injection**: Add memory attachments, file change notifications, skill discovery results, and queued command outputs.

6. **Continue or return**: Build the next `State` and `continue`, or `return` a `Terminal`.

The key insight is that steps 2 and 4 can overlap when streaming tool execution is enabled. The model is still streaming tokens while the first tool calls are already running. This is a major latency win.

### The Streaming Loop

The inner streaming loop (starting around line 659) iterates over messages from the API:

```typescript
// src/query.ts:659-708 (simplified)
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  thinkingConfig: toolUseContext.options.thinkingConfig,
  tools: toolUseContext.options.tools,
  signal: toolUseContext.abortController.signal,
  options: { model: currentModel, fallbackModel, ... },
})) {
  // Handle streaming fallback (model switch mid-stream)
  // Backfill tool_use inputs for SDK observers
  // Withhold recoverable errors
  // Track assistant messages and tool_use blocks
  // Feed tool_use blocks to StreamingToolExecutor
  // Yield completed tool results mid-stream
}
```

One detail that took me a while to appreciate: the code carefully clones assistant messages before yielding them to SDK consumers, but keeps the originals for the API message history. This is because `backfillObservableInput` adds derived fields (like expanded file paths) that SDK consumers want to see, but sending them back to the API would break prompt cache hits due to byte mismatches. That level of cache awareness is threaded throughout the entire system.

## 3. Streaming Tool Execution

The `StreamingToolExecutor` (`src/services/tools/StreamingToolExecutor.ts`) is one of the more impressive pieces of the codebase. It executes tools *while the model is still streaming its response*.

### Concurrency Model

The executor tracks each tool with a status: `'queued'` -> `'executing'` -> `'completed'` -> `'yielded'`. The concurrency rules are:

```typescript
// src/services/tools/StreamingToolExecutor.ts:129-135
private canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executingTools = this.tools.filter(t => t.status === 'executing')
  return (
    executingTools.length === 0 ||
    (isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
  )
}
```

Translation: concurrent-safe tools (reads, searches) can run in parallel with each other. Non-concurrent tools (writes, bash commands) get exclusive access. This is determined per-tool via the `isConcurrencySafe()` method on each tool definition.

When a new tool_use block arrives from the stream, it's immediately added to the executor:

```typescript
// src/query.ts:838-844
if (streamingToolExecutor && !toolUseContext.abortController.signal.aborted) {
  for (const toolBlock of msgToolUseBlocks) {
    streamingToolExecutor.addTool(toolBlock, message)
  }
}
```

And completed results are polled during the same streaming loop:

```typescript
// src/query.ts:851-862
for (const result of streamingToolExecutor.getCompletedResults()) {
  if (result.message) {
    yield result.message
    toolResults.push(...)
  }
}
```

### Sibling Error Cancellation

Here's where things get really thoughtful. When a Bash command fails, it aborts sibling tools via a dedicated `siblingAbortController`:

```typescript
// src/services/tools/StreamingToolExecutor.ts:357-364
if (tool.block.name === BASH_TOOL_NAME) {
  this.hasErrored = true
  this.erroredToolDescription = this.getToolDescription(tool)
  this.siblingAbortController.abort('sibling_error')
}
```

But only Bash errors trigger this. Read, WebFetch, and other independent tools can fail without affecting siblings. The comment explains the reasoning: "Bash commands often have implicit dependency chains (e.g. mkdir fails -> subsequent commands pointless). Read/WebFetch/etc are independent -- one failure shouldn't nuke the rest." This kind of nuanced error handling is what separates production code from demos.

### The Legacy Path

When streaming tool execution is disabled (or for older code paths), the system falls back to `runTools()` from `src/services/tools/toolOrchestration.ts`. This uses a simpler batch model:

```typescript
// src/services/tools/toolOrchestration.ts:91-116
function partitionToolCalls(
  toolUseMessages: ToolUseBlock[],
  toolUseContext: ToolUseContext,
): Batch[] {
  return toolUseMessages.reduce((acc: Batch[], toolUse) => {
    const tool = findToolByName(toolUseContext.options.tools, toolUse.name)
    const isConcurrencySafe = /* ... parse and check ... */
    if (isConcurrencySafe && acc[acc.length - 1]?.isConcurrencySafe) {
      acc[acc.length - 1]!.blocks.push(toolUse)
    } else {
      acc.push({ isConcurrencySafe, blocks: [toolUse] })
    }
    return acc
  }, [])
}
```

It partitions tool calls into consecutive batches of concurrent-safe or serial tools, then executes each batch appropriately. The max concurrency is configurable via `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` (defaulting to 10).

## 4. Error Recovery

The error recovery in `query.ts` is extensive. I count at least seven distinct recovery mechanisms, each addressing a different failure mode.

### Model Fallback

When the primary model is unavailable (rate limited, overloaded), a `FallbackTriggeredError` triggers a mid-stream model switch:

```typescript
// src/query.ts:894-950
} catch (innerError) {
  if (innerError instanceof FallbackTriggeredError && fallbackModel) {
    currentModel = fallbackModel
    attemptWithFallback = true
    // Clear all state from the failed attempt
    assistantMessages.length = 0
    toolResults.length = 0
    // Discard streaming tool executor and create fresh one
    if (streamingToolExecutor) {
      streamingToolExecutor.discard()
      streamingToolExecutor = new StreamingToolExecutor(...)
    }
    // Strip thinking signatures (model-bound, would cause 400s)
    messagesForQuery = stripSignatureBlocks(messagesForQuery)
    continue
  }
}
```

The `stripSignatureBlocks` call is interesting -- thinking block signatures are cryptographically bound to the model that produced them. Replaying a Capybara thinking block to Opus would cause a 400 error. This kind of edge case only surfaces in production.

### Max Output Tokens Recovery

When the model hits its output token limit mid-response, the system does an escalating retry:

1. **First**: If using the default 8k cap, retry at 64k (`ESCALATED_MAX_TOKENS`) with no meta message -- the same request, just a bigger output budget.
2. **Then**: Inject a meta message telling the model to resume: "Output token limit hit. Resume directly -- no apology, no recap." Up to 3 retries (`MAX_OUTPUT_TOKENS_RECOVERY_LIMIT`).
3. **Finally**: Surface the error to the user.

The recovery message is carefully worded to prevent the model from wasting tokens apologizing or summarizing what it already said. That wording was almost certainly tuned through painful iteration.

### Reactive Compaction

When the API returns a prompt-too-long error, the system doesn't just fail. It first tries context collapse (draining staged collapses), then falls back to reactive compaction -- summarizing the conversation to fit within the context window. This all happens transparently:

```typescript
// src/query.ts:1119-1166
if ((isWithheld413 || isWithheldMedia) && reactiveCompact) {
  const compacted = await reactiveCompact.tryReactiveCompact({
    hasAttempted: hasAttemptedReactiveCompact,
    messages: messagesForQuery,
    cacheSafeParams: { systemPrompt, userContext, systemContext, ... },
  })
  if (compacted) {
    const postCompactMessages = buildPostCompactMessages(compacted)
    for (const msg of postCompactMessages) yield msg
    state = { ...state, messages: postCompactMessages,
              hasAttemptedReactiveCompact: true,
              transition: { reason: 'reactive_compact_retry' } }
    continue
  }
}
```

The `hasAttemptedReactiveCompact` flag prevents infinite loops -- if compaction fails to bring the context under the limit, the error surfaces on the next iteration rather than compacting again.

### Streaming Fallback Tombstones

When a streaming fallback occurs (model switch mid-response), orphaned assistant messages -- including their thinking blocks with model-bound cryptographic signatures -- must be removed. The system yields "tombstone" messages:

```typescript
// src/query.ts:713-727
for (const msg of assistantMessages) {
  yield { type: 'tombstone' as const, message: msg }
}
assistantMessages.length = 0
```

SDK consumers and the UI interpret tombstones as "delete this message from the transcript." Without this, you'd get cryptic "thinking blocks cannot be modified" API errors on the retry.

## 5. Sub-Agent Architecture

The `AgentTool` (`src/tools/AgentTool/runAgent.ts`) spawns child agents that run their own independent `query()` loops. Each sub-agent gets:

- **Its own `agentId`** -- a unique identifier for transcript recording, Perfetto tracing, and analytics scoping
- **An isolated `ToolUseContext`** -- created via `createSubagentContext()` from `src/utils/forkedAgent.ts`, which clones the parent's file state cache and content replacement state
- **Filtered tools** -- resolved via `resolveAgentTools()`, which applies agent-specific tool allow/deny lists
- **Its own MCP servers** -- agents can declare MCP servers in their frontmatter, which are connected at spawn and cleaned up on exit
- **A separate abort controller** -- so aborting a sub-agent doesn't kill the parent

The sub-agent runs `query()` directly:

```typescript
// src/tools/AgentTool/runAgent.ts:15
import { query } from '../../query.js'
```

This is the same `query` function. Sub-agents are full-fidelity agent loops, not some degraded inner mode. They get auto-compaction, streaming tool execution, error recovery -- everything. The only differences are scoping and isolation.

Agent definitions support several isolation modes:
- **Default**: Shares the parent's working directory and file system
- **Worktree**: Operates in a separate git worktree for parallel branch work
- Sub-agents can be **synchronous** (blocks the parent until complete) or **asynchronous** (runs in the background, reports results later)

The transcript recording is particularly clever. Each sub-agent's messages are recorded to a sidechain file under `subagents/`, and the parent can reconstruct the child's content replacement state from these records on resume. This means you can `/resume` a session and sub-agents pick up where they left off.

## 6. Auto-Compaction

Context window management is arguably the hardest problem in long-running agent sessions. Claude Code attacks it with four distinct mechanisms, applied in sequence on every loop iteration:

### 1. Snip Compaction

The lightest touch. Removes old, low-value messages from the middle of the conversation. Runs first because it's cheap (no API call) and might free enough tokens to skip heavier compaction.

```typescript
// src/query.ts:401-410
if (feature('HISTORY_SNIP')) {
  const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
  messagesForQuery = snipResult.messages
  snipTokensFreed = snipResult.tokensFreed
}
```

### 2. Microcompact

Compresses individual tool results that are too large, replacing them with summaries. This runs before auto-compact and can use cached results (the `CACHED_MICROCOMPACT` feature gate).

### 3. Context Collapse

A projection-based system that archives groups of messages into summaries. Unlike auto-compact, collapses are reversible -- they're stored in a separate "collapse store" and replayed via `projectView()` on each iteration. The comment in `query.ts` explains: "the collapsed view is a read-time projection over the REPL's full history."

### 4. Auto-Compact (Full Summarization)

The nuclear option. When token usage exceeds the threshold (context window minus a ~13k buffer), the entire conversation is summarized into a compact representation:

```typescript
// src/services/compact/autoCompact.ts:72-91
export function getAutoCompactThreshold(model: string): number {
  const effectiveContextWindow = getEffectiveContextWindowSize(model)
  const autocompactThreshold =
    effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS
  return autocompactThreshold
}
```

The compaction itself runs as a forked agent (it calls the Claude API to generate the summary), which is why `querySource === 'compact'` gets special treatment throughout the codebase -- you don't want the compact agent to trigger *its own* compaction check.

Auto-compact tracking includes a circuit breaker: after `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` (3) consecutive failures, it stops trying. This prevents runaway API spend from sessions that are irrecoverably stuck.

## 7. The Query Chain

The `queryTracking` field on `ToolUseContext` tracks the chain of API calls within a single user turn:

```typescript
// src/query.ts:347-355
const queryTracking = toolUseContext.queryTracking
  ? {
      chainId: toolUseContext.queryTracking.chainId,
      depth: toolUseContext.queryTracking.depth + 1,
    }
  : {
      chainId: deps.uuid(),
      depth: 0,
    }
```

Every API call in a single turn shares the same `chainId` but gets an incrementing `depth`. This is used for analytics (understanding how many round-trips a typical task takes) and for debugging (correlating log entries across a multi-step tool-use chain).

The `maxTurns` parameter caps how many iterations the loop can run:

```typescript
// src/query.ts:1704-1712
if (maxTurns && nextTurnCount > maxTurns) {
  yield createAttachmentMessage({
    type: 'max_turns_reached',
    maxTurns,
    turnCount: nextTurnCount,
  })
  return { reason: 'max_turns', turnCount: nextTurnCount }
}
```

Sub-agents typically have lower `maxTurns` than the main loop, preventing runaway agent spawns from consuming unbounded resources.

Between iterations, the loop also handles:
- **Tool refreshing**: MCP servers that connected mid-query become available via `refreshTools()`
- **Command queue draining**: Slash commands and task notifications queued during tool execution are injected as attachments
- **Memory prefetch consumption**: Relevant CLAUDE.md files prefetched at turn start are injected if the prefetch has settled
- **Skill discovery injection**: Prefetched skill matches are attached for the next iteration
- **Task summary generation**: For the `claude ps` command, periodic summaries are generated in the background

## 8. Cost Tracking

Cost tracking is split across two modules: `src/cost-tracker.ts` for the accounting logic and `src/bootstrap/state.ts` for the global state.

The core model is per-model usage tracking:

```typescript
// src/cost-tracker.ts:71-80
type StoredCostState = {
  totalCostUSD: number
  totalAPIDuration: number
  totalAPIDurationWithoutRetries: number
  totalToolDuration: number
  totalLinesAdded: number
  totalLinesRemoved: number
  lastDuration: number | undefined
  modelUsage: { [modelName: string]: ModelUsage } | undefined
}
```

Usage accumulates via `accumulateUsage()` in `src/services/api/claude.ts`, which is called after every API response. The `QueryEngine` (`src/QueryEngine.ts`) maintains a `totalUsage` field that aggregates across all turns in a session:

```typescript
// src/QueryEngine.ts:189
private totalUsage: NonNullableUsage
```

Cost state persists to the project config file and is keyed by session ID, so resuming a session picks up the previous cost totals. The `getStoredSessionCosts()` function checks that the session ID matches before returning stored costs -- a different session's costs are discarded.

The `taskBudget` feature adds a hard spending cap. The remaining budget is tracked across compaction boundaries -- when a compaction fires, the system captures the pre-compact context window size and subtracts it from the remaining budget, because the server can no longer see the summarized-away history:

```typescript
// src/query.ts:508-515
if (params.taskBudget) {
  const preCompactContext =
    finalContextTokensFromLastResponse(messagesForQuery)
  taskBudgetRemaining = Math.max(
    0,
    (taskBudgetRemaining ?? params.taskBudget.total) - preCompactContext,
  )
}
```

## 9. My Take

After spending significant time with this codebase, here's what stands out.

### What's Clever

**The streaming tool execution overlap.** Starting tool execution while the model is still streaming is a genuine innovation for agent systems. In a typical multi-tool call where the model emits 3 read operations followed by a write, the reads start running as soon as their blocks arrive -- potentially seconds before the model finishes generating the write call. For I/O-bound tools like file reads and web fetches, this can cut turn latency by 30-50%.

**The transition tracking.** Using `state.transition` to record *why* the loop continued is an elegant solution to a messy problem. Without it, the loop would need complex boolean flags to prevent recovery spirals (compact -> still too long -> compact -> ...). With it, each recovery mechanism can check what the previous iteration tried and make an informed decision about what to try next.

**The withholding pattern.** Recoverable errors (prompt-too-long, max-output-tokens, media size) are withheld from the yield stream during API streaming, then checked after streaming completes. This prevents SDK consumers from seeing an error message that the system is about to recover from. The alternative -- yielding the error and then yielding a "never mind" -- would be much messier.

**The comment at line 151.** I have to quote this:

> *The rules of thinking are lengthy and fortuitous. They require plenty of thinking of most long duration and deep meditation for a wizard to wrap one's noggin around.*

The three rules that follow about thinking block management are genuinely tricky to implement correctly. The wizard metaphor is a perfect way to signal "here be dragons" to future maintainers.

### What's Over-Engineered

**The number of compaction strategies.** Four distinct mechanisms (snip, microcompact, context collapse, auto-compact) plus reactive compaction as a fallback feels like a lot. Each one has its own feature gate, its own tracking state, and its own interaction with the others. The comment "both may run -- they are not mutually exclusive" at line 396 suggests the team isn't fully sure which combinations are optimal either. I suspect this will eventually consolidate.

**The State type.** Nine fields, seven `continue` sites, and a comment noting "Continue sites write `state = { ... }` instead of 9 separate assignments." The State object is doing the job of what would be a proper state machine in a more formal architecture. Each continue site constructs a new State with slightly different fields, and it's hard to verify that all the invariants hold across all paths.

### What Other Agent Builders Can Learn

1. **Use async generators, not callbacks.** The `async function*` pattern gives you backpressure for free. The caller controls the pace of consumption. You can `yield*` to delegate to sub-generators. And you get automatic cleanup via generator `.return()`. I've seen too many agent frameworks built on callback chains or event emitters that lose these properties.

2. **Error recovery is the feature.** A demo agent that works when the API is healthy is table stakes. The production difference is what happens on the 1-in-100 calls where you hit a rate limit, a context overflow, a model fallback, or an output truncation. Claude Code handles all of these transparently. Most open-source agent frameworks handle zero of them.

3. **Concurrency control matters.** The distinction between concurrency-safe and exclusive tools is simple but crucial. Running 10 file reads in parallel is a 10x speedup. Running 10 file writes in parallel is a race condition. Most agent frameworks don't even have a concept of tool concurrency safety.

4. **Track why you're looping.** The `transition` field pattern is something I'd steal for any loop-based agent. It turns implicit control flow into explicit data, which makes debugging and testing dramatically easier.

5. **Instrument everything.** The `queryCheckpoint()` calls, the `logEvent()` calls, the analytics metadata -- it's a lot of code, but it's what lets you actually understand how the system behaves at scale. The comment about 1,279 sessions with 50+ consecutive autocompact failures (line 69 of `autoCompact.ts`) could only have been written by a team that has real telemetry.

The `query.ts` file is not pretty code. It's long, it's dense, it has deeply nested control flow, and it carries the scars of every production incident the team has dealt with. But it works, at scale, on real tasks. And that's what matters.
