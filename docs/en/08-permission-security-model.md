# Permission and Security Model: An Independent Analysis

This document is an independent security analysis of Claude Code's permission system, sandbox integration, and command validation infrastructure. It is based on a thorough reading of the source code as of this codebase snapshot, covering the layered defenses that stand between a model-generated tool call and its execution on the user's machine.

---

## 1. Permission Modes

Claude Code defines a set of permission modes that govern how aggressively tool calls are auto-approved or gated behind user consent. These are defined in `src/types/permissions.ts`:

```typescript
// src/types/permissions.ts:16-22
export const EXTERNAL_PERMISSION_MODES = [
  'acceptEdits',
  'bypassPermissions',
  'default',
  'dontAsk',
  'plan',
] as const
```

Additionally, there are internal-only modes:

```typescript
// src/types/permissions.ts:28-29
export type InternalPermissionMode = ExternalPermissionMode | 'auto' | 'bubble'
```

Here is what each mode means in practice:

- **default**: The standard interactive mode. Every tool call that is not covered by a pre-existing allow rule triggers a permission prompt. The user must explicitly approve or deny each action. This is the safest mode for users who want full control.

- **plan**: A read-only planning mode. The model can read files and search the codebase but cannot write or execute. The UI displays a pause icon. This mode is useful for reviewing what the model intends to do before granting execution permissions.

- **acceptEdits**: An intermediate mode that auto-approves file edits and certain filesystem-mutating Bash commands (mkdir, touch, rm, rmdir, mv, cp, sed) within the allowed working directory, without prompting. Other operations (arbitrary Bash commands, network access) still require approval. Defined in `src/tools/BashTool/modeValidation.ts`:

```typescript
// src/tools/BashTool/modeValidation.ts:7-16
const ACCEPT_EDITS_ALLOWED_COMMANDS = [
  'mkdir',
  'touch',
  'rm',
  'rmdir',
  'mv',
  'cp',
  'sed',
] as const
```

- **bypassPermissions**: The full "YOLO" mode for external users. All tool calls are auto-approved without any prompt. This mode is gated behind a Statsig feature flag and can be remotely disabled via `src/utils/permissions/bypassPermissionsKillswitch.ts`. When the killswitch fires, the mode silently downgrades.

- **dontAsk**: Similar to bypassPermissions in that no prompts are shown, but instead of auto-approving, it auto-*denies* anything that would have triggered a prompt. The model gets a rejection message and must find another approach. This is the "headless CI" mode -- the agent can only use tools it has been explicitly pre-authorized for.

- **auto** (internal, Anthropic-only): The AI-classifier-mediated mode. Instead of prompting the user, a second model (the "YOLO classifier") evaluates whether the action should be allowed. This is gated behind the `TRANSCRIPT_CLASSIFIER` feature flag and is not available to external users.

- **bubble** (internal): An internal coordination mode for multi-agent swarm scenarios, not user-addressable.

The mode configuration lives in `src/utils/permissions/PermissionMode.ts` (lines 42-91) which maps each mode to a display title, symbol, color, and its external-facing equivalent. Notably, `auto` maps to the external mode `default`, meaning external API consumers never see it.

---

## 2. The Permission Decision Tree

The central permission check lives in `hasPermissionsToUseTool` in `src/utils/permissions/permissions.ts` (line 473). Every tool call passes through this function. The decision tree is roughly:

1. **Tool-specific checkPermissions**: Each tool implements its own `checkPermissions` method. For BashTool, this is a deep pipeline of validators in `src/tools/BashTool/bashPermissions.ts`. For FileEditTool, it checks path safety. The tool returns `allow`, `ask`, `deny`, or `passthrough`.

2. **Deny rules take priority**: The system checks deny rules from all sources (user settings, project settings, local settings, policy settings, CLI args, session rules). If any deny rule matches the tool and its arguments, the action is denied immediately. See `getDenyRuleForTool` at line 287.

3. **Allow rules**: If the tool and its arguments match an allow rule, the action is auto-approved. Rules are checked from all sources. For Bash, rules can be prefix-based (`Bash(git commit:*)`) or exact (`Bash(ls -la)`).

