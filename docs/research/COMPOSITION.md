# Composition concepts: source study 1

Analysis only. These are working terms to understand native systems, not adopted
Wire interfaces, implementation promises or guarantees. Keep the set small.

The user's design motivation is to preserve the coherent native behavior they
observe in Codex and Claude, study the failure modes they observe in Pi/opencode,
and only later consider fixes for Pi or opencode v2. That observation is a
hypothesis for trace analysis; this study did not run a model-behavior comparison.

## Start with four questions

| Question | Working term | What it helps separate |
| --- | --- | --- |
| Why is this work being requested? | Intent | User objective from a raw prompt string |
| Which prior work does the harness continue? | Context lineage | Native history/context from a rendered transcript |
| What is actually happening? | Activity | Admission, native attempts, child work and settlement |
| What has been recorded? | Evidence | Observed output, durable state and missing information |

These terms name observable relationships. They do not claim to represent the
model's internal understanding. No new agent loop follows from naming them.

## Native vocabulary, without pretending equivalence

| Relationship | Codex SDK | Claude SDKs | Pi Durable 1.0 |
| --- | --- | --- | --- |
| Conversation identity | Thread / thread ID | Session ID / SessionKey | ConversationId; Session owns shared storage |
| User input admission | turn_start RPC and response | query writes to the native transport | Submission with stored queued/placed/done/unanswered states |
| Work handle | TurnHandle; goal mode groups native turns | ClaudeSDKClient / Query; results plus native task/session state | Submission.wait and a run spanning generation/tool tasks |
| Output content | Typed ThreadItem / Notification | ContentBlock / SDKMessage; raw SessionStoreEntry for native archive | EntryRecord has distinct model and data payloads |
| Child work | Native subagent/thread relationships | Parent tool/agent IDs and background task messages | TaskId with owner, checkpoint, join and background fields |
| Projection | Thread.read and streamed turn items | get_session_messages and streamed messages | ConversationView / document watches / experimental AgentEvent stream |
| Storage ownership | Native runtime history; no store injection in inspected SDK client | SessionStore mirror plus native-format resume materialization | Storage is the Session's atomic persistence boundary |
| Branching | Native thread_fork | fork_session; native or store-backed session fork | Parent conversation + inclusive EntryId ancestry |

A mapping means similar purpose, not identical guarantees. Codex Python and
TypeScript also have different control surfaces: the inspected Python SDK drives
app-server JSON-RPC, while TypeScript drives `codex exec` output. Keep transport
and capability scope in the evidence.

## Storage is the first composition boundary

```text
Codex SDK          --> native runtime --> native thread history

Claude SDK         --> native runtime --> local native transcript
                                           |
                                      SessionStore mirror
                                           |
                              load native entries for resume

Pi Durable Session --> atomic Storage commit
                       entries + documents + tasks + submissions
                                           |
                                publish committed state
```

Claude's inspected SessionStore is present in both languages. It requires append
and load, with optional listing, summaries, deletion and subagent-key discovery.
The Python source describes an external copy after local native writes. Mirror
errors can leave gaps while native work continues. Resume loads raw native records
and materializes temporary JSONL; this is stronger than merely rendering messages,
but it is not permission to treat any normalized Wire archive as native context.

Pi's Storage is not the same abstraction. Session validates/serializes mutations;
storage commits record/document writes atomically. It also exposes task and
submission records. Its Seq orders commits and explicitly permits gaps. Missing
integer values alone cannot prove lost events. JSONL fsync is optional, SQLite uses
synchronous NORMAL, and the documented model is one process owning storage. Do not
infer a concurrent PostgreSQL contract by renaming this interface.

Open design question: does Wire need an archive interface, an execution-state
interface, or a small composition of the two? Do not decide by selecting a driver.

## History is not model context, and neither is a view

Pi makes this distinction visible: EntryRecord contains model-facing messages,
application-facing data, head markers and context edits. Reset/compaction can
change what is supplied to the model while old evidence remains addressable.
Claude's native archive contains more than the filtered user/assistant view.
Codex ThreadItem collections are not necessarily the exact provider prompt.

Working distinction:

- Evidence history: retained source records and their provenance.
- Context lineage: the native continuation/branch/reset boundary.
- View: a derived projection for a reader.

Pi's slow watches can replace pending frames with a current snapshot. A view stream
therefore must not be treated as an immutable audit log. Native records needed for
resume should remain opaque and versioned where their schema is not public.

## Request, wait, attempt and settlement differ

