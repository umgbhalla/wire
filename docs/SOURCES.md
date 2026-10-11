# Protocol and dependency evidence

Checked 2026-10-12. These dependencies exist and were checked in primary sources.
No Wire dependencies were installed or manifests created. Exact versions, feature
sets and platform checks remain future work.

| Future module | Candidate dependency/runtime | Purpose and limit |
| --- | --- | --- |
| wire-types | Serde, where encoding is needed | Boundary serialization; not domain semantics |
| wire-core | Standard Rust types first | Pure decisions/reduction; no SQL/Python/runtime dependency |
| wire / adapters | Tokio, serde_json at native JSON boundaries | Async subprocess I/O; acknowledgements are protocol-specific |
| wire-store SQL backends | SQLx with selected backend features | Real SQLite/PostgreSQL drivers, without an ORM-shaped contract |
| wire-store JSON-file | Rust filesystem APIs | Writer exclusion and durability remain design work |
| wire-python | PyO3, pyo3-async-runtimes, maturin tooling | Bindings, asyncio bridge, packaging; lifetime/cancellation need tests |
| wire-codex | Pinned Codex app-server | Native JSON-RPC; no assumed official Rust SDK crate |
| wire-claude | Pinned Claude Code CLI | Stream-json/native control; compare official Agent SDK semantics |
| wire-pi | Pinned Pi RPC runtime | JSONL process integration; prompt acceptance is not settlement |

Primary sources:

- [Serde](https://docs.rs/serde/latest/serde/)
- [Tokio](https://docs.rs/tokio/latest/tokio/)
- [SQLx](https://github.com/transact-rs/sqlx)
- [PyO3](https://pyo3.rs/)
- [PyO3 async runtimes](https://docs.rs/pyo3-async-runtimes/latest/pyo3_async_runtimes/)
- [Maturin](https://www.maturin.rs/)
- [Codex app-server](https://learn.chatgpt.com/docs/app-server)
- [Codex SDKs](https://learn.chatgpt.com/docs/codex-sdk)
- [Claude CLI](https://code.claude.com/docs/en/cli-reference)
- [Claude Python Agent SDK](https://code.claude.com/docs/en/agent-sdk/python)
- [Claude sessions](https://code.claude.com/docs/en/agent-sdk/sessions)
- [Claude file checkpointing](https://code.claude.com/docs/en/agent-sdk/file-checkpointing)
- [Pi RPC at inspected revision](https://github.com/earendil-works/pi/blob/8a9c051f80090895abbc64262bc90d95b42103c4/packages/coding-agent/docs/rpc.md)

The old Pi URL redirects from `badlogic/pi-mono` to `earendil-works/pi`. Its RPC
docs require correlation IDs, strict LF framing and continuous stdout reads.
A prompt response can mean started, queued or handled. `agent_end` may precede
retries or follow-up work; `agent_settled` is the documented settlement boundary.

Official Python SDKs are comparison sources, not proposed Rust dependencies.
Mindwire's adapters supply additional reference evidence. None of these sources
establish exact native context restoration from an arbitrary saved transcript.