4. **Mode-specific handling**: After the tool's own permission check returns `ask`:
   - In `dontAsk` mode, `ask` is converted to `deny` (line 508).
   - In `auto` mode, the system first tries the `acceptEdits` fast-path (line 601), then checks a safe-tool allowlist (line 660), and finally invokes the YOLO classifier (line 689).
   - In `bypassPermissions` mode, all `ask` results become `allow`.

5. **Hook integration**: If `shouldAvoidPermissionPrompts` is set (headless agents), the system tries `PermissionRequest` hooks before auto-denying. Hooks can allow, deny, or pass through (line 400-471).

6. **Denial tracking**: In auto mode, the system tracks consecutive and total denials. If the classifier denies 3 actions in a row or 20 total, the system falls back to prompting the user:

```typescript
// src/utils/permissions/denialTracking.ts:12-15
export const DENIAL_LIMITS = {
  maxConsecutive: 3,
  maxTotal: 20,
} as const
```

This is a safety valve: a classifier stuck in a deny loop does not silently stall the agent forever.

---

## 3. Bash Command Validation

Bash command validation is the most complex part of the security model. It operates at multiple layers.

### Layer 1: Security Checks (bashSecurity.ts)

`src/tools/BashTool/bashSecurity.ts` implements over 20 distinct validators, each assigned a numeric ID for telemetry. The validators run in sequence; if any returns `ask`, the command requires user approval. Key checks include:

**Command substitution detection** (line 16-41): Blocks `$()`, `${}`, backticks, process substitution `<()` / `>()`, Zsh-specific expansions like `=cmd` and `~[`:

```typescript
// src/tools/BashTool/bashSecurity.ts:16-41
const COMMAND_SUBSTITUTION_PATTERNS = [
  { pattern: /<\(/, message: 'process substitution <()' },
  { pattern: />\(/, message: 'process substitution >()' },
  { pattern: /=\(/, message: 'Zsh process substitution =()' },
  { pattern: /(?:^|[\s;&|])=[a-zA-Z_]/, message: 'Zsh equals expansion (=cmd)' },
  { pattern: /\$\(/, message: '$() command substitution' },
  { pattern: /\$\{/, message: '${} parameter substitution' },
  // ... more patterns
]
```

**Zsh-specific dangerous commands** (line 43-74): A hardcoded blocklist of Zsh builtins that can bypass security: `zmodload` (loads arbitrary kernel modules), `sysopen`/`syswrite` (direct file descriptor manipulation), `zpty` (pseudo-terminal execution), `ztcp`/`zsocket` (network exfiltration), and the `zf_*` builtins from `zsh/files` which bypass binary-level checks:

```typescript
// src/tools/BashTool/bashSecurity.ts:43-74
const ZSH_DANGEROUS_COMMANDS = new Set([
  'zmodload', 'emulate', 'sysopen', 'sysread', 'syswrite',
  'sysseek', 'zpty', 'ztcp', 'zsocket', 'mapfile',
  'zf_rm', 'zf_mv', 'zf_ln', 'zf_chmod', 'zf_chown',
  'zf_mkdir', 'zf_rmdir', 'zf_chgrp',
])
```

**Heredoc safety analysis** (line 317+): The validator has a careful heredoc parser that only allows `$(cat <<'DELIM' ...)` patterns where the delimiter is single-quoted (preventing expansion), the closing delimiter is on a line by itself, and there is non-whitespace before the `$(` (preventing use as a command name). This is documented as an "EARLY-ALLOW path" and the comments emphasize it must be "PROVABLY safe, not probably safe."

**Quote stripping and analysis**: The `extractQuotedContent` function (line 128) carefully strips quoted content while tracking single vs. double quotes and escape sequences. This produces three views of the command: with double-quote content preserved, fully unquoted, and unquoted but with quote characters preserved (for detecting quote-adjacent metacharacters).

### Layer 2: Read-Only Command Validation (readOnlyValidation.ts)

`src/tools/BashTool/readOnlyValidation.ts` maintains a large configuration of commands known to be read-only. These include `git` read operations (diff, log, status), `docker` inspection commands, `gh` read commands, `rg` (ripgrep), and many others. Each command has a `safeFlags` record specifying which flags are known-safe and whether they take arguments.

Commands matching these patterns are auto-approved without user interaction, forming the backbone of what makes Claude Code usable without constant approval fatigue.

### Layer 3: Compound Command Splitting

