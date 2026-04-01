# Memory and Instruction Layers in Claude Code

Claude Code does not rely on one single memory mechanism. It uses multiple layers of stored context, each serving a different purpose.

That distinction is critical. A production coding agent needs at least three different things:

- durable user and project instructions
- continuity across long sessions
- a way to compress history without losing the important parts

Claude Code solves this with a combination of instruction files, automatic memory directories, and session memory extraction paths. This document looks at how those layers interact.

---

## 1. Not All Memory Means the Same Thing

One of the easiest mistakes when reading the code is to treat all persisted context as "memory." Claude Code's architecture is more specific than that.

There are at least three categories:

### 1. Instructional memory

These are files that tell the model how to behave:

- `CLAUDE.md`
- `.claude/CLAUDE.md`
- `.claude/rules/*.md`
- `CLAUDE.local.md`
- managed instruction files

These are not session logs. They are instruction overlays.

### 2. Persistent working memory

This is the file-based memory system centered around `MEMORY.md` and topic files in the project-specific memory directory.

This layer is meant to carry durable project- or user-relevant facts across sessions.

### 3. Session memory

This is the more transient, runtime-oriented memory extraction path used to help compaction and long-session continuity.

This layer is closer to an evolving session summary than a stable instruction base.

Architecturally, those three layers solve different problems and should not be conflated.

---

## 2. CLAUDE.md Is an Instruction Overlay System

The CLAUDE.md subsystem is not just "load a prompt file." It is a layered instruction overlay mechanism.

The loader walks multiple locations and priority levels, including:

- managed machine-level instructions
- user-level instructions
- project-level instructions
- local project-specific instructions

That means Claude Code is designed to allow behavior shaping from several scopes at once:

- organizational policy
- user preference
- repository norms
- private local overrides

This is important because the runtime is trying to behave well in real environments where those scopes often conflict or at least differ.

### Why the Layering Matters

The layering lets Claude Code behave differently depending on:

- the machine it is running on
- the user invoking it
- the repository it is in
- the private local setup not meant to be committed

That makes CLAUDE.md much closer to a behavioral configuration system than to a simple note file.

---

## 3. Include Paths and Rules Make the System More Like Config Infrastructure

The CLAUDE.md loader supports things like:

- include directives
- rule files
- path-conditioned rule application
- comment stripping
- exclusion filters

This is a significant architectural signal.

It means Anthropic did not want a flat "one file of instructions" model. They wanted:

- modularity
- reuse
- scoping
- local control over instruction injection

At that point, the instruction system starts looking less like prompt content and more like configuration infrastructure for model behavior.

---

## 4. The Prompt Pipeline Gives Instruction Files Real Power

The only reason CLAUDE.md matters is that it is injected into the system prompt pipeline with strong wording.

This wording is important because it tells the model to treat those instructions as overriding local behavior guidance.

In practice, that means the instruction layer can meaningfully redirect the model's behavior around:

- code style
- workflow preferences
- project-specific constraints
- repository conventions
- "how to work here" norms

This is one of the most practical pieces of the Claude Code design. Teams often care less about global model intelligence than about whether the assistant behaves correctly within their actual environment.

The instruction system is the bridge between generic agent behavior and local working norms.

---

## 5. The File-Based Memory System Solves a Different Problem

Separate from instruction files, Claude Code includes an automatic memory directory system built around `MEMORY.md`.

This layer is not just for telling the model how to behave. It is for preserving useful long-lived facts in a structured way.

The architecture here is notable because `MEMORY.md` is treated as an index rather than an unlimited note dump.

The intended pattern is:

1. save actual content into specific topic files
2. keep `MEMORY.md` as a compact map of what exists

This is a very good design choice.

If the always-loaded memory entrypoint were allowed to grow without limits, it would quickly become a context-budget sink. By forcing `MEMORY.md` to act as a lightweight index, the system preserves navigability without spending all of its prompt budget on persistent notes.

---

## 6. Hard Limits Are Part of the Design, Not an Afterthought

Claude Code places explicit size and line limits on memory entrypoints.

This tells you the memory system has already encountered the obvious failure mode:

- users or models keep adding memory
- memory keeps growing
- always-loaded context becomes bloated
- useful session work gets crowded out

So the architecture contains hard limits and truncation behavior.

That is exactly the right decision.

Long-term memory in agent systems does not fail because there is too little of it. It often fails because it becomes too large, too stale, and too expensive to keep loading.

Claude Code's memory design clearly understands that risk.

---

## 7. Session Memory Exists Because Compaction Needs Something Better Than Repeated Summarization

The session memory path is especially interesting because it is tightly connected to compaction.

If the runtime had to summarize the entire conversation from scratch every time context pressure rose, it would incur:

- latency
- cost
- repeated information loss
- more opportunities for compaction failure

Session memory offers an alternative:

- incrementally preserve important session facts
- then use that preserved state during compaction

This is a more scalable approach for long-lived sessions.

Architecturally, it turns compaction from "compress everything again" into "replace old context with already-maintained memory plus recent messages."

That is a much better shape for a persistent coding agent.

---

## 8. Instruction Memory and Session Memory Must Stay Separate

One of the most important architectural boundaries in the codebase is the separation between:

- instructions that should always shape future behavior
- session-derived information that should preserve continuity

If you collapse those into one system, several bad things happen:

- ephemeral task details get treated like durable policy
- local preference files become clogged with session residue
- the model loses the difference between "how to behave" and "what has happened"

Claude Code avoids that by preserving the conceptual boundary:

- instruction files shape behavior
- memory files preserve durable facts
- session memory helps continuity and compaction

This is more disciplined than many agent systems.

---

## 9. Memory Is Also a Cost and Context Strategy

The memory architecture is not only about recall. It is also about resource allocation.

A coding agent with long sessions has to decide:

- what belongs in every prompt
- what belongs only when needed
- what should be summarized
- what should be indexed
- what should be discarded

The memory design in Claude Code answers those questions structurally:

- always-loaded instruction files are bounded
- always-loaded memory entrypoints are bounded
- detailed memory can live in separate files
- session memory can stand in for older history during compaction

That makes memory a context strategy, not just a persistence feature.

---

## 10. The Whole System Is Really About Behavioral Continuity

If you zoom out, the memory and instruction architecture is serving one broader goal:

Claude Code should continue behaving coherently across time.

That includes:

- remembering how to work in a repository
- preserving important user and project preferences
- carrying forward key session facts
- staying effective after compaction
- resuming work after interruption

This is behavioral continuity more than literal memory.

That distinction matters because continuity is what users actually experience. They do not care whether the system persisted a file. They care whether it still seems to understand the task, the codebase, and the workflow when work resumes.
