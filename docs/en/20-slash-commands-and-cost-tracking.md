# Slash Commands and Cost Tracking

## Opening: 80+ Commands Is a Lot

Opening the `commands.ts` registry file in Claude Code and counting the imports is an exercise in disbelief. There are over 80 static imports at the top of the file, and that is before you add feature-gated conditional imports, plugin commands, skill directory commands, MCP-provided commands, workflow commands, bundled skills, and dynamically discovered skills. The total command surface area in a fully loaded Claude Code session can easily exceed 100 entries.

That is a staggering number for a CLI tool. Most terminal applications top out around 15-20 commands before users start losing track. Claude Code is in a different category entirely -- it is not just a CLI, it is a full development environment that happens to live in your terminal. The slash command system is the primary control surface for everything from session management (`/resume`, `/session`, `/rename`) to cost tracking (`/cost`, `/usage`) to developer workflow (`/commit`, `/review`, `/compact`) to meta-configuration (`/config`, `/permissions`, `/hooks`, `/mcp`).

The central design question this system raises: how to keep 80+ commands discoverable without overwhelming the user? The answer involves a surprisingly sophisticated combination of fuzzy search, feature gating, availability filtering, visibility controls, and a skill system that blurs the line between commands and AI-invocable capabilities. The following sections walk through all of it.

## Command Registry: The Central Nervous System

Everything starts in `src/commands.ts`. This file is the single registration point for every slash command in the system. The architecture has three layers:

**Layer 1: Static imports.** About 80 commands are imported directly at the top of the file. These are the "always available" commands -- things like `/help`, `/compact`, `/cost`, `/clear`, `/model`, `/theme`, etc.

**Layer 2: Feature-gated imports.** A block of conditional `require()` calls loads commands only when specific feature flags are active:

```typescript
// src/commands.ts:62-122
const proactive =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('./commands/proactive.js').default
    : null
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
const forceSnip = feature('HISTORY_SNIP')
  ? require('./commands/force-snip.js').default
  : null
const workflowsCmd = feature('WORKFLOW_SCRIPTS')
  ? (
      require('./commands/workflows/index.js') as typeof import('./commands/workflows/index.js')
    ).default
  : null
```

Note the use of `require()` instead of `import()` here. This is deliberate -- these are synchronous conditional loads that happen at module initialization time. The `feature()` function reads from a compile-time flag system (via `stubs/bun-bundle.js`), enabling dead code elimination in the production build. If `VOICE_MODE` is not enabled in the build, the entire voice command module gets tree-shaken away.

**Layer 3: Dynamic command sources.** The `loadAllCommands()` function merges in commands from five additional sources:

```typescript
// src/commands.ts:449-469
const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => {
  const [
    { skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills },
    pluginCommands,
    workflowCommands,
  ] = await Promise.all([
    getSkills(cwd),
    getPluginCommands(),
    getWorkflowCommands ? getWorkflowCommands(cwd) : Promise.resolve([]),
  ])

  return [
    ...bundledSkills,
    ...builtinPluginSkills,
    ...skillDirCommands,
    ...workflowCommands,
    ...pluginCommands,
    ...pluginSkills,
    ...COMMANDS(),
  ]
})
```

The ordering matters: bundled skills come first, then plugins, then skill directories, then built-in commands. This means a plugin or skill can shadow a built-in command name. The `getCommands()` function that callers actually use applies two more filters on top -- `meetsAvailabilityRequirement()` and `isCommandEnabled()` -- and also merges in dynamically discovered skills that were found during file operations.

The whole thing is memoized by `cwd` using lodash's `memoize`, since loading is expensive (disk I/O, dynamic imports). There is also a `clearCommandsCache()` function that nukes every layer of the memoization hierarchy when commands need to be reloaded.

## Command Interface: The Three Types

Every command in the system conforms to the `Command` type defined in `src/types/command.ts`. The base shape is:

```typescript
// src/types/command.ts:175-203
export type CommandBase = {
  availability?: CommandAvailability[]
  description: string
  hasUserSpecifiedDescription?: boolean
  isEnabled?: () => boolean
  isHidden?: boolean
  name: string
  aliases?: string[]
  isMcp?: boolean
  argumentHint?: string
  whenToUse?: string
  version?: string
  disableModelInvocation?: boolean
  userInvocable?: boolean
  loadedFrom?: 'commands_DEPRECATED' | 'skills' | 'plugin' | 'managed' | 'bundled' | 'mcp'
  kind?: 'workflow'
  immediate?: boolean
  isSensitive?: boolean
  userFacingName?: () => string
}
```

There are three command types, and the distinction is architecturally important:

**`local` commands** execute synchronously in the Node process and return a `LocalCommandResult` -- either text, a compaction result, or a skip signal. These are things like `/cost`, `/compact`, `/clear`. They use lazy loading via a `load()` function that returns a promise of a module with a `call()` function. This keeps the startup cost low -- the actual command implementation is not loaded until the user types the command.

**`local-jsx` commands** render React (Ink) components. They get a richer context including `setMessages`, `canUseTool`, and UI callbacks like `onChangeAPIKey` and `onInstallIDEExtension`. Commands like `/help`, `/skills`, `/config`, `/mcp`, `/resume` are local-jsx because they need to render interactive UI elements.

**`prompt` commands** are the most interesting. They do not execute locally at all -- they generate content blocks that get injected into the conversation as a prompt to the model. This is how skills work: `/review` does not review code itself, it constructs a detailed prompt asking Claude to review code. The `PromptCommand` type has additional fields like `progressMessage`, `contentLength` (for token estimation), `allowedTools`, `model` override, and `context` ('inline' vs 'fork' for sub-agent execution).

The `source` field on prompt commands tracks provenance: `'builtin'`, `'mcp'`, `'plugin'`, `'bundled'`, or a `SettingSource` like `'userSettings'`, `'projectSettings'`, `'policySettings'`. This is displayed in the typeahead UI so users know where a command came from.

## Fuzzy Search: Fuse.js Does the Heavy Lifting

With 80+ commands, exact-match lookup would be unusable. Claude Code uses Fuse.js for fuzzy matching, implemented in `src/utils/suggestions/commandSuggestions.ts`.

The Fuse index is built once per command array identity (the array is memoized in the REPL) and cached:

```typescript
// src/utils/suggestions/commandSuggestions.ts:53-80
const fuse = new Fuse(commandData, {
  includeScore: true,
  threshold: 0.3, // relatively strict matching
  location: 0, // prefer matches at the beginning of strings
  distance: 100, // increased to allow matching in descriptions
  keys: [
    {
      name: 'commandName',
      weight: 3, // Highest priority for command names
    },
    {
      name: 'partKey',
      weight: 2, // Next highest priority for command parts
    },
    {
      name: 'aliasKey',
      weight: 2, // Same high priority for aliases
    },
    {
      name: 'descriptionKey',
      weight: 0.5, // Lower priority for descriptions
    },
  ],
})
```

The weight distribution tells you about the design intent: command names are 6x more important than description words. The `partKey` field is clever -- it splits hyphenated command names like `install-github-app` into `['install', 'github', 'app']` so typing "github" matches even though it is in the middle of the name.

The sorting after fuzzy search applies a strict priority cascade: exact name match > exact alias match > prefix name match > prefix alias match > fuzzy score with usage frequency as tiebreaker. There is also logic to handle hidden commands -- if a user types the exact name of a hidden command, it gets prepended to the results even though Fuse's index excluded it.

When the user just types `/` with no query, the system shows a categorized list: recently used skills (top 5 by usage score), then built-in commands, then user settings commands, then project commands, then policy commands, then everything else. Each category is sorted alphabetically.

The system also supports mid-input slash commands -- typing text then ` /com` in the middle of a message triggers the suggestion system for that embedded command reference. The `findMidInputSlashCommand()` function handles this by looking backwards from the cursor position for a `/` preceded by whitespace.

## Feature-Gated Commands: Three Layers of Hiding

Claude Code uses three separate mechanisms to control command visibility, and understanding when each applies is critical:

**`isEnabled()`** is a function that returns a boolean. It is re-evaluated on every `getCommands()` call. Commands that fail this check are completely invisible -- not just hidden, but removed from the command list entirely. Examples: `/compact` checks `!isEnvTruthy(process.env.DISABLE_COMPACT)`, `/voice` checks `isVoiceGrowthBookEnabled()`, `/fast` checks `isFastModeEnabled()`.

