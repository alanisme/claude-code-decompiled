# Remote Control and Killswitches in Claude Code: What Anthropic Can Do Without Asking

*Mapping every remote control mechanism found in Claude Code v2.1.88*

## The Short Version

Claude Code has a surprisingly extensive remote control infrastructure. Anthropic — and enterprise admins — can modify the tool's behavior remotely, often without any user notification. Some of this is standard SaaS practice. Some of it goes further than one might expect.

## 1. Remote Managed Settings

### How It Works

Every eligible session phones home to fetch settings:

```
GET /api/claude_code/settings
```

Source: `src/services/remoteManagedSettings/index.ts:105-107`

### Polling Frequency

```typescript
// src/services/remoteManagedSettings/index.ts:52-54
const SETTINGS_TIMEOUT_MS = 10000
const DEFAULT_MAX_RETRIES = 5
const POLLING_INTERVAL_MS = 60 * 60 * 1000 // 1 hour
```

Every hour, Claude Code checks in. Five retries on failure. This is persistent.

### Who Gets Managed

- **Console users** (API key): All of them
- **OAuth users**: Only Enterprise/C4E and Team subscribers

### The "Accept or Leave" Dialog

This is the part that stands out. When remote settings include what the code calls "dangerous" changes, the user gets a blocking dialog:

```typescript
// src/services/remoteManagedSettings/securityCheck.tsx:67-73
export function handleSecurityCheckResult(result: SecurityCheckResult): boolean {
  if (result === 'rejected') {
    gracefulShutdownSync(1)  // Exit with code 1
    return false
  }
  return true
}
```

Reject the settings and the application **terminates**. Your choices are: accept whatever Anthropic is pushing, or stop using Claude Code. There's no "keep working with previous settings" option.

### What Happens When the Server Is Down

Once you've accepted remote settings, they're cached to disk:

```typescript
// src/services/remoteManagedSettings/index.ts:433-436
if (cachedSettings) {
  logForDebugging('Remote settings: Using stale cache after fetch failure')
  setSessionCache(cachedSettings)
  return cachedSettings
}
```

So even if Anthropic's servers go offline, whatever settings they last pushed to you remain active. The settings are persistent — there's no automatic rollback.

## 2. Feature Flag Killswitches

This is where it gets interesting. Multiple features can be remotely toggled off via GrowthBook, and none of these require user consent.

### Bypass Permissions Killswitch

```typescript
// src/utils/permissions/bypassPermissionsKillswitch.ts
// Checks a Statsig gate to disable bypass permissions
```

Can revoke permission bypass capabilities remotely. Makes sense from a security standpoint, but worth knowing it exists.

### Auto Mode Circuit Breaker

```typescript
// src/utils/permissions/autoModeState.ts
// autoModeCircuitBroken state prevents re-entry to auto mode
```

Auto mode can be killed remotely. If you're depending on auto mode for a workflow, it could disappear mid-session.

### Fast Mode Killswitch

```typescript
// src/utils/fastMode.ts
// Fetches from /api/claude_code_penguin_mode
// Can permanently disable fast mode for a user
```

Notably, it can **permanently** disable fast mode for a specific user. Not temporarily. Permanently.

### Analytics Sink Killswitch

```typescript
// src/services/analytics/sinkKillswitch.ts:4
const SINK_KILLSWITCH_CONFIG_NAME = 'tengu_frond_boric'
```

Can remotely stop all analytics output. Interestingly, this means Anthropic has a way to turn off their own telemetry collection when needed — they just don't expose that capability to users.

### Agent Teams Killswitch

```typescript
// src/utils/agentSwarmsEnabled.ts
// Requires both env var AND GrowthBook gate 'tengu_amber_flint'
```

### Voice Mode Killswitch

```typescript
// src/voice/voiceModeEnabled.ts:21
// 'tengu_amber_quartz_disabled' — emergency off for voice mode
```

## 3. Model Override System

Anthropic can remotely dictate which model internal employees use:

```typescript
// src/utils/model/antModels.ts:32-33
// @[MODEL LAUNCH]: Update tengu_ant_model_override with new ant-only models
// @[MODEL LAUNCH]: Add the codename to scripts/excluded-strings.txt
```

The `tengu_ant_model_override` GrowthBook flag can:
- Force a specific default model
- Set the effort level
- Append text to the system prompt
- Define custom model aliases

This is internal-only, but it demonstrates the level of remote control the infrastructure supports. If they wanted to extend this to external users, the plumbing is already there.

## 4. Penguin Mode (Fast Mode Internals)

Fast mode has its own dedicated endpoint:

```typescript
// src/utils/fastMode.ts
// GET /api/claude_code_penguin_mode
// If API indicates disabled, permanently disabled for user
```

Multiple feature flags are involved:
- `tengu_penguins_off`
- `tengu_marble_sandcastle`

The fact that "penguin" is the internal codename for fast mode is a nice continuation of the animal naming theme.
