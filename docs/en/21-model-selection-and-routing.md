# Model Selection and Routing System

## Overview

Claude Code implements a sophisticated model selection and routing system spanning approximately 15 TypeScript modules under `src/utils/model/`. The system addresses a fundamental challenge: mapping abstract user intent ("use the best model" or "use Opus") to a concrete, provider-specific model identifier string that varies across four distinct API providers, multiple subscription tiers, and evolving model generations. This document provides a technical analysis of each subsystem.

---

## 1. Model Registry and Aliases

### The Config Registry

At the foundation of the model system lies `configs.ts`, which defines every known model as a `ModelConfig` object mapping each API provider to its provider-specific model identifier string:

```typescript
// src/utils/model/configs.ts:79-84
export const CLAUDE_SONNET_4_6_CONFIG = {
  firstParty: 'claude-sonnet-4-6',
  bedrock: 'us.anthropic.claude-sonnet-4-6',
  vertex: 'claude-sonnet-4-6',
  foundry: 'claude-sonnet-4-6',
} as const satisfies ModelConfig
```

All configs are collected into the `ALL_MODEL_CONFIGS` registry (configs.ts:87-99), keyed by short internal identifiers (`haiku35`, `sonnet46`, `opus46`, etc.). This registry is the single source of truth from which provider-specific model strings are derived at runtime.

A reverse mapping `CANONICAL_ID_TO_KEY` (configs.ts:113-118) maps canonical first-party model IDs back to their internal keys, enabling settings-based `modelOverrides` to replace any model string with an arbitrary provider-specific value such as a Bedrock ARN.

### Model Aliases

The alias system (`aliases.ts`) defines a compact set of user-facing model references:

```typescript
// src/utils/model/aliases.ts:1-9
export const MODEL_ALIASES = [
  'sonnet', 'opus', 'haiku', 'best',
  'sonnet[1m]', 'opus[1m]', 'opusplan',
] as const
```

These aliases serve as stable entry points that survive model version upgrades. When a user sets their model to `"sonnet"`, the `parseUserSpecifiedModel()` function in `model.ts` resolves it to the current default Sonnet model string (e.g., `claude-sonnet-4-6` on first-party, `us.anthropic.claude-sonnet-4-5-20250929-v1:0` on Bedrock). This decoupling means that upgrading the default Sonnet from 4.5 to 4.6 requires no user action -- the alias automatically resolves to the newer version.

A distinct `MODEL_FAMILY_ALIASES` set (`['sonnet', 'opus', 'haiku']`) provides wildcard matching semantics for the model allowlist system.

### Canonical Name Resolution

The function `firstPartyNameToCanonical()` (model.ts:217-270) strips dates and provider suffixes from any model identifier, reducing it to a family-level canonical form. For example, both `claude-sonnet-4-5-20250929` and `us.anthropic.claude-sonnet-4-5-20250929-v1:0` resolve to `claude-sonnet-4-5`. The matching is done via ordered substring checks with more specific versions checked first (4-6 before 4-5, 4-5 before bare 4) to avoid false matches.

The higher-level `getCanonicalName()` (model.ts:279-283) first resolves any model overrides (e.g., Bedrock ARNs mapped back to canonical IDs via `resolveOverriddenModel()`), then applies canonical name extraction. This two-step resolution ensures that custom deployment identifiers are correctly mapped back to their pricing and capability tiers.

### Model Selection Priority Chain

The primary model resolution function `getMainLoopModel()` (model.ts:92-98) implements a five-level priority chain:

```typescript
// src/utils/model/model.ts:80-98
// Model Selection Priority Order:
// 1. Model override during session (from /model command)
// 2. Model override at startup (from --model flag)
// 3. ANTHROPIC_MODEL environment variable
// 4. Settings (from user's saved settings)
// 5. Built-in default
```

