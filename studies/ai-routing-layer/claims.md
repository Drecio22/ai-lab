# Study-002 Claims

Allowed MVP states:

```text
unknown
hypothesis
supported
confirmed
refuted
contradictory
deprecated
```

CLAIM-001
Statement:
OpenCode resolves the effective model for a prompt through a static precedence chain (`input.model ?? agent.model ?? currentModel(sessionID)`) before LLM execution begins, and no dynamic routing occurs after that point.

State:
confirmed

Type:
observed

Confidence:
high

Evidence:
- OpenCode Study-001, CLAIM-020, CLAIM-021, CLAIM-023
- OpenCode Study-001, EVID-019, EVID-020, EVID-021

Notes:
Imported from Study-001. This is the foundational architectural fact that justifies the routing layer question.

CLAIM-002
Statement:
An intelligent routing layer inserted **before** OpenCode's model resolution can control the effective model without modifying OpenCode's source code.

State:
hypothesis

Type:
hypothesis

Confidence:
none

Evidence:
- pending

Notes:
This is the core hypothesis of Study-002. It must be validated by identifying a viable insertion point that does not require source modification.

CLAIM-003
Statement:
Agent model configuration is an existing OpenCode extension point that allows per-agent model assignment, but it provides static mapping, not dynamic routing.

State:
supported

Type:
observed

Confidence:
high

Evidence:
- OpenCode Study-001, CLAIM-001 (agent can define its own model)
- OpenCode Study-001, EVID-019 (agent model resolution in createUserMessage)
- Static nature: the `agent.model` is a fixed config value, not evaluated per-prompt context

Notes:
Agents can be used for routing only insofar as the caller chooses which agent to invoke. The routing intelligence must reside in the agent-selection layer, not in the agent config itself.

CLAIM-004
Statement:
OpenCode's `currentModel(sessionID)` fallback chain (session state → previous messages → `provider.defaultModel()`) is configurable through `model.json` and `opencode.json`, but offers no per-prompt routing granularity.

State:
supported

Type:
observed

Confidence:
medium

Evidence:
- OpenCode Study-001, EVID-019 (currentModel fallback chain)
- OpenCode Study-001, EVID-021 (runtime confirmation of currentModel fallback)

Notes:
Configuration injection is the least invasive approach but also the least expressive for dynamic routing.

CLAIM-005
Statement:
An external HTTP proxy placed between OpenCode and LLM provider APIs can intercept and redirect model requests without modifying OpenCode, but must preserve OpenCode-specific protocol features (streaming, tool calls, SSE event format).

State:
hypothesis

Type:
hypothesis

Confidence:
none

Evidence:
- pending

Notes:
Key unknowns: whether the proxy can maintain OpenCode-compatible streaming semantics, whether tool-call serialization survives proxy transformation, and whether OpenCode's provider authentication can be proxied transparently.

CLAIM-006
Statement:
Whether OpenCode provides a built-in plugin or middleware system suitable for routing hooks remains unknown.

State:
deprecated

Type:
hypothesis

Confidence:
none

Evidence:
- OpenCode Study-001, Q-020 (extension points question, still open)
- OpenCode Study-001, specs.md "Extension points" section (empty)

Notes:
Superseded by CLAIM-009..CLAIM-016 after EVID-002/EVID-003. OpenCode DOES have formal plugin and hook systems (V1 active, V2 emerging). The updated, sharper question is whether any of those hooks can redirect the PRIMARY model per-prompt — see CLAIM-012.

CLAIM-007
Statement:
A custom OpenCode provider implementation can act as a routing layer by intercepting `getModel`/`getLanguage` calls and delegating to different backend providers based on request context.

State:
contradictory

Type:
hypothesis

Confidence:
medium

Evidence:
- OpenCode Study-001, EVID-019 (Provider.getModel and Provider.getLanguage are the model materialization layer)
- OpenCode Study-002, EVID-002 (modelLoaders registry is closed built-in; V1 plugin API exposes only `models`, not `getModel`; V2 `aisdk.language` can intercept language-model creation but is not on the active path)

