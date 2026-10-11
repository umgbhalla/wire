> Superseded. This records the earlier daemon-client proposal. The current Wire direction is [the language-neutral algebraic spec](../SPEC.md), with Rust and PyO3 as future implementation targets.

# Archived Mindwire Python SDK study

Status: analysis only, 2026-10-12. No SDK implementation or harness run was performed.

## Decision

Build a Python SDK for the existing Mindwire daemon. Prioritize Claude Code and
Codex. Expose typed resource modules and immutable fluent turn builders. Make
history capture and reconstruction first-class features. Put all SDK persistence
behind one database protocol, with JSON-file, SQLite and PostgreSQL adapters.

Use Python `>=3.13`. Validate releases on ordinary CPython 3.13 and 3.14. Do not
require free-threaded Python or Python 3.14-only syntax. The daemon stays in Go.
This SDK database does not replace the daemon's internal registry or either
harness's native transcript storage.

## Evidence

Mindwire is pinned at `aa26e91f0dba38d6942415cfeaedd2e296eb060b`, v0.1.52.
Its checked-in OpenAPI has 138 paths and 177 HTTP operations. See
[the endpoint inventory](../analysis/ENDPOINTS.md).

CCCC 1.8.0 was downloaded locally and its archive SHA-256 was verified before use.
The broad scan covered 817 files and 7,802 functions with no parser errors. The
focused SDK-core scan covered 20 files and 340 functions: cognitive median 0,
p95 6, maximum 30. These metrics support a small Python client; they do not prove
correctness. See [the report](../analysis/REPORT.md) and raw JSON alongside it.

The core client is already fairly thin. Its difficult parts are binary lifecycle,
project-sync recovery, attachment validation, stream framing and terminal-state
checks. Do not translate CLI/controller complexity into the Python SDK.

## Feasibility matrix

| Requirement | Feasibility | Boundary |
| --- | --- | --- |
| Typed public modules and fluent turns | High | SDK work; existing HTTP contract |
| Claude Code and Codex turns, resume and forks | High | Native context remains owned by the harness |
| Paginated rich history, tools, attachments and compaction markers | High | Expose all data supplied by Mindwire; unknown/omitted data stays explicit |
| Observe and control runs started by Mindwire | High | Check advertised capabilities and native acknowledgements |
| Control an already-running external Codex session | Conditional | Mindwire advertises input/interrupt/respond; per-session capabilities are authoritative |
| Control an already-running external Claude session | Limited | Current adapter advertises status/observe only |
| JSON-file, SQLite and PostgreSQL history backends | High | One atomic commit contract; backend-specific concurrency limits |
| Reconstruct the SDK transcript at a recorded boundary | High, if capture is complete | Deterministic reducer; report gaps and partial snapshots |
| Edit old messages and resend as a native branch | High at safe boundaries | `beforeMessageId`, `canFork`, stable branch IDs; source remains unchanged |
| Restore an identical native session from arbitrary saved messages | Not established | No general transcript-replacement/import endpoint in Mindwire |
| Recover missing original reasoning/tool output | Not possible from absent data | Preserve supplied content; never invent hidden context |
| Rewind workspace files with chat history | Separate feature | Requires file checkpoints or project sync; conversation forks do not revert files |

## Native SDK comparison

The comparison concerns coding-agent harnesses, rather than the model-provider
Messages or Responses clients.

| Surface | Claude Code | Codex |
| --- | --- | --- |
| Official Python SDK | `claude-agent-sdk` | `openai-codex`; current official docs describe a stable Python SDK |
| Native history | Session listing and message retrieval | App-server thread reads and listing; some detailed pagination is experimental |
| Continue/resume/fork | Supported by native session options | Supported by thread lifecycle |
| Mindwire-run respond/input/interrupt | Supported | Supported |
| External-session attachment through Mindwire | Status and observation | Status, observation, input, interrupt and response, subject to live capabilities |
| Live model/permission changes through Mindwire | Advertised as supported | Advertised as unsupported; configure subsequent turns |
| Full system prompt / per-turn MCP | Supported | Supported |
| Claude hooks/settings/subagent options | Supported through Claude-specific options | Not interchangeable; reject unsupported requests |
| File rewind | Official Claude SDK checkpointing; not exposed as a general Mindwire route | Needs separately verified workspace recovery |
| Arbitrary native history replacement through Mindwire | Not exposed | Not exposed |

