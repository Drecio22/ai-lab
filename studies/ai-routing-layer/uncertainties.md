# Study-002 — Uncertainties

## Known Unknowns

### U-001: OpenCode extension points

**What is unknown**:
Whether OpenCode has any existing middleware, plugin, service-layer interceptor, or Effect-TS mechanism that allows injecting behavior without source modification.

**Why it matters**:
If such mechanisms exist, Position 4 (provider-level interceptor) could become low-invasiveness instead of medium. Position 6 (LLM execution wrapper) could become feasible without a fork.

**Current status**:
RESOLVED (this iteration). OpenCode has a formal, documented V1 plugin system with ~20 named hooks (CLAIM-009), and a second Effect-based V2 system with transform + runtime hooks including `aisdk.language` (CLAIM-010). Extension surfaces are classified into three tiers (CLAIM-016). The sharper open question is now U-006.

**What could resolve it**:
Done. See specs.md "Extension Points".

---

### U-002: Provider registration mechanism

**What is unknown**:
Whether OpenCode allows registering custom providers through configuration alone (e.g., `opencode.json` or `providers.json`) or requires code-level provider implementation and registration.

**Why it matters**:
Position 4 (provider-level interceptor) depends on this. If configuration-only registration is possible, invasiveness drops from medium to low.

**Current status**:
RESOLVED (this iteration). Configuration-only providers ARE supported for static OpenAI-compatible catalogs (CLAIM-013). But configuration cannot carry routing logic (functions); a static config provider is Position 3 behavior, not Position 4. Dynamic per-call routing requires code. The V1 plugin API exposes only `ProviderHook.models` (catalog), not `getModel`/modelLoader, so a V1 plugin cannot install a routing provider (CLAIM-012).

**What could resolve it**:
Done for V1. The V2 equivalent (`catalog.transform` + `aisdk.language`) is tracked in U-006/U-007.

---

### U-003: External proxy protocol compatibility

**What is unknown**:
Whether an HTTP proxy can transparently handle OpenCode's specific provider request/response formats, including:
- SSE streaming format (event types, delta encoding) for all providers
- Tool call request/response serialization for different providers
- Authentication forwarding or replacement
- Error response format standardization

**Why it matters**:
Position 5 (external proxy) is the only truly transparent approach, but its feasibility depends on protocol compatibility. If tool call schemas differ significantly between providers, the proxy must translate them — which may be equivalent to implementing a full provider gateway.

**Current status**:
Unknown. The proxy approach is common in practice (LiteLLM, OpenRouter, Portkey) but their compatibility with OpenCode's specific expectations is not validated.

**What could resolve it**:
Trace the exact HTTP requests and responses OpenCode exchanges with different providers. Compare tool call serialization formats. Validate streaming event shape expectations.

---

### U-004: Effect-TS service interception feasibility

**What is unknown**:
Whether OpenCode's use of Effect-TS provides natural interception points (e.g., service layers with `Layer`, middleware, or aspect-oriented patterns) that could serve as routing hooks.

**Why it matters**:
Effect-TS's `Layer` system is designed for dependency injection and service replacement. If OpenCode registers providers as Effect-TS services, replacing a provider service with a routing-aware wrapper may be possible through configuration alone.

**Current status**:
PARTIALLY RESOLVED (this iteration). OpenCode is built on Effect-TS and composes services via `Layer`/`Context.Service`/`Layer.provideMerge` (CLAIM-014). The V2 host is entirely Effect services and DOES expose service-level interception through `aisdk.language`/`aisdk.sdk` (CLAIM-010). However, this is an internal/V2 surface, not a documented public extension point, and it is not reached by the active V1 execution path (CLAIM-011). So Effect-TS provides composition points, but they are not a usable public routing hook today.

**What could resolve it**:
Resolve U-006 (does V2 execute live prompts?). If yes, the V2 Effect hooks become the practical interception surface.

---

### U-005: Multi-client router portability

