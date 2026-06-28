# Study-002 Evidence

This study's formal evidence is imported from Study-001 (EVID-001) and collected from source inspection (EVID-002, EVID-003) and a runtime experiment (EVID-004). Architectural analysis is documented in `specs.md`.

EVID-001
Type:
source-code / experiment

Source:
OpenCode Study-001, EVID-019, EVID-020, EVID-021

Date:
2026-06-28

Summary:
Study-001 confirmed the static model resolution chain in OpenCode:
- `SessionPrompt.createUserMessage` resolves model as `input.model ?? agent.model ?? currentModel(sessionID)`.
- `currentModel(sessionID)` falls back through: session current model → recent user message model → `provider.defaultModel()` → `cfg.model` → `model.json` → first available provider/model.
- `runLoop` calls `Provider.getModel` to materialize the resolved reference.
- `LLM.stream` executes the already-resolved model through native or AI SDK runtime.
- No dynamic routing or model re-selection occurs after `createUserMessage`.

This establishes the architectural constraint: any routing must intercept **before** or **at** `createUserMessage`.

Related claims:
- CLAIM-001

---

EVID-002
Type:
source-code

Source:
Static source inspection over anomalyco/opencode commit `ae53163cad0048b2351e258699e815f4f2110807` (local checkout at `C:\Users\danie\AppData\Local\Temp\opencode\opencode-src`). All paths below are relative to the repo root. Consulted 2026-06-28.

Summary:
OpenCode at this commit contains TWO coexisting plugin/extension systems plus a separate schema-first LLM package. The findings below map the real extension points relevant to routing.

### A. V1 plugin system (active in session execution)

- The published plugin contract is the `Hooks` interface in `packages/plugin/src/index.ts:222-335`. A plugin is `(input: PluginInput, options?) => Promise<Hooks>` (`packages/plugin/src/index.ts:74`).
- Plugins load from configuration: `cfg.plugin_origins` (the `plugin` array in `opencode.json`) and auto-discovered files under `.opencode/plugins`. Loader: `packages/opencode/src/plugin/loader.ts`; orchestration in `packages/opencode/src/plugin/index.ts:177-238` (`PluginLoader.loadExternal`), plus internal plugins (`index.ts:65-82`).
- Hook dispatch is centralized in `Plugin.Service.trigger(name, input, output)` (`packages/opencode/src/plugin/index.ts:280-293`): hooks run sequentially over all registered plugins, mutating the shared `output` object.
- V1 named hooks present in the type (selected): `chat.message`, `chat.params`, `chat.headers`, `experimental.chat.system.transform`, `experimental.chat.messages.transform`, `tool.execute.before`, `tool.execute.after`, `tool.definition`, `command.execute.before`, `permission.ask`, `shell.env`, `experimental.provider.small_model`, `experimental.session.compacting`, `experimental.compaction.autocontinue`, `experimental.text.complete`, plus capability hooks `provider` (ProviderHook), `auth`, `tool`, `config`, `event`, `dispose`.

### B. Hook invocation in the live LLM/session path (V1)

- `chat.params` is fired in `packages/opencode/src/session/llm/request.ts:114-132`. The `model` is passed in the **input** (read-only); the **output** only carries generation params (`temperature`, `topP`, `topK`, `maxOutputTokens`, `options`). The model therefore cannot be changed from this hook.
- `chat.headers` is fired at `packages/opencode/src/session/llm/request.ts:134-146` (headers are mutable; model is still read-only input).
- `experimental.chat.system.transform` at `packages/opencode/src/session/llm/request.ts:69-73` (system prompt is mutable).
- `experimental.provider.small_model` is fired in `packages/opencode/src/provider/provider.ts:1858-1869` inside `Provider.getSmallModel`; a plugin may set `output.model`. This is a model-selection interception point but is scoped to the small model only (compaction/titling/etc.), not the primary prompt model.
- Tool-call interception (`tool.execute.before/after`, `tool.definition`) is wired in `packages/opencode/src/session/tools.ts` and `packages/opencode/src/session/prompt.ts` (e.g. `prompt.ts:307`, `prompt.ts:389`).

### C. Provider registration and the model materialization layer (V1)

- Providers come from four sources, assembled in `packages/opencode/src/provider/provider.ts`:
  1. Config: `cfg.provider` (opencode.json), iterated at `provider.ts:1357` and `1395-1449`; each entry yields a provider with `source: "config"` and may declare `npm` (default `"@ai-sdk/openai-compatible"`), `api` URL, and a `models` map.
  2. Plugin `ProviderHook.models`: `packages/plugin/src/index.ts:214-217`; invoked at `provider.ts:1367-1392` — returns a per-provider model catalog.
  3. The `models.dev` catalog: `fromModelsDevProvider` (`provider.ts:1239-1271`), `source: "custom"`.
  4. (V2 only) `catalog.transform` (see section D).
