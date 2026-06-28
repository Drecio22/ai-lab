# Study-002 Questions

Format:

```text
Q-000
Question:
...

State:
open

Related to:
...
```

Q-001
Question:
Where should an intelligent routing layer be placed to control which LLM model OpenCode uses, without modifying OpenCode source code?

State:
open

Related to:
routing, architecture, model-selection, extension-points, proxy

---

Q-002
Question:
Does OpenCode provide real extension points — plugins, hooks, middleware, provider registration, or service-layer interception — that allow inserting a routing layer without modifying the core source? Specifically: can Position 4 (provider-level interceptor) move from hypothesis to a technically viable option without a fork?

State:
answered (this iteration)

Related to:
extension-points, plugins, hooks, middleware, provider-registration, effect-ts, Position-4

Answer summary (supported by EVID-002, EVID-003; details in specs.md "Extension Points"):

- Formal plugin system: YES. Two coexisting systems at commit `ae53163`: V1 (`@opencode-ai/plugin`, officially documented, active in session execution) and V2 (`@opencode-ai/plugin/v2/effect`, Effect-based, migration target).
- Formal hooks system: YES. V1 exposes a `Hooks` interface (~20 named hooks); V2 exposes replayable `transform` hooks and runtime `hook`s (`aisdk.sdk`, `aisdk.language`).
- Per-prompt model redirect: NO direct hook on the active V1 path. `chat.params`/`chat.message` expose `model` as read-only input. `experimental.provider.small_model` covers only the small model.
- Position 4 verdict: NOT viable on the active V1 path without core modification (the dynamic `modelLoaders` registry is a closed built-in map; V1 plugin API does not expose it). VIABLE on the V2 path via the `aisdk.language` runtime hook, but V2 is not yet the active execution path. See CLAIM-009..CLAIM-016.

---

## Derivation from Study-001

Study-001 confirmed that OpenCode's effective model resolution occurs in `SessionPrompt.createUserMessage` via:

```
input.model ?? agent.model ?? currentModel(sessionID)
```

This resolution happens **before** `runLoop`, `Provider.getModel`, `SessionProcessor.process`, and `LLM.stream` execute. After this point, no further model selection occurs — only materialization and execution.

Therefore, Q-001 asks: at what architectural layer can we inject routing logic **before** or **instead of** this static resolution?
