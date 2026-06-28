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

Q-003
Question:
How does OpenCode build the effective catalog of LLM models available to the user? Specifically: where does the catalog originate, how are providers registered, how are model lists merged and filtered, who consumes the resulting structure, and can a future Decision Engine reuse it?

State:
answered (static inspection only)

Related to:
model-catalog, provider-registration, model-selection, decision-engine, models.dev, V1-provider-service, V2-catalog-service

Answer summary (supported by EVID-005/EVID-006; details in specs.md "Model Catalog Construction"):

- The current active/V1 catalog is built by `Provider.Service` in `packages/opencode/src/provider/provider.ts` from `ModelsDev.Service`, config providers, plugin provider hooks, environment variables, stored auth, built-in provider customizers, and provider/model filters.
- The effective V1 catalog shape is not `AvailableModel[]`; it is `Record<ProviderID, Provider.Info>`, where each provider owns `models: Record<ModelID, Model>`. Flattened arrays are derived for ACP/model selectors, but the canonical V1 service returns provider-indexed data.
- The V2 catalog is a separate Effect service (`Catalog.Service`) with `provider.available()` and `model.available()` APIs returning flattened `ModelV2.Info[]`; it is used by the V2/server-next path, not by the active V1 prompt path confirmed in U-006.
- Model entries contain provider/model IDs, display name, provider API/materialization info, capabilities, modalities, variants, limits, pricing, status, family, release date, headers and options (V1); V2 normalizes similar information into schema-level `ModelV2.Info`.
- A future Decision Engine can reuse OpenCode's internal catalog infrastructure if it runs inside the same process/service graph. External plugins today do not have a documented public hook/API that directly exposes the complete active V1 catalog; they would need an API/client call or a core-supported service boundary.

---

## Derivation from Study-001

Study-001 confirmed that OpenCode's effective model resolution occurs in `SessionPrompt.createUserMessage` via:

```
input.model ?? agent.model ?? currentModel(sessionID)
```

This resolution happens **before** `runLoop`, `Provider.getModel`, `SessionProcessor.process`, and `LLM.stream` execute. After this point, no further model selection occurs — only materialization and execution.

Therefore, Q-001 asks: at what architectural layer can we inject routing logic **before** or **instead of** this static resolution?

---

Q-004
Question:
Can OpenCode's agent/subagent architecture, with fixed per-agent models, serve as a practical routing layer for daily work without adding an external router?

State:
answered (this iteration; source/docs evidence, no new runtime prompt test)

Related to:
agents, subagents, model-selection, practical-routing, task-tool, app-oposiciones

Answer summary (supported by EVID-007, EVID-008; runtime caveat in U-009):

- Official docs define two agent types: primary agents used directly, and subagents invoked by primary agents or by user `@` mentions.
- Agents/subagents can define a fixed `model`. Official docs state that a subagent without a model uses the invoking primary agent's model.
- Source code confirms stronger precedence for task delegation: `TaskTool` uses `next.model ?? parent assistant message model`, then passes that model explicitly into the child session prompt. Therefore a subagent's configured model prevails over the parent session model for that child run.
- Invocation is both automatic and manual: automatic through the `task` tool exposed to the model using subagent descriptions, and manual through `@agent` mention.
- Tools are governed by each subagent's permissions plus inherited deny/external-directory constraints from the parent session. Results return as task tool output and child sessions record agent/model metadata.
- Practical verdict: yes, for a human-in-the-loop workflow this is a better near-term answer than a generic external router. It replaces opaque per-request routing with explicit, inspectable work lanes: legal/corpus audit, JSON audit, question linking, Next.js refactor, senior review, and release validation.
