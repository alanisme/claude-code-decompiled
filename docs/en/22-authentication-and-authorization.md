# Authentication and Authorization System

## Technical Analysis of Claude Code's Credential Management Architecture

---

## 1. Authentication Methods

Claude Code supports a multi-layered authentication architecture designed to serve diverse deployment contexts -- from individual developers using API keys, to enterprise teams authenticating through OAuth against claude.ai or the Anthropic Console, to cloud-native deployments running on AWS Bedrock, Google Cloud Vertex AI, or Microsoft Azure Foundry. The system resolves which authentication method to use through a prioritized evaluation chain defined primarily in `src/utils/auth.ts`.

### 1.1 Direct API Key Authentication

The simplest authentication method involves setting the `ANTHROPIC_API_KEY` environment variable. This bypasses all OAuth machinery and sends the key directly as an `x-api-key` header. The function `getAnthropicApiKeyWithSource()` (file: `src/utils/auth.ts:226`) implements the resolution logic:

```typescript
export function getAnthropicApiKeyWithSource(
  opts: { skipRetrievingKeyFromApiKeyHelper?: boolean } = {},
): {
  key: null | string
  source: ApiKeySource
}
```

The `ApiKeySource` type enumerates four possible origins:

```typescript
export type ApiKeySource =
  | 'ANTHROPIC_API_KEY'
  | 'apiKeyHelper'
  | '/login managed key'
  | 'none'
```

API keys from environment variables receive the highest priority in CI environments. Outside CI, keys must be approved via the `customApiKeyResponses.approved` list in global config before they are honored -- a security measure preventing arbitrary environment variable injection from overriding established credentials.

### 1.2 OAuth Authentication (Claude.ai and Console)

OAuth 2.0 with PKCE (Proof Key for Code Exchange) serves as the primary authentication mechanism for interactive users. Two distinct OAuth paths exist:

- **Claude.ai OAuth**: For subscribers (Pro, Max, Team, Enterprise). Grants inference scopes (`user:inference`) allowing direct model access without an API key.
- **Console OAuth**: For API customers. Grants `org:create_api_key` scope, which the system uses to mint an API key server-side after login.

The determination of which auth system applies is governed by `shouldUseClaudeAIAuth()` (file: `src/services/oauth/client.ts:38`):

```typescript
export function shouldUseClaudeAIAuth(scopes: string[] | undefined): boolean {
  return Boolean(scopes?.includes(CLAUDE_AI_INFERENCE_SCOPE))
}
```

### 1.3 Environment Variable Token Injection

For managed contexts (Claude Code Remote, Claude Desktop), OAuth tokens can be injected via `CLAUDE_CODE_OAUTH_TOKEN` or through file descriptors (`CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR`). These tokens are treated as inference-only with no refresh capability, represented as:

```typescript
// src/utils/auth.ts:1260-1269
if (process.env.CLAUDE_CODE_OAUTH_TOKEN) {
  return {
    accessToken: process.env.CLAUDE_CODE_OAUTH_TOKEN,
    refreshToken: null,
    expiresAt: null,
    scopes: ['user:inference'],
    subscriptionType: null,
    rateLimitTier: null,
  }
}
```

### 1.4 Managed OAuth Context Guard

A critical architectural decision prevents credential confusion in managed sessions. The function `isManagedOAuthContext()` (file: `src/utils/auth.ts:91-96`) returns `true` when running inside Claude Code Remote or Claude Desktop:

```typescript
function isManagedOAuthContext(): boolean {
  return (
    isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) ||
    process.env.CLAUDE_CODE_ENTRYPOINT === 'claude-desktop'
  )
}
```

When this returns `true`, user-level settings such as `apiKeyHelper`, `ANTHROPIC_API_KEY`, and `ANTHROPIC_AUTH_TOKEN` are ignored. This prevents a stale API key from a terminal session from leaking into managed sessions where OAuth should be the sole authentication mechanism.

### 1.5 Auth Enable/Disable Logic

The function `isAnthropicAuthEnabled()` (file: `src/utils/auth.ts:100-149`) determines whether first-party Anthropic OAuth should be active. Authentication is disabled when:

