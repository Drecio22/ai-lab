# Study-002 Evidence

This study's formal evidence is imported from Study-001 (EVID-001), collected from source inspection (EVID-002, EVID-005), official documentation (EVID-003, EVID-006), and a runtime experiment (EVID-004). Architectural analysis is documented in `specs.md`.

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

---

EVID-005
Type:
source-code

Source:
Static source inspection over anomalyco/opencode commit `ae53163cad0048b2351e258699e815f4f2110807` (local checkout at `C:\Users\danie\AppData\Local\Temp\opencode\opencode-src`). All paths below are relative to the repo root. Consulted 2026-06-28.

Summary:
OpenCode has two catalog implementations relevant to models: the active V1 `Provider.Service` used by the current prompt path, and the V2 `Catalog.Service` used by the core/server-next path.

### A. models.dev source and refresh/cache behavior

- `packages/core/src/models-dev.ts` defines `ModelsDev.Service` with `get()` and `refresh(force?)` (`models-dev.ts:116-121`).
- The upstream source is `Flag.OPENCODE_MODELS_URL || "https://models.dev"`; the fetched endpoint is `${source}/api.json` (`models-dev.ts:138`, `153-159`).
- Local cache path is `Global.Path.cache/models.json` or a hash-specific filename for non-default sources (`models-dev.ts:139-142`). TTL freshness check is 5 minutes (`models-dev.ts:143-151`).
- Loading order inside `populate`: disk (`OPENCODE_MODELS_PATH` or cache) -> embedded `OPENCODE_MODELS_DEV` snapshot -> fetch unless `OPENCODE_DISABLE_MODELS_FETCH` (`models-dev.ts:162-209`).
- `get()` is an indefinitely cached Effect (`Effect.cachedInvalidateWithTTL(populate, Duration.infinity)`), invalidated only by `refresh()` (`models-dev.ts:211-224`). `refresh()` is scheduled hourly when model fetching is enabled (`models-dev.ts:233-235`).
- The raw models.dev schema includes provider fields `id`, `name`, `env`, optional `api`, optional `npm`, and `models: Record<string, Model>` (`models-dev.ts:101-108`). Each raw model includes `id`, `name`, optional `family`, `release_date`, booleans for `attachment`, `reasoning`, `temperature`, `tool_call`, optional `interleaved`, optional `cost`, `limit`, optional `modalities`, optional `experimental.modes`, optional `status`, and optional provider overrides (`models-dev.ts:47-98`).

### B. Active V1 catalog construction (`Provider.Service`)

- `packages/opencode/src/provider/provider.ts` defines the active V1 model shape at `provider.ts:1018-1033` and provider shape at `provider.ts:1035-1044`.
- V1 model fields are: `id`, `providerID`, `api: { id, url, npm }`, `name`, optional `family`, `capabilities`, `cost`, `limit`, `status`, `options`, `headers`, `release_date`, optional `variants` (`provider.ts:1018-1032`).
- V1 provider fields are: `id`, `name`, `source` (`env`/`config`/`custom`/`api`), `env`, optional `key`, `options`, and `models: Record<string, Model>` (`provider.ts:1035-1044`).
- `fromModelsDevModel` normalizes raw models.dev models into V1 `Model`, mapping raw modalities to `capabilities.input/output`, `tool_call` to `capabilities.toolcall`, cost/limits/status/release date/provider API metadata, and defaulting `npm` to `@ai-sdk/openai-compatible` (`provider.ts:1188-1237`).
- `fromModelsDevProvider` converts each raw models.dev provider into V1 provider `Info` with `source: "custom"` and a model map. It also expands `model.experimental.modes` into additional synthetic model IDs (`${model.id}-${mode}`) with mode-specific cost/options/headers (`provider.ts:1239-1271`).
- The service state initializes `modelsDev = modelsDevSvc.get()`, `catalog = mapValues(modelsDev, fromModelsDevProvider)`, and `database = mapValues(catalog, toPublicInfo)` (`provider.ts:1316-1320`). `catalog` is retained separately from `providers` for suggestions/fallbacks (`provider.ts:1626-1633`, `1777-1797`).
- `providers` is the active effective catalog returned by `Provider.list()` (`provider.ts:1321`, `1637`). It is provider-indexed, not a flattened array.

### C. Provider registration and merge/filter order in V1

Observed V1 construction sequence inside `Provider.Service` (`provider.ts:1302-1634`):

