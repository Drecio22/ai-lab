# Study-002 — Next Investigation Proposal

## Current State

The architectural map is complete for the conceptual analysis phase. Eight positions have been identified and documented with their tradeoffs.

The strongest candidates are:
1. **Client-side preprocessor** (Position 1) — best invasiveness/expressiveness tradeoff
2. **Agent config routing** (Position 2) — most OpenCode-native for static routing
3. **External HTTP proxy** (Position 5) — most transparent for dynamic routing

## Proposal: Resolve the Extension Points and Provider Architecture

Before designing or implementing any router, the following uncertainties must be resolved because they affect which positions are viable:

### Priority 1: Investigate OpenCode's Extension Point Architecture (U-001, U-002, U-004)

**Why**: Without knowing whether OpenCode has service-layer interception, custom provider registration, or middleware support, we cannot evaluate Position 4's true viability. This is the cheapest investigation that could open a new low-invasiveness routing position.

**Approach**:
- Inspect OpenCode's Effect-TS `Layer`/`Service` architecture.
- Find how providers are registered and wired.
- Determine if a custom provider can be added via configuration alone.
- Look for any existing middleware, plugin, or hook system.
- Check the OpenCode documentation and issues for "extension", "plugin", "middleware", "hook" mentions.

**Expected output**: Updated knowledge about extension points; clear evaluation of Position 4 feasibility.

### Priority 2: Trace OpenCode's Provider HTTP Protocol (U-003)

**Why**: Without knowing the exact HTTP request/response shapes OpenCode uses with each provider, we cannot evaluate Position 5 (external proxy) feasibility. This is needed before any proxy implementation.

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

1. **First**: Resolve U-001, U-002, U-004 (extension points + provider architecture).
   - Pure source inspection; no experiments, no proxy, no product investigation.
   - Could open Position 4 as a viable low-invasiveness option.

2. **Second**: Resolve U-003 (HTTP protocol compatibility).
   - May require a lightweight trace experiment (capture real OpenCode provider traffic).
   - Evaluates Position 5 feasibility.

3. **Third**: Resolve U-005 (client entry points).
   - Source inspection across client repositories.
   - Evaluates Position 1 universality.

**After all three**: converge on the recommended routing position and begin implementation-phase investigation (concrete tools, design, prototyping).

## What this investigation explicitly defers

- Product evaluation (LiteLLM, OpenRouter, etc.)
- Implementation design
- Prototyping
- Performance benchmarking
- Production deployment considerations
