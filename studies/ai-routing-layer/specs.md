# Study-002 — Specs: AI Routing Architecture

## Reference Flow

From Study-001, OpenCode's model resolution and execution flow:

```
User prompt
  │
  ▼
SessionPrompt.prompt
  │
  ├── createUserMessage        ◄── MODEL RESOLUTION
  │     input.model ?? agent.model ?? currentModel(sessionID)
  │     stores {providerID, modelID} on user message
  │
  ├── runLoop
  │     ├── Provider.getModel  ◄── MATERIALIZATION
  │     ├── create assistant message with resolved model
  │     └── SessionProcessor.process
  │           └── LLM.stream
  │                 ├── Provider.getLanguage  ◄── LANGUAGE LOADING
  │                 ├── (native runtime) or (AI SDK streamText)
  │                 └── HTTP request to provider API  ◄── NETWORK
  │
  └── events → SSE → client store → UI
```

The resolution chain decomposes into distinct architectural layers, each a potential interception point:

| Layer | Component | What it does | Can route? |
|---|---|---|---|
| Configuration | `Config`, `model.json`, agent configs | Static model/provider definitions | Static only |
| Reference resolution | `createUserMessage` | `input.model ?? agent.model ?? currentModel(sessionID)` | **Before commitment** |
| Model materialization | `Provider.getModel` | Resolves `{providerID, modelID}` → `Provider.Model` | After commitment |
| Language model loading | `Provider.getLanguage` | `Provider.Model` → AI SDK `LanguageModelV3` | After commitment |
| Runtime adapter | `LLM.stream` | Selects native vs AI SDK path | After commitment |
| Network | HTTP request to provider API | Actual API call | External proxy only |

Additionally, Study-001 identified that the `TaskTool` subagent delegation path (CLAIM-022, EVID-019) selects `next.model` for child sessions. This is an existing form of model routing, but it is limited to subagent delegation, not primary prompt routing.

```
User prompt
  │
  ▼
SessionPrompt.prompt
  │
  ├── createUserMessage        ◄── MODEL RESOLUTION
  │     input.model ?? agent.model ?? currentModel(sessionID)
  │     stores {providerID, modelID} on user message
  │
  ├── runLoop
  │     ├── Provider.getModel  ◄── MATERIALIZATION
  │     ├── create assistant message with resolved model
  │     └── SessionProcessor.process
  │           └── LLM.stream
  │                 ├── Provider.getLanguage  ◄── LANGUAGE LOADING
  │                 ├── (native runtime) or (AI SDK streamText)
  │                 └── HTTP request to provider API  ◄── NETWORK
  │
  └── events → SSE → client store → UI
```

**Key architectural constraint**: Model is resolved once in `createUserMessage` and never re-evaluated. The routing layer must intervene **at or before** that point to dynamically control the model.

---

## Extension Points (EVID-002, EVID-003)

This section records the real extension surfaces found in OpenCode at commit `ae53163`. They determine which routing positions are reachable without a fork. Observed facts, inferences, and hypotheses are labeled.

### Two coexisting plugin systems

| System | Package | Status | Active in session execution? |
|---|---|---|---|
| V1 plugin | `@opencode-ai/plugin` (`packages/plugin/src/index.ts`) | Officially documented, published | **Yes** (hooks dispatched via `Plugin.Service.trigger`) |
| V2 plugin | `@opencode-ai/plugin/v2/effect` (`packages/plugin/src/v2/effect/*`) | "Implementation plan, not documentation for the current API" (`PLAN.md:5`) | Host wired (`packages/core/src/plugin/host.ts`, `plugin.ts:140-153`), feeds catalog/agent/skill, but its runtime hooks are **not** on the live LLM path |
| `@opencode-ai/llm` | `packages/llm` | Schema-first LLM core; `DESIGN.md` proposes replacing it with `@opencode-ai/ai` | Not on the active path analyzed in Study-001 |

