# Native SDK complexity evidence

Seven scoped CCCC 1.8.0 scans. These measure reference source control flow, not Wire types or model behavior. All source pins and exact scopes are in [provenance.json](provenance.json).

| Scope | Files | Functions | Parse errors | Cognitive median / p95 / max | Cyclomatic max |
| --- | ---: | ---: | ---: | --- | ---: |
| codex-python | 18 | 283 | 0 | 0 / 9 / 29 | 23 |
| codex-typescript | 10 | 34 | 0 | 1 / 20 / 50 | 44 |
| claude-python | 26 | 317 | 0 | 2 / 27 / 95 | 61 |
| claude-typescript-examples | 9 | 40 | 0 | 1 / 22 / 30 | 14 |
| pi-durable-1.0 | 54 | 1184 | 0 | 1 / 12 / 145 | 96 |
| pi-storage-session | 13 | 388 | 0 | 1 / 15 / 145 | 82 |
| claude-store-resume | 4 | 56 | 0 | 3 / 22 / 37 | 24 |

## Hotspots inspected

| Scope | Function | Source location reported by CCCC | Cognitive | Cyclomatic |
| --- | --- | --- | ---: | ---: |
| codex-python | `process` | `references/codex/sdk/python/src/openai_codex/_goal.py:260` | 29 | 23 |
| codex-python | `_is_server_overloaded` | `references/codex/sdk/python/src/openai_codex/errors.py:61` | 27 | 16 |
| codex-python | `validate_initialize_metadata` | `references/codex/sdk/python/src/openai_codex/_initialize_metadata.py:19` | 19 | 18 |
| codex-typescript | `run` | `references/codex/sdk/typescript/src/exec.ts:94` | 50 | 44 |
| codex-typescript | `toTomlValue` | `references/codex/sdk/typescript/src/exec.ts:314` | 20 | 12 |
| codex-typescript | `flattenConfigOverrides` | `references/codex/sdk/typescript/src/exec.ts:274` | 19 | 12 |
| claude-python | `_build_command` | `references/claude-python/src/claude_agent_sdk/_internal/transport/subprocess_cli.py:570` | 95 | 61 |
| claude-python | `_read_messages` | `references/claude-python/src/claude_agent_sdk/_internal/query.py:388` | 74 | 35 |
| claude-python | `list_sessions_from_store` | `references/claude-python/src/claude_agent_sdk/_internal/sessions.py:1672` | 59 | 25 |
| pi-durable-1.0 | `recover` | `references/pi-durable-1.0/packages/durable/src/storage/jsonl/storage.ts:534` | 145 | 82 |
| pi-durable-1.0 | `translate` | `references/pi-durable-1.0/packages/durable/src/harness/events.ts:195` | 91 | 96 |
| pi-durable-1.0 | `#assemble` | `references/pi-durable-1.0/packages/durable/src/session/transaction.ts:753` | 72 | 47 |

## What the scores suggest for the study

- Pi recovery and publication-to-event translation carry substantial branch cost. Inspect the record/commit protocol before copying the API shape.
- Claude command assembly and inbound message routing mix many feature/version cases. Keep native option translation outside the common algebra.
- Codex logical goal handling distinguishes multiple native turns from one logical operation. Its small SDK is not the whole native runtime.
- The Claude TypeScript GitHub repository exposes examples and release history, not the complete readable runtime source. That scan covers only examples, including demos. Published 0.3.296 declarations were inspected separately; minified runtime was not scored.
- Generated Codex models were excluded from complexity scoring but inspected for contract structure. Pi testing helpers were excluded from the production scan.
- Whole-source and focused scans overlap. Do not sum their file/function populations or compare scores as performance/behavior ratings.

## Reproduce

Use the exact source paths and exclusions in provenance with:

```sh
.tools/cccc/cccc --no-config --no-cache --no-ignore <paths-and-exclusions> --pretty
```

No test suite, native model run or crash experiment was executed. Source contracts and actual runtime proof remain distinct.
