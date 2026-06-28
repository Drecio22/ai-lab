# EXP-001-run-001: Runtime Plugin Path Tracer

## Meta

- **Date**: 2026-06-28
- **Commit**: `ae53163cad0048b2351e258699e815f4f2110807`
- **Status**: completed
- **Executed by**: opencode agent (Study-002)

## Purpose

Determine at runtime whether V1 `chat.params` and/or V2 `aisdk.language` hooks fire during a live prompt.

## Setup

### V1 test plugin

File: `opencode-src/.opencode/plugins/exp-001-v1-plugin.ts`

A minimal V1 plugin registered via `opencode.jsonc` `plugin` array. Its `server()` function returns hooks with a `chat.params` handler that writes a timestamped signal to `hook-signals.log`.

### V2 test plugin (instrumentation)

File: `opencode-src/packages/core/src/plugin/exp-001-v2-observer.ts`

A minimal V2 internal plugin registered in `packages/core/src/plugin/internal.ts` (added to the boot sequence). Its `effect` registers `aisdk.language` and `aisdk.sdk` hooks that write timestamped signals to `hook-signals.log`.

### Instrumentation changes

The following files were modified from the original commit:

1. `packages/core/src/plugin/internal.ts` — added `yield* add(Exp001V2Observer)` in the boot sequence (minimal, test-only).
2. `packages/core/src/plugin/provider.ts` — added `Exp001V2Observer` import and entry (minimal, test-only).
3. `opencode-src/.opencode/opencode.jsonc` — added `"plugin"` array with the V1 test plugin path.

These changes are purely for observation; no product logic is altered.

## Execution

Command:

```
bun run --cwd packages/opencode --conditions=browser src/index.ts --pure run "say hello"
```

The `--pure` flag disables external plugins (leaving only built-in and the test V1 plugin).