Mindwire's adapters and API routes define what this Python SDK can actually do.
An official harness feature is not automatically reachable through Mindwire.
Current Codex app-server docs still describe deprecated `thread/rollback`, while
the current changelog reports its removal. Do not build a stable rollback API on
it. Discover and pin the supported protocol before considering native extensions.

Keep Mindwire as the first execution backend. Direct wrappers around the official
Python SDKs are a separate decision if their native-only controls are required.
They must not start competing owners of the same live session.

Sources: [Claude Python reference](https://code.claude.com/docs/en/agent-sdk/python),
[Claude sessions](https://code.claude.com/docs/en/agent-sdk/sessions),
[Claude file checkpointing](https://code.claude.com/docs/en/agent-sdk/file-checkpointing),
[Codex SDK](https://learn.chatgpt.com/docs/codex-sdk),
[Codex app-server](https://learn.chatgpt.com/docs/app-server),
[Codex changelog](https://learn.chatgpt.com/docs/changelog).

## Public Python shape

The following is a proposed API, not runnable code in this repository.

```python
from mindwire import AsyncMindwire
from mindwire.storage.sqlite import SQLiteStore
from mindwire.targets import Remote

async with SQLiteStore("history.db") as store:
    async with AsyncMindwire(
        target=Remote(daemon_url, token=daemon_token),
        store=store,
    ) as mw:
        chat = mw.agents.use("codex").chats.use("demo")
        turn = (
            chat.turn("Add a health endpoint and a test")
            .with_model(model_id)
            .with_reasoning_effort("high")
            .with_request_id("health-endpoint-1")
        )
        run = await turn.start()
        async with run.events() as events:
            async for event in events:
                if event.type == "text":
                    print(event.text, end="", flush=True)
        result = await run.wait()

        await chat.history.sync()  # explicit complete pagination and reconciliation
        transcript = await chat.history.reconstruct()
        checkpoint = await chat.history.checkpoint(label="before revision")
        branch = await chat.fork(
            before_message_id=user_message_id,
            new_chat_id="demo-revised",
        )
        revised_run = await branch.turn("Use a different route name").start()
```

`use()` constructs a resource handle without network access. Builder methods
return new values. `.start()` is the explicit mutation boundary; reusing a request
ID is only valid for the same request. No generic `.execute()` or magic attribute
dispatch. Model names and reasoning levels come from the selected agent's catalog.
Do not silently change a saved chat's agent profile.

The event context manager closes the subscription even on early exit. It does
not cancel the remote run. `run.cancel()` requests a hard stop;
`run.interrupt()` requests a soft stop. Local task cancellation propagates after
cleanup and does not silently issue a remote cancel.

```text
src/mindwire/
|-- __init__.py          # small stable exports: AsyncMindwire, errors
|-- client.py            # connection ownership and resource access
|-- agents.py            # agent scope, capabilities, setup, settings, models
|-- chats.py             # handles, listing, rename, fork, compact, delete
|-- runs.py              # turn builder, run handles, wait and live controls
|-- history.py           # sync, pages, search, snapshots, branch records
|-- reconstruction.py    # pure ordered transcript reducer and completeness
|-- native_sessions.py   # external-owner activity, frames and receipts
|-- auth.py              # methods, login flow, status, logout
|-- workspace.py         # projects and native conversation discovery
|-- models/              # public typed requests, events, history, receipts
|-- targets/             # Remote first; Local process lifecycle later
|-- storage/
|   |-- protocol.py      # public HistoryStore and atomic batch types
|   |-- json.py          # optional JSON-file adapter
|   |-- sqlite.py        # optional standard-library SQLite adapter
|   `-- postgres.py      # optional PostgreSQL adapter
|-- _http.py             # private HTTPX request/auth/error logic
|-- _sse.py              # private framing, bounds and stream cleanup
|-- errors.py            # public API, protocol, storage and gap errors
`-- py.typed             # shipped typing marker
```

These are ownership boundaries, not a requirement to create empty files up front.
Keep each resource's request methods together. Add prompts, MCP, attachments,
project sync, execution, computer and voice modules only in their planned phases.
Export public types from their modules. Avoid wildcard root exports or exporting
the private HTTP/SSE plumbing.

## History and reconstruction controls

| Proposed API | Authority and behavior |
| --- | --- |
| `chat.history.page(before=..., limit=..., max_bytes=...)` | Daemon native/fallback message page, with revision and completeness |
| `chat.history.pages()` | Explicit page iteration; no hidden unbounded prefetch |
| `chat.history.sync()` | Fetch all needed pages, reconcile by stable message ID and revision, commit progress |
| `chat.history.search(query)` | Search captured structured history through the store; return scope/completeness |
| `chat.history.reconstruct(checkpoint=...)` | Rebuild the SDK transcript without running an agent |
| `chat.history.checkpoint(label=...)` | Save a versioned SDK projection and exact covered stream cursors |
| `chat.history.export()` / `import_archive(...)` | Typed versioned archive; import restores SDK records, not a native harness session |
| `chat.fork(before_message_id=..., new_chat_id=...)` | Native branch through Mindwire; safe edit-and-resend boundary |
| `chat.compact()` | Native compaction; preserve its boundary and prior captured history |
| `chat.native.activity()` / `events()` / `action(...)` | External owner controls, explicitly capability-gated |
| `chat.history.delete_archive()` | Delete SDK-owned capture only |
| `chat.delete()` | Explicit native/daemon deletion; expose partial purge outcomes |

Do not expose a misleading native `messages.replace()` API. Offline edits create a
new archive branch with lineage; they do not mutate native agent files. Resuming a
native branch uses the retained native identity. If that identity is absent or
gone, return an unsupported/not-found result. A future explicit context-seeded
new session is different from exact resume and requires separate feasibility work.

Reconstruction must preserve ordered rich parts, tool invocation/result IDs,
attachments, native turn IDs, interactions and resolutions, compaction and
continuation boundaries, usage metadata when supplied, input receipts, timestamps,
and unknown future event fields. Raw means the payload Mindwire supplied; it is
not a claim to contain every native protocol field or hidden agent state.

```text
history pages ----> versioned messages -----------+
                                                |
run events ------> (run_id, sequence) -----------+|
native frames ---> (connection_id, sequence) ---+||
                                              vvv
                                  atomic store commit
                                   records + cursor
                                         |
                                         v
                            deterministic reconstruction
                          complete / partial / gaps / stale
                                         |
                               SDK snapshot or archive

native resume / fork ---> Mindwire ---> existing harness session
```

Run cursors and native attachment cursors are different domains. Never merge them
as one integer sequence. Native replacement snapshots replace their live
projection; resume frames retain it. Stream markers lack content sequences.
Advance a durable cursor only in the same commit as its applied records.

Messages and stream projections can describe the same assistant output. Reconcile
with message IDs, native turn IDs and item/tool IDs; do not append the text twice.
Repeated events with the same identity and same payload are no-ops. Conflicting
payloads at one identity raise a protocol/storage conflict.

History pagination is not a database snapshot. Compare head revisions before and
after a multi-page sync; restart a bounded number of times if they change. Preserve
the previous good snapshot on failure. Older services without revisions cannot
provide the same consistency claim. Report that limit.

Check for incomplete capture, expired replay buffers, daemon restart, reordered or
missing events, compacted history, inaccessible attachments, and omitted native
parts. `reconstruct(strict=True)` raises `HistoryGapError` when exact reconstruction
of the recorded SDK projection cannot be established. Non-strict reconstruction
returns structured gap/completeness information. It never reruns tools.

## Pluggable database interface

Use a domain-specific async `Protocol`, not a generic ORM or an SQL-shaped client.
The protocol is justified by the three requested backends.

```python
class HistoryStore(Protocol):
    async def commit(
        self, batch: HistoryBatch, *, expected_revision: int
    ) -> CommitResult: ...

    async def read(
        self, query: HistoryQuery
    ) -> HistoryPage: ...

    async def snapshot(
        self, chat: ChatKey, *, checkpoint_id: str | None = None
    ) -> HistorySnapshot | None: ...

    async def delete(
        self, chat: ChatKey, *, expected_revision: int
    ) -> DeleteArchiveResult: ...

    async def aclose(self) -> None: ...
```

`HistoryBatch` carries typed events/messages, native links, branch/checkpoint
records, receipts and cursor updates. `commit` applies all supplied changes or
none, with an expected revision. The implementation must enforce these invariants
at the database/file boundary. The protocol sketch is a proposed contract, not an
implementation. Define and test the exact batch fields in phase 3.

Keys include an explicit workspace identity, harness and chat/session identity.
A native ID alone is insufficient across machines or accounts. Store connection
epochs and run IDs separately. Save schema/reducer versions with snapshots.
Checkpoint and branch identifiers are stable across retries.

| Backend | Dependency | Concurrency and durability |
| --- | --- | --- |
| JSON-file | Standard library | One writer process; reject a second writer. Per-chat atomic file replacement, fsync and recovery validation. Cross-chat writes are separate commits |
| SQLite | Standard-library `sqlite3` | Serialized writer, WAL, foreign keys, busy timeout, explicit transactions; synchronous work off the event loop |
| PostgreSQL | Optional `psycopg` extra | Transactions, unique event keys, row locking or revision compare-and-swap; parameterized SQL; migrations |
| User backend | Implements `HistoryStore` | Must pass the same atomicity, replay, revision and reconstruction contract suite |

The common transaction scope is one chat. Define cross-chat branch creation as a
retryable operation with an idempotent receipt, rather than claiming a file backend
supports a distributed transaction. Large attachments remain referenced artifacts;
do not duplicate their bytes into every event row or JSON document. Missing artifact
content reduces the completeness of an export.

No backend is forced. With `store=None`, the SDK reads daemon history and holds only
the live projection in memory; cross-process offline reconstruction is unavailable.
Injected stores and HTTP clients are caller-owned; SDK-created ones are SDK-owned.
Do not put daemon tokens, OAuth secrets or API keys in history records. Tool content
can contain private data; provide an explicit redaction policy and mark redacted
archives as intentionally incomplete.

JSON is one optional backend and the upstream wire format. SQLite is one optional
backend. Neither becomes the public history model. PostgreSQL is an optional driver,
not a service that the SDK starts or requires.

## Python features and dependency decisions

| Feature | Use |
| --- | --- |
| `async with`, async iterators, `asyncio.timeout` | Explicit ownership, streams and unary deadlines |
| `Self`, `Protocol`, `Literal`, `Annotated`, `type` aliases | Fluent return types and typed contracts; most predate 3.13 |
| Frozen/slots dataclasses | Small immutable builders; copy mutable nested inputs on construction |
| `TaskGroup`, `ExceptionGroup` | Only when related reader/writer tasks are needed; propagate cancellation |
| 3.13 `Queue.shutdown` | Clean shutdown if capture uses a bounded worker queue; no queue just for feature use |
| 3.13 `ReadOnly` / type parameter defaults | Narrow typed payload contracts and reusable typed pages where useful |
| 3.14 deferred annotations | Test introspection and Pydantic schema generation on both supported versions |
| Free threading, `InterpreterPoolExecutor` | Defer: this client is mainly I/O and must not assume shared-state safety |
| 3.14 t-strings and Zstandard | Defer: neither is needed for the core SDK or archive contract |

Recommend HTTPX for pooled async HTTP and streaming. Recommend Pydantic 2 for
runtime validation, aliases and typed nested payloads. These are substantial
capabilities, not dependencies added to save a few lines. Use strict validation for
identifiers, cursors and request fields; preserve unknown response fields and event
kinds. Snake-case Python fields map explicitly to the upstream camel-case wire.
Distinguish unset from null on patch/update requests. Validate builder updates;
unchecked `model_copy(update=...)` is not a validation boundary.

No automatic retry of mutations after an uncertain outcome. Reuse the same request
ID only where the upstream contract supports idempotency. One bounded token-refresh
retry can mirror upstream auth behavior. Unary requests have a total deadline;
SSE has connect/write/pool limits and separate optional idle limits. A missing
terminal run record cannot be reported as successful completion.

Sources: [Python typing](https://docs.python.org/3.13/library/typing.html),
[asyncio tasks](https://docs.python.org/3.13/library/asyncio-task.html),
[queue shutdown](https://docs.python.org/3.13/library/asyncio-queue.html),
[Python 3.14 changes](https://docs.python.org/3.14/whatsnew/3.14.html),
[free-threading limits](https://docs.python.org/3.14/howto/free-threading-python.html),
[HTTPX async](https://www.python-httpx.org/async/),
[HTTPX timeouts](https://www.python-httpx.org/advanced/timeouts/),
[Pydantic models](https://docs.pydantic.dev/latest/concepts/models/),
[Pydantic aliases](https://docs.pydantic.dev/latest/concepts/alias/).

## Implementation sequence, after authorization

| Phase | Deliverable | Exit evidence |
| --- | --- | --- |
| 1. Contract | Public imports, typed core requests/responses, Claude/Codex capability matrix, endpoint coverage manifest | Example type checks; wire aliases/nullability fixtures; every core operation mapped |
| 2. Remote client | HTTP/auth/errors, fluent turns, runs, SSE, chat history pages, fork/compact, native-session controls | MockTransport contract tests; framing/chunk/Unicode tests; early-close/cancellation/truncated-stream checks |
| 3. History engine | Store protocol, deterministic reducer, checkpoints, branch lineage, sync/export/import | Duplicate/conflict/gap tests; paginated revision-change tests; snapshot+tail equals full reduction |
| 4. Storage plugins | JSON-file, SQLite and PostgreSQL implementations | One shared backend contract suite; crash-before/after commit; concurrent writer/CAS checks; real temporary PostgreSQL |
| 5. Native proof | Real Claude Code and Codex round trips through Mindwire | Create -> stream -> disconnect -> reattach -> history -> fork -> resume; external attachment controls tested separately |
| 6. Local lifecycle | Verified pinned binary, loopback token, owned process cleanup | Checksum rejection, health/version gate, failed-start cleanup, borrowed daemon remains running |
| 7. Wider parity | Auth/settings/MCP/prompts/subagents, workspaces, attachments and remaining requested controls | Endpoint coverage manifest and adapter capability fixtures; destructive action tests |

Phases 1-5 define the first meaningful release for this request. Phase 6 adds the
zero-server local experience. Phase 7 is explicit parity work, not grounds to delay
history correctness. Direct native SDK wrappers, computer-service CLI, browser UI,
Grok/opencode, voice and remote provisioning are outside the first release.

CI: CPython 3.13/3.14, Ruff, a single strict type checker, pytest, package wheel
import check, public API examples, and CCCC on the new Python source. Proposed
complexity review thresholds: cognitive 15 and cyclomatic 10; exceptions require
behavior tests and a reason. These are review heuristics, not architecture goals.

## Limits of this study

Source and docs were inspected; CCCC and submodule verification were run. No native
harness was launched, no billable model call was made, no SDK code was built, and
no PostgreSQL/file durability experiment was run. Dependency versions and Python
3.14 wheels must be pinned and smoke-tested at implementation time.

The plan is feasible as a Python client with reliable recorded-history control.
It cannot promise identical hidden native context, missing content, or file rollback
from a database transcript alone. Those limits remain visible in the public API.