[observed: EVID-002 §A, §D, §E]

### V1 hooks vs. routing intent

| Hook | Fires at (V1) | Can it change the primary model? | Notes |
|---|---|---|---|
| `chat.message` | new user message | **No** — `model` is read-only input; output mutates message/parts | `packages/plugin/src/index.ts:234-243` |
| `chat.params` | LLM request preparation (`llm/request.ts:114`) | **No** — `model` read-only input; output is generation params only | Decisive negative evidence for routing via this hook |
| `chat.headers` | LLM request preparation (`llm/request.ts:134`) | No | Headers only |
| `experimental.chat.system.transform` | LLM request preparation (`llm/request.ts:69`) | No | System prompt only |
| `experimental.chat.messages.transform` | prompt loop (`prompt.ts:1254`) | No | Message list only |
| `experimental.provider.small_model` | `Provider.getSmallModel` (`provider.ts:1858`) | **Only the small model** | Not the primary prompt model |
| `provider` (ProviderHook `.models`) | provider assembly (`provider.ts:1367`) | No (catalog-level) | Defines which models a provider offers; static per load |
| `tool.execute.before/after`, `tool.definition` | tool dispatch | No | Tool interception |
| `experimental.session.compacting`, `experimental.compaction.autocontinue`, `experimental.text.complete` | compaction / completion | No | Auxiliary flows |

[observed: EVID-002 §B, §C]

**Inference**: No V1 hook exposes the primary model as a writable value before or at commitment. The model is resolved in `createUserMessage` (Study-001, CLAIM-020) and is thereafter read-only input to every downstream hook.

### Provider registration and the model materialization layer

Providers are assembled in `packages/opencode/src/provider/provider.ts` from four sources [observed: EVID-002 §C]:

1. **Config** (`cfg.provider`, opencode.json) — static definitions, `source: "config"`. A custom OpenAI-compatible endpoint is configurable without code (`provider.ts:1395-1449`, default npm `@ai-sdk/openai-compatible`). [CLAIM-013]
2. **Plugin `ProviderHook.models`** — returns a model catalog per provider (`provider.ts:1367-1392`). Catalog-level only; not per-prompt.
3. **`models.dev` catalog** — built-in data, `source: "custom"` (`provider.ts:1239`).
4. **V2 `catalog.transform`** — can `provider.update/remove`, `model.update/remove`, `model.default.set` (`packages/plugin/src/v2/effect/catalog.ts`). V2 only; domain-rebuild, not per-prompt.

The dynamic point — `Provider.getLanguage` (`provider.ts:1801`) turning a `Model` into an AI SDK `LanguageModelV3` — uses the internal `resolveSDK` (`provider.ts:1639`) and `s.modelLoaders[model.providerID]` (`provider.ts:1811`). The `modelLoaders` map is populated **only** from the closed built-in `custom(dep)` registry (`provider.ts:168`, applied at `1535-1551`). The V1 `ProviderHook` exposes `models`, not `getModel`, so a V1 plugin cannot install a routing model loader. [CLAIM-012]

`resolveSDK` also wraps the transport: a provider/model `fetch` option is wrapped in-place (`provider.ts:1703-1734`). This is an HTTP-level interception point, but because `fetch` is a function it requires code and behaves like Position 5 running in-process, not like a provider-abstraction interceptor.

### V2 interception points (emerging)

The V2 host exposes runtime hooks that ARE designed for provider-abstraction interception [observed: EVID-002 §D]:

- `aisdk.sdk` — intercepts AI SDK creation; can set `event.sdk`. Used by `DynamicProviderPlugin` to load any `@ai-sdk/*` package on demand (`packages/core/src/plugin/provider/dynamic.ts`).
- `aisdk.language` — intercepts `LanguageModelV3` creation; can set `event.language`. **This is the V2 equivalent of `Provider.getLanguage` and the natural home for Position 4.** [CLAIM-015, hypothesis]
- `catalog.transform` — provider/model/default manipulation at the catalog layer.