The built-in default itself varies by subscription tier. `getDefaultMainLoopModelSetting()` (model.ts:178-200) returns Opus 4.6 for Max and Team Premium subscribers, and Sonnet 4.6 for all other users (Pro, Team Standard, Enterprise, PAYG). The `[1m]` suffix is conditionally appended based on whether the Opus 1M merge feature is enabled.

---

## 2. Multi-Provider Architecture

### Provider Detection

The `providers.ts` module implements a simple environment-variable-based provider detection:

```typescript
// src/utils/model/providers.ts:6-14
export function getAPIProvider(): APIProvider {
  return isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)
    ? 'bedrock'
    : isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX)
      ? 'vertex'
      : isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY)
        ? 'foundry'
        : 'firstParty'
}
```

The four providers (`firstParty`, `bedrock`, `vertex`, `foundry`) form a discriminated union type `APIProvider` used throughout the codebase for provider-conditional logic.

### Model String Resolution

The `modelStrings.ts` module bridges the static config registry and runtime provider context. On initialization, it calls `getBuiltinModelStrings(provider)` to extract the appropriate column from each `ModelConfig`. For Bedrock specifically, the system performs an asynchronous API call (`getBedrockInferenceProfiles()` in `bedrock.ts:7-41`) to list available cross-region inference profiles, then matches each model config against available profiles:

```typescript
// src/utils/model/modelStrings.ts:47-54
const out = {} as ModelStrings
for (const key of MODEL_KEYS) {
  const needle = ALL_MODEL_CONFIGS[key].firstParty
  out[key] = findFirstMatch(profiles, needle) || fallback[key]
}
```

This enables automatic discovery of region-specific Bedrock model strings (e.g., `eu.anthropic.claude-opus-4-6-v1` instead of the default `us.anthropic.claude-opus-4-6-v1`).

### Bedrock Region Prefix Handling

Bedrock cross-region inference profiles use region prefixes (`us`, `eu`, `apac`, `global`) prepended to the model identifier. The `bedrock.ts` module provides utilities for extracting (`getBedrockRegionPrefix()`), applying (`applyBedrockRegionPrefix()`), and managing these prefixes. Crucially, when subagents are spawned, the parent model's region prefix is inherited to ensure IAM permission consistency:

```typescript
// src/utils/model/agent.ts:58-67
const applyParentRegionPrefix = (
  resolvedModel: string,
  originalSpec: string,
): string => {
  if (parentRegionPrefix && getAPIProvider() === 'bedrock') {
    if (getBedrockRegionPrefix(originalSpec)) return resolvedModel
    return applyBedrockRegionPrefix(resolvedModel, parentRegionPrefix)
  }
  return resolvedModel
}
```

If the original model specification already carries its own region prefix, the system preserves it to avoid silent data-residency violations.

### Model Overrides via Settings

Users can define `modelOverrides` in their settings, keyed by canonical first-party model ID and mapping to arbitrary provider-specific strings (typically Bedrock inference profile ARNs). The `applyModelOverrides()` function in `modelStrings.ts:63-76` layers these overrides on top of the provider-derived model strings. The reverse mapping (`resolveOverriddenModel()`) ensures that overridden model IDs can be resolved back to canonical names for pricing and capability lookups.

### Provider-Specific Model Defaults

Different providers receive different default model versions due to availability lag. The `getDefaultSonnetModel()` function (model.ts:119-128) illustrates this:

```typescript
// src/utils/model/model.ts:119-128
export function getDefaultSonnetModel(): ModelName {
  if (process.env.ANTHROPIC_DEFAULT_SONNET_MODEL) {
    return process.env.ANTHROPIC_DEFAULT_SONNET_MODEL
  }
  if (getAPIProvider() !== 'firstParty') {
    return getModelStrings().sonnet45  // 3P gets older stable version
  }
  return getModelStrings().sonnet46    // 1P gets latest
}
```

This pattern -- first-party providers receiving the latest model version while third-party providers default to the previous stable version -- repeats across all model families. Environment variables (`ANTHROPIC_DEFAULT_SONNET_MODEL`, `ANTHROPIC_DEFAULT_OPUS_MODEL`, `ANTHROPIC_DEFAULT_HAIKU_MODEL`) allow administrators to pin specific versions regardless of provider.