Commands joined by `&&`, `||`, `;`, or pipes are split into subcommands, each checked individually. There is a complexity cap at 50 subcommands (`MAX_SUBCOMMANDS_FOR_SECURITY_CHECK` in `bashPermissions.ts` line 103) to prevent event-loop starvation from pathologically complex commands.

### Layer 4: Path Validation in Bash

`src/tools/BashTool/pathValidation.ts` extracts file paths from commands like `sed`, `mv`, `cp`, `rm`, and `tee`, then validates each path against the working directory restrictions and permission rules.

---

## 4. File System Protection

Path validation is a multi-step process defined primarily in `src/utils/permissions/pathValidation.ts` and `src/utils/permissions/filesystem.ts`.

### Dangerous Files and Directories

The system maintains explicit blocklists of sensitive targets:

```typescript
// src/utils/permissions/filesystem.ts:57-79
export const DANGEROUS_FILES = [
  '.gitconfig', '.gitmodules', '.bashrc', '.bash_profile',
  '.zshrc', '.zprofile', '.profile', '.ripgreprc',
  '.mcp.json', '.claude.json',
] as const

export const DANGEROUS_DIRECTORIES = [
  '.git', '.vscode', '.idea', '.claude',
] as const
```

These files are protected from auto-editing because they can be used for code execution (shell configs execute on login, `.gitconfig` can set `core.fsmonitor` hooks) or data exfiltration (MCP configs can add malicious servers).

### Path Resolution and TOCTOU Prevention

The `validatePath` function (pathValidation.ts line 373) implements several anti-bypass measures:

1. **Tilde expansion**: Only `~` and `~/` are expanded. Variants like `~root`, `~+`, `~-` are rejected because they create a TOCTOU gap -- the validator would resolve them differently than the shell.

2. **Shell expansion rejection** (line 423-436): Any path containing `$`, `%`, or starting with `=` is rejected. These trigger shell expansion at execution time but are literal strings during validation.

3. **UNC path protection**: Windows UNC paths (`\\server\share`) that could leak credentials via NTLM authentication are blocked.

4. **Glob pattern handling**: Globs in write operations are blocked entirely. For read operations, the glob's base directory is validated.

5. **Case normalization** (filesystem.ts line 90): On case-insensitive filesystems (macOS, Windows), paths are lowercased before comparison to prevent bypasses via mixed-case paths like `.cLauDe/Settings.locaL.json`.

### Dangerous Removal Protection

`isDangerousRemovalPath` (pathValidation.ts line 331) blocks removal of critical paths: the root directory, home directory, Windows drive roots, direct children of root (`/usr`, `/tmp`), and any wildcard path (`*`, `/path/to/dir/*`).

### Working Directory Enforcement

Files outside the declared working directories require explicit permission rules. The working directory set includes the original CWD, any directories added via `--add-dir` or `/add-dir`, and sandbox write allowlist entries. Each is resolved through symlinks to prevent path aliasing attacks.

---

## 5. Sandbox Integration

The sandbox system is built on `@anthropic-ai/sandbox-runtime`, adapted for Claude Code in `src/utils/sandbox/sandbox-adapter.ts`. The `SandboxManager` class wraps the runtime with Claude-specific integrations.

### Configuration Translation

`convertToSandboxRuntimeConfig` (sandbox-adapter.ts line 172) translates Claude Code's settings into sandbox-runtime's config format. It extracts:

- **Network restrictions**: Allowed/denied domains from `WebFetch` permission rules and `sandbox.network.allowedDomains` settings
- **Filesystem write restrictions**: From `FileEdit` allow/deny rules, plus always-deny entries for settings files and `.claude/skills`
- **Filesystem read restrictions**: From `FileRead` deny rules and `sandbox.filesystem.denyRead`

### Sandbox Decision Logic

`shouldUseSandbox` in `src/tools/BashTool/shouldUseSandbox.ts` determines whether a command runs sandboxed:

```typescript
// src/tools/BashTool/shouldUseSandbox.ts:130-153
export function shouldUseSandbox(input: Partial<SandboxInput>): boolean {
  if (!SandboxManager.isSandboxingEnabled()) return false
  if (input.dangerouslyDisableSandbox &&
      SandboxManager.areUnsandboxedCommandsAllowed()) return false
  if (!input.command) return false
  if (containsExcludedCommand(input.command)) return false
  return true
}
```

