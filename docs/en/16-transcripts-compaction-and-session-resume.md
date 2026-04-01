# Transcripts, Compaction Boundaries, and Session Resume in Claude Code

Long-lived agent sessions only feel coherent if the system can stop, persist, compress, and later resume without losing the thread. Claude Code takes that problem seriously.

This document looks at the session-lifecycle mechanics behind that continuity:

- transcript storage
- parent-linked message chains
- sidechains and branching
- compaction boundaries
- resume and continue behavior
- why these mechanisms are tightly coupled

This is one of the less glamorous parts of the codebase, but it is one of the reasons the product feels like a working environment rather than a disposable chat window.

---

## 1. Session History Is Not Just a Flat Log

A naive assistant would store conversation history as a single append-only array. Claude Code does not.

Its transcript system stores messages with chain structure:

- message UUID
- parent UUID
- additional logical parent information
- sidechain state
- session metadata

That means the transcript is not merely "a list of past messages." It is a record of conversational structure.

This matters because agent sessions do not always move in one straight line.

Users can:

- rewind
- continue from a different point
- compact
- fork
- resume later

A flat log is too weak to represent those transitions cleanly.

---

## 2. Why Parent-Linked Chains Matter

The parent-linked transcript model solves a real product problem: branching history.

Imagine this sequence:

1. the user asks for a plan
2. the model explores files
3. the conversation rewinds
4. a different approach is taken

If you only keep a flat ordered list, the session either:

- loses the earlier branch entirely, or
- keeps it in a way that is hard to distinguish from the active thread

Claude Code's transcript chain makes it possible to preserve historical branches while still reconstructing the active conversation path at load time.

This is a very practical design for a tool-oriented agent, where exploratory dead ends are normal.

---

## 3. Session Materialization Is Delayed on Purpose

One of the cleaner design choices in the storage subsystem is that a session file is not immediately created just because the runtime starts.

Instead, entries can be buffered until a real user or assistant message exists.

This avoids "ghost sessions" where:

- the tool starts
- the user exits immediately
- the filesystem gets littered with empty or meaningless session files

Small choice, but exactly the kind of one that signals the code has been used at scale.

Once you start and quit a terminal assistant a lot, storage hygiene matters.

---

## 4. Compaction Changes the Meaning of History

Compaction is not just a token-saving transformation. It changes the shape of the stored conversation.

After compaction, the system does not want a resumed session to behave as if the full pre-compact history were still active in ordinary form.

So Claude Code introduces explicit compact boundaries and associated metadata.

That metadata can include:

- preserved segment references
- boundary markers
- relationships to prior message chains
- pointers needed to reconstruct the post-compact active state

This is essential because compaction is not equivalent to deletion. It is replacement plus continuity.

The runtime has to remember:

- that compaction happened
- what was preserved directly
- what was summarized
- where the resumed conversation should continue from

Without those markers, session resume after compaction would be ambiguous and error-prone.

---

## 5. Compaction Boundaries Are a Semantic Marker

The compact boundary should be understood as a semantic event, not just a storage artifact.

It means:

- old history has been transformed
- the active future thread now depends on a summary or session memory in place of full prior detail
- some storage cleanup and cache reset work must happen
- downstream consumers such as SDK integrations need to understand that a state shift occurred

This is why the compact boundary logic touches:

- transcript persistence
- runtime state
- compaction metadata
- SDK message adaptation
- cleanup and cache invalidation

If compaction were treated as a cosmetic message insertion, the rest of the system would drift out of sync with reality.

---

## 6. Resume Works Because Persistence and Compaction Are Coupled

Session resume is easy only if session history is simple. Claude Code's history is not simple, because the system supports:

- streaming tool interactions
- stateful session metadata
- sidechains
- compaction
- memory extraction
- forked session behaviors

So resume must reconstruct more than "the last N messages."

It needs to reconstruct:

- the active conversation chain
- the current session identity
- which branch is authoritative
- how compaction changed message continuity
- what post-compact state should look like

This is why persistence and compaction are so tightly coupled. A resume system that ignores compaction semantics would rehydrate the wrong state.

---

## 7. Sidechains Preserve Reality Without Polluting the Active Thread

The sidechain concept is one of the more elegant storage ideas in the codebase.

It lets the transcript keep branches that were once real without forcing them into the active session reconstruction path.

This gives Claude Code two useful properties at once:

- it can preserve actual history
- it can still load a clean active conversation

For an agent product, that is important. Real work is messy. The user changes direction. The model tries things that do not pan out. The system needs a storage model that acknowledges that mess without letting it destroy session continuity.

Sidechains are the compromise.

---

## 8. Compaction Is Also a Persistence Problem

It is tempting to think of compaction as belonging only to context management. But in Claude Code, compaction is just as much a persistence problem.

Once a session is compacted, the storage layer must preserve answers to questions like:

- what part of the old chain is still directly present?
- what was summarized away?
- what metadata is needed for future resume?
- what old messages should no longer be considered part of the active state?

This is one of the reasons Claude Code feels architecturally mature. Its designers seem to understand that "context" and "storage" are not separate worlds in a long-lived agent.

The way you compress conversation history determines how you must persist and reload it later.

---

## 9. Resume, Continue, and Fork Are Different Continuity Operations

From the user's perspective, "resume" may sound like one action. Internally, there are different continuity modes:

- resume the same session
- continue from prior history
- fork into a new branch or session identity

These are not interchangeable.

A system that supports all three needs to be very careful about:

- session IDs
- parent relationships
- overwritten metadata
- what should and should not be inherited

This is why the storage layer has logic that re-stamps or rewrites metadata in some flows. The stored messages may originate in one context but become part of another active session lineage.

That is subtle, but necessary.

---

## 10. Why This Subsystem Is More Important Than It Looks

Users often evaluate AI coding products on:

- model quality
- tool quality
- latency
- UX polish

But once they start using them for serious work, continuity becomes just as important.

A system that cannot reliably:

- stop
- resume
- compact
- preserve work
- stay coherent after branching

will feel brittle, no matter how strong the model is.

Claude Code's transcript and compaction-boundary architecture exists to avoid that brittleness.

It is part of what makes the product feel operationally durable.