1. The `--bare` flag is set (API-key-only mode)
2. Third-party providers (Bedrock/Vertex/Foundry) are configured
3. External auth tokens (`ANTHROPIC_AUTH_TOKEN`, `apiKeyHelper`) are present outside managed contexts
4. External API keys are present outside managed contexts

A special case exists for SSH remote sessions: when `ANTHROPIC_UNIX_SOCKET` is set, only the presence of `CLAUDE_CODE_OAUTH_TOKEN` determines auth enablement, because the local auth-injecting proxy handles actual credentials.

---

## 2. API Key Management

### 2.1 Storage Mechanisms

API keys are stored through a platform-specific hierarchy. On macOS, the preferred storage is the system keychain, accessed via the `security` command-line utility. The function `saveApiKey()` (file: `src/utils/auth.ts:1094-1160`) implements this:

```typescript
export async function saveApiKey(apiKey: string): Promise<void> {
  if (!isValidApiKey(apiKey)) {
    throw new Error(
      'Invalid API key format. API key must contain only alphanumeric characters, dashes, and underscores.',
    )
  }
  await maybeRemoveApiKeyFromMacOSKeychain()
  let savedToKeychain = false
  if (process.platform === 'darwin') {
    // Convert to hexadecimal to avoid any escaping issues
    const hexValue = Buffer.from(apiKey, 'utf-8').toString('hex')
    const command = `add-generic-password -U -a "${username}" -s "${storageServiceName}" -X "${hexValue}"\n`
    await execa('security', ['-i'], { input: command, reject: false })
    savedToKeychain = true
  }
  // ...
}
```

The hex encoding and use of `security -i` (interactive/stdin mode) are deliberate security choices. By piping the credential via stdin rather than passing it as a command-line argument, process monitors (such as CrowdStrike) only see `security -i`, not the credential itself.

On non-macOS platforms, the API key falls back to the global config file (`~/.claude.json`) under the `primaryApiKey` field.

### 2.2 Validation

API key validation uses a conservative regex pattern (file: `src/utils/auth.ts:1089`):

```typescript
function isValidApiKey(apiKey: string): boolean {
  return /^[a-zA-Z0-9-_]+$/.test(apiKey)
}
```

Runtime verification against the Anthropic API is performed by `verifyApiKey()` in `src/services/api/claude.ts`, invoked through the `useApiKeyVerification` hook (file: `src/hooks/useApiKeyVerification.ts`). This hook manages a state machine with states `loading`, `valid`, `invalid`, `missing`, and `error`.

### 2.3 Key Normalization and Approval

When an API key is saved, only the last 20 characters are stored in the approval list (file: `src/utils/authPortable.ts:17-19`):

```typescript
export function normalizeApiKeyForConfig(apiKey: string): string {
  return apiKey.slice(-20)
}
```

This normalization prevents storing full keys in the config while still allowing the system to recognize previously approved keys when presented via the `ANTHROPIC_API_KEY` environment variable.

### 2.4 The apiKeyHelper Mechanism

For environments requiring dynamic credential fetching, the `apiKeyHelper` setting specifies a shell command that outputs an API key. The execution employs a stale-while-revalidate (SWR) caching pattern with epoch-based invalidation (file: `src/utils/auth.ts:452-536`). Key properties include:

- **Default TTL**: 5 minutes (`DEFAULT_API_KEY_HELPER_TTL`), configurable via `CLAUDE_CODE_API_KEY_HELPER_TTL_MS`
- **Cold cache**: blocks until the helper completes (up to 10 minutes)
- **Warm cache**: returns the stale value immediately, refreshes in the background
- **Epoch tracking**: cache invalidation increments an epoch counter; in-flight requests from previous epochs are discarded
- **Security guard**: helpers sourced from project settings require workspace trust acceptance before execution

---

## 3. OAuth Flow

### 3.1 PKCE Authorization Code Flow

The OAuth implementation follows the Authorization Code flow with PKCE, implemented in the `OAuthService` class (file: `src/services/oauth/index.ts`). The cryptographic primitives are in `src/services/oauth/crypto.ts`:

```typescript
export function generateCodeVerifier(): string {
  return base64URLEncode(randomBytes(32))
}

export function generateCodeChallenge(verifier: string): string {
  const hash = createHash('sha256')
  hash.update(verifier)
  return base64URLEncode(hash.digest())
}

export function generateState(): string {
  return base64URLEncode(randomBytes(32))
}
```