**Inference**: If V2 were the active execution path, Position 4 would be viable without a fork by returning a delegating `LanguageModelV3` from `aisdk.language`. Today the live path is V1 and does not reach these hooks (CLAIM-011).

### Effect-TS composition

OpenCode is built on Effect-TS (`Context.Service`, `Layer`, `Layer.provideMerge`, `Effect.fn`). The V2 host and all V2 domains are composed this way (`packages/core/src/plugin.ts:145-153`). This is a real dependency-injection/composition system, but it is an internal/V2 concern: the V1 active session path exposes extension through the `Hooks` trigger pattern, not through user-facing Effect layer replacement. [CLAIM-014]

### Extension tiers

| Tier | Examples | Routing utility |
|---|---|---|
| Officially supported/documented | V1 plugins (npm/local), documented hooks (`event`, `tool.execute.*`, `shell.env`, `experimental.session.compacting`, custom `tool`), config providers, agents, MCP | Static catalog (Position 2/3); tool/event interception (Position 7) |
| Public type, undocumented | `chat.params`, `chat.headers`, `provider` ProviderHook, `experimental.provider.small_model`, `experimental.chat.*.transform`, `tool.definition`, `permission.ask`, `command.execute.before` | Auxiliary; small-model only; catalog-level |
| Internal / not public API | V2 host + hooks (`aisdk.*`, `catalog.transform`), `modelLoaders`/`custom()`, `resolveSDK`, `BUNDLED_PROVIDERS` | Position 4 lives here; requires V2 activation or core mod |

[observed: EVID-002, EVID-003; CLAIM-016]

---

## Position 1: Client-Side Preprocessor

### Description
Insert routing logic between the user interface (TUI, CLI, desktop, IDE) and the `promptAsync` API call. The preprocessor examines the prompt, session context, or task characteristics and decides which `input.model` to pass.

### Ubicación
```
User → [Router CLI/TUI layer] → promptAsync({ model: decided })
                                  │
                                  ▼
                                OpenCode (unchanged)
```

### Ventajas
- **Zero invasiveness**: OpenCode source is never touched.
- **Full control**: can examine prompt, parts, agent, session, and any external signals.
- **Supports any strategy**: cost-based, capability-based, latency-based, user-preference-based.
- **Independent lifecycle**: the router can evolve, fail, or be replaced without affecting OpenCode.
- **Works with any client**: can wrap CLI, TUI, desktop, or SDK calls.

### Limitaciones
- **Duplicate context**: the router may need to replicate some OpenCode state (session info, agent config, available providers) to make good routing decisions.
- **Race conditions**: if the router and OpenCode both try to decide the model, there is no single source of truth.
- **Not transparent to the user**: the router's decision happens "before" OpenCode, so the user sees a potentially unexpected model in the UI.
- **Client-dependent**: each client (TUI, CLI, IDE plugin) may need its own preprocessor integration; not a single universal solution.

### Grado de invasividad
**Ninguno** — no modification to OpenCode.

### Compatibilidad con OpenCode
- Fully compatible. Sets `input.model` which OpenCode already supports.
- Works with both v1 `promptAsync` and v2 `session.prompt` paths (both accept `model`).
- No risk of breaking streaming, tool calls, or SSE events.

### Impacto en mantenimiento
- Low for OpenCode (zero changes).
- Medium for the router itself: must stay compatible with the `model` parameter shape across OpenCode versions.

---

## Position 2: Agent Configuration Routing

### Description
Leverage OpenCode's existing agent→model mapping. Define multiple agents, each configured with a specific model. The routing decision becomes: **which agent to invoke** for a given prompt. The router selects the agent, and OpenCode picks up `agent.model` from the configuration.