- `Provider.getModel` materializes `{providerID, modelID}` into a `Provider.Model`. `Provider.getLanguage` (`provider.ts:1801-1830`) turns a `Model` into an AI SDK `LanguageModelV3` via `resolveSDK` (`provider.ts:1639`) and `s.modelLoaders[model.providerID]` (`provider.ts:1811`); if no custom loader exists it falls back to `sdk.languageModel(model.api.id)` (`provider.ts:1821`).
- The `modelLoaders` map is populated ONLY from the internal built-in `custom(dep)` registry (`provider.ts:168`, applied at `1535-1551`). This registry is a hardcoded map of provider factories (anthropic, opencode, azure, …). The V1 `ProviderHook` exposes only `models`, NOT `getModel`, so a V1 plugin cannot register a custom model loader.
- `resolveSDK` resolves the AI SDK factory from `BUNDLED_PROVIDERS[model.api.npm]` (`provider.ts:110-134` and `1736-1745`). It does NOT call any V2 hook.
- HTTP-level interception exists inside `resolveSDK`: the provider/model option `fetch` is wrapped (`provider.ts:1703-1734`, `customFetch`). Because `fetch` is a function, it is not serializable config; it requires code (e.g. a plugin setting provider options, or a custom provider) and effectively behaves like Position 5 (network proxy) running in-process.

### D. V2 plugin system (Effect-based; wired but not the active execution path)

- Public contract: `packages/plugin/src/v2/effect/index.ts`; `define({ id, effect })` at `packages/plugin/src/v2/effect/plugin.ts:9-11`. `PLAN.md:5` explicitly states it is "an implementation plan, not documentation for the current API."
- Capabilities: replayable `transform` hooks for the domains `agent`, `catalog`, `command`, `integration`, `reference`, `skill` (`packages/plugin/src/v2/effect/README.md:50-59`), and runtime `hook`s. The `catalog` editor can `provider.update/remove`, `model.update/remove`, and `model.default.set` (`packages/plugin/src/v2/effect/catalog.ts:9-25`). The `aisdk` runtime hooks intercept SDK creation (`aisdk.sdk`) and language-model creation (`aisdk.language`), letting a plugin set `event.language` (`packages/plugin/src/v2/effect/aisdk.ts`).
- Runtime host: `PluginHost.make` is implemented in `packages/core/src/plugin/host.ts:20-219` (catalog wiring at `72-98`, aisdk wiring at `44-71`). It is constructed inside the core plugin layer (`packages/core/src/plugin.ts:140`) and composed into the app via `locationLayer` (`packages/core/src/plugin.ts:145-153`) with `AgentV2`, `AISDK`, `Catalog`, etc. All built-in providers are registered as V2 plugins (`packages/core/src/plugin/provider.ts`, added in `packages/core/src/plugin/internal.ts:105-118`).
- V2 is referenced from the V1 app: `packages/opencode/src/agent/agent.ts:33` imports `PluginV2` and `agent.ts:104` awaits a V2 plugin (`PluginV2.ID.make("core/config-reference")`); `packages/opencode/src/skill/index.ts:9` imports `SkillPlugin`. So the V2 host runs and feeds the catalog/agent/skill domains.
- CRITICAL: the V2 `aisdk.language`/`aisdk.sdk` hooks are NOT reached by the V1 execution path. The active `Provider.getLanguage` (`provider.ts:1801`) calls `resolveSDK` + `modelLoaders`, neither of which invokes the V2 AISDK service. Study-001 (same commit, runtime-confirmed via EXP-006/008) established that the live prompt path is V1 (`SessionPrompt.prompt` → `LLM.stream` → AI SDK `streamText`) and that V1 events are the active event system.

### E. Separate `@opencode-ai/llm` package

- `packages/llm` is a schema-first LLM core ("one typed request/response/event/tool language; provider quirks live in adapters") with `Route.make`, `protocols/`, `providers/`, and staged hooks. Its `DESIGN.md:10-15` describes it as the current private API to be replaced by a proposed `@opencode-ai/ai`, with stable staged hooks (request/body/transport/event/error; `DESIGN.md:740-776`). This package is not on the active session path analyzed in Study-001; it is the direction of travel for provider/protocol abstraction.

### Inferences (labeled, not observed)

- INFERENCE: Because `chat.params`/`chat.message` expose `model` read-only and `modelLoaders` is a closed built-in registry, there is no V1-supported way to redirect the primary model per-prompt. Any per-call redirection on V1 must happen at HTTP level (`customFetch`) or after commitment.
- INFERENCE: V2 `aisdk.language` is, by design, a provider-abstraction interception point that could return a delegating `LanguageModelV3`; this would make Position 4 viable without a fork — but only once V2 is the active execution path.
- INFERENCE: The V2 catalog transform (`model.default.set`) can change the default model dynamically but is a domain-rebuild (static-ish) mechanism, not per-prompt routing.