**`isHidden`** is a property (often a getter) that controls typeahead visibility. Hidden commands still work if you type the exact name -- they just do not appear in suggestions. This is used for internal/debug commands like `/heapdump` (`isHidden: true`) and commands whose visibility is conditional, like `/cost` which hides itself for Claude.ai subscribers:

```typescript
// src/commands/cost/index.ts:12-17
get isHidden() {
  if (process.env.USER_TYPE === 'ant') {
    return false
  }
  return isClaudeAISubscriber()
},
```

**`availability`** is a static declaration of which auth/provider environments support the command. It uses a `CommandAvailability` type with values `'claude-ai'` and `'console'`. The `meetsAvailabilityRequirement()` function in `commands.ts` checks this before any command appears in the list. This is separate from `isEnabled()` because availability is about who you are (auth context), while enablement is about what is turned on (feature flags, env vars).

There is also a fourth layer specific to internal development: the `INTERNAL_ONLY_COMMANDS` array in `commands.ts` contains about 20 commands that only load when `process.env.USER_TYPE === 'ant'` and `!process.env.IS_DEMO`. These include `/commit`, `/commit-push-pr`, `/issue`, `/mock-limits`, `/share`, and various debugging tools.

## Skills as Commands: The Blurry Line

One of the most architecturally interesting decisions in Claude Code is that skills and slash commands share the same `Command` type. A skill is just a prompt-type command with some extra metadata. The system loads skills from multiple sources:

1. **Bundled skills** (`src/skills/bundledSkills.ts`) are compiled into the CLI. They are registered via `registerBundledSkill()` which wraps a `BundledSkillDefinition` into a full `Command` object with `loadedFrom: 'bundled'` and `source: 'bundled'`.

2. **Skill directory skills** are markdown files loaded from `~/.claude/skills/`, `.claude/skills/` (project-level), and managed settings paths. The `loadSkillsDir.ts` module reads these files, parses frontmatter for metadata (description, arguments, model override, allowed tools, hooks), and wraps them as prompt commands.

3. **Plugin skills** come from installed plugins and get `source: 'plugin'` with plugin manifest metadata attached.

4. **MCP skills** come from MCP servers and get `loadedFrom: 'mcp'`.

5. **Dynamic skills** are discovered during file operations when the model touches files matching a skill's `paths` glob patterns. These are tracked in state and merged in at `getCommands()` time.

The `getSkillToolCommands()` function filters the full command list to just the prompt-type commands that the model can invoke via the Skill tool. The filtering logic is nuanced -- it excludes `source: 'builtin'` commands (those are user-facing, not model-facing) and requires either a user-specified description or a `whenToUse` field for plugin/MCP commands. Bundled skills and skill directory skills always pass because they get auto-derived descriptions.

The `disableModelInvocation` flag lets you create skills that only the user can invoke (via typing `/skill-name`), while `userInvocable` controls the reverse -- whether users can invoke a skill that was designed for the model.

## Cost Tracking Architecture: Every Token Counted

The cost tracking system is split across three files: `src/cost-tracker.ts` for the high-level API, `src/utils/modelCost.ts` for per-model pricing calculations, and `src/bootstrap/state.ts` for the actual state storage.

The global state in `bootstrap/state.ts` tracks:
- `totalCostUSD` -- running USD total
- `totalAPIDuration` / `totalAPIDurationWithoutRetries` -- time spent waiting for API responses
- `totalToolDuration` -- time spent executing tools
- `totalLinesAdded` / `totalLinesRemoved` -- code change metrics
- `modelUsage` -- a `{ [modelName: string]: ModelUsage }` map with per-model token breakdowns
- OpenTelemetry counters for cost and tokens (attributed by model and speed tier)

The `ModelUsage` type tracks seven dimensions per model: input tokens, output tokens, cache read tokens, cache creation tokens, web search requests, cumulative USD cost, context window size, and max output tokens.

The core cost accumulation function is `addToTotalSessionCost()` in `cost-tracker.ts`:

```typescript
// src/cost-tracker.ts:278-323
export function addToTotalSessionCost(
  cost: number,
  usage: Usage,
  model: string,
): number {
  const modelUsage = addToTotalModelUsage(cost, usage, model)
  addToTotalCostState(cost, modelUsage, model)

  const attrs =
    isFastModeEnabled() && usage.speed === 'fast'
      ? { model, speed: 'fast' }
      : { model }

  getCostCounter()?.add(cost, attrs)
  getTokenCounter()?.add(usage.input_tokens, { ...attrs, type: 'input' })
  getTokenCounter()?.add(usage.output_tokens, { ...attrs, type: 'output' })
  // ...cache tokens...

  let totalCost = cost
  for (const advisorUsage of getAdvisorUsage(usage)) {
    const advisorCost = calculateUSDCost(advisorUsage.model, advisorUsage)
    totalCost += addToTotalSessionCost(advisorCost, advisorUsage, advisorUsage.model)
  }
  return totalCost
}
```

Notice the recursive call for "advisor" usage -- this handles cases where the primary model delegates to a secondary model (the advisor system), ensuring those costs are tracked separately but included in the total. Each advisor call gets its own analytics event with `tengu_advisor_tool_token_usage`.

The USD conversion in `src/utils/modelCost.ts` maintains a lookup table of per-model pricing tiers:

```typescript
// src/utils/modelCost.ts:36-42
export const COST_TIER_3_15 = {
  inputTokens: 3,
  outputTokens: 15,
  promptCacheWriteTokens: 3.75,
  promptCacheReadTokens: 0.3,
  webSearchRequests: 0.01,
} as const satisfies ModelCosts
```

These are prices per million tokens. The system defines tiers like `COST_TIER_3_15` (Sonnet-class: $3/$15), `COST_TIER_15_75` (Opus 4/4.1: $15/$75), `COST_TIER_5_25` (Opus 4.5/4.6: $5/$25), and `COST_TIER_30_150` (Opus 4.6 fast mode: $30/$150). When an unknown model is encountered, the system falls back to the default main loop model's pricing and sets a `hasUnknownModelCost` flag that adds a warning to the cost display.

The fast mode pricing for Opus 4.6 is particularly interesting -- the `getModelCosts()` function checks `usage.speed === 'fast'` on the API response to dynamically select between the $5/$25 and $30/$150 tiers for the same model.

## Cost Persistence: Surviving Session Boundaries

Cost state is ephemeral by default -- it lives in the in-memory `State` object in `bootstrap/state.ts`. But Claude Code needs costs to survive across session resumes. The persistence mechanism uses the project config file (`.claude/config.json`).

`saveCurrentSessionCosts()` serializes the entire cost state:

```typescript
// src/cost-tracker.ts:143-175
export function saveCurrentSessionCosts(fpsMetrics?: FpsMetrics): void {
  saveCurrentProjectConfig(current => ({
    ...current,
    lastCost: getTotalCostUSD(),
    lastAPIDuration: getTotalAPIDuration(),
    // ... all the metrics ...
    lastModelUsage: Object.fromEntries(
      Object.entries(getModelUsage()).map(([model, usage]) => [
        model,
        {
          inputTokens: usage.inputTokens,
          outputTokens: usage.outputTokens,
          // ... stripped of runtime-only fields ...
        },
      ]),
    ),
    lastSessionId: getSessionId(),
  }))
}
```

The restore path in `restoreCostStateForSession()` only restores if the session ID matches -- you do not accidentally inherit costs from a different session. When restoring, it re-derives `contextWindow` and `maxOutputTokens` from the current model configs rather than trusting stale values from disk.

The `useCostSummary()` React hook in `src/costHook.ts` wires this into the process lifecycle:

```typescript
// src/costHook.ts:6-22
export function useCostSummary(
  getFpsMetrics?: () => FpsMetrics | undefined,
): void {
  useEffect(() => {
    const f = () => {
      if (hasConsoleBillingAccess()) {
        process.stdout.write('\n' + formatTotalCost() + '\n')
      }
      saveCurrentSessionCosts(getFpsMetrics?.())
    }
    process.on('exit', f)
    return () => { process.off('exit', f) }
  }, [])
}
```