---

## 3. Model Capabilities

### Dynamic Capability Discovery

The `modelCapabilities.ts` module implements a cached capability discovery system. For eligible users (internal, first-party API), it fetches the model list from the Anthropic API and caches the results to `~/.claude/cache/model-capabilities.json`:

```typescript
// src/utils/model/modelCapabilities.ts:75-83
export function getModelCapability(model: string): ModelCapability | undefined {
  if (!isModelCapabilitiesEligible()) return undefined
  const cached = loadCache(getCachePath())
  if (!cached || cached.length === 0) return undefined
  const m = model.toLowerCase()
  const exact = cached.find(c => c.id.toLowerCase() === m)
  if (exact) return exact
  return cached.find(c => m.includes(c.id.toLowerCase()))
}
```

The cache is sorted by ID length (longest first) to prefer the most specific match when using substring matching. Each capability record contains `id`, `max_input_tokens`, and `max_tokens`, which override the static defaults for context window and output token calculations.

### Context Window Determination

Context window size follows a multi-layered resolution in `context.ts`:

1. **Environment override** (`CLAUDE_CODE_MAX_CONTEXT_TOKENS`) -- ant-only, caps effective context
2. **`[1m]` suffix** -- explicit opt-in to 1M token context window
3. **Dynamic capability** -- from cached API response (`modelCapabilities.ts`)
4. **Beta header** -- server-side 1M beta flag
5. **Experiment treatment** -- GrowthBook-controlled Sonnet 1M experiment
6. **Static default** -- 200,000 tokens

### Output Token Limits

The `getModelMaxOutputTokens()` function (context.ts:149-210) returns model-specific default and upper-limit output token counts:

| Model Family | Default | Upper Limit |
|---|---|---|
| Opus 4.6 | 64,000 | 128,000 |
| Sonnet 4.6 | 32,000 | 128,000 |
| Opus 4.5 / Sonnet 4.x / Haiku 4.x | 32,000 | 64,000 |
| Opus 4/4.1 | 32,000 | 32,000 |
| Claude 3.7 Sonnet | 32,000 | 64,000 |
| Claude 3.5 Sonnet/Haiku | 8,192 | 8,192 |
| Claude 3 Opus/Haiku | 4,096 | 4,096 |

A slot-reservation optimization caps the default to 8,000 tokens (`CAPPED_DEFAULT_MAX_TOKENS`) with an escalation path to 64,000 tokens on retry, since the p99 output in production is approximately 4,911 tokens.

### Feature Detection Per Model

Thinking support, interleaved streaming protocol (ISP), adaptive thinking, structured outputs, web search, and context management are all gated per model and per provider. The pattern is consistent: check a 3P capability override first, then apply known allowlists/denylists, then fall back to provider-specific defaults. For example, from `thinking.ts`:

```typescript
// src/utils/thinking.ts:102-110
const canonical = getCanonicalName(model)
const provider = getAPIProvider()
// 1P and Foundry: all Claude 4+ models
if (provider === 'foundry' || provider === 'firstParty') {
  return !canonical.includes('claude-3-')
}
// 3P (Bedrock/Vertex): only Opus 4+ and Sonnet 4+
return canonical.includes('sonnet-4') || canonical.includes('opus-4')
```

### 3P Capability Overrides

The `modelSupportOverrides.ts` module allows third-party providers to declare model capabilities via environment variables. Each model tier (opus, sonnet, haiku) has a paired `ANTHROPIC_DEFAULT_*_MODEL_SUPPORTED_CAPABILITIES` environment variable containing a comma-separated list of capability strings (`effort`, `max_effort`, `thinking`, `adaptive_thinking`, `interleaved_thinking`). When a model string matches a pinned tier, the override takes precedence over all static detection.

---

## 4. Agent Model Selection

### Subagent Model Resolution

The `agent.ts` module handles model selection for subagents (tool-invoked child conversations). The resolution follows this priority:

1. **`CLAUDE_CODE_SUBAGENT_MODEL` env var** -- global override for all subagents
2. **Tool-specified model** -- the tool's `model` frontmatter (e.g., `model: opus`)
3. **Agent-configured model** -- the agent's `model` field from configuration
4. **Default: `inherit`** -- use the same model as the parent thread

### Tier Matching Optimization

A key optimization prevents surprising downgrades when an alias matches the parent's tier. The `aliasMatchesParentTier()` function (agent.ts:110-122) checks whether a bare family alias (`opus`, `sonnet`, `haiku`) matches the parent model's canonical family:

```typescript
// src/utils/model/agent.ts:110-122
function aliasMatchesParentTier(alias: string, parentModel: string): boolean {
  const canonical = getCanonicalName(parentModel)
  switch (alias.toLowerCase()) {
    case 'opus':
      return canonical.includes('opus')
    case 'sonnet':
      return canonical.includes('sonnet')
    case 'haiku':
      return canonical.includes('haiku')
    default:
      return false
  }
}
```

When a match occurs, the subagent inherits the parent's exact model string rather than resolving the alias through `getDefaultOpusModel()`, which could resolve to a different version on 3P providers. This prevents a Vertex user on Opus 4.6 from having subagents silently downgraded to whatever the 3P default happens to be.

### Skill Model Override Resolution

The `resolveSkillModelOverride()` function (model.ts:523-536) handles a subtle interaction between skill frontmatter and 1M context windows. When a user is on `opus[1m]` at 230K tokens and invokes a skill with `model: opus`, the bare alias would normally drop the effective context window from 1M to 200K, triggering premature auto-compact. The function carries the `[1m]` suffix forward when the target model family supports it:

```typescript
// src/utils/model/model.ts:523-536
export function resolveSkillModelOverride(
  skillModel: string,
  currentModel: string,
): string {
  if (has1mContext(skillModel) || !has1mContext(currentModel)) {
    return skillModel
  }
  if (modelSupports1M(parseUserSpecifiedModel(skillModel))) {
    return skillModel + '[1m]'
  }
  return skillModel
}
```

### Runtime Model Switching

The `getRuntimeMainLoopModel()` function (model.ts:145-167) applies context-dependent model switching at runtime. The `opusplan` alias uses Opus for plan mode and Sonnet for execution mode. The `haiku` alias upgrades to Sonnet for plan mode to ensure adequate reasoning capability:

```typescript
// src/utils/model/model.ts:152-166
if (
  getUserSpecifiedModelSetting() === 'opusplan' &&
  permissionMode === 'plan' &&
  !exceeds200kTokens
) {
  return getDefaultOpusModel()
}
if (getUserSpecifiedModelSetting() === 'haiku' && permissionMode === 'plan') {
  return getDefaultSonnetModel()
}
```

---

## 5. Deprecation and Migration

### Deprecation Registry

The `deprecation.ts` module maintains a registry of deprecated models with per-provider retirement dates:

```typescript
// src/utils/model/deprecation.ts:33-61
const DEPRECATED_MODELS: Record<string, DeprecationEntry> = {
  'claude-3-opus': {
    modelName: 'Claude 3 Opus',
    retirementDates: {
      firstParty: 'January 5, 2026',
      bedrock: 'January 15, 2026',
      vertex: 'January 5, 2026',
      foundry: 'January 5, 2026',
    },
  },
  'claude-3-7-sonnet': {
    modelName: 'Claude 3.7 Sonnet',
    retirementDates: {
      firstParty: 'February 19, 2026',
      bedrock: 'April 28, 2026',
      vertex: 'May 11, 2026',
      foundry: 'February 19, 2026',
    },
  },
  // ...
}
```

Retirement dates differ by provider, reflecting the staggered availability timelines of Bedrock and Vertex. Models are matched via case-insensitive substring matching against the model ID. The `getModelDeprecationWarning()` function returns a formatted warning message or `null` for non-deprecated models.