1. Load plugins before config providers so plugin `config()` hooks can affect `cfg.provider` (`provider.ts:1353-1357`).
2. Build allow/deny sets from `cfg.disabled_providers` and `cfg.enabled_providers` (`provider.ts:1357-1365`).
3. For V1 plugins with `provider.models`, locate the matching provider in `database`, call the hook, and replace that provider's model map with the hook result, coercing IDs/providerIDs (`provider.ts:1367-1392`). This requires a provider already present in `database`.
4. Extend `database` from `cfg.provider` (`provider.ts:1394-1486`). Config providers can create new provider entries, inherit existing provider/model data, set provider `name`, `env`, `options`, `api`, `npm`, and add/override models. Each configured model is normalized with merged capabilities, cost, limits, headers, options, family, release date, and variants (`provider.ts:1395-1483`).
5. Load providers from environment variables: if any configured provider env var is present, merge a provider with `source: "env"` and optional key into `providers` (`provider.ts:1488-1499`).
6. Load providers from stored auth (`auth.all()`), merging API-key auth as `source: "api"` (`provider.ts:1501-1512`).
7. Run plugin auth loaders to compute provider options after auth exists (`provider.ts:1514-1533`).
8. Run built-in `custom(dep)` provider customizers. They may set model loaders, vars loaders, discovery loaders, options, and can autoload providers (`provider.ts:1535-1551`; custom registry starts at `provider.ts:168`).
9. Re-apply config provider metadata/options after customizers (`provider.ts:1553-1561`).
10. Run provider-specific model discovery for entries with a `discoverModels` loader (observed hardcoded `gitlab`) and append missing discovered models (`provider.ts:1563-1575`).
11. Final filter pass over active `providers`: enforce enabled/disabled providers; delete invalid aliases for known chat-only IDs; remove alpha models unless `runtimeFlags.enableExperimentalModels`; remove deprecated models; apply per-provider `blacklist` and `whitelist`; fill/merge variants; remove empty providers (`provider.ts:1577-1624`).

### D. Duplicates, aliases, variants

- Models are keyed by `provider.models[modelID]`; within a provider, later writes to the same key overwrite/merge the previous entry. Across providers there is no global model dedupe because provider ID is part of the identity.
- Config parsing can separate model key from provider API ID: `parsedModel.id` is `modelID`, while `parsedModel.api.id` can be `model.id` or inherited API ID (`provider.ts:1406-1426`). This supports aliases/custom deployment names.
- Explicit alias deletion is narrow and hardcoded for known invalid chat aliases (`gpt-5-chat-latest`, `openai/gpt-5-chat`) in selected providers (`provider.ts:1586-1597`).
- Variants are generated by `ProviderTransform.variants(model)` and then merged with configured variants; disabled variants are removed (`provider.ts:1478-1482`, `1606-1617`).
- `fromModelsDevProvider` expands raw `experimental.modes` into separate synthetic models, while `ProviderTransform.variants` creates per-model variant options; these are two different mechanisms (`provider.ts:1243-1261`, `1478-1482`).

### E. Active V1 catalog accessors and consumers

- `Provider.Service` exposes `list`, `getProvider`, `getModel`, `getLanguage`, `closest`, `getSmallModel`, and `defaultModel` (`provider.ts:1129-1140`, returned at `provider.ts:1948`).
- `Provider.list()` returns the active effective catalog as `Record<ProviderID, Provider.Info>` (`provider.ts:1637`).
- `Provider.getModel(providerID, modelID)` resolves one model from `s.providers`; if provider/model is missing it uses `s.catalog` and fuzzy matching for suggestions (`provider.ts:1777-1799`).
- `Provider.defaultModel()` consults config `model`, then recent `model.json`, then the first provider/model using internal sorting (`provider.ts:1913-1946`; sort at `provider.ts:1964-1973`).
- V1 HTTP provider list handler returns `all`, `default`, and `connected` by merging raw models.dev-derived providers with active connected providers: `providers = Object.assign(mapValues(filtered, fromModelsDevProvider), connected)` (`packages/opencode/src/server/routes/instance/httpapi/handlers/provider.ts:40-58`). This endpoint includes providers not currently connected, unlike `Provider.list()`.
- ACP directory builds `Snapshot.providers`, flattened `modelOptions`, `variantsByModel`, and `defaultModel` from `Provider.list()` and `Provider.defaultModel()` (`packages/opencode/src/acp/directory.ts:60-103`, `117-135`).
- ACP config model options flatten provider/model/variant choices into `{ value, name }` entries (`packages/opencode/src/acp/config-option.ts:166-190`).
- Active prompt and agent flows consume resolved models via `Provider.getModel`/`defaultModel`: agent resolution (`packages/opencode/src/agent/agent.ts:373-374`), SessionPrompt (`packages/opencode/src/session/prompt.ts:119`, `219-221`, `599`, `632`, `864`), compaction (`packages/opencode/src/session/compaction.ts:330-331`, `452-460`), plan tool (`packages/opencode/src/tool/plan.ts:51`), share path (`packages/opencode/src/share/share-next.ts:190`, `288`).

### F. V2 catalog service