### Ubicación
```
User → [Router] → promptAsync({ agent: "code-fast" })
                    │
                    ▼
                  OpenCode
                    │
                    ├── agent.model = "claude-sonnet-4"   (from config)
                    └── createUserMessage → fallback → agent.model
```

### Ventajas
- **Uses existing OpenCode feature** — agent→model mapping is already supported, confirmed by Study-001.
- **Configuration-driven**: no code changes in OpenCode; only config files.
- **Natural separation**: different agents can have different instructions, tools, and permissions, not just different models.
- **Granularity per task type**: "code generation agent uses model X, debugging agent uses model Y".

### Limitaciones
- **Static mapping**: agent→model is fixed in configuration. Dynamic routing requires changing which agent is selected, not the model per prompt.
- **Configuration overhead**: each (routing decision, model) combination needs a distinct agent definition. For N routing criteria × M models, you may need N×M agents.
- **Agent semantics overload**: using agents purely for model routing conflates agent identity (role, tools, instructions) with model selection.
- **Routing intelligence is external**: the router decides which agent; OpenCode just follows the static mapping.

### Grado de invasividad
**Bajo** — requires creating agent config files; no source modification.

### Compatibilidad con OpenCode
- Fully compatible. Agents with `model` are a documented, tested feature.
- No risk of breaking streaming, tools, or events.

### Impacto en mantenimiento
- Low for OpenCode.
- Medium-high for the agent definitions: each change in routing strategy requires updating agent configs.

---

## Position 3: Configuration / Fallback Injection

### Description
Influence the `currentModel(sessionID)` fallback chain by modifying `model.json`, `opencode.json`, environment variables, or provider configuration. This changes the **default** model that OpenCode uses when neither `input.model` nor `agent.model` is specified.

### Ubicación
```
OpenCode startup
  │
  ├── model.json         ◄── set default model
  ├── opencode.json      ◄── set default provider/model
  └── env vars           ◄── set provider defaults
        │
        ▼
      createUserMessage → currentModel(sessionID)
                           ├── session.currentModel (set by config)
                           ├── recent user message model
                           └── provider.defaultModel()
                                 ├── cfg.model
                                 ├── model.json
                                 └── first available provider/model
```

### Ventajas
- **Minimal invasiveness**: configuration-only; no code or proxy.
- **Quick to change**: update a JSON file or env var.
- **Works globally**: affects all prompts that don't specify an explicit model.

### Limitaciones
- **No per-prompt granularity**: the configuration applies globally; dynamic routing per task/prompt is impossible.
- **Static**: changing the default requires a config reload or restart.
- **Limited expressiveness**: can only set one default; cannot route based on prompt content, cost, latency, or capability.
- **Cannot override explicit `input.model`**: if a caller sets `input.model`, the fallback is never reached.
- **Pollution of state**: setting `session.currentModel` affects subsequent prompts in the same session.

### Grado de invasividad
**Mínimo** — configuration only.

### Compatibilidad con OpenCode
- Fully compatible, designed to work this way.
- No risk to streaming, tools, or events.

### Impacto en mantenimiento
- Very low. Configuration schema is stable.
- Risk of invisible config drift: multiple config files can override each other in non-obvious ways.

---

## Position 4: Provider-Level Interceptor

### Description
Implement a custom OpenCode provider (or wrap an existing one) that intercepts `getModel` and `getLanguage` calls. The custom provider can inspect the request context and delegate to different backend providers/models dynamically.

### Ubicación
```
createUserMessage → { providerID: "router", modelID: "dynamic" }
                            │
                            ▼
                      Provider.getModel("router", "dynamic")
                            │
                            ▼
                      [Custom Router Provider]
                            │
                            ├── analyze request context
                            ├── decide: "claude-sonnet" or "gpt-4o" or "gemini-2.5"
                            └── delegate to real provider
                                  │
                                  ▼
                            Provider.getLanguage(realModel)
                                  │
                                  ▼
                            LLM.stream → real provider API
```

