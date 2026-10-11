# Wire: specification work plan

Specification only. [SPEC.md](SPEC.md) is the design draft;
[AGENTS.md](../AGENTS.md) preserves the scope. No implementation is authorized.

Design a language-neutral algebra for database commits, activity, commands,
observed state, history reconstruction and compatibility. Rust crates and PyO3
are future targets. Codex, Claude Code and Pi will be independent adapter crates.
Mindwire supplies research evidence, not Wire's runtime architecture.

| Gate | Spec deliverable | Required evidence |
| --- | --- | --- |
| 1. Vocabulary | Minimal identities and sum/product families | Every field has an owner, scope and unit |
| 2. Activity | Commands, facts, receipts and complete transition tables | Start, queue, approval, interrupt, cancel, disconnect and unknown-outcome traces |
| 3. Storage | Commit/query algebra, CAS, receipts, durability and erasure | Same observable outcomes across all three backends; crash boundaries modeled |
| 4. History | Reducer, revisions, checkpoints, branches and coverage | Full replay equals checkpoint plus tail; dedupe/gap/compaction/revision scenarios |
| 5. Compatibility | Guarantees and protocol/version negotiation | Native/emulated/unavailable/unknown matrix for the three harnesses |
| 6. Language mapping | Rust and Python projection | Variants/errors map without loss; ownership and cancellation described |
| 7. Adapters | Separate protocol contracts and real dependencies | Version-pinned evidence; native acceptance and settlement distinguished |
| 8. Review | Smallest coherent spec with explicit unresolved questions | Each law has positive/negative examples; unnecessary types removed |

These gates produce documents and trace tables only. Passing them does not
implicitly authorize implementation.

Use CCCC repeatedly when source changes or a narrower scope adds evidence. Keep
source SHA, tool version, exclusions, parser errors and function scores. Do not
rescan an unchanged tree hundreds of times or hide branch cost inside helpers.
CCCC cannot score this spec. Review concept count, ambiguous states, duplicated
operations, dependencies and exceptions to laws with short traces and tables.

Once implementation is explicitly authorized, compare CCCC before/after real
changes and retain behavior checks. Existing Mindwire scores are reference evidence,
not a Wire implementation baseline.

Keep out: UI, CLI, deployment model, general workflow framework, ORM, convenience
wrappers, magic dispatch and automatic mutation retries. Keep validation, durable
commits, error propagation, unknown outcomes and fidelity reporting.

The architecture is feasible based on primary sources: Rust can drive native
subprocess protocols, SQLx supports both SQL backends, and PyO3 supports Rust/Python
bindings. Integration is not tested. The hard part is semantic agreement across
native idempotency, settlement, history, ownership and restoration. See
[SOURCES.md](SOURCES.md). Draft 0 records these unresolved boundaries.