The flow proceeds as follows:

1. A localhost HTTP server (`AuthCodeListener`) starts on an OS-assigned port
2. PKCE code verifier/challenge and state parameter are generated
3. Two authorization URLs are built -- one for automatic browser redirect, one for manual code entry
4. The browser opens the automatic URL; the manual URL is displayed as fallback
5. The `AuthCodeListener` (file: `src/services/oauth/auth-code-listener.ts`) captures the redirect
6. State parameter validation prevents CSRF attacks
7. The authorization code is exchanged for tokens

### 3.2 Authorization URL Construction

The `buildAuthUrl()` function (file: `src/services/oauth/client.ts:46-105`) constructs the authorization URL with parameters:

- `client_id`: Registered OAuth application ID (`9d1c250a-e61b-44d9-88ed-5944d1962f5e` in production)
- `response_type`: `code`
- `redirect_uri`: `http://localhost:{port}/callback` (automatic) or the platform's manual redirect URL
- `scope`: All scopes requested upfront (union of Console and Claude.ai scopes)
- `code_challenge` / `code_challenge_method`: S256 PKCE
- `state`: CSRF protection
- Optional: `orgUUID`, `login_hint`, `login_method` (e.g., `sso`)

### 3.3 Token Exchange

The function `exchangeCodeForTokens()` (file: `src/services/oauth/client.ts:107-144`) POSTs to the token endpoint with the authorization code, code verifier, and redirect URI. The response includes `access_token`, `refresh_token`, `expires_in`, and `scope`.

### 3.4 Token Refresh

Token refresh is handled by `refreshOAuthToken()` (file: `src/services/oauth/client.ts:146-274`). An optimization avoids redundant profile fetches: if the global config already contains `billingType`, `accountCreatedAt`, `subscriptionCreatedAt`, and secure storage has valid `subscriptionType` and `rateLimitTier`, the `/api/oauth/profile` round-trip is skipped entirely. This optimization is estimated to eliminate approximately 7 million daily requests fleet-wide.

### 3.5 Scope Management

Scopes are defined in `src/constants/oauth.ts`:

```typescript
export const CLAUDE_AI_OAUTH_SCOPES = [
  'user:profile',
  'user:inference',
  'user:sessions:claude_code',
  'user:mcp_servers',
  'user:file_upload',
] as const

export const CONSOLE_OAUTH_SCOPES = [
  'org:create_api_key',
  'user:profile',
] as const
```

All scopes are requested at login time (`ALL_OAUTH_SCOPES` is the union). The backend supports scope expansion on refresh, meaning tokens issued before new scopes were added can gain those scopes on the next refresh without requiring re-login.

### 3.6 Environment Variable Fast Path

The `authLogin` handler (file: `src/cli/handlers/auth.ts:140-186`) supports a non-interactive login path via `CLAUDE_CODE_OAUTH_REFRESH_TOKEN`. When set (along with `CLAUDE_CODE_OAUTH_SCOPES`), the browser flow is bypassed entirely -- the refresh token is exchanged directly for access tokens. This enables automated provisioning in CI/CD and managed environments.

---

## 4. Credential Storage

### 4.1 Secure Storage Architecture

The credential storage system uses a strategy pattern with three implementations and a fallback wrapper:

1. **macOS Keychain** (`src/utils/secureStorage/macOsKeychainStorage.ts`): The preferred storage on macOS. Credentials are serialized as JSON, hex-encoded, and stored via the `security` CLI. A cache with TTL prevents excessive keychain queries.

2. **Plain Text** (`src/utils/secureStorage/plainTextStorage.ts`): A JSON file at `~/.claude/.credentials.json` with `0o600` permissions. Used on Linux and as a fallback on macOS when the keychain is unavailable.

3. **Fallback Composite** (`src/utils/secureStorage/fallbackStorage.ts`): Wraps the primary and secondary storage. On read, it tries primary first, then secondary. On write, if primary fails, it writes to secondary and deletes stale primary data to prevent shadowing.