Notes:
The claim is true in principle but split by generation:
- On the ACTIVE V1 path: NOT achievable without core modification. The dynamic interception point (`modelLoaders`/`getLanguage`) is a closed built-in registry (`provider.ts:168`, `1535-1551`), and the V1 `ProviderHook` exposes only `models`, not `getModel`. So a V1 plugin cannot install a routing model loader. See CLAIM-012.
- On the V2 path: achievable in principle via the `aisdk.language` runtime hook returning a delegating `LanguageModelV3`. See CLAIM-015.
Marked contradictory because the same statement is refuted for V1 and supported (not yet confirmed at runtime) for V2.

CLAIM-009
Statement:
OpenCode has a formal, officially supported plugin system (V1) that loads plugins from configuration (the `plugin` array in `opencode.json` and the `.opencode/plugins/` / `~/.config/opencode/plugins/` directories) and dispatches a defined set of named hooks sequentially via `Plugin.Service.trigger`.

State:
confirmed

Type:
observed

Confidence:
high

Evidence:
- EVID-002 (V1 plugin contract `packages/plugin/src/index.ts:222-335`; loader `packages/opencode/src/plugin/loader.ts`; orchestration `packages/opencode/src/plugin/index.ts:177-238`; trigger `index.ts:280-293`)
- EVID-003 (official plugins documentation confirms loading sources, npm install, and `@opencode-ai/plugin` types)

Notes:
This resolves the "does a plugin system exist" half of U-001. It does not by itself imply routing capability (see CLAIM-012).

CLAIM-010
Statement:
OpenCode has a second, Effect-based plugin system (V2) with replayable domain `transform` hooks (agent, catalog, command, integration, reference, skill) and runtime hooks (`aisdk.sdk`, `aisdk.language`); its host is implemented and wired into the app via `locationLayer`, but its own plan document states it is a migration target rather than the current public API.

State:
confirmed

Type:
observed

Confidence:
high

Evidence:
- EVID-002 (`packages/plugin/src/v2/effect/*`; `PluginHost.make` at `packages/core/src/plugin/host.ts:20-219`; composed at `packages/core/src/plugin.ts:140-153`; provider plugins registered at `packages/core/src/plugin/internal.ts:105-118`; "implementation plan, not documentation for the current API" at `packages/plugin/src/v2/effect/PLAN.md:5`; referenced from app at `packages/opencode/src/agent/agent.ts:33,104` and `packages/opencode/src/skill/index.ts:9`)

Notes:
V2 is real and wired, but it is an emerging API. Whether its runtime hooks fire during a live prompt is a separate question (CLAIM-011).

CLAIM-011
Statement:
On the active V1 execution path at commit `ae53163`, the V2 runtime hooks (`aisdk.language`, `aisdk.sdk`) are NOT reached: `Provider.getLanguage` resolves the language model through the internal `resolveSDK` + `modelLoaders` path, which does not call into the V2 `AISDK` service.

State:
confirmed

Type:
observed

Confidence:
high

Evidence:
- EVID-002 (`Provider.getLanguage` at `provider.ts:1801-1830`; `resolveSDK` at `provider.ts:1639-1745`; `modelLoaders` populated only from `custom(dep)` at `provider.ts:168` and `1535-1551`; none reference the V2 AISDK service)
- OpenCode Study-001, EVID-013..EVID-015 and EXP-006/EXP-008 (runtime confirmation that the live prompt path is V1 and V1 events are active)
- EVID-004 (runtime tracer experiment: V1 plugin loaded, V1 `chat.params`/`chat.message` fired; V2 `aisdk.sdk`/`aisdk.language` registered but DID NOT fire during live prompt; only V2 `catalog.transform` fired during initialization)

Notes:
Runtime evidence confirms static analysis. U-006 is resolved. V2 hooks are registered with the AISDK service but never invoked because `AISDK.runSDK`/`AISDK.runLanguage` are only called from `AISDK.language()`, which is not reached by the V1 `Provider.getLanguage` path.