### Ventajas
- **Centralized routing logic**: all model decisions go through one provider.
- **Access to full context**: the provider has access to the model reference, and through the surrounding system, potentially to prompt content and session state.
- **Clean abstraction**: OpenCode's provider interface is designed for this kind of extension.
- **Dynamic at materialization time**: can route per-call, not per-configuration.

### Limitaciones
- **After model resolution**: the model reference (`providerID/modelID`) is already chosen by `createUserMessage`. The provider intercepts during materialization, not selection. The routing decision is constrained by the original reference.
- **Provider registration may require source changes**: it is unknown whether OpenCode supports registering custom providers through configuration alone (CLAIM-006).
- **Effect-TS dependency injection**: OpenCode uses Effect-TS services; overriding a provider may require understanding its service layer.
- **Tool call compatibility**: the routed-to provider must support the same tool-call interface that OpenCode expects.
- **Streaming must work through the delegation**: the custom provider must correctly forward or reconstruct streaming responses.

### Grado de invasividad
**Medio** — requires implementing a provider; may need configuration changes; may need service-layer understanding.

### Compatibilidad con OpenCode
- Functionally compatible if the provider interface is respected.
- Risk area: tool call format differences between upstream providers.
- Risk area: streaming event format differences.

### Impacto en mantenimiento
- Medium: custom provider must track OpenCode's provider interface changes.
- Medium: must handle provider-specific error modes, rate limits, and auth.

### Veredicto actualizado (EVID-002, CLAIM-007, CLAIM-011, CLAIM-012, CLAIM-015)

This position was the principal target of the Q-002 investigation. The refined verdict, split by generation:

- **Active V1 path (today, commit `ae53163`): NOT viable without core modification.** The provider abstraction's only dynamic point (`Provider.getLanguage` via `modelLoaders`) is a closed built-in registry (`provider.ts:168`); the V1 plugin API exposes only `models`, not `getModel`. No V1 hook can redirect the primary model per-prompt (CLAIM-012). The sole code-reachable per-call interception on V1 is the `customFetch` wrapper in `resolveSDK` (`provider.ts:1703-1734`), which is HTTP-level interception (i.e. Position 5 in-process), not a provider-abstraction interceptor.
- **V2 path: viable without fork, conditional on activation.** The V2 `aisdk.language` runtime hook can set `event.language` to a delegating `LanguageModelV3`, which is exactly Position 4 (CLAIM-015, hypothesis). But V2's runtime hooks are not reached by the live prompt path today (CLAIM-011, supported; residual runtime uncertainty in U-006).

**Net**: Position 4 graduates from pure hypothesis to *conditionally viable*. Today it requires either (a) a core modification to open `modelLoaders`/add a V1 model-redirect hook, or (b) waiting for/migrating to the V2 execution path. It is not a no-fork option on the current active path.

---

## Position 5: External HTTP Proxy

### Description
Place an HTTP proxy between OpenCode and the LLM provider APIs. The proxy intercepts outgoing requests, inspects the request body, rewrites the `model` field and possibly the endpoint URL, and forwards to the target provider. The response (including streams) is passed back transparently.

### Ubicación
```
OpenCode → HTTP request (e.g., POST https://api.openai.com/v1/chat/completions)
              │
              ▼
          [External Proxy]
              │
              ├── inspect request body
              ├── rewrite model: "gpt-4o" → "claude-sonnet-4-20250514"
              ├── rewrite endpoint: openai.com → anthropic.com (or use unified API)
              ├── handle auth translation
              └── forward to selected provider
                    │
                    ▼
              [Real Provider API]
                    │
                    ▼
              Response → Proxy → OpenCode
```

### Ventajas
- **Zero modification to OpenCode**: works at the network level.
- **Transparent**: OpenCode is unaware of the proxy; models appear as the configured provider.
- **Can route on any HTTP-visible signal**: prompt content, token count, user agent, session cookie.
- **Can implement any strategy**: cost, latency, capability, fallback, load balancing.
- **Independent lifecycle**: deploy, scale, and maintain separately from OpenCode.