Codex's JSON-RPC ID routes a response. It is not by itself a durable intent key.
Its Python goal transformer can suppress intermediate native completions while
representing one logical operation. Claude's receive_response convenience stops
at ResultMessage, while its internal reader also tracks session state/background
work. Pi distinguishes submission settlement from task settlement and model-turn
completion. Waiting cancellation in Pi does not cancel the work.

Pi durable submission deduplication is scoped to conversation/requestId. The
inspected admission code returns an existing same-type submission; it does not
compare every input payload field before returning it. Wire's proposed stronger
same-ID/same-intent check is a separate design question, not an inherited fact.

## Restart is not automatic safe replay

Pi's tool task commits an execution checkpoint first. On recovery it reruns only
when both the saved intent and current tool registration declare replay safe.
Otherwise it records interruption with possible partial effects. Task outcomes
include failed, aborted, orphaned and faulted; they should not all become one
undifferentiated error. A decided outcome can remain completing until owned work
settles. Definition/version migration and missing registrations add further states.

Candidate questions for the common types:

- Is the command merely delivered, natively accepted, or durably admitted?
- Is a checkpoint for rendering history or for resuming executable work?
- Which native source can establish settlement?
- Which effect is safe to retry, and who supplies that evidence?
- Does a history gap affect the view, native resume, or both?

No answer is fixed yet. Keep native semantics rather than rewriting a generic
agent loop just to make the names uniform.

## Cognitive-load findings

CCCC found complexity concentrated in JSONL recovery, event translation, option
construction, history extraction and logical-run grouping. See
[the scoped report](../analysis/native-sdks/REPORT.md). The seven scans all parsed
without errors; they are not behavior tests or proof of low reading effort.

Study one relationship at a time. Keep native-specific options out of common
commands. Keep transport identities out of storage revisions. Separate archive
resume from task recovery. Do not copy Pi's full extension/task framework merely
because its algebra is useful reference material.

## Source anchors

Pins and reproducible worktree instructions: [WORKTREES.md](WORKTREES.md).

- [Codex Python public handles](https://github.com/openai/codex/blob/9f3d25588b3f0bdf452d4eda919f2c2365a89bba/sdk/python/src/openai_codex/api.py)
- [Codex logical goal transformer](https://github.com/openai/codex/blob/9f3d25588b3f0bdf452d4eda919f2c2365a89bba/sdk/python/src/openai_codex/_goal.py)
- [Codex TypeScript Thread](https://github.com/openai/codex/blob/9f3d25588b3f0bdf452d4eda919f2c2365a89bba/sdk/typescript/src/thread.ts)
- [Claude Python types and SessionStore](https://github.com/anthropics/claude-agent-sdk-python/blob/b6e9d12fe1cc98dde988ab7b7713c1feeee50c6c/src/claude_agent_sdk/types.py)
- [Claude mirror batcher](https://github.com/anthropics/claude-agent-sdk-python/blob/b6e9d12fe1cc98dde988ab7b7713c1feeee50c6c/src/claude_agent_sdk/_internal/transcript_mirror_batcher.py)
- [Claude resume materialization](https://github.com/anthropics/claude-agent-sdk-python/blob/b6e9d12fe1cc98dde988ab7b7713c1feeee50c6c/src/claude_agent_sdk/_internal/session_resume.py)
- [Claude native message reader](https://github.com/anthropics/claude-agent-sdk-python/blob/b6e9d12fe1cc98dde988ab7b7713c1feeee50c6c/src/claude_agent_sdk/_internal/query.py)
- [Claude TypeScript reference stores](https://github.com/anthropics/claude-agent-sdk-typescript/blob/85d8f8e0772199ec7965a0c2e874ac88343f2a01/examples/session-stores/README.md)
- [Pi Durable record/storage types](https://github.com/earendil-works/pi/blob/a13d35a742c6ef8462812a28fbe1d8c8b7431c32/packages/durable/src/types.ts)
- [Pi submission admission](https://github.com/earendil-works/pi/blob/a13d35a742c6ef8462812a28fbe1d8c8b7431c32/packages/durable/src/harness/submissions.ts)
- [Pi tool recovery](https://github.com/earendil-works/pi/blob/a13d35a742c6ef8462812a28fbe1d8c8b7431c32/packages/durable/src/harness/tool.ts)
- [Pi committed-state event translation](https://github.com/earendil-works/pi/blob/a13d35a742c6ef8462812a28fbe1d8c8b7431c32/packages/durable/src/harness/events.ts)

Claude TypeScript's public repository is not the full readable runtime. Published
0.3.296 declarations were additionally inspected from an integrity-checked npm
archive; no package/runtime was executed. Primary repo examples are reference code,
not production-tested adapters. Codex source package versions are development
placeholders; pins are commit identities, not claims about published version parity.