CLAIM-012
Statement:
No V1 plugin hook can redirect the primary (prompt) model per-prompt. `chat.params` and `chat.message` expose `model` as read-only input; `experimental.provider.small_model` only selects the small model; and the dynamic model-loader registry used by `Provider.getLanguage` is a closed built-in map not exposed to plugins.

State:
confirmed

Type:
observed

Confidence:
high

Evidence:
- EVID-002 (`chat.params` input/output shape at `packages/opencode/src/session/llm/request.ts:114-132`; `experimental.provider.small_model` scope at `provider.ts:1858-1869`; closed `modelLoaders` registry at `provider.ts:168` and `1535-1551`; `ProviderHook` exposes only `models` at `packages/plugin/src/index.ts:214-217`)

Notes:
This is the decisive negative result for Position 4 on the V1 path. Per-prompt primary-model routing on V1 requires either core modification (open `modelLoaders` to plugins, or add a model-redirect hook), or falling back to HTTP-level interception via `customFetch` (which is effectively Position 5 in-process).

CLAIM-013
Statement:
A custom OpenAI-compatible provider can be added through configuration alone (the `provider` section of `opencode.json`, defaulting its npm package to `@ai-sdk/openai-compatible`), with no code; but such a provider is a static model catalog with no per-request routing logic.

State:
confirmed

Type:
observed

Confidence:
high

Evidence:
- EVID-002 (config providers iterated at `provider.ts:1357` and `1395-1449`; default npm at `provider.ts:1197` and `1414`; `source: "config"`)
- EVID-003 (official Providers documentation confirms configuration as a supported feature)

Notes:
This refines CLAIM-004 and answers Q5: configuration-only providers are supported, but they provide static mapping, not dynamic routing. Dynamic routing logic requires code (a plugin) or an external/in-process proxy.

CLAIM-014
Statement:
OpenCode is built on Effect-TS services (`Context.Service`, `Layer`, `Layer.provideMerge`, `Effect.fn`), and this is the internal composition mechanism for the V2 plugin host and all V2 domains; however, the V1 active session path exposes extension through the `Hooks` trigger pattern, not through user-facing service replacement.

State:
confirmed

Type:
observed

Confidence:
high

Evidence:
- EVID-002 (`PluginHost.make` built entirely on Effect services at `packages/core/src/plugin/host.ts:20-219`; `locationLayer` composition at `packages/core/src/plugin.ts:145-153`; V1 `Plugin.Service` as `Context.Service` at `packages/opencode/src/plugin/index.ts:58`)

Notes:
Effect-TS does offer composition points, but they are primarily an internal/V2 concern. A user today extends via V1 hooks, not by replacing Effect layers. This partially answers U-004: Effect Layers exist, but they are not a documented public extension surface for routing.

CLAIM-015
Statement:
On the V2 path, the `aisdk.language` runtime hook is a genuine provider-abstraction interception point: a plugin can set `event.language` to a `LanguageModelV3` that delegates to a different backend, enabling per-call model routing without modifying core. This makes Position 4 technically viable without a fork — conditional on V2 becoming the active execution path.

State:
supported

Type:
observed

Confidence:
medium

Evidence:
- EVID-002 (V2 `aisdk.language` hook shape at `packages/plugin/src/v2/effect/aisdk.ts`; host wiring at `packages/core/src/plugin/host.ts:58-71`; `DynamicProviderPlugin` demonstrates on-demand SDK loading via `aisdk.sdk` at `packages/core/src/plugin/provider/dynamic.ts`)
- EVID-004 (standalone dispatch test proved `runLanguage(...)` fires the V2 hook and the returned `LanguageModelV3` is intercepted by the caller)

Notes:
Standalone dispatch test (EVID-004) confirms the interception chain works when invoked. The remaining open condition is U-007: protocol transparency — whether a delegating `LanguageModelV3` preserves streaming, tool-call, and usage semantics. V2 activation per-prompt is separately blocked by U-006 (now resolved: V2 path not active).