The fallback storage includes sophisticated conflict resolution (file: `src/utils/secureStorage/fallbackStorage.ts:42-58`): if the primary write fails but primary still holds an older entry, that stale entry would shadow fresh data in secondary (since `read()` prefers primary). The system best-effort deletes the stale primary to prevent this, addressing a class of bugs where rotated refresh tokens caused infinite login loops.

### 4.2 Keychain Cache Behavior

The macOS keychain implementation features a stale-while-error cache (file: `src/utils/secureStorage/macOsKeychainStorage.ts:51-64`): if a keychain read fails transiently (e.g., a `security` process spawn failure), the previous cached value is served rather than caching `null`. This prevents a single transient failure from surfacing as "Not logged in" across all subsystems. Explicit invalidation (logout, key deletion) uses `clearKeychainCache()` which sets `data=null`, ensuring intentional deletions are honored.

### 4.3 CCR (Claude Code Remote) Token Persistence

In remote execution environments, credentials arrive via file descriptors. The `getCredentialFromFd()` function (file: `src/utils/authFileDescriptor.ts:97-166`) implements a two-tier read strategy:

1. **File descriptor** (legacy): Read from a pipe FD passed by the Go environment manager via `cmd.ExtraFiles`. The pipe is drained on first read and does not survive across `exec`/`tmux` boundaries.
2. **Well-known file**: Written by the system on successful FD read at `/home/claude/.claude/remote/.oauth_token` (or `.api_key`). Covers subprocesses that inherit the environment variable but not the FD itself.

Token files are written with restrictive permissions (`0o600` for files, `0o700` for directory) and only within CCR containers:

```typescript
// src/utils/authFileDescriptor.ts:35-36
if (!isEnvTruthy(process.env.CLAUDE_CODE_REMOTE)) {
  return
}
```

---

## 5. Multi-Provider Authentication

### 5.1 AWS Bedrock

Bedrock authentication supports multiple credential sources, configured in `src/services/api/client.ts:153-189`:

- **Bearer Token**: Via `AWS_BEARER_TOKEN_BEDROCK`, which sets an `Authorization: Bearer` header and skips standard AWS auth signing.
- **Credential Export Helper**: Via the `awsCredentialExport` settings key, which runs a shell command expected to produce `aws sts` JSON output. The output is validated against the `AwsStsOutput` type guard (file: `src/utils/aws.ts:25-47`).
- **Auth Refresh Helper**: Via the `awsAuthRefresh` settings key, which runs an interactive auth command (e.g., `aws sso login`) with a 3-minute timeout.
- **Standard AWS SDK Chain**: Falls through to `@aws-sdk/credential-providers` when no helpers are configured.

The `refreshAndGetAwsCredentials()` function (file: `src/utils/auth.ts:787-807`) is memoized with a 1-hour TTL matching the default AWS STS session duration. It chains auth refresh, credential export, and INI cache clearing in sequence.

A prefetch mechanism (`prefetchAwsCredentialsAndBedRockInfoIfSafe()`, file: `src/utils/auth.ts:1023-1048`) starts credential resolution early for trusted workspaces, but defers it for untrusted ones until the trust dialog is accepted.

### 5.2 Google Cloud Vertex AI

Vertex authentication uses `google-auth-library`'s `GoogleAuth` with the `cloud-platform` scope. The system implements:

- **GCP Auth Refresh Helper**: Via `gcpAuthRefresh` setting (e.g., `gcloud auth application-default login`)
- **Credential Validity Check**: `checkGcpCredentialsValid()` (file: `src/utils/auth.ts:847-866`) attempts to get an access token with a 5-second timeout to avoid the ~12s GCE metadata server hang outside GCP
- **TTL-based caching**: 1-hour credential TTL matching typical ADC token lifetime

### 5.3 Microsoft Azure Foundry

Foundry authentication (file: `src/services/api/client.ts:191-220`) supports two paths:

- **API Key**: Via `ANTHROPIC_FOUNDRY_API_KEY` (read by the SDK automatically)
- **Azure AD**: When no API key is present, `DefaultAzureCredential` from `@azure/identity` is used with `getBearerTokenProvider` scoped to `https://cognitiveservices.azure.com/.default`

A skip-auth mode (`CLAUDE_CODE_SKIP_FOUNDRY_AUTH`) provides a mock token provider for testing and proxy scenarios.

