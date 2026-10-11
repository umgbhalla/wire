# CCCC evidence

Upstream commit: `aa26e91f0dba38d6942415cfeaedd2e296eb060b`. CCCC: `1.8.0`. Date: 2026-10-12.

Scores describe control-flow complexity. They do not prove defects, speed, or security. Nested functions are included in the function population. Source locations below are reported by CCCC.

| Scan | Files | Functions | Parse errors | Cognitive median / p95 / max | Cyclomatic median / p95 / max |
| --- | ---: | ---: | ---: | --- | --- |
| cccc-all | 817 | 7802 | 0 | 1 / 18 / 282 | 2 / 16 / 182 |
| cccc-production | 466 | 4711 | 0 | 1 / 20 / 282 | 2 / 17 / 182 |
| cccc-sdk-core | 20 | 340 | 0 | 0 / 6 / 30 | 1 / 8 / 29 |

## Focused SDK hotspots

| Function | File | Line | Cognitive | Cyclomatic |
| --- | --- | ---: | ---: | ---: |
| `ensureDaemonBinary` | `packages/sdk/src/daemon-binary.ts` | 41 | 30 | 29 |
| `switchTo.work` | `packages/sdk/src/project-sync.ts` | 228 | 20 | 19 |
| `relayCheckpoint.visit` | `packages/sdk/src/project-sync.ts` | 190 | 18 | 13 |
| `fetchWithTimeout` | `packages/sdk/src/http.ts` | 144 | 15 | 12 |
| `uploadAttachment` | `packages/sdk/src/client.ts` | 353 | 14 | 17 |
| `waitHealthy` | `packages/sdk/src/embedded.ts` | 224 | 14 | 8 |
| `wait` | `packages/sdk/src/run.ts` | 183 | 14 | 10 |
| `readSSE` | `packages/sdk/src/sse.ts` | 13 | 13 | 9 |
| `parseEvent` | `packages/sdk/src/sse.ts` | 75 | 13 | 9 |
| `resolveBinary` | `packages/sdk/src/embedded.ts` | 161 | 9 | 9 |
| `stream` | `packages/sdk/src/voice.ts` | 57 | 9 | 11 |
| `spawnDaemon` | `packages/sdk/src/embedded.ts` | 81 | 8 | 10 |

## Interpretation for the Python SDK

- Keep normalization in the Go daemon. The Python client consumes its wire contract.
- Use HTTPX for deadlines, pooling, streaming, and test transport injection. Do not port the browser AbortController code.
- Implement SSE framing separately from typed decoding and history reduction.
- Port the failure rules in `Run.wait`: truncated streams cannot count as success.
- Keep project synchronization recovery in its own module and phase.
- Binary download and local-process lifecycle need separate verification.
- CLI dispatch and computer-service supervision are outside the Python SDK MVP. Their high scores must not be mistaken for core client complexity.

## Reproduce

```sh
.tools/cccc/cccc --version
.tools/cccc/cccc --no-config --no-cache vendor/mindwire/daemon vendor/mindwire/packages/sdk vendor/mindwire/apps/console vendor/mindwire/apps/web --pretty > docs/analysis/cccc-all.json
.tools/cccc/cccc --no-config --no-cache vendor/mindwire/daemon vendor/mindwire/packages/sdk vendor/mindwire/apps/console --exclude "**/*_test.go" --exclude "**/*.test.ts" --exclude "**/*.e2e.test.ts" --exclude "**/test/**" --exclude "**/scripts/**" --pretty > docs/analysis/cccc-production.json
.tools/cccc/cccc --no-config --no-cache vendor/mindwire/packages/sdk/src --exclude cli.ts --exclude "computer/**" --exclude "target/**" --exclude "catalog/**" --pretty > docs/analysis/cccc-sdk-core.json
```

The full scan covers the four named source roots, including their tests. It is not a scan of every root-level release script. The production scan excludes test and script paths and excludes the marketing/docs app by selecting only daemon, SDK and console roots. The focused scan excludes CLI, computer controllers, target implementations, and the model catalog. It still includes local binary lifecycle and project sync.

CCCC binary: [release v1.8.0](https://github.com/moznion/cccc/releases/tag/v1.8.0). The macOS ARM64 archive was checked against its published SHA-256 before execution. `.tools/` is local and ignored.