CLAIM-016
Statement:
OpenCode extension surfaces divide into three tiers: (a) officially supported/documented (V1 plugins via npm/local files; documented hooks; config providers; agents; MCP); (b) present in the public `@opencode-ai/plugin` type but undocumented and without stability guarantees (`chat.params`, `provider` ProviderHook, `experimental.provider.small_model`, the `experimental.chat.*` transforms, `tool.definition`, etc.); and (c) internal/not public API (V2 host and hooks, `modelLoaders`/`custom()` registry, `resolveSDK`, `BUNDLED_PROVIDERS`).

State:
confirmed

Type:
observed

Confidence:
high

Evidence:
- EVID-002 (type vs. invocation sites vs. internal registries)
- EVID-003 (official docs document only a subset of V1 hooks and omit all V2 hooks)

Notes:
This answers Q8. Tier (b) is usable today but carries migration risk; tier (c) requires either waiting for V2 to stabilize or modifying core.

CLAIM-008
Statement:
The MCP (Model Context Protocol) server mechanism in OpenCode is not designed for model routing and cannot serve as a primary routing interception point.

State:
hypothesis

Type:
hypothesis

Confidence:
none

Evidence:
- pending

Notes:
MCP servers in OpenCode extend tool/context capabilities, not model selection. This is an intuitive assessment, not evidence-backed.

CLAIM-017
Statement:
The active OpenCode/V1 model catalog is built by `Provider.Service` from a models.dev-derived database plus config providers, V1 provider plugin hooks, environment credentials, stored auth, built-in provider customizers, discovery loaders, and final provider/model filters.

State:
confirmed

Type:
observed

Confidence:
high

Evidence:
- EVID-005 (V1 construction sequence in `packages/opencode/src/provider/provider.ts:1302-1634`)
- EVID-006 (official docs state Models.dev + AI SDK, provider config, credentials, `/models`, and provider/model filters)

Notes:
This answers Q-003 items 1, 2, 3, 6, 7, and 9 for the active path. The effective runtime catalog is not simply the raw models.dev data; it is filtered and personalized by config/auth/env/plugins/runtime flags.

CLAIM-018
Statement:
On the active V1 path, the canonical effective model inventory is provider-indexed (`Record<ProviderID, Provider.Info>` with nested `models: Record<ModelID, Model>`), not a first-class flattened `AvailableModel[]`; flattened model option lists are derived for ACP/model-picker consumers.

State:
confirmed

Type:
observed

Confidence:
high

Evidence:
- EVID-005 (`Provider.Service.list()` returns `Record<ProviderID, Info>` at `provider.ts:1129-1140` and `1637`; ACP derives flattened `modelOptions` at `acp/directory.ts:60-103`)

Notes:
This answers Q-003 item 4 for V1. A Decision Engine can flatten the provider-indexed catalog itself, but OpenCode's V1 canonical service boundary is provider-indexed.

CLAIM-019
Statement:
OpenCode model records carry enough metadata for capability-aware routing: provider/model IDs, display name, API/materialization ID and package, capabilities/modalities, tool/reasoning/temperature/attachment/interleaving flags, context/input/output limits, cost including cache and tier variants, status, family, release date, headers/options, and variants.

State:
confirmed

Type:
observed

Confidence:
high

Evidence:
- EVID-005 (V1 `Model` schema at `provider.ts:1018-1033`; raw models.dev schema at `models-dev.ts:47-98`; V2 `ModelV2.Info` schema at `schema/src/model.ts:59-87`)
- EVID-006 (official docs expose provider/model IDs, model config, variants, custom providers, filters)

Notes:
This answers Q-003 item 5. The catalog is suitable for a Decision Engine's static capability/cost checks, but not sufficient by itself for live latency, quota, health, or benchmark-based decisions.

