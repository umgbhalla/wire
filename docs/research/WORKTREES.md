# Local reference worktrees

Four independent repositories are checked out as detached reference worktrees under `references/`. Their bare repositories live in ignored `.tools/repositories/`. These are not Wire implementation source and are not copied into the public repository.

| Worktree | Repository | Pin | Scope |
| --- | --- | --- | --- |
| `references/codex` | [codex](https://github.com/openai/codex) | `9f3d25588b3f0bdf452d4eda919f2c2365a89bba` | Sparse SDK/protocol scope |
| `references/claude-python` | [claude-python](https://github.com/anthropics/claude-agent-sdk-python) | `b6e9d12fe1cc98dde988ab7b7713c1feeee50c6c` | Repository checkout |
| `references/claude-typescript` | [claude-typescript](https://github.com/anthropics/claude-agent-sdk-typescript) | `85d8f8e0772199ec7965a0c2e874ac88343f2a01` | Repository checkout |
| `references/pi-durable-1.0` | [pi-durable-1.0](https://github.com/earendil-works/pi) | `a13d35a742c6ef8462812a28fbe1d8c8b7431c32` | Sparse SDK/protocol scope |

Pi is pinned to the requested `v1.0.0`, not current main. Codex and both Claude repositories are pinned main snapshots. Codex includes Python/TypeScript SDKs and relevant Rust protocol/server source. Pi includes durable, Chord and model-access packages.

Published Claude TypeScript declarations: `@anthropic-ai/claude-agent-sdk@0.3.296`, under `.tools/claude-published/package`. The downloaded archive was verified against npm SHA-512 integrity. No lifecycle scripts were run.

## Recreate

```sh
mkdir -p .tools/repositories references
git clone --bare https://github.com/openai/codex.git .tools/repositories/codex.git
git --git-dir=.tools/repositories/codex.git worktree add --detach references/codex 9f3d25588b3f0bdf452d4eda919f2c2365a89bba
git clone --bare https://github.com/anthropics/claude-agent-sdk-python.git .tools/repositories/claude.git
git --git-dir=.tools/repositories/claude.git worktree add --detach references/claude-python b6e9d12fe1cc98dde988ab7b7713c1feeee50c6c
git clone --bare https://github.com/anthropics/claude-agent-sdk-typescript.git .tools/repositories/claude-typescript.git
git --git-dir=.tools/repositories/claude-typescript.git worktree add --detach references/claude-typescript 85d8f8e0772199ec7965a0c2e874ac88343f2a01
git clone --bare https://github.com/earendil-works/pi.git .tools/repositories/pi.git
git --git-dir=.tools/repositories/pi.git worktree add --detach references/pi-durable-1.0 a13d35a742c6ef8462812a28fbe1d8c8b7431c32
```

The commands above recreate full worktrees. The current Codex/Pi worktrees use sparse checkout to limit local files; their Git repositories still identify the full source revision. Worktrees are unmodified.
