# Context Window Management in Claude Code

## 1. The Context Pressure Problem

If you have spent any serious time with Claude Code, you know the feeling. You are forty minutes into a complex refactoring session, the model is making good progress, and then you see the context usage bar tick past 80%. The conversation is about to hit the wall. You either compact now and lose nuance, or you keep going and hope the auto-compaction does not destroy something important.

A 200K token context window sounds enormous in the abstract. In practice, it fills up fast. A single large file read can consume 10-15K tokens. A `grep` across a monorepo might return 5K tokens of results. The system prompt, tool definitions, CLAUDE.md memory files, and attachment metadata eat a baseline of 20-40K tokens before the user even types their first message. Add in extended thinking blocks (which can be substantial on complex reasoning tasks), and a productive coding session can burn through 200K tokens in 15-20 conversational turns.

The problem is compounded by the architecture of tool-using conversations. Every tool call creates at minimum two messages: the assistant's `tool_use` block and the user's `tool_result` block. A single "read the file, edit it, then verify" cycle produces six messages. Code exploration is even worse -- searching for a pattern, reading matched files, checking related tests -- a dozen tool interactions before any actual code changes happen. Each of those interactions persists in the context window forever (or at least until something clears them).

The default context window is defined in `src/utils/context.ts`:

```typescript
// src/utils/context.ts:9
export const MODEL_CONTEXT_WINDOW_DEFAULT = 200_000
```

Some models support 1M context windows, detected via the `[1m]` suffix or capability metadata:

```typescript
// src/utils/context.ts:35-39
export function has1mContext(model: string): boolean {
  if (is1mContextDisabled()) {
    return false
  }
  return /\[1m\]/i.test(model)
}
```

But even 1M tokens is not infinite. And larger context windows mean higher API costs and slower responses. The real engineering challenge is not getting a bigger window -- it is managing the one you have.

## 2. Token Counting

Claude Code uses a layered approach to token estimation that avoids expensive API calls for routine accounting. The canonical function for measuring context size is `tokenCountWithEstimation` in `src/utils/tokens.ts`:

```typescript
// src/utils/tokens.ts:226
export function tokenCountWithEstimation(messages: readonly Message[]): number {
  let i = messages.length - 1
  while (i >= 0) {
    const message = messages[i]
    const usage = message ? getTokenUsage(message) : undefined
    if (message && usage) {
      // Walk back past any earlier sibling records split from the same API
      // response (same message.id) so interleaved tool_results between them
      // are included in the estimation slice.
      const responseId = getAssistantMessageId(message)
      if (responseId) {
        let j = i - 1
        while (j >= 0) {
          const prior = messages[j]
          const priorId = prior ? getAssistantMessageId(prior) : undefined
          if (priorId === responseId) {
            i = j
          } else if (priorId !== undefined) {
            break
          }
          j--
        }
      }
      return (
        getTokenCountFromUsage(usage) +
        roughTokenCountEstimationForMessages(messages.slice(i + 1))
      )
    }
    i--
  }
  return roughTokenCountEstimationForMessages(messages)
}
```

The strategy is clever: take the last known-accurate token count from an API response (which includes `input_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`, and `output_tokens`), then estimate the delta for any messages added since that response. This avoids calling the tokenizer API on every turn while staying reasonably accurate.

The rough estimation itself is simple character division in `src/services/tokenEstimation.ts`:

```typescript
// src/services/tokenEstimation.ts:203-208
export function roughTokenCountEstimation(
  content: string,
  bytesPerToken: number = 4,
): number {
  return Math.round(content.length / bytesPerToken)
}
```

Four characters per token is a decent heuristic for English text and code. JSON gets a tighter ratio of 2 bytes per token because of all the single-character structural tokens (`{`, `}`, `:`, `,`, `"`):

