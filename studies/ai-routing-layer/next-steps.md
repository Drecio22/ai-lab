# Study-002 — Next Investigation Proposal

## Current State

The architectural map is complete for the conceptual analysis phase. Eight positions have been identified and documented with their tradeoffs.

The strongest candidates are:
1. **Client-side preprocessor** (Position 1) — best invasiveness/expressiveness tradeoff
2. **Agent config routing** (Position 2) — most OpenCode-native for static routing
3. **External HTTP proxy** (Position 5) — most transparent for dynamic routing

The Q-002 iteration (EVID-002/EVID-003) resolved U-001, U-002, U-004, with runtime confirmation from EVID-004 resolving U-006. Position 4 is now confirmed **not viable on the active V1 path** without core modification (CLAIM-012 ⇒ confirmed). V2 `aisdk.language` is confirmed functional (standalone test) but unreachable during live prompts. The next open question shifts to protocol transparency (U-007) for Position 4 on V2, and to the remaining positions (1, 2, 5) for near-term feasibility.

## Recommended next investigation

### Priority 1: Protocol transparency for delegating LanguageModelV3 (U-007)

**Why**: With U-006 resolved, Position 4 is viable on V2 but blocked from active use. The next question is whether it COULD work once V2 becomes active — which depends on whether a delegating `LanguageModelV3` preserves streaming, tool-call, and usage semantics.

**Approach**: Build a V2 plugin that returns a delegating `LanguageModelV3` and test with real prompts (on a throwaway V2-enabled branch or after V2 migration). Document which semantics survive and which break.

**Status**: PENDING — blocked on V2 activation for live prompts.

### Priority 2: Client-side preprocessor (Position 1) feasibility

**Why**: Position 1 remains the strongest candidate (low invasiveness + high expressiveness). Now that we have confidence about Position 4 being blocked, effort should shift to validating Position 1 across clients.

**Approach**: Map all client prompt entry points (addresses U-005). Determine if a shared pre-SDK-hook interception exists.

### Priority 3: HTTP protocol trace (Position 5 / U-003)

**Why**: The external proxy approach remains the most transparent option. Protocol trace determines whether it requires provider-specific handling or can work generically.

**Approach**: Capture actual HTTP requests OpenCode sends to different providers. Document format, streaming, and tool-call serialization differences.

### Priority 2: Trace OpenCode's Provider HTTP Protocol (U-003)

**Why**: Without knowing the exact HTTP request/response shapes OpenCode uses with each provider, we cannot evaluate Position 5 (external proxy) feasibility. This is needed before any proxy implementation. The same protocol-translation problem applies to Position 4 on V2 (U-007).

**Approach**:
- Capture actual HTTP requests OpenCode sends to different providers.
- Document request format (JSON schema, endpoint path, auth headers).
- Document response format (SSE event types, delta encoding, tool call serialization).
- Compare across providers (OpenAI, Anthropic, Google, etc.) to find a common denominator.

**Expected output**: Protocol compatibility matrix; clear evaluation of Position 5 feasibility.

### Priority 3: Map All Client Prompt Entry Points (U-005)

**Why**: Position 1's viability depends on whether a single client-side interception point exists across all OpenCode clients.

**Approach**:
- Identify all OpenCode clients and their prompt submission code paths.
- Determine if they share a common SDK call (e.g., `promptAsync`) or diverge.
- Map where a preprocessor could be inserted per client.

**Expected output**: Client entry point map; clear evaluation of Position 1 portability.

## Recommended Sequence

1. **First**: Priority 0 (runtime path tracer, U-006) — settles Position 4 for the current commit; pure runtime observation, minimal effort.
2. **Second**: Priority 2 (HTTP protocol compatibility, U-003) — may require a lightweight trace experiment; evaluates Position 5 and informs U-007.
3. **Third**: Priority 3 (client entry points, U-005) — source inspection across client repositories; evaluates Position 1 universality.

**After all three**: converge on the recommended routing position and begin implementation-phase investigation (concrete tools, design, prototyping).

## What this investigation explicitly defers

- Product evaluation (LiteLLM, OpenRouter, etc.)
- Implementation design
- Prototyping
- Performance benchmarking
- Production deployment considerations
