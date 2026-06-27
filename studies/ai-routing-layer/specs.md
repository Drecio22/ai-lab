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
