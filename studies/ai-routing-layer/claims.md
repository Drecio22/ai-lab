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
unknown

Type:
hypothesis

Confidence:
none

Evidence:
- OpenCode Study-001, Q-020 (extension points question, still open)
- OpenCode Study-001, specs.md "Extension points" section (empty)

Notes:
Study-001 identified Q-020 about extension points but did not investigate it. The absence of documented extension points does not prove they don't exist. Resolving this is critical for evaluating Position 4 (provider-level interceptor) and Position 6 (LLM execution wrapper) viability.

CLAIM-007
Statement:
A custom OpenCode provider implementation can act as a routing layer by intercepting `getModel`/`getLanguage` calls and delegating to different backend providers based on request context.

State:
hypothesis

Type:
hypothesis

Confidence:
none

Evidence:
- OpenCode Study-001, EVID-019 (Provider.getModel and Provider.getLanguage are the model materialization layer)

Notes:
OpenCode's provider abstraction may be a natural interception point. The degree of invasiveness depends on whether custom providers can be registered through configuration alone.

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