### 5.4 Security for Cloud Auth Helpers

All cloud auth helpers (`awsAuthRefresh`, `awsCredentialExport`, `gcpAuthRefresh`) follow the same security pattern when sourced from project settings: execution is blocked until workspace trust has been established. This prevents a malicious `.claude/settings.local.json` from executing arbitrary commands before the user has reviewed and approved the project:

```typescript
// src/utils/auth.ts:620-630 (representative pattern)
if (isAwsAuthRefreshFromProjectSettings()) {
  const hasTrust = checkHasTrustDialogAccepted()
  if (!hasTrust && !getIsNonInteractiveSession()) {
    logAntError('awsAuthRefresh invoked before trust check', error)
    return false
  }
}
```

---

## 6. Session Token Management

### 6.1 Token Lifecycle

OAuth tokens follow a well-defined lifecycle:

1. **Acquisition**: Via OAuth flow (`OAuthService.startOAuthFlow()`) or refresh token exchange
2. **Storage**: In secure storage (keychain or plaintext file) via `saveOAuthTokensIfNeeded()`
3. **Read**: Via `getClaudeAIOAuthTokens()`, memoized for performance
4. **Refresh**: Triggered by expiration check or server-side 401
5. **Invalidation**: On logout, credential change, or cross-process token update

### 6.2 Expiration Check with Buffer

Token expiration is checked with a 5-minute safety buffer (file: `src/services/oauth/client.ts:344-353`):

```typescript
export function isOAuthTokenExpired(expiresAt: number | null): boolean {
  if (expiresAt === null) {
    return false
  }
  const bufferTime = 5 * 60 * 1000
  const now = Date.now()
  const expiresWithBuffer = now + bufferTime
  return expiresWithBuffer >= expiresAt
}
```

Tokens with `expiresAt === null` (env var tokens, file descriptor tokens) are never considered expired, since they have no refresh capability.

### 6.3 Cross-Process Token Synchronization

Multiple Claude Code instances may run simultaneously. The function `invalidateOAuthCacheIfDiskChanged()` (file: `src/utils/auth.ts:1320-1336`) detects when another process has written new tokens by comparing the `mtime` of `.credentials.json`. If the file's modification time has changed, in-memory caches are cleared, forcing a re-read from disk.