### Limitaciones
- **Protocol compatibility risk**: OpenCode's provider integration may use non-standard request/response shapes, streaming SSE formats, or custom headers. The proxy must preserve these perfectly.
- **Tool call serialization**: different providers have different tool call schemas. A proxy routing from OpenAI-formatted requests to Anthropic must also translate the tool call format — or require that OpenCode already uses a normalized format.
- **Streaming must be preserved**: SSE streams with custom events must pass through without corruption.
- **Authentication complexity**: the proxy must either forward OpenCode's API keys or manage its own credentials per backend provider.
- **Latency overhead**: additional network hop.
- **Stateful routing decisions**: if the proxy needs session context (beyond a single HTTP request), it must maintain or reconstruct state.
- **Not portable**: requires infrastructure (proxy server) to deploy and operate.

### Grado de invasividad
**Ninguno** — no modification to OpenCode. However, it may require changes to OpenCode's provider configuration (change the base URL to point to the proxy).

### Compatibilidad con OpenCode
- Potentially compatible, but **unvalidated**.
- Streaming, tool calls, event formats, and authentication are all risk areas.
- OpenCode's provider configuration must be pointed at the proxy.

### Impacto en mantenimiento
- High for the proxy: must track provider API changes, tool call format evolution, OpenCode-specific protocol behaviors.
- Low for OpenCode: no changes needed.

---

## Position 6: LLM Execution Wrapper

### Description
Wrap or replace the `LLM.stream` function, the AI SDK's `streamText`, or the native runtime to intercept the already-resolved model and redirect execution to a different provider/model.

### Ubicación
```
runLoop → SessionProcessor.process
              │
              ▼
          LLM.stream(input)
              │
              ├── (intercepted) change input.model
              ├── native path or AI SDK streamText
              └── HTTP request
```

### Ventajas
- **Full control at the last responsible moment**: can override the model just before the API call.
- **Access to the full stream API**: can transform, log, or augment the response.
- **Can implement fallback logic**: if the primary model fails, retry with a different model.

### Limitaciones
- **After model resolution**: the model is already chosen by `createUserMessage` and materialized by `Provider.getModel`. Wrapping at `LLM.stream` means routing against an already-committed decision.
- **High invasiveness**: requires hooking into OpenCode's internal module/service — likely requires source modification or deep monkey-patching.
- **Effect-TS dependency**: `LLM.stream` is likely an Effect-TS function or service; wrapping it requires understanding the Effect runtime.
- **Two runtime adapters**: native and AI SDK have different code paths; a wrapper must handle both.
- **Maintenance burden**: internal APIs change more frequently than external ones.

### Grado de invasividad
**Alto** — requires modifying or monkey-patching OpenCode internals.

### Compatibilidad con OpenCode
- Technically possible but fragile.
- Risk of breaking with any OpenCode update that changes `LLM.stream` internals.

### Impacto en mantenimiento
- High: tracks OpenCode's internal changes.
- High: must handle both native and AI SDK runtime paths.

---

## Position 7: MCP Server / Tool-Based Intercept

### Description
Use OpenCode's MCP server or tool system to influence model selection indirectly. A tool could analyze the task and return a model recommendation, or an MCP server could expose model information that affects agent selection.

### Ubicación
```
User → prompt
  │
  ├── router tool executes before main LLM call
  │     └── tool returns "recommend model: claude-sonnet-4"
  │
  └── (agent may or may not act on tool output)
```

### Ventajas
- **Non-invasive**: tools and MCP servers are supported extension mechanisms.
- **Can use model information as context**: the agent can see model recommendations and act on them.

