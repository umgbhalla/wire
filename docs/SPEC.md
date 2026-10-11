# Wire algebra: design draft 0

Specification only. Types below use language-neutral algebraic notation. They are
not executable code or a claim of native protocol parity. The future codebase is
Rust; Python is a thin PyO3 projection of the same semantics.

## Study motivation

Preserve the native behavior the user values in Codex and Claude. Study the
observed spiralling in Pi/opencode through intent, context lineage, activity and
evidence before considering fixes. This is a user-observed hypothesis, not a
measured behavior conclusion. Types describe observable relationships, not the
model's hidden understanding.

The [first SDK composition study](research/COMPOSITION.md) distinguishes transcript
archives from execution-state stores. It adds questions, not adopted guarantees.
Pi Durable 1.0 is the durability reference; coding-agent RPC is a separate layer.

## Identity

| Type | Meaning | Distinct from |
| --- | --- | --- |
| ChatId | Wire conversation within a WorkspaceId | Native session ID |
| NativeRef | Harness + host/account scope + native identity | Arbitrary path or executable |
| ActivityId | Logical work | Low-level attempt |
| CommandId | Identity of one caller intent | Native idempotency guarantee |
| Epoch | Lifetime of a source sequence | Database revision |
| Position | Source + epoch + sequence | Timestamp |
| Revision | Committed Wire state version per chat | Native transcript revision |
| CheckpointId | Stored projection boundary | Filesystem restore point |

Every record has an owner and stable identity. Timestamps are metadata, not causal
ordering. Native evidence may be incomplete; an adapter reports its fidelity.

## Minimum families

```text
Wire
|-- Identity       Workspace, Chat, NativeRef, Activity, Command, Position
|-- Content        Message, Part, ArtifactRef, Interaction
|-- Activity       State, Command, Fact, Receipt
|-- History        Record, Checkpoint, Branch, Coverage
|-- Storage        Commit, ExpectedRevision, CommitOutcome, Query
`-- Compatibility Protocol, Capability, Requirement, Assessment
```

Use explicit sums/products. Each variant carries only meaningful fields. Boundary
decoding validates them. Unknown content is retained with source/version; it is not
silently coerced into a known variant.

## Activity algebra

Separate execution, connection and delivery. One status string cannot express
these independently without ambiguity.

```text
Execution = Idle
          | Active(ActivityId, Phase)
          | Settled(ActivityId, Outcome)

Phase = Starting | Running | Waiting(InteractionId) | Stopping
Outcome = Completed(ResultRef) | Failed(Failure) | Cancelled

Connection = Attached(Epoch) | Detached(LastPosition) | Unavailable(Reason)

Delivery = Pending(CommandId)
         | Acknowledged(CommandId, Acceptance)
         | Rejected(CommandId, Reason)
         | Unknown(CommandId, ReconcileFrom)

Acceptance = Started(ActivityId) | Queued(QueueRef) | Handled(ReceiptRef)
```

Detached does not imply cancelled. Acknowledged does not imply completed. In Pi,
a prompt response can mean started, queued or handled; `agent_end` is not necessarily
logical settlement. Adapters own those mappings. They cannot emit Completed without
an established terminal guarantee.

```text
Idle --accepted start--> Active(Starting)
Starting --started--> Running --interaction--> Waiting
Waiting --resolved--> Running
Starting / Running / Waiting --stop requested--> Stopping
Active --authoritative terminal fact--> Settled
Settled --new accepted activity--> Active(Starting)

connection loss changes Connection; it does not invent an execution transition
```

Admission, transport delivery and native acceptance differ. A lost acknowledgement
can mean Unknown. Resolve it with a receipt/state lookup. Never convert uncertainty
into definite rejection or retry the mutation with a fresh ID.

This draft requires complete transition tables before approval. Queues and native
retries must not be disguised as additional global states without evidence.

## Commands and facts

```text
Command = Start(TurnIntent)
        | SendInput(ActivityId, Input)
        | Respond(ActivityId, InteractionId, Answer)
        | Interrupt(ActivityId)
        | Cancel(ActivityId)
        | Configure(SettingsPatch, ApplyAt)
        | Fork(NativeBoundary, TargetChatId)
        | Compact(ChatId)

ApplyAt = NextActivity | Live(ActivityId)

Fact = ActivityStarted | ContentChanged | InteractionOpened
     | InteractionResolved | InputAccepted | CompactionRecorded
     | ActivitySettled | NativeLinkChanged | CoverageChanged
```

Command envelopes contain identity, owner, expected state/revision and typed intent.
Fact envelopes contain source identity and position. A request is not proof of its
effect. Version-specific native details stay in adapters.

The future pure decision function maps `(state, command, compatibility)` to
Rejected(reason) or Admitted(intent). The reducer maps `(state, fact)` to
Applied(state) or a typed conflict/gap. I/O happens outside both. No callback runs
inside a transaction or pure reducer. Exact payload types remain a design gate.

## History and reconstruction

```text
HistoryRecord = MessageVersion | ObservedFact | NativeLink
              | BranchCreated | CheckpointSaved | ArchiveDeleted

Coverage = Complete(Boundary)
         | Partial(Boundary, NonEmpty<Gap>)
         | Unknown(Reason)

