# Study-002 — Experiments

## EXP-001: Runtime Plugin Path Tracer (U-006)

**Objective**: Determine which plugin system (V1, V2) participates in a live prompt execution at commit `ae53163`.

**Hypothesis**:
- V1 `chat.params` hook fires during a real prompt (V1 path is active).
- V2 `aisdk.language` hook does NOT fire during a real prompt (V2 path is not reached).

**Approach**:
1. Register a V1 plugin via config with a `chat.params` hook that writes to a signal file.
2. Register a V2 plugin via internal plugin list with `aisdk.language` and `aisdk.sdk` hooks that write to a signal file.
3. Run a minimal non-interactive prompt.
4. Check which hooks left signals.

**Execution constraints**:
- No modify product logic beyond minimal instrumentation.
- No commit.
- All temporary changes documented and reversible.

**Outcome constraints**:
- If V1 fires → confirms V1 execution path is active.
- If V2 fires → confirms V2 execution path is also/live.
- If V2 does NOT fire → confirms V2 is not reached on the V1 prompt path.
- If both fire → reveals V1/V2 coexistence on the same path.

**Runs**:
- `EXP-001-run-001` — first attempt with minimal instrumentation.

### EXP-001-run-001 Results

**Date**: 2026-06-28

**Setup**:
- Dual registration: V1 plugin via `opencode-dev.json` (`plugin` array), V2 observer plugin via temporary internal plugin registration in `packages/opencode/src/plugin/plugin.dev.ts`.
- Both plugins write timestamps to a shared `exp-001-signals.json` log file.
- Prompt executed: `opencode run "test" --model ollama/qwen2.5:7b-instruct`
- Also ran a standalone dispatch test to verify both V1 and V2 dispatch mechanisms function independently.

**Signals captured during live prompt**:
| Signal | Fired? |
|---|---|
| V1_PLUGIN_LOADED | Yes |
| V1_CHAT_MESSAGE_FIRED | Yes |
| V1_CHAT_PARAMS_FIRED | Yes (with error stack trace confirming V1 path) |
| V2_PLUGIN_LOADED | Yes |
| V2_CATALOG_TRANSFORM_FIRED | Yes (during initialization) |
| V2_AISDK_SDK_FIRED | **No** |
| V2_AISDK_LANGUAGE_FIRED | **No** |
| V1_PLUGIN_DISPOSED | Yes |

**Standalone dispatch test results**:
| Dispatch call | Hook fired? | Output intercepted? |
|---|---|---|
| `Plugin.Service.trigger("chat.params", ...)` | Yes | Yes |
| `AISDK.runSDK(event)` | Yes | N/A (no output swap) |
| `AISDK.runLanguage(event)` | Yes | Yes (LanguageModelV3 replaced) |

**Conclusion**: Hypothesis confirmed. V1 path is the exclusive execution path for live prompts. V2 `aisdk.sdk`/`aisdk.language` hooks are registered but never called during prompt execution. Standalone test proves Position 4 interception is viable when V2 dispatch is reached.

**Evidence**: EVID-004

## EXP-002: Delegating LanguageModelV3 Protocol Transparency (U-007)

(Planned — see next-steps.md)