**What is unknown**:
Whether a single router design can serve all OpenCode clients (TUI, CLI, desktop, IDE plugin, headless SDK) or whether each client requires its own router integration.

**Why it matters**:
Position 1 (client-side preprocessor) must answer this to determine whether a single router implementation is sufficient.

**Current status**:
Unknown. Different OpenCode clients may call `promptAsync` (v1) or `session.prompt` (v2), use different SDK bindings, or have different plugin/extension mechanisms (IDE vs standalone).

**What could resolve it**:
Map the prompt submission entry points for each client. Determine if there is a common SDK layer that all clients use.

---

### U-006: Which execution path is truly active at runtime (V1 vs V2)

**What is unknown**:
At commit `ae53163`, with the V2 host wired and consumed for catalog/agent/skill, does a live prompt still execute entirely on the V1 path (`SessionPrompt.prompt` → `LLM.stream` → AI SDK `streamText`, V1 events), or does any segment — specifically `Provider.getLanguage` / language-model creation — route through the V2 `AISDK` service and therefore through the `aisdk.language`/`aisdk.sdk` hooks?

**Why it matters**:
This is the single decisive question for Position 4 viability today. Static evidence says V1 is active and does NOT call V2 hooks (CLAIM-011, supported). If a runtime tracer confirms that, Position 4 is definitively not reachable on the current path without core mod or V2 migration. If the tracer shows V2 hooks firing, Position 4 becomes immediately testable.

**Current status**:
RESOLVED. EVID-004 executed a runtime tracer experiment: dual V1+V2 plugin registered; V1 `chat.params`/`chat.message` FIRED during prompt; V2 `aisdk.sdk`/`aisdk.language` did NOT fire; only `catalog.transform` fired (during initialization). Standalone dispatch tests confirmed both V1 and V2 dispatch mechanisms are functional individually. The V1 path is the exclusive execution path for live prompts at this commit.

**What could resolve it**:
Done. See EVID-004 and CLAIM-011 (now confirmed).

---

### U-007: Delegating LanguageModelV3 semantics (V2 Position 4)

**What is unknown**:
If a V2 `aisdk.language` hook returns a `LanguageModelV3` that delegates to a different backend provider, whether OpenCode's expectations survive: SSE streaming event shape, tool-call serialization, usage/cost accounting, error model, and the V1-facing `LLMEvent` normalization (Study-001 EVID-020).

**Why it matters**:
Position 4 on V2 is only useful if the delegating language model is transparent to OpenCode. If protocol translation is required, Position 4 collapses into the same complexity as Position 5 (external proxy).

**Current status**:
Unknown. Not exercised. Related to U-003 (proxy protocol compatibility) — the same provider-quirk translation problem applies.

**What could resolve it**:
A V2 plugin prototype that returns a delegating `LanguageModelV3` (e.g. an OpenAI-compatible backend behind an Anthropic-declared model) and a runtime check of streaming, tool calls, and usage.

---

### U-008: Public catalog access for a Decision Engine

**What is unknown**:
Which stable/public boundary should a future Decision Engine use to read the complete effective model catalog without coupling to internal OpenCode services: V1 instance HTTP provider endpoint, V2 server `model.list`, SDK client, a new plugin hook, or a new first-class catalog service API.

**Why it matters**:
Q-003 confirms OpenCode already constructs a rich internal catalog. But reuse differs by deployment shape. An in-process core feature can call `Provider.Service.list()` or V2 `Catalog.Service.model.available()` directly; a V1 plugin or external router cannot rely on undocumented internal Effect services without migration risk.

**Current status**:
Partially resolved. Static inspection confirms the internal catalog exists and identifies candidate accessors (EVID-005, CLAIM-022). The officially documented plugin context does not expose the complete active catalog directly. No runtime/API compatibility test was run in this iteration.

**What could resolve it**:
Inspect generated SDK routes and server API schemas for provider/model endpoints, then run a non-mutating API call against a live instance to verify which endpoint returns the effective active catalog needed by a Decision Engine.