CLAIM-020
Statement:
OpenCode has an internal V2 catalog service that does expose flattened available models (`catalog.model.available(): ModelV2.Info[]`), but that service belongs to the V2/server-next path and is not the active V1 prompt catalog confirmed by U-006.

State:
confirmed

Type:
observed

Confidence:
high

Evidence:
- EVID-005 (`Catalog.Service` interface and `model.available()` at `packages/core/src/catalog.ts:47-60`, `200-213`; server handlers at `packages/server/src/handlers/model.ts:12-13`; V2 runner at `packages/core/src/session/runner/model.ts:185-197`)
- EVID-004 (V2 runtime hooks are not on live V1 prompt path)

Notes:
This answers Q-003 items 4, 8, and 10 for V2. `Catalog.Service` is the cleanest shape for a future Decision Engine after V2 activation, but using it today would not automatically match V1 prompt execution.

CLAIM-021
Statement:
The effective catalog is semi-dynamic: models.dev is cached and periodically refreshable, while `Provider.Service` computes an instance-local effective provider catalog at service initialization/reload boundaries; it is not recomputed per prompt.

State:
confirmed

Type:
observed

Confidence:
high

Evidence:
- EVID-005 (`ModelsDev.get()` cached indefinitely until refresh; cache TTL/fetch behavior at `models-dev.ts:138-235`; `Provider.Service` builds `InstanceState` once at `provider.ts:1313-1634`)
- EVID-006 (docs describe startup availability, `/connect`, config, model loading priority)

Notes:
This answers Q-003 item 9. The raw upstream database can refresh, but the active decision surface is a service-state snapshot, not a per-request discovery process.

CLAIM-022
Statement:
A future in-process Decision Engine could reuse OpenCode's internal catalog infrastructure, but a V1 plugin cannot directly and officially reuse the complete active catalog through the documented plugin API; external/direct reuse requires either an OpenCode client/server endpoint or a new exposed service/hook.

State:
supported

Type:
inference

Confidence:
medium

Evidence:
- EVID-005 (`Provider.Service.list()` is an internal Effect service; documented plugin context does not expose it; HTTP handlers expose provider/model lists with path-specific shapes)
- EVID-003/EVID-006 (official plugin docs expose plugin hooks/client context but do not document a complete provider catalog hook/API)

Notes:
This answers Q-003 item 11. The answer depends on where the Decision Engine runs: inside core can reuse `Provider.Service`/`Catalog.Service`; outside core can consume server/client APIs if available; ordinary V1 plugin code has no documented direct catalog service injection.

CLAIM-023
Statement:
OpenCode agents are first-class configured assistants with `mode`, prompt, permissions, model, generation options, visibility, and step limits; `mode` determines whether an agent is primary, subagent, or both.

State:
confirmed

Type:
observed

Confidence:
high

Evidence:
- EVID-007 (official Agents docs: primary agents, subagents, JSON/Markdown config, options)
- EVID-008 (`Agent.Info` schema and config merge in `packages/opencode/src/agent/agent.ts:35-55`, `267-294`)

Notes:
This directly answers Q-004 item 1. The public docs and code agree on the core agent model.

CLAIM-024
Statement:
Each agent/subagent can have its own fixed model; for subagent delegation, the subagent model overrides the parent session model, while an unconfigured subagent inherits the model of the invoking assistant message.

State:
confirmed

Type:
observed

Confidence:
high