### Legacy Model Remapping

Opus 4.0 and 4.1 are no longer available on the first-party API. The system silently remaps these to the current Opus default (model.ts:477-483):

```typescript
// src/utils/model/model.ts:538-543
const LEGACY_OPUS_FIRSTPARTY = [
  'claude-opus-4-20250514',
  'claude-opus-4-1-20250805',
  'claude-opus-4-0',
  'claude-opus-4-1',
]
```

This remapping applies only on first-party (since 3P providers may still serve these models) and can be disabled via `CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP=1`.

### 3P Fallback Suggestions

When model validation fails on a third-party provider (typically a 404 indicating the model does not exist), the `get3PFallbackSuggestion()` function (validateModel.ts:144-159) suggests the previous generation:

| Requested | Suggestion |
|---|---|
| opus-4-6 | Opus 4.1 |
| sonnet-4-6 | Sonnet 4.5 |
| sonnet-4-5 | Sonnet 4.0 |

### Model Family Upgrade Hints

The model picker detects when a user has pinned a specific older version and a newer one is available via the alias. The `getModelFamilyInfo()` function (modelOptions.ts:385-424) maps a full model name to its family alias and the marketing name of the version the alias currently resolves to. When these differ, the UI shows "Newer version available -- select Sonnet for Sonnet 4.6."

---

## 6. Pricing Tiers

### Cost Table

The `modelCost.ts` module defines per-model pricing tiers in dollars per million tokens:

```typescript
// src/utils/modelCost.ts:35-87
// Sonnet family (all versions): $3 input / $15 output
export const COST_TIER_3_15 = {
  inputTokens: 3, outputTokens: 15,
  promptCacheWriteTokens: 3.75, promptCacheReadTokens: 0.3,
  webSearchRequests: 0.01,
}

// Opus 4/4.1: $15 input / $75 output
export const COST_TIER_15_75 = { ... }

// Opus 4.5 and Opus 4.6 (standard): $5 input / $25 output
export const COST_TIER_5_25 = { ... }

// Opus 4.6 fast mode: $30 input / $150 output
export const COST_TIER_30_150 = { ... }

// Haiku 3.5: $0.80 input / $4 output
export const COST_HAIKU_35 = { ... }

// Haiku 4.5: $1 input / $5 output
export const COST_HAIKU_45 = { ... }
```

The `MODEL_COSTS` record (modelCost.ts:104-126) maps canonical model short names to their cost tiers. All Sonnet variants (3.5, 3.7, 4.0, 4.5, 4.6) share the same `$3/$15` tier.

### Opus 4.6 Dynamic Pricing

Opus 4.6 uniquely has two pricing tiers depending on whether fast mode was active for a given request. The `getModelCosts()` function (modelCost.ts:144-164) checks the `usage.speed` field returned by the API:

```typescript
// src/utils/modelCost.ts:148-153
if (
  shortName === firstPartyNameToCanonical(CLAUDE_OPUS_4_6_CONFIG.firstParty)
) {
  const isFastMode = usage.speed === 'fast'
  return getOpus46CostTier(isFastMode)
}
```

Standard Opus 4.6 requests cost `$5/$25` per Mtok; fast mode requests cost `$30/$150` per Mtok -- a 6x multiplier.

### Unknown Model Fallback

When a model is not found in `MODEL_COSTS`, the system falls back to the default main loop model's cost tier, or `COST_TIER_5_25` as the final fallback. An analytics event (`tengu_unknown_model_cost`) is logged to track unrecognized models.

---

## 7. Model-Specific Behavioral Configuration

### Beta Header Assembly

Rather than applying prompt text patches, Claude Code adjusts model behavior through API beta headers assembled per-model in `betas.ts`. The `getAllModelBetas()` function (betas.ts:234-369) constructs a model-specific array of beta headers:

- **Interleaved thinking** (`interleaved-thinking-2025-04-14`): enabled for all ISP-supporting models unless `DISABLE_INTERLEAVED_THINKING` is set
- **Thinking redaction** (`redact-thinking-2025-06-01`): enabled for interactive (non-SDK) sessions on first-party/Foundry to suppress thinking summaries
- **Context management** (`context-management-2025-06-01`): enabled for Claude 4+ models on first-party/Foundry
- **Structured outputs** (`structured-outputs-2025-07-31`): enabled for specific models when the strict tools experiment gate passes
- **Token-efficient tools**: mutually exclusive with structured outputs, enabled for internal users via GrowthBook
- **Web search** (`web-search-2025-03-05`): provider-dependent (Vertex requires Claude 4+, Foundry always enabled)
- **1M context** (`context-window-1m-2025-06-01`): enabled when the model string contains `[1m]`
- **Prompt caching scope**: always sent on first-party/Foundry (no-op without a scope field)

Bedrock receives special handling: certain headers that Bedrock passes as extra body parameters (rather than HTTP headers) are separated out by `getBedrockExtraBodyParamsBetas()`.

### Thinking Configuration

The thinking system (`thinking.ts`) configures three modes: `adaptive` (model decides thinking depth), `enabled` with a fixed budget, or `disabled`. Adaptive thinking is supported only by Opus 4.6 and Sonnet 4.6 models. The `modelSupportsAdaptiveThinking()` function defaults to `true` for unknown models on first-party and Foundry, erring on the side of enabling the feature since "newer models (4.6+) are all trained on adaptive thinking and MUST have it enabled for model testing."

---

## 8. Fast Mode and Effort System

### Fast Mode Architecture

Fast mode is a first-party-only feature that provides priority processing for Opus 4.6 requests at 6x the standard price. The system architecture consists of:

1. **Eligibility check** (`isFastModeAvailable()`): Validates provider (first-party only), organization status (prefetched from `/api/claude_code_penguin_mode`), subscription tier, and feature flags
2. **Model gate** (`isFastModeSupportedByModel()`): Only Opus 4.6 models qualify (fastMode.ts:167-176)
3. **Runtime state machine**: Tracks `active` or `cooldown` states, where cooldowns are triggered by rate limits or overload responses
4. **Organization status**: Prefetched at startup and cached in memory; falls back to persistent cache (`penguinModeOrgEnabled` in global config) when the network request fails

The fast mode model is always resolved to `'opus'` (or `'opus[1m]'` when the 1M merge is enabled), as shown in `fastMode.ts:145-147`.

### Cooldown Mechanism

When the API returns a rate limit (429) or overloaded response, the system enters a cooldown period:

```typescript
// src/utils/fastMode.ts:214-233
export function triggerFastModeCooldown(
  resetTimestamp: number,
  reason: CooldownReason,
): void {
  runtimeState = { status: 'cooldown', resetAt: resetTimestamp, reason }
  // ...
}
```

During cooldown, requests fall back to standard speed. The cooldown expires automatically when `Date.now() >= resetAt`, and signal-based listeners notify the UI of state changes.

### The Effort System

The effort system (`effort.ts`) controls the model's reasoning depth through four named levels (`low`, `medium`, `high`, `max`) and, for internal users, numeric values. The resolution chain is:

1. **`CLAUDE_CODE_EFFORT_LEVEL` env var** -- highest priority; `"unset"` or `"auto"` disables effort entirely
2. **App state `effortValue`** -- set via `/effort` command or CLI `--effort` flag
3. **Model default** -- computed by `getDefaultEffortForModel()`

Model defaults vary by subscription tier and model:

```typescript
// src/utils/effort.ts:308-319
if (model.toLowerCase().includes('opus-4-6')) {
  if (isProSubscriber()) {
    return 'medium'
  }
  if (
    getOpusDefaultEffortConfig().enabled &&
    (isMaxSubscriber() || isTeamSubscriber())
  ) {
    return 'medium'
  }
}
```

Opus 4.6 defaults to `medium` effort for Pro, Max, and Team subscribers. When the "ultrathink" feature is enabled, all effort-supporting models default to `medium` (with ultrathink bumping to `high` on demand).