Related claims:
- CLAIM-006, CLAIM-007, CLAIM-009, CLAIM-010, CLAIM-011, CLAIM-012, CLAIM-013, CLAIM-014, CLAIM-015, CLAIM-016

---

EVID-003
Type:
official-docs

Source:
OpenCode official documentation, `https://opencode.ai/docs/plugins/` and `https://opencode.ai/docs/providers/` (site nav), consulted 2026-06-28.

Summary:
- Plugins are an officially supported, documented extension mechanism. Loading sources: local files in `.opencode/plugins/` (project) and `~/.config/opencode/plugins/` (global), auto-loaded at startup; and npm packages declared in the `plugin` array of `opencode.json` (installed automatically via Bun). Types are imported from `@opencode-ai/plugin` (`import type { Plugin }`).
- The documented examples exercise these hooks: `event` (subscribe), `tool.execute.before`/`tool.execute.after`, `shell.env`, `experimental.session.compacting`, and custom `tool` registration.
- The docs do NOT document `chat.params`, `chat.headers`, `provider` (ProviderHook), `experimental.provider.small_model`, `experimental.chat.messages.transform`, `experimental.chat.system.transform`, `experimental.compaction.autocontinue`, `experimental.text.complete`, `tool.definition`, `permission.ask`, `command.execute.before`, or any V2 hook (`aisdk.*`, `catalog.transform`). These exist in the `@opencode-ai/plugin` type but are undocumented; they are usable but carry no documented stability guarantee.
- No documented hook or feature provides per-prompt model redirection for the primary model. A dedicated "Providers" doc page exists, confirming that adding providers via configuration is an officially supported feature.

Related claims:
- CLAIM-009, CLAIM-013, CLAIM-014, CLAIM-016

---

EVID-004
Type:
experiment

Source:
EXP-001-run-001 (runtime tracer). Run on commit `ae53163cad0048b2351e258699e815f4f2110807`, 2026-06-28.

Summary:
A runtime tracer experiment registered two plugins simultaneously:
- V1 test plugin (via config) with `chat.params`, `chat.message` hooks that write to a shared signal log.
- V2 observer plugin (via temporary internal plugin registration) with `aisdk.sdk`, `aisdk.language`, `catalog.transform` hooks that write to the same signal log.

A single non-interactive prompt (`opencode run "test" --model ollama/qwen2.5:7b-instruct`) was executed. The signal log captured:

**V1 hooks that fired:**
- `V1_PLUGIN_LOADED` (plugin server() called, hooks registered)
- `V1_CHAT_MESSAGE_FIRED` (sessionID=ses_0f2dae0b9ffeURvcZ8Ynftx2xX, model=ollama/qwen2.5:7b-instruct)
- `V1_CHAT_PARAMS_FIRED` (confirmed by error stack trace: `at chat.params (...exp-001-v1-plugin.ts:33)`)
- `V1_PLUGIN_DISPOSED` (cleanup after session ended)

**V2 hooks that fired:**
- `V2_PLUGIN_LOADED` (plugin effect() called, hooks registered)
- `V2_CATALOG_TRANSFORM_FIRED` (catalog.transform hook invoked during catalog initialization)
- `V2_PLUGIN_HOOKS_REGISTERED` (aisdk.sdk, aisdk.language callbacks registered)

**V2 hooks that did NOT fire during the prompt:**
- `V2_AISDK_SDK_FIRED` — **NOT present** in the log
- `V2_AISDK_LANGUAGE_FIRED` — **NOT present** in the log

**In addition**, a standalone dispatch test (EXP-001-run-001, standalone-test) proved that both V1 `trigger("chat.params", ...)` and V2 `runLanguage(...)` / `runSDK(...)` dispatch mechanisms are individually functional when called directly:
- V1 dispatch: hook fired, output returned correctly
- V2 dispatch (sdk): hook fired
- V2 dispatch (language): hook fired AND the returned `LanguageModelV3` was intercepted (proving Position 4 viability on V2)

**Conclusion**: The runtime evidence confirms the static analysis:
1. V1 hooks (`chat.params`, `chat.message`) ARE on the active prompt execution path.
2. V2 `aisdk.sdk` and `aisdk.language` hooks are NOT on the V1 prompt execution path. They are registered but never called during a live prompt.
3. V2 `catalog.transform` IS called during startup/initialization but is a catalog-level transform, not a per-prompt routing point.
4. Position 4 is NOT viable on the active V1 path without modifying core.
5. Position 4 IS viable on the V2 path (confirmed by standalone dispatch test) but V2 is not yet the active execution path for prompts.

Related claims:
- CLAIM-009, CLAIM-011, CLAIM-012, CLAIM-014, CLAIM-015
- U-006 (resolved)