```typescript
// src/services/tokenEstimation.ts:215-224
export function bytesPerTokenForFileType(fileExtension: string): number {
  switch (fileExtension) {
    case 'json':
    case 'jsonl':
    case 'jsonc':
      return 2
    default:
      return 4
  }
}
```

Images and PDF documents are estimated at a flat 2000 tokens regardless of actual content -- a conservative constant shared between the estimation and microcompact systems.

The parallel tool call handling in `tokenCountWithEstimation` deserves attention. When the model makes multiple tool calls in a single response, the streaming code splits each content block into a separate `AssistantMessage` record, all sharing the same `message.id`. Tool results get interleaved between these split records. If the function naively grabbed only the last assistant record, it would miss all the interleaved tool results from earlier splits. The walk-back logic (lines 235-250) finds the first sibling with the same response ID so every interleaved tool result is included in the estimate.

There is also a real tokenizer API call path (`countTokensWithAPI` and `countTokensViaHaikuFallback` in `src/services/tokenEstimation.ts`) used for precise counts when needed -- for example, during the `/context` command's analysis. But for the hot path of deciding whether to auto-compact, the estimation approach is sufficient.

## 3. Auto-Compaction Strategy

Auto-compaction is the primary defense against context overflow. The system is defined in `src/services/compact/autoCompact.ts` and operates on a threshold model.

The effective context window is calculated by subtracting reserved output tokens from the model's context window:

```typescript
// src/services/compact/autoCompact.ts:33-49
export function getEffectiveContextWindowSize(model: string): number {
  const reservedTokensForSummary = Math.min(
    getMaxOutputTokensForModel(model),
    MAX_OUTPUT_TOKENS_FOR_SUMMARY,
  )
  let contextWindow = getContextWindowForModel(model, getSdkBetas())
  const autoCompactWindow = process.env.CLAUDE_CODE_AUTO_COMPACT_WINDOW
  if (autoCompactWindow) {
    const parsed = parseInt(autoCompactWindow, 10)
    if (!isNaN(parsed) && parsed > 0) {
      contextWindow = Math.min(contextWindow, parsed)
    }
  }
  return contextWindow - reservedTokensForSummary
}
```

The `MAX_OUTPUT_TOKENS_FOR_SUMMARY` is 20,000 tokens, calibrated against the p99.99 of actual compact summary outputs (17,387 tokens). This reserves headroom for the summary generation itself.

Auto-compaction fires when usage crosses a threshold calculated as the effective window minus a 13K token buffer:

```typescript
// src/services/compact/autoCompact.ts:62-64
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
export const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
export const MANUAL_COMPACT_BUFFER_TOKENS = 3_000
```

For a standard 200K context window model, the math works out to: effective window = 200K - 20K = 180K, auto-compact threshold = 180K - 13K = 167K tokens. So compaction fires around 83% of the raw context window.

The `shouldAutoCompact` function has several guard rails:

```typescript
// src/services/compact/autoCompact.ts:160-238
export async function shouldAutoCompact(
  messages: Message[],
  model: string,
  querySource?: QuerySource,
  snipTokensFreed = 0,
): Promise<boolean> {
  // Recursion guards
  if (querySource === 'session_memory' || querySource === 'compact') {
    return false
  }
  // ... additional guards for context collapse, reactive compact experiments
  const tokenCount = tokenCountWithEstimation(messages) - snipTokensFreed
  const threshold = getAutoCompactThreshold(model)
  // ...
}
```

The recursion guards are critical. Without them, a forked compaction agent could trigger its own compaction, which would trigger another, and so on. The system also has a circuit breaker that stops retrying after 3 consecutive failures:

```typescript
// src/services/compact/autoCompact.ts:68-70
// BQ 2026-03-10: 1,279 sessions had 50+ consecutive failures (up to 3,272)
// in a single session, wasting ~250K API calls/day globally.
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

That comment tells a story. Before the circuit breaker, some sessions were hammering the API with over 3,000 doomed compaction attempts. At scale, that was a quarter million wasted API calls per day.

## 4. Compression Algorithm

The actual compaction is a two-stage process. First, the system tries session memory compaction (a lighter-weight approach). If that is not available or does not apply, it falls back to full conversation compaction.

### Session Memory Compaction

Session memory compaction (`src/services/compact/sessionMemoryCompact.ts`) is the newer, more efficient path. Rather than summarizing the entire conversation, it leverages an ongoing session memory extraction process that has already been capturing key decisions and context throughout the session. It keeps recent messages (configurable, defaults to at least 5 text-block messages and 10K-40K tokens) and replaces older messages with the accumulated session memory content.

### Full Conversation Compaction

The full path sends the entire conversation to a forked agent with a carefully designed prompt. The summary prompt in `src/services/compact/prompt.ts` asks for a 9-section structured summary:

1. Primary Request and Intent
2. Key Technical Concepts
3. Files and Code Sections (with full code snippets)
4. Errors and fixes
5. Problem Solving
6. All user messages (non-tool-result)
7. Pending Tasks
8. Current Work (with emphasis on recency)
9. Optional Next Step

The prompt uses a chain-of-thought pattern with a stripped scratchpad:

```typescript
// src/services/compact/prompt.ts:31-44
const DETAILED_ANALYSIS_INSTRUCTION_BASE = `Before providing your final summary,
wrap your analysis in <analysis> tags to organize your thoughts and ensure you've
covered all necessary points. In your analysis process:

1. Chronologically analyze each message and section of the conversation...
2. Double-check for technical accuracy and completeness...`
```

The `<analysis>` block is a thinking scratchpad that gets stripped from the final summary by `formatCompactSummary`:

```typescript
// src/services/compact/prompt.ts:311-335
export function formatCompactSummary(summary: string): string {
  let formattedSummary = summary
  // Strip analysis section
  formattedSummary = formattedSummary.replace(
    /<analysis>[\s\S]*?<\/analysis>/,
    '',
  )
  // Extract and format summary section
  const summaryMatch = formattedSummary.match(/<summary>([\s\S]*?)<\/summary>/)
  if (summaryMatch) {
    const content = summaryMatch[1] || ''
    formattedSummary = formattedSummary.replace(
      /<summary>[\s\S]*?<\/summary>/,
      `Summary:\n${content.trim()}`,
    )
  }
  return formattedSummary.trim()
}
```

This is a practical trick: the analysis block improves the quality of the summary (forcing the model to reason through the conversation chronologically), but including it in the final compacted context would waste tokens on meta-reasoning.

The prompt aggressively instructs the model not to call any tools:

```typescript
// src/services/compact/prompt.ts:19-26
const NO_TOOLS_PREAMBLE = `CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.
`
```

This is reinforced with a trailer at the end of the prompt. The comment explains why: on Sonnet 4.6+ with adaptive thinking, the model sometimes attempts tool calls despite instructions, and with `maxTurns: 1`, a denied tool call means no text output.

### Partial Compaction

There is also partial compaction, which can operate in two directions: `from` (summarize messages after a pivot point, keep earlier ones) and `up_to` (summarize messages before a pivot, keep later ones). The `from` direction preserves prompt cache for the kept prefix, while `up_to` invalidates it since the summary precedes the kept messages.

### Prompt-Too-Long Retry

If the compaction request itself is too large for the model, the system has a retry mechanism that progressively drops the oldest message groups:

```typescript
// src/services/compact/compact.ts:243-291
export function truncateHeadForPTLRetry(
  messages: Message[],
  ptlResponse: AssistantMessage,
): Message[] | null {
  const groups = groupMessagesByApiRound(input)
  if (groups.length < 2) return null
  const tokenGap = getPromptTooLongTokenGap(ptlResponse)
  let dropCount: number
  if (tokenGap !== undefined) {
    // Drop enough groups to cover the gap
    // ...
  } else {
    dropCount = Math.max(1, Math.floor(groups.length * 0.2))
  }
  dropCount = Math.min(dropCount, groups.length - 1)
  // ...
}
```

Up to 3 retries are allowed, each time dropping more history from the head. If the token gap is unparseable (some Vertex/Bedrock error formats differ), it falls back to dropping 20% of groups per retry.

## 5. Tool Result Management

Tool results are the biggest consumers of context space. A single `cat` of a large file or a broad `grep` search can inject tens of thousands of tokens into the conversation. Claude Code manages this through a multi-layered system called "microcompact."

### Microcompact

The microcompact system (`src/services/compact/microCompact.ts`) targets tool results specifically. It only compacts results from specific tools:

```typescript
// src/services/compact/microCompact.ts:41-50
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,
  ...SHELL_TOOL_NAMES,
  GREP_TOOL_NAME,
  GLOB_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME,
  WEB_FETCH_TOOL_NAME,
  FILE_EDIT_TOOL_NAME,
  FILE_WRITE_TOOL_NAME,
])
```

There are two microcompact paths operating at different layers:

**Cached Microcompact** uses the cache editing API to remove old tool results without invalidating the cached prompt prefix. This is the preferred path when available -- it does not modify local message content but instead uses `cache_reference` and `cache_edits` at the API layer. This preserves the server-side prompt cache, avoiding expensive cache recreation.

**Time-Based Microcompact** fires when there is a significant time gap between assistant messages (indicating the cache has already expired). Since the cache is cold anyway, it directly clears old tool result content in the local messages:

```typescript
// src/services/compact/microCompact.ts:470-492
let tokensSaved = 0
const result: Message[] = messages.map(message => {
  // ...
  const newContent = message.message.content.map(block => {
    if (
      block.type === 'tool_result' &&
      clearSet.has(block.tool_use_id) &&
      block.content !== TIME_BASED_MC_CLEARED_MESSAGE
    ) {
      tokensSaved += calculateToolResultTokens(block)
      touched = true
      return { ...block, content: TIME_BASED_MC_CLEARED_MESSAGE }
    }
    return block
  })
  // ...
})
```

Cleared tool results are replaced with the string `'[Old tool result content cleared]'`. Both paths keep the most recent N tool results intact (configurable, with a floor of 1) so the model always has working context for its most recent actions.

### Image Stripping

Before sending messages for compaction, all image and document blocks are replaced with text markers:

```typescript
// src/services/compact/compact.ts:145-200
export function stripImagesFromMessages(messages: Message[]): Message[] {
  return messages.map(message => {
    // ... replaces image blocks with [image], document blocks with [document]
  })
}
```

This prevents the compaction API call from hitting prompt-too-long limits, especially in sessions where users frequently attach screenshots.

### Token Estimation for Tool Results

Tool result token estimation handles different content types:

```typescript
// src/services/compact/microCompact.ts:138-157
function calculateToolResultTokens(block: ToolResultBlockParam): number {
  if (typeof block.content === 'string') {
    return roughTokenCountEstimation(block.content)
  }
  return block.content.reduce((sum, item) => {
    if (item.type === 'text') {
      return sum + roughTokenCountEstimation(item.text)
    } else if (item.type === 'image' || item.type === 'document') {
      return sum + IMAGE_MAX_TOKEN_SIZE  // 2000
    }
    return sum
  }, 0)
}
```

The `estimateMessageTokens` function in the same file adds a 4/3 padding multiplier to be conservative, since character-based estimation tends to undercount.

## 6. Message Pruning

Beyond compaction, Claude Code has several mechanisms for pruning messages from the conversation.

### Post-Compact Cleanup

After any compaction, `runPostCompactCleanup` (`src/services/compact/postCompactCleanup.ts`) resets a broad set of caches and tracking state:

```typescript
// src/services/compact/postCompactCleanup.ts:31-77
export function runPostCompactCleanup(querySource?: QuerySource): void {
  const isMainThreadCompact =
    querySource === undefined ||
    querySource.startsWith('repl_main_thread') ||
    querySource === 'sdk'

  resetMicrocompactState()
  // Reset context collapse state (main thread only)
  // Clear getUserContext cache and memory file cache
  clearSystemPromptSections()
  clearClassifierApprovals()
  clearSpeculativeChecks()
  clearBetaTracingState()
  clearSessionMessagesCache()
}
```

The function is careful about which state to reset for sub-agents versus the main thread. Sub-agents run in the same process and share module-level state -- resetting the main thread's context collapse state when a sub-agent compacts would corrupt the main conversation.

Notably, invoked skill content is intentionally NOT cleared. The comment explains that skill content must survive across multiple compactions so that skill text can be re-included in subsequent compaction attachments.

### Post-Compact File Restoration

After compaction strips the conversation, the system re-injects the most important file contexts:

```typescript
// src/services/compact/compact.ts:122-131
export const POST_COMPACT_MAX_FILES_TO_RESTORE = 5
export const POST_COMPACT_TOKEN_BUDGET = 50_000
export const POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000
export const POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000
export const POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000
```

Up to 5 recently-read files are re-attached (capped at 5K tokens each, 50K total budget), along with skill content (up to 25K tokens total, 5K per skill), plan mode instructions, deferred tool listings, agent listings, and MCP instructions. This ensures the model does not lose awareness of files it was actively working on.

### Compact Boundary Messages

Every compaction inserts a `SystemCompactBoundaryMessage` that marks where the old conversation ended and the summary begins. This boundary carries metadata including:
- Whether it was auto or manual compaction
- The pre-compact token count
- The UUID of the last pre-compact message
- Pre-compact discovered tool names (so the post-compact schema filter continues sending already-loaded deferred tool schemas)

## 7. The System Reminder Mechanism

System reminders are a mechanism for injecting context that must survive compaction and remain visible to the model regardless of where they appear in the conversation. They use a simple XML wrapper:

```typescript
// src/utils/messages.ts:3097-3099
export function wrapInSystemReminder(content: string): string {
  return `<system-reminder>\n${content}\n</system-reminder>`
}
```

The system prompt explicitly tells the model about these tags:

```typescript
// src/constants/prompts.ts:132-133
`Tool results and user messages may include <system-reminder> tags.
<system-reminder> tags contain useful information and reminders.
They are automatically added by the system, and bear no direct relation
to the specific tool results or user messages in which they appear.`
```

System reminders are used for:
- Memory file content (CLAUDE.md files) injected as attachments
- Hook output messages
- Stop hook blocking errors
- Various system notifications

The `ensureSystemReminderWrap` function in `src/utils/messages.ts` ensures all text content in attachment-origin messages carries the wrapper. This makes the prefix a reliable discriminator for the post-pass "smoosh" operation that combines adjacent system-reminder siblings into a single tool result, reducing the message count without losing information.

Memory files have their own injection budget. Up to 5 files per turn are injected via system-reminder attachments, with a per-file byte cap (enforced by `readFileInRange`'s `truncateOnByteLimit` option) and a line cap of 200 lines. This keeps aggregate injection bounded at roughly 20KB per turn, even when many memory files are relevant.

## 8. Performance Observations

### Token Usage Patterns

The compaction system generates significant API traffic of its own. A full compaction sends the entire conversation (minus images) as input and generates up to 20K tokens of summary output. For a conversation at the 167K auto-compact threshold, this means the compaction call itself consumes roughly 187K tokens. On a $3/MTok input / $15/MTok output model, that is about $0.50-$0.80 per compaction event.

The telemetry events (`tengu_compact`) track both the "compact call total tokens" and the "true post-compact token count" (the actual size of the resulting context). The gap between them reveals how much headroom compaction creates. The system also tracks whether compaction will retrigger on the next turn -- a signal that the summary plus restored attachments already exceed the threshold, indicating a potential compaction loop.

### Prompt Cache Economics

The microcompact system's distinction between cached and time-based paths reflects a deep awareness of prompt cache economics. Cached microcompact uses `cache_edits` to delete tool results server-side without invalidating the cached prefix. This means subsequent API calls still get cache hits on the unchanged prefix, saving significant money at scale. Time-based microcompact only fires when the cache is already cold (the gap since the last assistant message exceeds a threshold), so the cache-invalidation cost is moot.

The compact system itself tries to reuse the main conversation's prompt cache via a forked-agent path. A comment notes an experiment from January 2026 confirming that the alternative (no cache sharing) resulted in 98% cache miss, consuming about 0.76% of fleet-wide cache creation.

### Circuit Breaker Impact

The consecutive failure circuit breaker (max 3 retries) was added after discovering that 1,279 sessions had 50+ consecutive failures in a single session. The worst offender hit 3,272 consecutive failed compaction attempts. This was burning approximately 250K API calls per day globally -- pure waste on sessions where the context was irrecoverably over the limit.

## 9. My Take

Having traced through the full context management pipeline, a few observations stand out.

**What works well:** The layered approach is sound engineering. Microcompact handles the common case (old tool results clogging the context) cheaply and incrementally. Session memory compaction provides a lighter alternative to full summarization. Full compaction is the heavy artillery, reserved for when smaller measures cannot keep up. The circuit breaker and recursion guards show battle-tested production awareness.

**The summary quality problem:** The fundamental weakness of compaction is information loss. No matter how good the summary prompt is (and it is quite thorough with its 9-section structure), compressing a 167K token conversation into a ~15K token summary necessarily loses detail. Specific error messages, exact code snippets, nuanced user feedback -- some of this will be paraphrased or dropped. The system mitigates this by restoring recently-read files and providing a transcript path for recovery, but the model's working memory of the pre-compact conversation is permanently degraded.

**The post-compact token budget is generous but blunt.** Restoring 5 files at 5K tokens each plus 25K of skill content plus delta attachments can easily push the post-compact context to 50-80K tokens. If the auto-compact threshold is 167K and the summary is 15K, the system has about 152K of headroom -- but 50-80K of that immediately goes to restored context. In sessions with many active files and skills, this can create a compaction loop where the system compacts, restores context, immediately exceeds the threshold, and compacts again.

**The analysis scratchpad is a clever hack.** Using `<analysis>` tags as a chain-of-thought scratchpad that gets stripped before the summary enters context is a pragmatic solution. It improves summary quality at the cost of output tokens but does not waste context space. I would be curious to see ablation data on how much the analysis block actually improves summary fidelity.

**What I would change:** I would want to see more aggressive prioritization in tool result management. Rather than keeping the last N tool results uniformly, the system could score tool results by relevance to the current task -- results from files being actively edited are worth more than results from a grep that was just exploration. I would also want compaction to be more incremental -- instead of the binary "keep everything" / "summarize everything" model, a rolling summarization that compresses older turns at progressively coarser granularity would provide more graceful degradation.

The partial compaction feature (compacting only before or after a specific message) is a step in this direction, and the context collapse system (gated behind a feature flag) appears to be exploring exactly this kind of incremental approach. Whether it graduates from experiment to default will probably depend on how well it handles the edge cases that the current all-or-nothing compaction has been battle-tested against.

Overall, the context management system is a sophisticated piece of production engineering that reflects years of real-world usage data and hard-won lessons about failure modes. The code is dense with comments explaining not just what each piece does, but why it exists -- usually referencing specific bugs, metrics, or production incidents. That is the kind of institutional knowledge that makes a system maintainable, and it reveals just how much engineering effort goes into making a "simple" chat interface work reliably at scale.