Evidence:
- EVID-007 (official docs: `model` option; subagents without model use the invoking primary agent's model)
- EVID-008 (`TaskTool` computes `model = next.model ?? { modelID: msg.info.modelID, providerID: msg.info.providerID }` and passes it to `ops.prompt` at `tool/task.ts:167-198`; `createUserMessage` uses `input.model ?? ag.model ?? currentModel` at `session/prompt.ts:635-667`)

Notes:
This answers Q-004 items 2 and 3. The subagent model is not dynamic routing; it is fixed lane selection. But it is enough to remove repeated manual model choice when tasks map naturally to specialists.

CLAIM-025
Statement:
Subagents can be invoked both automatically by the model through the Task tool and manually by the user through `@agent` mentions.

State:
confirmed

Type:
observed

Confidence:
high

Evidence:
- EVID-007 (official docs: automatic invocation based on descriptions and manual `@` mention)
- EVID-008 (`ToolRegistry.describeTask` injects available subagents and descriptions into the task tool description at `tool/registry.ts:252-264`; `SessionPrompt.resolveUserPart` converts `agent` parts into a synthetic instruction to call the task tool at `session/prompt.ts:974-990`)

Notes:
This answers Q-004 items 4 and 5. Automatic invocation quality depends on descriptions and permissions; manual `@` remains the deterministic override.

CLAIM-026
Statement:
A Task-tool subagent run creates or resumes a child session with `parentID`, title, agent name, derived permissions, and an explicit model; its result is returned to the parent as the task tool output, and background mode can later inject a synthetic result message.

State:
confirmed

Type:
observed

Confidence:
high

Evidence:
- EVID-008 (`sessions.create({ parentID, title, agent, permission })` at `tool/task.ts:142-158`; child prompt input at `tool/task.ts:186-199`; task result rendering at `tool/task.ts:64-78`, foreground return at `303-320`, background injection at `202-228`)

Notes:
This answers Q-004 items 6 and 8. The child receives the prompt generated by the parent/tool call, not an implicit full copy of the parent transcript.

CLAIM-027
Statement:
Subagent tool access is determined primarily by the subagent's own permissions, with parent session deny rules and external-directory rules inherited; parent task permissions determine which subagents are exposed to the parent via the Task tool.

State:
confirmed

Type:
observed

Confidence:
high

Evidence:
- EVID-007 (official docs: `permission`, `permission.task`, hidden agents, task permissions)
- EVID-008 (`deriveSubagentSessionPermission` inherits parent deny/external-directory rules at `agent/subagent-permissions.ts:14-27`; `ToolRegistry.describeTask` filters by `Permission.evaluate("task", item.name, agent.permission)` at `tool/registry.ts:252-256`; `SessionTools.resolve` passes agent/session permissions to tool execution at `session/tools.ts:56-94`)

Notes:
This answers Q-004 item 7. It is suitable for safe specialization, e.g. read-only legal auditor vs full-access refactor agent.

CLAIM-028
Statement:
OpenCode records enough runtime metadata to verify which model an agent/subagent used: session rows include `agent`, `model`, and `parentID`; user and assistant messages carry provider/model IDs; Task tool metadata includes parent/child session IDs and selected model; LLM logs include provider/model/session/agent/mode.

State:
supported

Type:
observed

Confidence:
medium-high

Evidence:
- EVID-008 (`Session.Info` fields and row mapping at `session/session.ts:80-149`, `220-273`; user message model at `session/prompt.ts:656-689`; Task metadata at `tool/task.ts:171-181`; LLM logs at `session/llm.ts:85-93`, `244-258`, `271-290`; system prompt includes exact model ID at `session/system.ts:58-66`)

Notes:
This answers Q-004 items 9 and 10 from source inspection. No new runtime prompt test was run in this iteration, so exact user-facing log/API retrieval workflow remains U-009.

CLAIM-029
Statement:
For daily OpenCode usage, fixed-model agents/subagents are a more practical near-term solution than a generic external router when the user's tasks are recurring, typed work lanes rather than ambiguous one-off prompts.

State:
supported

Type:
inference

Confidence:
medium-high

Evidence:
- EVID-001 (model resolution is static and supports agent model as an input)
- EVID-007/EVID-008 (agents/subagents have fixed models, descriptions, permissions, manual/automatic invocation, and runtime traceability)
- CLAIM-012 (no V1 plugin hook can dynamically redirect the primary prompt model)

Notes:
This answers Q-004 items 11 and 12. It does not solve arbitrary automatic model selection; it solves practical routing by making the work type select the model lane.
