# Claude Code's "Undercover Mode": How Anthropic Hides AI Authorship in Open Source

*Dissecting the most controversial feature found in the Claude Code v2.1.88 source*

## What It Is

Of everything in the Claude Code source, this is the feature that stands out most starkly. Undercover mode is a system that, when active, instructs the AI to strip all traces of AI involvement from its output — commit messages, PR descriptions, everything. The model is told to present its work as if a human developer wrote it.

Source: `src/utils/undercover.ts`

## When It Activates

Here's the activation logic, and it's worth reading carefully:

```typescript
// src/utils/undercover.ts:28-37
export function isUndercover(): boolean {
  if (process.env.USER_TYPE === 'ant') {
    if (isEnvTruthy(process.env.CLAUDE_CODE_UNDERCOVER)) return true
    // Auto: active unless positively confirmed internal repo
    return getRepoClassCached() !== 'internal'
  }
  return false
}
```

The key details:

- **It only applies to Anthropic employees** (`USER_TYPE === 'ant'`) — external users never trigger this
- **It's on by default** for any repo that isn't on Anthropic's internal allowlist
- **There is no force-off switch.** The code comments explicitly state: "There is NO force-OFF. This guards against model codename leaks"
- **Dead-code-eliminated** in external builds, so it never even runs outside Anthropic

That "no force-off" detail is telling. They're more worried about accidental codename leaks than about giving employees the choice to attribute AI contributions.

## The Actual Prompt

Here is what the model receives when undercover mode is active. The full text is quoted because the framing matters:

```typescript
// src/utils/undercover.ts:39-69
export function getUndercoverInstructions(): string {
  return `## UNDERCOVER MODE — CRITICAL

You are operating UNDERCOVER in a PUBLIC/OPEN-SOURCE repository. Your commit
messages, PR titles, and PR bodies MUST NOT contain ANY Anthropic-internal
information. Do not blow your cover.

NEVER include in commit messages or PR descriptions:
- Internal model codenames (animal names like Capybara, Tengu, etc.)
- Unreleased model version numbers (e.g., opus-4-7, sonnet-4-8)
- Internal repo or project names (e.g., claude-cli-internal, anthropics/…)
- Internal tooling, Slack channels, or short links (e.g., go/cc, #claude-code-…)
- The phrase "Claude Code" or any mention that you are an AI
- Any hint of what model or version you are
- Co-Authored-By lines or any other attribution

Write commit messages as a human developer would — describe only what the code
change does.

GOOD:
- "Fix race condition in file watcher initialization"
- "Add support for custom key bindings"

BAD (never write these):
- "Fix bug found while testing with Claude Capybara"
- "1-shotted by claude-opus-4-6"
- "Generated with Claude Code"
- "Co-Authored-By: Claude Opus 4.6 <…>"`
}
```

"Do not blow your cover." That's the phrase that stands out. The model is being framed as a covert operative, not a tool being used transparently.

## How the Attribution System Supports It

The attribution system (`src/utils/attribution.ts`, `src/utils/commitAttribution.ts`) works hand-in-hand with undercover mode. When the model name might be a codename, it falls back to a safe public name:

```typescript
// src/utils/attribution.ts:70-72
// @[MODEL LAUNCH]: Update the hardcoded fallback model name below
// (guards against codename leaks).
// For external repos, fall back to "Claude Opus 4.6" for unrecognized models.
```

There's even a masking function for model names that might slip through:

```typescript
// src/utils/model/model.ts:386-392
function maskModelCodename(baseName: string): string {
  // e.g. capybara-v2-fast → cap*****-v2-fast
  const [codename = '', ...rest] = baseName.split('-')
  const masked = codename.slice(0, 3) + '*'.repeat(Math.max(0, codename.length - 3))
  return [masked, ...rest].join('-')
}
```

Multiple layers of defense. They really don't want codenames getting out.