On process exit, it saves costs to disk and optionally prints the summary. The FPS metrics piggyback on this same save -- a clever reuse of the exit hook.

## Billing Integration: Who Sees What

The billing logic in `src/utils/billing.ts` determines who gets to see cost information. There are two separate functions for two different billing contexts:

`hasConsoleBillingAccess()` checks if you are a direct Anthropic Console API user (not Claude.ai, not third-party) with admin or billing roles at the org or workspace level. If `DISABLE_COST_WARNINGS` is set, it returns false. This gates the exit-time cost summary.

`hasClaudeAiBillingAccess()` checks if you are a Claude.ai subscriber with appropriate permissions. Consumer plans (Max/Pro) always have access. Team/Enterprise plans require admin, billing, owner, or primary_owner org roles.

The `/cost` command itself has another layer of logic. For Claude.ai subscribers, it shows subscription status rather than dollar amounts (since they are on a subscription, not pay-per-token). But there is an escape hatch for internal users:

```typescript
// src/commands/cost/cost.ts:18-19
if (process.env.USER_TYPE === 'ant') {
  value += `\n\n[ANT-ONLY] Showing cost anyway:\n ${formatTotalCost()}`
}
```

The formatted cost display aggregates usage by canonical model name (so `claude-sonnet-4-20250514` and any alias resolve to the same line), shows per-model breakdowns with input/output/cache-read/cache-write token counts, and includes web search request counts when non-zero.

## Slash Command Parsing: Simpler Than You Would Think

Given all the complexity above, the actual parsing of slash commands is refreshingly simple. `src/utils/slashCommandParsing.ts` is a 60-line file:

```typescript
// src/utils/slashCommandParsing.ts:25-59
export function parseSlashCommand(input: string): ParsedSlashCommand | null {
  const trimmedInput = input.trim()
  if (!trimmedInput.startsWith('/')) {
    return null
  }

  const withoutSlash = trimmedInput.slice(1)
  const words = withoutSlash.split(' ')

  if (!words[0]) {
    return null
  }

  let commandName = words[0]
  let isMcp = false
  let argsStartIndex = 1

  // Check for MCP commands (second word is '(MCP)')
  if (words.length > 1 && words[1] === '(MCP)') {
    commandName = commandName + ' (MCP)'
    isMcp = true
    argsStartIndex = 2
  }

  const args = words.slice(argsStartIndex).join(' ')
  return { commandName, args, isMcp }
}
```

Split on spaces, first word is the command, rest is args. MCP commands get special handling because they carry a `(MCP)` suffix marker. That is it. The sophistication lives in the lookup and fuzzy matching, not the parsing.

The `findCommand()` function in `commands.ts` does a simple linear scan checking name, user-facing name, and aliases:

```typescript
// src/commands.ts:688-698
export function findCommand(
  commandName: string,
  commands: Command[],
): Command | undefined {
  return commands.find(
    _ =>
      _.name === commandName ||
      getCommandName(_) === commandName ||
      _.aliases?.includes(commandName),
  )
}
```

No index, no hash map -- just `.find()`. With ~100 commands this is fine. The Fuse.js index handles the performance-sensitive typeahead path; the final lookup after selection is a one-time linear scan.

## Bridge and Remote Safety

One detail worth calling out: not all commands are safe to run in all contexts. Two sets -- `REMOTE_SAFE_COMMANDS` and `BRIDGE_SAFE_COMMANDS` -- whitelist commands for remote/mobile usage. The bridge safety predicate is type-aware:

```typescript
// src/commands.ts:672-676
export function isBridgeSafeCommand(cmd: Command): boolean {
  if (cmd.type === 'local-jsx') return false   // renders Ink UI
  if (cmd.type === 'prompt') return true        // expands to text
  return BRIDGE_SAFE_COMMANDS.has(cmd)          // explicit allowlist for 'local'
}
```

This exists because someone once tried to run `/model` from their phone, which tried to pop an interactive Ink picker on the server terminal. The fix was a principled type-based safety system: prompt commands are safe by construction (they produce text), JSX commands are always blocked (they render UI), and local commands need explicit opt-in.

