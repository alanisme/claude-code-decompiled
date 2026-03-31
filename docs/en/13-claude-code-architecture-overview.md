# Claude Code Architecture Overview: How the Whole System Fits Together

If you read the existing reports one by one, you can reconstruct most of Claude Code's internals. But there is still a gap between understanding the parts and understanding the whole. This document is meant to close that gap.

It is a synthesis of the codebase as a production AI coding agent: how input enters the system, how the prompt is assembled, how the model loop runs, how tools execute, how permissions are enforced, how context pressure is managed, how state is persisted, and how external systems like MCP plug into the runtime.

The goal here is not to repeat every lower-level detail from the topic-specific reports. It is to show how the major subsystems fit together as one coherent design.

---

## 1. The System at a Glance

At a high level, Claude Code is not just a chat client with tool calls attached. It is a layered runtime for long-lived, tool-using, permission-aware, stateful agent sessions.

You can think of the architecture as seven major layers:

1. **Entry and session bootstrap**
2. **Prompt assembly and environment injection**
3. **Main agent loop**
4. **Tool runtime and permission enforcement**
5. **Context management and compaction**
6. **Persistence and memory**
7. **Integration surfaces and remote control**

The reason the codebase feels large is that each of those layers is real. None of them are thin wrappers.

### A Working Mental Model

```text
User / SDK / IDE input
  -> session bootstrap and runtime state
  -> system prompt assembly
  -> query loop
  -> model streaming
  -> tool discovery / permission checks / execution
  -> tool results back into conversation
  -> compaction / persistence / memory updates
  -> next turn or terminal state
```

This cycle repeats until the system reaches a terminal condition such as completion, abort, max turns, prompt-too-long failure, model failure, or an explicit stop path.

---

## 2. Entry Layer: More Than a CLI Wrapper

The obvious external face of Claude Code is the terminal UI, but internally there are multiple entry paths:

- CLI usage
- SDK or headless usage
- bridge and remote-session paths
- internal feature-gated modes

The entry layer sets up:

- the working directory and project root
- session identity
- permission mode
- model settings
- telemetry providers
- state stores
- command context
- UI and streaming observers

This is important because Claude Code is not stateless. Before the first model request is made, the runtime has already decided where the session lives, how permissions will behave, what directories are in scope, and what integrations are available.

In other words, the first prompt is not the first thing that happens. The runtime environment is established first.

---

## 3. Prompt Assembly: The Real Front Door

The true front door to Claude Code is the system prompt pipeline.

What reaches the model is not a single handwritten prompt, but a composed array of prompt segments that includes:

- base identity and safety framing
- behavioral coding instructions
- tool descriptions
- environment information
- memory and CLAUDE.md content
- session-specific guidance
- MCP instructions
- model-specific behavior patches
- dynamic runtime metadata

This matters because Claude Code's behavior is not controlled primarily by the UI. It is controlled by the runtime prompt assembly process.

### Why the Prompt Pipeline Is Architecturally Important

The prompt system is doing at least five jobs at once:

1. **Defining policy**  
   What the model is allowed to do, how cautious it should be, and how it should communicate.

2. **Teaching the model the runtime contract**  
   How tools behave, how permission prompts work, what system reminders mean, and how compaction affects the session.

3. **Injecting local instructions**  
   CLAUDE.md and memory files can override or refine default behavior.

4. **Shaping model behavior around known weaknesses**  
   Internal comments clearly show prompt patches added to compensate for specific model tendencies.

5. **Optimizing cost and latency through prompt caching**  
   The static/dynamic boundary in the prompt is not cosmetic. It is an infrastructure choice.

Prompt engineering here is not a thin product layer. It is one of the central control planes of the system.

---

## 4. The Main Agent Loop: The Operational Core

Once the runtime and prompt are ready, the system enters the main query loop.

This loop is where:

- user messages enter
- model requests are sent
- streaming responses are consumed
- tool requests are detected
- errors are recovered
- context pressure is handled
- next-turn state is decided