The `dangerouslyDisableSandbox` parameter is a per-tool-call escape hatch, but it only works if the sandbox policy allows unsandboxed commands. The `containsExcludedCommand` function is explicitly documented as NOT a security boundary -- it is a convenience feature, and the comment at line 20-21 states: "It is not a security bug to be able to bypass excludedCommands."

### Bare Git Repository Attack Prevention

A noteworthy defense is the bare-git-repo attack mitigation (sandbox-adapter.ts lines 259-280). Git's `is_git_directory()` treats any directory with `HEAD`, `objects/`, and `refs/` as a bare repository. An attacker who plants these files (plus a `config` with `core.fsmonitor`) could achieve code execution when git runs outside the sandbox. The sandbox blocks writes to these files and scrubs any newly-planted ones after command execution.

### Settings File Protection

The sandbox always denies writes to all settings files across all sources:

```typescript
// src/utils/sandbox/sandbox-adapter.ts:232-235
const settingsPaths = SETTING_SOURCES.map(source =>
  getSettingsFilePathForSource(source),
).filter((p): p is string => p !== undefined)
denyWrite.push(...settingsPaths)
```

This prevents a sandboxed command from modifying Claude Code's own configuration, which would be a sandbox escape vector (e.g., adding malicious allow rules).

---

## 6. YOLO Mode / Auto Mode Analysis

The auto-approve classifier (colloquially "YOLO mode") is the most architecturally interesting part of the security model. It replaces human approval with an AI classifier that reads the full conversation transcript and the pending tool call, then decides whether to allow or block.

### Classifier Architecture

The classifier is implemented in `src/utils/permissions/yoloClassifier.ts`. It is a side-query to a separate model instance. The system prompt is loaded from template files:

- `yolo-classifier-prompts/auto_mode_system_prompt.txt`: Base instructions
- `yolo-classifier-prompts/permissions_external.txt`: Default rules for external users
- `yolo-classifier-prompts/permissions_anthropic.txt`: Stricter rules for Anthropic internal use

In the open-source build, these template files are empty (stubbed out), and the classifier feature is gated behind `feature('TRANSCRIPT_CLASSIFIER')` which evaluates to false.

### Decision Flow

The auto mode decision flow (permissions.ts lines 519-700+) follows a precise sequence:

1. **Safety check immunity**: Non-classifier-approvable safety checks (e.g., Windows path bypasses) are never auto-approved, even in auto mode.

2. **Accept-edits fast path**: Before invoking the classifier, the system checks if the action would be allowed in `acceptEdits` mode. If so, it skips the expensive API call. This is the "fast path."

3. **Safe tool allowlist**: Tools like FileRead, Grep, Glob, and task management tools are on a hardcoded allowlist (`SAFE_YOLO_ALLOWLISTED_TOOLS` in classifierDecision.ts line 56-94) and bypass the classifier entirely.

4. **Classifier invocation**: The full conversation transcript is sent to the classifier model, which evaluates the pending action against the configured rules.

5. **Fail-closed behavior**: If the classifier API call fails, the system fails closed -- the action is denied or falls back to prompting.

### User-Configurable Rules

Users can customize the classifier's behavior through `settings.autoMode`:

```typescript
// src/utils/permissions/yoloClassifier.ts:85-89
export type AutoModeRules = {
  allow: string[]
  soft_deny: string[]
  environment: string[]
}
```

The `allow` rules tell the classifier what actions to auto-approve. The `soft_deny` rules tell it what to block. The `environment` section provides context about the user's setup.

### Dangerous Pattern Stripping

When entering auto mode, the system strips permission rules that would be dangerous if auto-approved. `src/utils/permissions/dangerousPatterns.ts` defines patterns like interpreters (`python`, `node`, `ruby`), package runners (`npx`, `bunx`), shells (`bash`, `sh`, `zsh`), and execution wrappers (`eval`, `exec`, `env`, `xargs`, `sudo`):

```typescript
// src/utils/permissions/dangerousPatterns.ts:18-42
export const CROSS_PLATFORM_CODE_EXEC = [
  'python', 'python3', 'python2', 'node', 'deno', 'tsx',
  'ruby', 'perl', 'php', 'lua',
  'npx', 'bunx', 'npm run', 'yarn run', 'pnpm run', 'bun run',
  'bash', 'sh', 'ssh',
] as const
```