The `max` effort level is restricted to Opus 4.6 only. On non-Opus models, it is automatically downgraded to `high`:

```typescript
// src/utils/effort.ts:163-165
if (resolved === 'max' && !modelSupportsMaxEffort(model)) {
  return 'high'
}
```

Effort persistence is carefully scoped: `max` is session-only for external users (internal users can persist it), and numeric values are never persisted. The `toPersistableEffort()` function (effort.ts:95-105) enforces these constraints at the settings write boundary.

### Ultrathink

The "ultrathink" feature (`thinking.ts:19-24`) is a keyword-triggered mechanism. When the word "ultrathink" appears in user input, it signals a request for deeper reasoning. When the feature is enabled at the build level and via GrowthBook, it sets the default effort to `medium` (allowing the ultrathink keyword to bump it to `high`). This creates a dynamic two-tier reasoning system where standard requests get medium effort and ultrathink-tagged requests get high effort.

---

## 9. Model Allowlist System

### Tiered Matching

The `modelAllowlist.ts` module implements an allowlist system via the `availableModels` setting. When configured, only listed models appear in the model picker and are accepted by `getUserSpecifiedModelSetting()`. The matching operates in three tiers:

1. **Family aliases** (`"opus"`, `"sonnet"`, `"haiku"`) -- wildcard for the entire family, unless more specific entries exist
2. **Version prefixes** (`"opus-4-5"`, `"claude-opus-4-5"`) -- any build of that version
3. **Full model IDs** (`"claude-opus-4-5-20251101"`) -- exact match only

A notable edge case: when the allowlist contains both `"opus"` and `"opus-4-5"`, the family wildcard is ignored and only Opus 4.5 is permitted. The `familyHasSpecificEntries()` function (modelAllowlist.ts:65-87) detects this narrowing.

### Model Validation

The `validateModel()` function (validateModel.ts:20-82) performs live API validation by sending a minimal request (`max_tokens: 1`, content: `"Hi"`). Results are cached to avoid redundant API calls. The validation checks the allowlist before making any network request and accepts all known aliases without validation.

---

## 10. Subscription-Tier Model Selection

The model picker presented to users varies dramatically by subscription tier, as implemented in `getModelOptionsBase()` (modelOptions.ts:271-376):

- **Max / Team Premium**: Default is Opus 4.6 (with optional 1M); Sonnet and Haiku available as downgrades
- **Pro / Team Standard / Enterprise**: Default is Sonnet 4.6; Opus available as an upgrade (with pricing displayed)
- **PAYG first-party**: Default is Sonnet 4.6; full model range with per-token pricing shown
- **PAYG third-party**: Default is Sonnet 4.5 (older stable); supports custom model strings via `ANTHROPIC_DEFAULT_*_MODEL` environment variables with custom names and descriptions

The 1M context variants are conditionally shown based on `checkOpus1mAccess()` and `checkSonnet1mAccess()` (check1mAccess.ts), which gate on both the `CLAUDE_CODE_DISABLE_1M_CONTEXT` environment variable and, for subscribers, whether extra usage billing is enabled.

---

## Summary

The model selection and routing system represents one of Claude Code's most complex subsystems, addressing the combinatorial explosion of model versions, API providers, subscription tiers, capability flags, and user preferences. The architecture follows several consistent design principles:

- **Alias indirection**: Users interact with stable family names; version pinning is an explicit opt-in
- **Provider abstraction**: A single config registry maps each model to all provider formats
- **Fail-safe defaults**: Unknown models default to safe assumptions (thinking enabled, adaptive thinking enabled on 1P)
- **Progressive enhancement**: Capabilities are detected per-model with override mechanisms at every level
- **Subscription awareness**: Model defaults, pricing display, and feature access all vary by tier

Code markers (`@[MODEL LAUNCH]`) throughout the codebase annotate every site that requires updates when a new model generation is introduced, providing a mechanical checklist for launch operations.