### Limitaciones
- **Indirect and non-deterministic**: the agent/tool has no binding authority over model selection. It can only suggest.
- **Not suitable as primary router**: cannot guarantee a specific model is used.
- **Adds latency and token cost**: the tool runs within the LLM loop, consuming context and generation time.
- **Complex interaction**: the routing logic competes with the agent's own decisions.
- **MCP servers are not designed for this**: MCP is for tools and context, not model routing.

### Grado de invasividad
**Bajo** — uses supported extension mechanisms.

### Compatibilidad con OpenCode
- Compatible as an advisory mechanism.
- Not suitable for deterministic routing.

### Impacto en mantenimiento
- Low: uses standard MCP/tool interfaces.
- Adds operational complexity: must deploy and maintain an MCP server.

---

## Position 8: Fork / Source Modification

### Description
Fork the OpenCode repository and add a routing hook directly into the source code at the desired interception point (e.g., inside `createUserMessage`, `runLoop`, or `LLM.stream`).

### Ubicación
```
[Forked OpenCode]
  │
  ├── createUserMessage: [NEW] call router before model resolution
  ├── runLoop: [NEW] allow model re-selection
  └── LLM.stream: [NEW] routing-aware execution
```

### Ventajas
- **Full control**: can route at any point, with full access to all internal state.
- **Clean integration**: the routing logic is part of the system, not an external bolt-on.

### Limitaciones
- **Highest maintenance burden**: must rebase on every OpenCode release.
- **Diverges from upstream**: security fixes, features, and bug patches require manual merging.
- **Not a realistic option for most teams** unless they are OpenCode maintainers.
- **Requires deep understanding of OpenCode internals**.

### Grado de invasividad
**Total** — modifies OpenCode source.

### Compatibilidad con OpenCode
- N/A — it is a modified OpenCode.

### Impacto en mantenimiento
- Very high: fork maintenance, rebase burden, divergence risk.

---

## Summary Comparison

| Position | Invasiveness | Dynamic | Before resolution | Streaming safe | Tool call safe | Config only |
|---|---|---|---|---|---|---|
| 1. Client-side preprocessor | None | Yes | **Yes** | Yes | Yes | No |
| 2. Agent config routing | Low | Static mapping | **At resolution** | Yes | Yes | Yes |
| 3. Configuration injection | Minimal | No | **At resolution** | Yes | Yes | Yes |
| 4. Provider-level interceptor | Medium | Yes | No (at materialization) | Unknown | Unknown | Unknown |
| 5. External HTTP proxy | None | Yes | No (at network) | Unknown | Unknown | No |
| 6. LLM execution wrapper | High | Yes | No (at execution) | Unknown | Unknown | No |
| 7. MCP/tool-based | Low | Advisory | No | Yes | Yes | No |
| 8. Fork/modify source | Total | Yes | **Yes** | Yes | Yes | No |

**Key insight**: Only Positions 1 (client-side), 2 (agent config), and 3 (config injection) intercept **before** model resolution. Position 8 (fork) also can but at maximum cost.

For **dynamic, non-invasive routing**, Position 1 (client-side preprocessor) is the strongest candidate because it operates before model resolution with zero invasiveness and full expressiveness.

For **configuration-driven static routing**, Position 2 (agent config) is the most OpenCode-native approach.

For **transparent, protocol-level routing** with potential full dynamic capability, Position 5 (external proxy) is the strongest candidate but carries significant protocol compatibility risk.

**Updated insight after Q-002 (EVID-002/003)**: Position 4 (provider-level interceptor) is *conditionally viable* — not on the active V1 path without core modification (the `modelLoaders` registry is closed and no V1 hook can redirect the primary model; CLAIM-012), but viable on the V2 path via `aisdk.language` once V2 is the active execution path (CLAIM-015). Position 6 (LLM execution wrapper) remains high-invasiveness: the only code-reachable per-call interception on V1 is `resolveSDK`'s `customFetch` wrapper (HTTP-level, i.e. Position 5 in-process), not a stable public hook.