For Anthropic-internal users, the list is even longer, adding `gh`, `curl`, `wget`, `git`, `kubectl`, `aws`, `gcloud`, and `gsutil`. This is grounded in empirical data from internal sandbox telemetry, as the comments explain.

The rationale is sound: a rule like `Bash(python:*)` effectively allows arbitrary code execution, which the auto-mode classifier should not silently approve.

---

## 7. Network Access Control

### WebFetch Domain Restrictions

The `WebFetch` tool has a layered domain approval system:

1. **Preapproved hosts** (`src/tools/WebFetchTool/preapproved.ts`): A hardcoded set of documentation domains (docs.python.org, react.dev, developer.mozilla.org, etc.) that are allowed for GET requests only. The comment at lines 1-12 is explicit:

```
// SECURITY WARNING: These preapproved domains are ONLY for WebFetch (GET requests only).
// The sandbox system deliberately does NOT inherit this list for network restrictions,
// as arbitrary network access (POST, uploads, etc.) to these domains could enable
// data exfiltration.
```

2. **User-configured domains**: Users can add `WebFetch(domain:example.com)` to their allow rules. These are respected by both the application-level permission system and the sandbox network proxy.

3. **Domain check**: For non-preapproved domains, a preflight check verifies the domain is safe before fetching.

### SSRF Guard

`src/utils/hooks/ssrfGuard.ts` implements an SSRF guard for HTTP hooks. It blocks private, link-local, and non-routable address ranges:

```typescript
// src/utils/hooks/ssrfGuard.ts:42-53 (summary)
// Blocked: 0.0.0.0/8, 10.0.0.0/8, 100.64.0.0/10, 169.254.0.0/16,
//          172.16.0.0/12, 192.168.0.0/16, fc00::/7, fe80::/10
// Allowed: 127.0.0.0/8 (loopback), ::1
```

Notably, loopback is explicitly allowed because local dev servers are a primary HTTP hook use case. The guard also handles IPv4-mapped IPv6 addresses (`::ffff:169.254.169.254`) to prevent bypasses through address format tricks.

The implementation uses a dns.lookup callback (`ssrfGuardedLookup` at line 216) that validates the resolved IP before the connection is established, eliminating DNS rebinding attacks.

### Sandbox Network Proxy

When the sandbox is enabled, all network traffic from sandboxed commands goes through a network proxy. The proxy enforces its own domain allowlist, derived from the user's permission rules and `sandbox.network.allowedDomains` settings. The `shouldAllowManagedSandboxDomainsOnly` setting (sandbox-adapter.ts line 152) allows policy-level lockdown where only domains from policy settings are permitted.

---

## 8. MCP Server Security

MCP (Model Context Protocol) servers represent a significant attack surface because they are third-party code running with tool-call privileges.

### Permission Namespacing

MCP tools are namespaced as `mcp__serverName__toolName`. Permission rules can target specific tools or entire servers:
- `mcp__server1__tool1`: Specific tool
- `mcp__server1` or `mcp__server1__*`: All tools from a server

This namespacing prevents a malicious MCP tool from masquerading as a built-in tool. The `getToolNameForPermissionCheck` function ensures that in skip-prefix mode (where MCP tools have unprefixed display names), rules targeting builtins do not match MCP replacements.

### Channel Permission Relay

`src/services/mcp/channelPermissions.ts` implements permission approval via external channels (Telegram, iMessage, Discord). The security model here is carefully documented:

- The approval requires a structured event from the server (`notifications/claude/channel/permission`), not raw text parsing
- Each request gets a unique 5-character ID from a 25-letter alphabet (excluding 'l' to avoid confusion with 1/I)
- The trust boundary is the server allowlist (`tengu_harbor_ledger`), not the channel itself
- The code explicitly acknowledges (lines 17-23) that a compromised channel server could fabricate approvals, but argues this is accepted risk since a compromised channel already has unlimited conversation-injection turns

### Sandbox Integration for MCP

The sandbox network restrictions apply to MCP tool execution. The sandbox converts `WebFetch(domain:*)` permission rules into network allowlist entries, but preapproved WebFetch domains are deliberately NOT included in the sandbox network allowlist. This separation is a defense-in-depth measure.

---

## 9. Security Observations

### What is Well-Designed