This addresses a critical scenario: Terminal 1 logs in, Terminal 2 logs in (revoking Terminal 1's refresh token server-side), and without mtime checking, Terminal 1's memoized cache would serve the now-invalid token indefinitely.

### 6.4 Concurrent Refresh Deduplication

The token refresh mechanism employs multiple deduplication layers:

1. **Promise deduplication**: `pendingRefreshCheck` (file: `src/utils/auth.ts:1425`) ensures concurrent non-retry, non-force calls share a single promise
2. **File-based locking**: `lockfile.lock(claudeDir)` acquires an exclusive lock before refreshing, with up to 5 retries and 1-2 second jittered backoff between attempts
3. **Post-lock validation**: After acquiring the lock, tokens are re-read to check if another process already refreshed

```typescript
// src/utils/auth.ts:1484-1502 (lock acquisition with retry)
try {
  release = await lockfile.lock(claudeDir)
} catch (err) {
  if ((err as { code?: string }).code === 'ELOCKED') {
    if (retryCount < MAX_RETRIES) {
      await sleep(1000 + Math.random() * 1000)
      return checkAndRefreshOAuthTokenIfNeededImpl(retryCount + 1, force)
    }
    return false
  }
}
```

---

## 7. Auth State Machine

### 7.1 Token Source Resolution

The `getAuthTokenSource()` function (file: `src/utils/auth.ts:153-206`) implements a prioritized state resolution chain. The evaluation order is:

1. **Bare mode**: Only `apiKeyHelper` from `--settings` flag
2. **`ANTHROPIC_AUTH_TOKEN`**: External bearer token (skipped in managed OAuth contexts)
3. **`CLAUDE_CODE_OAUTH_TOKEN`**: Injected OAuth token (env var)
4. **File descriptor OAuth token**: From CCR pipe or well-known file
5. **`apiKeyHelper`**: Configured helper command (skipped in managed OAuth contexts)
6. **Claude.ai OAuth**: Tokens from secure storage with `user:inference` scope

Each source returns a typed discriminated union:

```typescript
{ source: 'ANTHROPIC_AUTH_TOKEN', hasToken: true }
{ source: 'CLAUDE_CODE_OAUTH_TOKEN', hasToken: true }
{ source: 'CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR', hasToken: true }
{ source: 'CCR_OAUTH_TOKEN_FILE', hasToken: true }
{ source: 'apiKeyHelper', hasToken: true }
{ source: 'claude.ai', hasToken: true }
{ source: 'none', hasToken: false }
```

### 7.2 Auth Method Determination

The `authStatus` handler (file: `src/cli/handlers/auth.ts:232-319`) resolves the current auth method for user-facing display. The mapping is:

| Condition | Auth Method |
|-----------|-------------|
| Third-party provider active | `third_party` |
| Token source is `claude.ai` | `claude.ai` |
| Token source is `apiKeyHelper` | `api_key_helper` |
| Token source is anything else (not `none`) | `oauth_token` |
| API key from `ANTHROPIC_API_KEY` | `api_key` |
| API key from `/login managed key` | `claude.ai` |
| Nothing | `none` |

### 7.3 Login Flow State Transitions

The `authLogin` handler (file: `src/cli/handlers/auth.ts:112-229`) orchestrates state transitions:

1. `IDLE` -> Determine login method (claude.ai vs Console, based on `forceLoginMethod` setting or `--console`/`--claudeai` flags)
2. Check for `CLAUDE_CODE_OAUTH_REFRESH_TOKEN` fast path -> If present, exchange directly and skip browser flow
3. Start `OAuthService.startOAuthFlow()` -> Opens browser
4. Receive tokens -> Call `installOAuthTokens()`
5. `installOAuthTokens()` performs:
   - `performLogout({ clearOnboarding: false })` -- clears old state
   - Fetches and stores profile info
   - For Claude.ai auth: stores tokens in secure storage
   - For Console auth: creates and stores an API key via `createAndStoreApiKey()`
   - Fetches and stores user roles
   - Clears all auth-related caches

### 7.4 Subscription Type Resolution

The subscription type flows from the OAuth profile into secure storage and the global config. Recognized types include `max`, `pro`, `enterprise`, and `team`, mapped from the organization type field in the profile response (file: `src/services/oauth/client.ts:369-387`). Accessor functions such as `isMaxSubscriber()`, `isTeamSubscriber()`, `isEnterpriseSubscriber()`, and `isProSubscriber()` (file: `src/utils/auth.ts:1679-1699`) expose this state to the rest of the system.

### 7.5 Organization Validation

Enterprise deployments can enforce organization membership via the `forceLoginOrgUUID` policy setting. The `validateForceLoginOrg()` function (file: `src/utils/auth.ts:1923`) validates after every login that the authenticated user belongs to the required organization. This validation fails closed -- if the organization cannot be determined (network error, missing profile data), authentication is rejected.

---

## 8. Error Handling

### 8.1 OAuth 401 Recovery

When the API returns a 401 indicating token expiration, `handleOAuth401Error()` (file: `src/utils/auth.ts:1360-1392`) executes a recovery sequence:

1. **Deduplication**: Concurrent 401 errors for the same `failedAccessToken` share a single handler via `pending401Handlers` Map
2. **Cache clearing**: All in-memory caches are cleared
3. **Cross-process check**: Async re-read from keychain to check if another process already refreshed
4. **Conditional refresh**: Only refreshes if the keychain still holds the same failed token

```typescript
async function handleOAuth401ErrorImpl(failedAccessToken: string): Promise<boolean> {
  clearOAuthTokenCache()
  const currentTokens = await getClaudeAIOAuthTokensAsync()
  if (!currentTokens?.refreshToken) return false
  if (currentTokens.accessToken !== failedAccessToken) {
    logEvent('tengu_oauth_401_recovered_from_keychain', {})
    return true
  }
  return checkAndRefreshOAuthTokenIfNeeded(0, true)
}
```

This design prevents the "thundering herd" problem where multiple concurrent API requests all receiving 401s would each attempt independent token refreshes.

### 8.2 API Key Helper Error Recovery

The apiKeyHelper uses a resilient SWR approach (file: `src/utils/auth.ts:513-535`). On failure during a background refresh (SWR path), the stale value is preserved and its timestamp is bumped to prevent retry hammering. On failure during a cold start, a sentinel value (`' '` -- a single space) is cached to prevent fallback to OAuth, which would be incorrect for environments that configured an apiKeyHelper intentionally.

### 8.3 Keychain Lock Detection

On macOS, the system detects locked keychains (common in SSH sessions) via `isMacOsKeychainLocked()` (file: `src/utils/secureStorage/macOsKeychainStorage.ts:211-231`). Exit code 36 from `security show-keychain-info` indicates a locked keychain. This result is cached for the process lifetime since keychain lock state does not change during a CLI session.

### 8.4 SSL and Network Errors

Login failure messages include SSL error hints when applicable (file: `src/cli/handlers/auth.ts:180-184`):

```typescript
const sslHint = getSSLErrorHint(err)
process.stderr.write(
  `Login failed: ${errorMessage(err)}\n${sslHint ? sslHint + '\n' : ''}`,
)
```

### 8.5 Token Refresh Lock Contention

When the file lock for token refresh cannot be acquired (`ELOCKED`), the system retries up to 5 times with jittered backoff (1-2 seconds per retry). If the maximum retry count is reached, the refresh silently fails and the system continues with the existing token, which may trigger a server-side 401 and invoke the 401 recovery path described above.

### 8.6 Logout and Credential Cleanup

The logout process (file: `src/commands/logout/logout.tsx`) follows a specific ordering:

1. Flush telemetry **before** clearing credentials (prevents organization data leakage to the wrong context)
2. Remove API key from keychain and config
3. Delete all secure storage data
4. Clear all auth-related caches (OAuth tokens, trusted device tokens, betas, tool schemas, GrowthBook, Grove config, remote managed settings, policy limits)
5. Reset global config (clear `oauthAccount`, optionally clear onboarding state)

The cache clearing sequence is order-sensitive: user data caches must be cleared before GrowthBook refresh so it picks up fresh (absent) credentials rather than stale ones.

### 8.7 Custom OAuth URL Security

The system supports redirecting OAuth to alternative endpoints (for FedStart/PubSec deployments) via `CLAUDE_CODE_CUSTOM_OAUTH_URL`, but enforces a strict allowlist (file: `src/constants/oauth.ts:179-183`):

```typescript
const ALLOWED_OAUTH_BASE_URLS = [
  'https://beacon.claude-ai.staging.ant.dev',
  'https://claude.fedstart.com',
  'https://claude-staging.fedstart.com',
]
```

Any URL not in this list causes an immediate throw, preventing OAuth tokens from being sent to arbitrary endpoints.

---

## Summary of Key Files

| File | Role |
|------|------|
| `src/utils/auth.ts` | Central auth orchestration: token resolution, key management, refresh logic, subscription queries |
| `src/cli/handlers/auth.ts` | CLI auth commands: login, logout, status |
| `src/services/oauth/index.ts` | OAuth PKCE flow orchestration |
| `src/services/oauth/client.ts` | OAuth token exchange, refresh, profile fetching |
| `src/services/oauth/auth-code-listener.ts` | Localhost HTTP server for OAuth redirect capture |
| `src/services/oauth/crypto.ts` | PKCE cryptographic primitives |
| `src/constants/oauth.ts` | OAuth configuration (URLs, scopes, client IDs) |
| `src/utils/authFileDescriptor.ts` | File descriptor and well-known file credential reading for CCR |
| `src/utils/authPortable.ts` | Cross-platform key normalization and keychain removal |
| `src/utils/secureStorage/macOsKeychainStorage.ts` | macOS keychain read/write with caching |
| `src/utils/secureStorage/plainTextStorage.ts` | Plaintext JSON file credential storage |
| `src/utils/secureStorage/fallbackStorage.ts` | Primary/secondary storage composite with conflict resolution |
| `src/utils/aws.ts` | AWS STS credential validation and cache management |
| `src/services/api/client.ts` | Anthropic SDK client factory with multi-provider auth |
| `src/hooks/useApiKeyVerification.ts` | React hook for API key validation state |
| `src/commands/logout/logout.tsx` | Logout procedure with ordered cache invalidation |