- `packages/core/src/catalog.ts` defines `Catalog.Service` with `provider.get/all/available`, `model.get/all/available/default/small`, and transform/reload inherited from `State.Transformable<Draft>` (`catalog.ts:47-60`).
- V2 catalog state is `providers: Map<ProviderID, ProviderRecord>` plus optional `defaultModel`; each `ProviderRecord` contains `provider: ProviderV2.MutableInfo` and `models: Map<ModelID, ModelV2.MutableInfo>` (`catalog.ts:13-27`).
- The draft API allows provider/model update/remove and default model set (`catalog.ts:29-45`, implementation at `105-159`). This is what V2 `catalog.transform` plugins edit.
- `model.all()` returns a flattened, release-date-sorted `ModelV2.Info[]`; `model.available()` filters that list to available providers and enabled models (`catalog.ts:200-213`). This is the closest observed structure to `AvailableModel[]`.
- V2 `projectModel` resolves provider/model API and request inheritance before returning `ModelV2.Info` (`catalog.ts:78-97`).
- `Catalog.Service` is consumed by the separate `packages/server` handlers: `server.model/model.list` returns `catalog.model.available()` and `server.provider/provider.list` returns `catalog.provider.available()` (`packages/server/src/handlers/model.ts:12-13`; `packages/server/src/handlers/provider.ts:14-22`).
- V2 runner model selection uses `catalog.model.available()` (`packages/core/src/session/runner/model.ts:185-197`).
- This V2 catalog is not the active V1 prompt path per EVID-004/U-006.

### G. V2 schema fields

- `packages/schema/src/model.ts` defines `ModelV2.Info` with fields: `id`, `providerID`, optional `family`, `name`, `api`, `capabilities`, `request`, `variants`, `time.released`, `cost`, `status`, `enabled`, and `limit` (`schema/src/model.ts:59-87`).
- `ModelV2.Capabilities` contains `tools`, `input: string[]`, `output: string[]` (`schema/src/model.ts:24-30`).
- `ModelV2.Cost` contains optional context tier, input/output prices, and cache read/write (`schema/src/model.ts:31-43`).
- `ProviderV2.Info` contains `id`, optional `integrationID`, `name`, optional `disabled`, `api`, and `request` (`schema/src/provider.ts:52-60`).

Inferences (labeled, not observed):
- INFERENCE: For current OpenCode, the reusable internal inventory for a Decision Engine is `Provider.Service.list()` plus `Provider.getModel()`/`defaultModel()`, not `Catalog.Service.model.available()`, because V1 remains the active execution path.
- INFERENCE: A Decision Engine running outside the process can use the HTTP provider/model endpoints only if it accepts endpoint-specific differences: the V1 instance provider endpoint returns a provider-indexed `all/default/connected` structure and includes disconnected models.dev providers; the V2 server model endpoint returns flattened available models but belongs to the V2/server-next path.
- INFERENCE: A V1 plugin cannot directly consume `Provider.Service.list()` through the documented plugin context; direct reuse by plugins would require a client/server API call or a new exposed service/hook.

Related claims:
- CLAIM-017, CLAIM-018, CLAIM-019, CLAIM-020, CLAIM-021, CLAIM-022
- U-008

---

EVID-006
Type:
official-docs

Source:
OpenCode official documentation, `https://opencode.ai/docs/models/`, `https://opencode.ai/docs/providers/`, `https://opencode.ai/docs/config/`, consulted 2026-06-28.

Summary:
- Official docs state that OpenCode uses the AI SDK and Models.dev to support 75+ LLM providers and local models (`/docs/models/`, `/docs/providers/`).
- The docs describe model selection via `/models` and default model configuration via `model: "provider_id/model_id"` (`/docs/models/`). For custom providers, `provider_id` is the key under config `provider`, and `model_id` is the key under `provider.models`.
- The docs state that popular providers are preloaded by default, and providers become available when credentials are added through `/connect` (`/docs/models/`).
- Config docs document `provider`, `model`, and `small_model`; `small_model` is for lightweight tasks and falls back to the main model if no cheaper provider model is available (`/docs/config/`).
- Provider docs document provider config, custom providers, custom base URLs, and `models` maps for custom providers. Examples show `npm`, `name`, `options.baseURL`, and `models` as user-facing provider/model declaration fields (`/docs/providers/`).
- Provider docs document `blacklist` and `whitelist` as model picker filters; config docs document `disabled_providers` and `enabled_providers` as provider-level filters. Docs state disabled providers' models do not appear in the model selection list (`/docs/providers/`, `/docs/config/`).
- Models docs document variants: built-in variants, custom variants, disabled variants, and cycling variants (`/docs/models/`).
- Models docs describe loading model priority as command-line `--model`/`-m`, config model list/default, last used model, then first model using internal priority (`/docs/models/`). This documentation is broadly consistent with Study-001 model resolution and `Provider.defaultModel()` fallback, though the exact internal code path includes `cfg.model`, recent `model.json`, and sorted first provider/model.

Related claims:
- CLAIM-017, CLAIM-018, CLAIM-019, CLAIM-020, CLAIM-021