**Defense in depth**: The system does not rely on any single layer. A Bash command passes through security checks (bashSecurity.ts), read-only validation, mode-based checks, path validation, permission rules, the classifier (if in auto mode), and finally the sandbox. Each layer catches different attack vectors.

**Fail-closed philosophy**: Throughout the codebase, errors default to blocking. Classifier API failures trigger denials. Unparseable commands require approval. Ambiguous permission situations result in prompts.

**TOCTOU awareness**: The path validation code shows sophisticated awareness of time-of-check-to-time-of-use vulnerabilities. Shell expansion syntax (`$HOME`, `%TEMP%`), tilde variants (`~root`, `~+`), and symlink traversals are all handled. The case-normalization for macOS/Windows is a detail that many security systems miss.

**Zsh attack surface coverage**: The `ZSH_DANGEROUS_COMMANDS` set in bashSecurity.ts is remarkably thorough. Blocking `zmodload` at the command level prevents an entire class of module-based attacks (mapfile, zpty, ztcp, zsh/files builtins). The defense-in-depth approach of also blocking the individual module builtins covers the case where a module is pre-loaded.

**Bare git repository attack prevention**: The sandbox-adapter.ts code that prevents bare-git-repo attacks is an example of the kind of obscure attack vector that most tools would miss entirely. The comment explains the exact mechanism (git's `is_git_directory()` + `core.fsmonitor`) and handles the edge case where blocking writes to non-existent files would create stubs.

**Settings file self-protection**: The sandbox unconditionally denies writes to its own settings files and `.claude/skills`, preventing sandboxed commands from modifying Claude Code's configuration.

### Potential Concerns

**Complexity as attack surface**: The bashSecurity.ts file alone handles 23+ distinct security check categories. The interaction between quote stripping, heredoc parsing, compound command splitting, and the various shell-specific expansions creates a very large validation surface. The comments themselves acknowledge edge cases (e.g., the `stripSafeRedirections` security note at line 177 about `/dev/nullo` bypass potential).

**Classifier as security boundary**: In auto mode, an AI model makes security decisions. The classifier sees the full conversation transcript and pending action, but it is fundamentally a probabilistic system making deterministic security decisions. The denial tracking limits (3 consecutive, 20 total) provide a safety valve, but the classifier itself could be prompt-injected by a malicious repository's contents already in the conversation context.

**External build stubbing**: In the open-source build, the bash classifier (`bashClassifier.ts`) is completely stubbed out -- `isClassifierPermissionsEnabled()` always returns false, and `classifyBashCommand` always returns `{matches: false}`. The YOLO classifier prompts are empty files. This means external users running from source do not get the classifier protections that Anthropic internal users get, and must rely on the static rule system and sandbox alone.

**excludedCommands is not a security boundary**: The `shouldUseSandbox` code explicitly documents (line 20-21) that `excludedCommands` is a user convenience feature, not a security control. A user might misconfigure this thinking it provides security isolation.

**Preapproved domains include upload-capable sites**: The preapproved hosts list includes domains like nuget.org and others that support file uploads. While the preapproval only applies to WebFetch (GET requests), the explicit warning in the code suggests awareness that this is a risk boundary worth monitoring.

### Comparison to Other Agent Systems

Compared to other coding agents:

- **vs. Cursor/Windsurf**: Most IDE-based agents do not implement OS-level sandboxing. Claude Code's `@anthropic-ai/sandbox-runtime` integration (likely using bubblewrap/bwrap on Linux, seatbelt on macOS) provides a real isolation boundary rather than just application-level permission checks.

- **vs. Devin**: While both implement permission modes, Claude Code's permission rule system is more granular (tool-level, prefix-based, wildcard patterns) and allows rules from multiple sources with clear precedence (policy > project > user > session).

- **vs. OpenAI Codex CLI**: Claude Code's bash security analysis is significantly more thorough, covering Zsh-specific attack vectors, heredoc edge cases, and shell metacharacter escapes that simpler implementations would miss.

The main architectural differentiator is the layered approach: application-level permission rules, an AI classifier, and OS-level sandboxing all operate simultaneously. A bypass at one layer (e.g., tricking the classifier) is still caught by another (the sandbox blocks the file write). This is the correct architecture for a system where the threat model includes both accidental damage and adversarial prompt injection.