The key architectural decision is that Claude Code treats the model interaction as a **streaming state machine**, not as a simple request/response operation.

### Why This Design Matters

A production coding agent has to handle:

- partial model output
- mid-stream tool calls
- interrupted sessions
- changing context budgets
- fallback model paths
- withheld recoverable errors
- tool failures that may or may not invalidate sibling work

A single function call abstraction is too weak for this. That is why the main loop carries explicit state across turns and transitions.

This is also why `query.ts` is huge. Its size is not just accidental sprawl. It reflects the fact that the system's control flow is concentrated there.

---

## 5. Tool Runtime: Claude Code Is Really a Tool-Oriented Agent

At the product level, Claude Code looks like a coding assistant. At the runtime level, it is much closer to a tool-oriented execution engine driven by a model.

The tool system provides:

- a shared tool contract
- schema validation
- prompt descriptions for model-side usage
- permission integration
- concurrency metadata
- result mapping
- large-result persistence
- deferred tool loading

This means tool calls are not just loosely attached capabilities. They are first-class runtime units.

### The Real Design Choice

The most important tool-system choice in Claude Code is not that there are many tools. It is that every tool must describe itself in a structured way:

- what input it accepts
- whether it is read-only
- whether it is concurrency-safe
- how it should appear to the model
- how permissions should be checked
- how results should return to the conversation

That creates a real execution runtime instead of a bag of ad hoc functions.

### Streaming Execution Changes the User Experience

When streaming tool execution is enabled, the system does not wait for the model to fully finish before starting tools. Instead:

- tool-use blocks are detected as they stream in
- concurrency-safe tools can begin immediately
- completed results can be yielded while the assistant is still streaming

That reduces latency and makes the product feel much more responsive.

This is one of the most important implementation details in the whole codebase because it directly changes the perceived intelligence of the system.

---

## 6. Permissions and Sandboxing: The Safety Boundary

Without the permission system, Claude Code would just be a model with local machine access. That is not a usable product.

The permission model adds several layers:

- high-level permission modes
- allow rules and deny rules
- tool-specific permission checks
- path validation
- command parsing and read-only detection
- sandbox integration
- internal classifier-assisted decisions in some modes

The big design principle is that permission handling is not centralized in one place only. It is layered:

- some policies are global
- some are tool-specific
- some are path-based
- some are shell-command-specific
- some depend on runtime mode

This layered design is necessary because risk is not uniform. A file read is not a shell command. A shell command is not an MCP tool. An MCP tool is not a CLAUDE.md read. Each surface needs its own guardrails.

### Why Bash Gets Special Treatment

Bash is the most dangerous and least structured surface in the runtime. That is why its validation path is far deeper than other tools.

This is not overengineering. It is the minimum needed to make shell access tolerable in an AI-driven environment.

---

## 7. Context Management: Long Sessions Are a First-Class Problem

Claude Code clearly assumes that long-running sessions are normal, not exceptional.

That single assumption changes the architecture in major ways.

The system needs to manage:

- tool-heavy conversations
- growing message history
- dynamic attachments
- model context limits
- prompt cache efficiency
- auto-compaction triggers
- fallback when compaction itself fails

This is why context management shows up in so many places:

- token estimation
- pre-request pruning
- microcompact and snip logic
- reactive and automatic compact paths
- session memory compaction
- post-compact cleanup

In many AI applications, context handling is a side utility. In Claude Code, it is a core subsystem.

### The Deep Design Insight

Claude Code does not treat the context window as static capacity. It treats it as a fluctuating runtime resource that must be budgeted, monitored, and periodically restructured.

That is exactly the right mental model for long-lived coding agents.

---

## 8. Persistence and Memory: Sessions Are Meant to Survive

Claude Code persists much more than casual users might assume.

It stores:

- conversation transcripts
- session metadata
- settings
- permissions
- memory files
- project-scoped history
- compaction boundaries
- resume and fork context

This makes sense because the product is not trying to behave like a stateless chatbot. It is trying to act like a working environment.

