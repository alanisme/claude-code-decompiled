# What Claude Code Actually Sends Home: A Deep Dive into Its Telemetry

*Reverse-engineering the data pipeline in Claude Code v2.1.88*

## The Big Picture

After thorough tracing through the decompiled telemetry code in Claude Code, the short version is: it runs a two-tier analytics pipeline that collects a *lot* of environment and usage metadata. There is no evidence of keylogging or source code exfiltration — that's the good news. The less-good news is the sheer breadth of what gets collected, and how hard it is to actually opt out.

## How the Data Pipeline Works

### First-Party Logging (Anthropic's Own)

This is the main telemetry channel, and the engineering here is pretty robust — maybe too robust if you value privacy:

- **Endpoint**: `https://api.anthropic.com/api/event_logging/batch`
- **Protocol**: OpenTelemetry with Protocol Buffers
- **Batch size**: Up to 200 events per batch, flushed every 10 seconds
- **Retry logic**: Quadratic backoff, up to 8 attempts, and here's the kicker — failed events are **persisted to disk** at `~/.claude/telemetry/` so they can be retried later

That persistence detail is notable. It means even if your network goes down mid-session, those telemetry events are waiting patiently on disk to be shipped out the next time you connect.

Source: `src/services/analytics/firstPartyEventLoggingExporter.ts`

### Third-Party Logging (Datadog)

On top of Anthropic's own pipeline, data also flows to Datadog:

- **Endpoint**: `https://http-intake.logs.us5.datadoghq.com/api/v2/logs`
- **Scope**: Limited to 64 pre-approved event types (so at least it's not a full firehose)
- **Token**: `pubbbf48e6d78dae54bceaa4acf463299bf`

Source: `src/services/analytics/datadog.ts`

## What Exactly Gets Collected

### Your Environment Fingerprint

Every single telemetry event carries this metadata payload (from `src/services/analytics/metadata.ts:417-452`):

```
- platform, platformRaw, arch, nodeVersion
- terminal type
- installed package managers and runtimes
- CI/CD detection, GitHub Actions metadata
- WSL version, Linux distro, kernel version
- VCS (version control system) type
- Claude Code version and build time
- deployment environment
```

This is surprising. The level of environment profiling goes well beyond what seems necessary for a coding assistant. Knowing your terminal type and installed package managers? That starts to feel like product analytics doing market research, not just debugging telemetry.

### Process Metrics (`metadata.ts:457-467`)

```
- uptime, rss, heapTotal, heapUsed
- CPU usage and percentage
- memory arrays and external allocations
```

This part is standard — most desktop apps collect performance data. No complaints here.

### User and Session Tracking (`metadata.ts:472-496`)

```
- model in use
- session ID, user ID, device ID
- account UUID, organization UUID
- subscription tier (max, pro, enterprise, team)
- repository remote URL hash (SHA256, first 16 chars)
- agent type, team name, parent session ID
```

The repo URL hashing is worth pausing on. They take your git remote URL, SHA256 it, and send the first 16 characters. This means Anthropic can correlate sessions working on the same repo, even though they can't (trivially) reverse the hash to get the actual URL. It's clever from a product analytics perspective, but it does mean your repos are being fingerprinted.

### Tool Input Logging — And Its Backdoor

By default, tool inputs are truncated, which is reasonable:

```
- Strings: truncated at 512 chars, displayed as 128 + ellipsis
- JSON: limited to 4,096 chars
- Arrays: max 20 items
- Nested objects: max 2 levels deep
```

Source: `metadata.ts:236-241`

**However** — and this is important — when the environment variable `OTEL_LOG_TOOL_DETAILS=1` is set, **full tool inputs are logged without any truncation**. That means your entire file contents, code snippets, whatever the tool was processing, all goes into the telemetry stream.

Source: `metadata.ts:86-88`

The circumstances under which this gets enabled are worth investigating. If it's only for internal debugging, fine. But it's worth knowing it exists.

### File Extension Tracking

When you run bash commands through Claude Code, if the command involves any of these tools — `rm, mv, cp, touch, mkdir, chmod, chown, cat, head, tail, sort, stat, diff, wc, grep, rg, sed` — the file extensions of their arguments are extracted and logged.

Source: `metadata.ts:340-412`

So Anthropic knows you're working with `.py` files and `.ts` files even if they don't see the contents. This is standard product analytics stuff for understanding user workflows, but still good to be aware of.

## The Opt-Out Problem (Or Lack Thereof)

Here's where things get uncomfortable. The first-party logging pipeline **cannot be disabled** if you're using the direct Anthropic API:

```typescript
// src/services/analytics/firstPartyEventLogger.ts:141-144
export function is1PEventLoggingEnabled(): boolean {
  return !isAnalyticsDisabled()
}
```

Digging into `isAnalyticsDisabled()`, it only returns true for:
- Test environments
- Third-party cloud providers (Bedrock, Vertex)
- A global telemetry opt-out flag that **isn't exposed in the settings UI**

So there is technically a mechanism — but no user-facing way to reach it. If you're a direct API user, telemetry is effectively mandatory.

## A/B Testing Without Consent

The source also reveals GrowthBook integration for experiment assignment. Users get bucketed into A/B test groups, and the system sends user attributes including:

```
- id, sessionId, deviceID
- platform, organizationUUID, subscriptionType
```

Source: `src/services/analytics/growthbook.ts`

This is standard practice in SaaS products, but typically users at least know they might be in experiments. There's no visible indicator here.

