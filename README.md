# Wire

Language-neutral algebraic design for agent activity, state control, history,
storage and compatibility.

**Specification only. Nothing is implemented.** The future codebase will use Rust
with PyO3 bindings for Python. Codex, Claude Code and Pi will be separate crates.

- [Type and behavioral specification](docs/SPEC.md)
- [Design work plan](docs/PLAN.md)
- [Visual design](docs/show-me-wire.html)
- [Protocol and dependency evidence](docs/SOURCES.md)
- [CCCC reference analysis](docs/analysis/REPORT.md)

```text
language-neutral types + laws
             |
       future Rust core
       |           |
       |      Python / PyO3
       |
       +-- Codex adapter
       +-- Claude adapter
       +-- Pi adapter
       |
       +-- HistoryStore: JSON-file / SQLite / PostgreSQL
```

The design separates commands, facts and outcomes. It distinguishes transcript
reconstruction from native resume and file restoration. Backends obey one commit
contract. Unsupported capabilities remain explicit.

Mindwire is a reference submodule at `vendor/mindwire`, pinned to
`aa26e91f0dba38d6942415cfeaedd2e296eb060b`. It is not a Wire runtime dependency.

```sh
git clone --recurse-submodules https://github.com/umgbhalla/wire.git
```

The [earlier Python daemon-client study](docs/research/mindwire-python-study.md)
is historical research. It does not define the current architecture.