### Two Different Kinds of Memory

The architecture distinguishes between:

- **instructional memory**  
  CLAUDE.md and related files that shape future model behavior

- **operational memory**  
  session transcripts, extracted memory, and persistence mechanisms that preserve continuity

This distinction matters because not all persisted state serves the same role.

One kind of memory tells the model what to care about. The other allows the runtime to keep going across time.

---

## 9. MCP and External Integrations: Claude Code as a Platform

The MCP subsystem reveals an important architectural truth: Claude Code is not only a local coding assistant. It is also a client platform for external tool ecosystems.

Through MCP it can:

- discover external tools
- connect through multiple transports
- authenticate with OAuth
- refresh tool and resource inventories
- integrate external capabilities into the model's tool space

This pushes Claude Code beyond a monolithic application and toward a platform model.

### Why This Matters Strategically

Once MCP tools are first-class citizens in the runtime:

- the local built-in tools are no longer the full capability boundary
- the product can expand without shipping every tool inside the base package
- the model's world can be extended dynamically per environment

That is a significant architectural direction, not just a protocol feature.

---

## 10. Remote Settings, Flags, and Control Planes

Several reports highlight a broader point: Claude Code is not controlled only by local code and local settings.

There are multiple control planes:

- static code
- system prompts
- local config
- project config
- managed config
- feature flags
- remote settings

This makes the runtime more operationally flexible, but it also makes it more complex to reason about. Behavior can change because of:

- code changes
- prompt changes
- feature gates
- remote flags
- admin-managed settings
- connected MCP servers

That is exactly how modern SaaS-style agent products behave in practice.

---

## 11. What the Architecture Optimizes For

After reading across the codebase, Claude Code appears to be optimized for the following priorities:

### 1. Real-world robustness over conceptual neatness

The code frequently chooses practical error recovery over elegant abstraction.

### 2. Tool-use effectiveness over pure chat quality

The system is structured around tool execution, not around conversational minimalism.

### 3. Long-session continuity over short-turn simplicity

Persistence, compaction, and memory exist because sessions are expected to last.

### 4. Operational control over static product behavior

Feature flags, prompt patches, and remote settings all point in this direction.

### 5. Safety layers over a single trust mechanism

Permissions, validation, rules, classifiers, and sandboxing are all stacked rather than collapsed into one gate.

These are the priorities of a production agent, not a demo.

---

## 12. Where the Complexity Comes From

The system's complexity is not coming from one source. It comes from the interaction of several hard problems:

- model non-determinism
- tool execution
- local-machine risk
- long conversations
- state persistence
- prompt behavior shaping
- external integrations
- UI and SDK support

A codebase like this gets complex because each layer multiplies the complexity of the others.

For example:

- streaming matters more once tools can execute mid-stream
- permissions matter more once tools can modify the machine
- persistence matters more once sessions can be resumed
- compaction matters more once transcripts are long-lived
- prompt design matters more once tools and policies are injected dynamically

That interaction effect is the real architectural story.

---

## 13. Final Assessment

Claude Code is best understood as a **production-grade agent runtime** with a terminal interface, not as a terminal UI with a model attached.

Its architecture combines:

- prompt control
- stateful execution
- tool runtime design
- permission enforcement
- context compression
- persistence
- external integration
- operational control surfaces

The result is not a small or elegant toy. It is a deeply practical system shaped by product pressure, failure recovery, latency concerns, and the realities of letting a model act on a developer's machine.

That is what makes it worth studying.

For deeper dives into specific subsystems, continue with:

- [06-agent-loop-deep-dive.md](06-agent-loop-deep-dive.md)
- [07-tool-system-architecture.md](07-tool-system-architecture.md)
- [08-permission-security-model.md](08-permission-security-model.md)
- [09-system-prompt-engineering.md](09-system-prompt-engineering.md)
- [10-mcp-integration.md](10-mcp-integration.md)
- [11-context-window-management.md](11-context-window-management.md)
- [12-state-management.md](12-state-management.md)
