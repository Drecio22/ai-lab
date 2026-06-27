# Study-002 — Uncertainties

## Known Unknowns

### U-001: OpenCode extension points

**What is unknown**:
Whether OpenCode has any existing middleware, plugin, service-layer interceptor, or Effect-TS mechanism that allows injecting behavior without source modification.

**Why it matters**:
If such mechanisms exist, Position 4 (provider-level interceptor) could become low-invasiveness instead of medium. Position 6 (LLM execution wrapper) could become feasible without a fork.

**Current status**:
Study-001 registered Q-020 about extension points but did not investigate. No evidence available.

**What could resolve it**:
Source inspection of OpenCode's Effect-TS service layer, provider registration mechanism, and any documented plugin/middleware APIs.

---

### U-002: Provider registration mechanism

**What is unknown**:
Whether OpenCode allows registering custom providers through configuration alone (e.g., `opencode.json` or `providers.json`) or requires code-level provider implementation and registration.

**Why it matters**:
Position 4 (provider-level interceptor) depends on this. If configuration-only registration is possible, invasiveness drops from medium to low.

**Current status**:
Unknown. Study-001 inspected `Provider.getModel` and `Provider.getLanguage` behavior but not provider registration mechanisms.

**What could resolve it**:
Source inspection of provider loading, registration, and configuration schemas.

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
Unknown. Study-001 noted OpenCode uses Effect-TS but did not investigate its service architecture for extensibility.

**What could resolve it**:
Inspect OpenCode's Effect-TS service layer, specifically how providers are registered, how `Provider.getModel` and `Provider.getLanguage` are wired through the Effect runtime.

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
