# Inside Claude Code's Hidden Features, Codenames, and Two-Tier User System

*What the decompiled source of Claude Code v2.1.88 reveals about internal model names, secret feature flags, and how Anthropic treats its own employees differently*

## The Animal Codename System

One of the more entertaining findings from digging through the source: Anthropic names their models after animals. And they go to considerable lengths to keep those names from leaking.

### Codenames I Found

| Codename | What It Is | How I Found It |
|----------|-----------|----------------|
| **Tengu** (天狗) | Product/telemetry prefix — appears in 250+ analytics event names and feature flags | Everywhere in the codebase |
| **Capybara** | Sonnet-series model, currently at v8 | `capybara-v2-fast[1m]`, plus prompt patches referencing v8 behavior issues |
| **Fennec** (耳廓狐) | Predecessor to Opus 4.6 | Migration code: `fennec-latest` → `opus` |
| **Numbat** (袋食蚁兽) | Next model launch | A comment literally says "Remove this section when we launch numbat" |

### How They Protect Codenames

The undercover mode (covered in detail in my separate write-up) explicitly lists what must never leak:

```typescript
// src/utils/undercover.ts:48-49
NEVER include in commit messages or PR descriptions:
- Internal model codenames (animal names like Capybara, Tengu, etc.)
- Unreleased model version numbers (e.g., opus-4-7, sonnet-4-8)
```

The build system uses `scripts/excluded-strings.txt` to scan compiled output for accidentally included codenames. What caught my eye is a particularly clever workaround in the buddy system (the virtual pet feature). One of the pet species is "capybara" — which collides with the model codename. So the developers construct the species name at runtime using `String.fromCharCode()` to avoid tripping the build scanner:

```typescript
// src/buddy/types.ts:10-13
// One species name collides with a model-codename canary in excluded-strings.txt.
// The check greps build output (not source), so runtime-constructing the value keeps
// the literal out of the bundle while the check stays armed for the actual codename.
```

This is the kind of accidental-complexity-meets-clever-hack that makes reverse engineering fun.

### Capybara v8 Is Having a Rough Time

The source code is surprisingly candid about Capybara v8's behavioral issues. I found dedicated prompt patches for five distinct problems:

1. **Stop sequence false triggers** — roughly 10% of the time when `<functions>` appears at the end of the prompt. That's a high rate for something that should be reliable. (`src/utils/messages.ts:2141`)

2. **Empty tool_result causes zero output** — the model just goes silent. (`src/utils/toolResultStorage.ts:281`)

3. **Over-commenting** — bad enough that they wrote a dedicated anti-comment prompt. (`src/constants/prompts.ts:204`)

4. **High false-claims rate** — v8 hits 29-30% vs v4's 16.7%. Nearly double. That's a meaningful regression. (`src/constants/prompts.ts:237`)

5. **Insufficient verification** — they had to add a "thoroughness counterweight" prompt to make it actually check its work. (`src/constants/prompts.ts:210`)

Seeing these numbers baked into the source code is fascinating. It gives you a real sense of the tradeoffs involved in shipping newer model versions — faster isn't always better if reliability takes a hit.

## Feature Flags: Deliberately Obscure

All feature flags use the `tengu_` prefix followed by **random word pairs**. This is intentional — it prevents anyone outside Anthropic from guessing what a flag controls just by reading its name. Here's what I managed to map:

| Flag | What It Actually Does |
|------|----------------------|
| `tengu_onyx_plover` | Auto Dream (background memory consolidation) |
| `tengu_coral_fern` | Memdir feature |
| `tengu_moth_copse` | Another memdir switch |
| `tengu_herring_clock` | Team memory |
| `tengu_passport_quail` | Path feature |
| `tengu_slate_thimble` | Another memdir switch |
| `tengu_sedge_lantern` | Away Summary |
| `tengu_frond_boric` | Analytics kill switch |
| `tengu_amber_quartz_disabled` | Voice mode kill switch |
| `tengu_amber_flint` | Agent teams |
| `tengu_hive_evidence` | Verification agent |

The naming convention loosely follows an adjective/material + nature/object pattern. It's a solid approach to operational security — if one of these flag names shows up in network traffic, you can't infer anything about the feature it controls.

## The Internal vs. External User Split

This is probably the most significant finding in the entire codebase. Anthropic employees (`USER_TYPE === 'ant'`) get a substantially different experience.

### Prompt Differences (`src/constants/prompts.ts`)

| What Changes | External Users | Internal (ant) Users |
|-------------|---------------|---------------------|
| Output style | "Be extra concise" | "Err on the side of more explanation" |
| False-claims mitigation | None | Dedicated Capybara v8 patches |
| Length constraints | None | "≤25 words between tools, ≤100 words final" |
| Verification | None | Required verification agent for non-trivial changes |
| Comment guidance | Generic | Dedicated anti-over-commenting prompt |
| Proactive correction | None | "If user has misconception, say so" |

Let that sink in for a second. External users get a stripped-down "be concise" experience, while internal users get the model that actually pushes back on mistakes, verifies its own work, and provides detailed explanations. The verification agent alone is a significant quality-of-life difference — external users don't get that safety net.

### Internal-Only Tools

Internal users also have access to tools that external users simply can't use:

- **REPLTool** — REPL mode
- **SuggestBackgroundPRTool** — background PR suggestions
- **TungstenTool** — performance monitoring panel
- **VerifyPlanExecutionTool** — plan verification
- **Agent nesting** — agents spawning agents

The engineering reasoning probably makes sense (you test features internally before rolling them out), but it does mean the product Anthropic's engineers use daily is meaningfully better than what paying customers get.

## Hidden Slash Commands

I also found some undocumented commands:

| Command | Status | What It Does |
|---------|--------|-------------|
| `/btw` | Active | Ask side questions without breaking the conversation flow |
| `/stickers` | Active | Opens a browser link to order Claude Code stickers (yes, really) |
| `/thinkback` | Active | 2025 Year in Review |
| `/effort` | Active | Set model effort level |
| `/good-claude` | Stub | Placeholder, not yet implemented |
| `/bughunter` | Stub | Placeholder, not yet implemented |

The `/btw` command is genuinely useful if you know it exists. The `/stickers` one is just charming.