Reconstruction = { projection, covered_positions, revision, coverage }
```

Complete means complete for a stated source scope and boundary, not all hidden
agent state. Gaps require the source's continuity semantics: Pi commit sequences permit gaps,
so missing integers alone do not prove lost records. Other gaps include inaccessible history, omitted content,
redaction and missing artifact bytes.

Reconcile history and stream output by stable message/part/tool identities. Same
identity and same evidence is a no-op; conflicting evidence is a conflict. Do not
append one response twice. A new source epoch cannot replace an old retry cursor.

Checkpoint plus tail must equal full replay through the same boundary. Reconstruction
never executes tools or starts a native session. Snapshot replacement differs from
an incremental update. Unknown facts are retained; if their effects cannot be
understood, coverage cannot remain Complete.

| Operation | Wire archive | Native session | Files |
| --- | --- | --- | --- |
| Reconstruct checkpoint | Read only | No change | No change |
| Archive branch/edit | New lineage | No change | No change |
| Native resume | Record new evidence | Continues when input is sent | May change through tools |
| Native fork | Record returned linkage | New branch | Does not undo prior edits |
| File restore | Record evidence separately | Separate protocol | Restores covered files |

Archive import is not native restoration. Exact restoration from arbitrary messages
is unproven. Fork capabilities must identify safe boundaries and context guarantees.

## Database algebra

Backends receive typed records. JSON-file, SQLite and PostgreSQL are encodings and
storage engines, not competing public models. No backend is selected implicitly.

```text
Commit = {
  chat: ChatKey,
  id: CommitId,
  expected: Revision,
  records: NonEmpty<HistoryRecord>,
  positions: SourcePositions
}

CommitOutcome = Committed(Revision)
              | AlreadyCommitted(Revision)
              | Conflict(CurrentRevision)
              | Rejected(StorageViolation)
              | Unknown(CommitId)

Store operation = Commit | Read(Query) | LoadCheckpoint | LookupCommit
```

Records, projection and cursor positions commit atomically for one chat. A commit
ID binds to its canonical batch intent; different content under the same ID rejects.
Revision CAS does not implement native exactly-once execution. A native effect and
a local write are not one transaction. Track their reconciliation explicitly.

Deletion is revisioned. Physical erasure of older checkpoints/artifacts is a separate
contract to define before advertising it. Cross-chat branch linkage uses durable
retryable receipts; the common interface does not promise cross-chat transactions.

| Backend | Shared law | Backend limit |
| --- | --- | --- |
| JSON-file | Atomic per-chat commit and receipt lookup | Enforced single writer; atomic replacement and durable sync |
| SQLite | Same outcomes and replay | Transactions, unique identities, serialized writer, explicit busy policy |
| PostgreSQL | Same outcomes and replay | Transactions, unique identities, revision CAS or row locks |

Crash before commit preserves the old revision. After durable commit, LookupCommit
recovers the receipt. Lost acknowledgements remain Unknown until reconciled. Errors
distinguish unavailable, corrupt, conflict and unknown outcome. Native credentials
never belong in transcript records.

## Compatibility algebra

```text
Support<T> = Native(T) | Emulated(T, Limits) | Unavailable(Reason) | Unknown

Assessment = Compatible(NegotiatedContract)
           | Degraded(NegotiatedContract, NonEmpty<Limit>)
           | Incompatible(NonEmpty<Reason>)

Requirement = Operation + RequiredGuarantees
```

Support describes guarantees, not just method presence. Separate observation/control,
live/next-turn settings, correlation/idempotency, cancellation request/settlement,
fork/exact restore, and transcript/file recovery.

Assessments record harness/protocol/schema versions, owner scope and evidence.
Unknown does not imply supported. Reject unsupported commands before effects where
possible; preserve native rejection after local admission. Lossy emulation is an
explicit degraded result. Never pass arbitrary native command strings through a
common public control bag.

## Future crate boundaries

These are dependency directions, not directories to scaffold now.

```text
wire-types       identities and sums/products; no I/O or Python types
    ^
wire-core        pure decisions, reduction, history and compatibility
    ^
    +-- wire-store      database contract and backend modules
    +-- wire-codex      Codex protocol translation
    +-- wire-claude     Claude protocol translation
    `-- wire-pi         Pi protocol translation
             ^
           wire         ownership and effect coordination
             ^
         wire-python    PyO3 projections and async lifetime bridge
```

Adapters do not import each other. Core does not depend on adapters, drivers or
Python. SQL backends can be optional wire-store features; split crates only when
dependencies/ownership justify it. Extract shared framing only after actual protocols
prove the same requirements.

Python preserves variant meanings and failure cases. Initial Python target: 3.13,
with 3.14 checks. Rust owns runtime semantics. The binding specifies owned/borrowed
lifetimes, cancellation and asyncio context. No Python reference is held across a
Rust await without an explicit policy. Fluent construction creates immutable typed
commands; one terminal operation submits them. The algebra does not depend on
builder syntax, interpreter or storage engine.

## Laws to resolve before implementation

| Law | Counterexample |
| --- | --- |
| Command is not fact | Failed stdin write recorded as accepted input |
| Acceptance is not completion | Pi prompt receipt counted as finished work |
| Detach is not cancel | Reader disconnect kills native work |
| Exact duplicate is harmless | Replayed tool output appended twice |
| Conflicting duplicate is visible | Same position contains different content |
| Checkpoint replay equals full replay | Text delta applied twice on resume |
| Cursor follows durable content | Cursor committed while records are lost |
| Backend substitution preserves outcomes | JSON silently overwrites revision conflict |
| Capability includes scope | External observation treated as write control |
| Branch preserves source | Archive edit rewrites native source |
| Unknown remains unknown | Timeout reported as definite native rejection |
| Binding preserves semantics | Python task cancellation cancels remote work |

Draft 0 does not prove these laws. Complete transition tables, native boundary
rules, variant payloads, serialization, protocol pinning and erasure semantics are
unfinished specification tasks. Do not implement while these contracts are unsettled.
