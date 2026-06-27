# OpenCode Claims

This file tracks verifiable technical claims for the OpenCode study.

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

Format:

```text
CLAIM-000
Statement:
...

State:
unknown

Type:
observed / recommended / hypothesis

Confidence:
none / low / medium / high

Evidence:
- pending

Experiments:
- pending

Notes:
...
```

CLAIM-001
Statement:
OpenCode allows assigning a specific model to an agent.

State:
unknown

Type:
hypothesis

Confidence:
none

Evidence:
- pending

Experiments:
- EXP-001

Notes:
Needs validation in documentation, source code, and runtime behavior.

CLAIM-002
Statement:
An agent-specific model takes precedence over the global default model.

State:
unknown

Type:
hypothesis

Confidence:
none

Evidence:
- pending

Experiments:
- EXP-001

Notes:
Precedence must be verified directly.

CLAIM-003
Statement:
OpenCode supports user-defined agents through project or user configuration.

State:
unknown

Type:
hypothesis

Confidence:
none

Evidence:
- pending

Experiments:
- pending

Notes:
Need to identify configuration sources and schema.

CLAIM-004
Statement:
OpenCode has a delegation mechanism that can invoke subagents.

State:
unknown

Type:
hypothesis

Confidence:
none

Evidence:
- pending

Experiments:
- EXP-002

Notes:
Need to determine whether subagent is a formal construct or invocation pattern.

CLAIM-005
Statement:
A subagent inherits some context from the parent agent.

State:
unknown

Type:
hypothesis

Confidence:
none

Evidence:
- pending

Experiments:
- EXP-002

Notes:
The scope of inherited context must be measured.

CLAIM-006
Statement:
A subagent inherits tools and permissions from the parent agent.

State:
unknown

Type:
hypothesis

Confidence:
none

Evidence:
- pending

Experiments:
- pending

Notes:
This may depend on implementation or configuration.

CLAIM-007
Statement:
OpenCode can use LiteLLM as an OpenAI-compatible endpoint.

State:
unknown

Type:
hypothesis

Confidence:
none

Evidence:
- pending

Experiments:
- EXP-003

Notes:
Need to distinguish native LiteLLM support from generic OpenAI-compatible support.

CLAIM-008
Statement:
OpenCode can use Ollama either directly or via an OpenAI-compatible endpoint.

State:
unknown

Type:
hypothesis

Confidence:
none

Evidence:
- pending

Experiments:
- pending

Notes:
Need to verify provider path and tool support.

CLAIM-009
Statement:
OpenCode performs dynamic internal routing between models based on task characteristics.

State:
unknown

Type:
hypothesis

Confidence:
none

Evidence:
- pending

Experiments:
- pending

Notes:
Need to distinguish real routing from static config resolution.

CLAIM-010
Statement:
OpenCode exposes extension points where an intelligent model router could be implemented.

State:
unknown

Type:
hypothesis

Confidence:
none

Evidence:
- pending

Experiments:
- pending

Notes:
Potential extension points must be found in code or documented APIs.

CLAIM-011
Statement:
The user-facing OpenCode app submits normal prompts through `client.session.promptAsync`.

State:
supported

Type:
observed

Confidence:
medium

Evidence:
- EVID-004

Experiments:
- pending

Notes:
Supported by app source code. Runtime confirmation is still needed to mark as confirmed.

CLAIM-012
Statement:
The `promptAsync` HTTP endpoint accepts the prompt, starts prompt processing asynchronously, and returns immediately with no content.

State:
supported

Type:
observed

Confidence:
medium

Evidence:
- EVID-005

Experiments:
- pending

Notes:
Supported by endpoint source and OpenAPI annotation. Needs runtime verification for user-visible behavior.

CLAIM-013
Statement:
OpenCode also exposes a v2 `session.prompt` endpoint that durably admits a session input and schedules agent-loop execution unless `resume` is false.

State:
supported

Type:
observed

Confidence:
medium

Evidence:
- EVID-006
- EVID-007

Experiments:
- pending

Notes:
This is a separate v2/core flow. Its relationship to the current app `promptAsync` path remains uncertain.

CLAIM-014
Statement:
During prompt processing, OpenCode creates a user message before starting the assistant loop.

State:
supported

Type:
observed

Confidence:
medium

Evidence:
- EVID-008

Experiments:
- pending

Notes:
Supported by `SessionPrompt.prompt` calling `createUserMessage` and updating session state before `loop`.

CLAIM-015
Statement:
OpenCode streams provider output through an internal LLM event stream and persists text deltas/parts as they arrive.

State:
supported

Type:
observed

Confidence:
medium

Evidence:
- EVID-009
- EVID-010
- EVID-011

Experiments:
- pending

Notes:
Supported by source code in both current `opencode` session processor and v2 core runner publisher. UI display path still requires evidence.

CLAIM-016
Statement:
The exact path from persisted streamed parts/events to visible screen updates is not yet established.

State:
refuted

Type:
observed

Confidence:
medium

Evidence:
- EVID-013
- EVID-014
- EVID-015

Experiments:
- EXP-005

Notes:
Refuted for the normal app path by static call trace (EVID-013+014) and runtime-confirmed for server-side segment (EVID-015). The path is fully mapped statically and runtime-validated for the server-to-store segments.

CLAIM-017
Statement:
For the normal OpenCode app path at commit `ae53163`, a prompt can be statically traced from UI submission through `promptAsync`, session prompt processing, LLM streaming, event publication, SSE delivery, session store updates, timeline projection, `MessagePart`, `TextPartDisplay`, `Markdown`, and DOM update for non-empty text parts.

State:
confirmed

Type:
observed

Confidence:
high

Evidence:
- EVID-013 (static trace — full app-side chain)
- EVID-014 (static trace — MessagePart to DOM)
- EVID-015 (runtime — server-side prompt→event)
- httpapi-event.test.ts (runtime — SSE delivery)
- server-session.test.ts (runtime — client store processing)
- EVID-017 (runtime — MessagePart→Markdown→DOM via happy-dom test)
- EXP-007-run-001 (executed, 2/2 pass, 12 expect assertions)

Experiments:
- EXP-006
- EXP-007

Notes:
All segments of the static trace now have runtime evidence:
1. Server prompt → event publishing — EVID-015 ✅
2. SSE delivery — httpapi-event.test.ts ✅
3. Client session-store processing — server-session.test.ts ✅
4. UI rendering (MessagePart → Markdown → DOM) — EVID-017 ✅

The full chain from `promptAsync` call to DOM content under `[data-component="markdown"]` is both statically traced and validated per segment. No single end-to-end test covers all segments in one execution.

CLAIM-018
Statement:
The static app prompt-to-screen trace applies unchanged to all OpenCode clients, including TUI, CLI, SDK, desktop, and IDE flows.

State:
unknown

Type:
hypothesis

Confidence:
none

Evidence:
- pending

Experiments:
- pending

Notes:
Out of scope for the current question iteration. The traced path is the app normal prompt path, not all clients.

CLAIM-019
Statement:
At the inspected OpenCode checkout, there is no existing automated unit/component test that validates DOM rendering for `MessagePart -> TextPartDisplay -> Markdown / PacedMarkdown -> DOM`.

State:
supported

Type:
observed

Confidence:
high

Evidence:
- EVID-016

Experiments:
- EXP-007

Notes:
Storybook provides manual rendering harnesses and app e2e tests cover timeline-level behavior, but no inspected test mounts the target components and asserts markdown DOM output directly.

CLAIM-020
Statement:
For the normal `SessionPrompt.prompt` path, `SessionPrompt.createUserMessage` decides and persists the effective model reference for a user prompt by choosing `input.model ?? agent.model ?? currentModel(sessionID)`.

State:
supported

Type:
observed

Confidence:
medium

Evidence:
- EVID-019 (source code)
- EVID-020 (run 001: explicit input.model confirmed)
- EVID-021 (run 003: fallback currentModel confirmed)
- EVID-022 (run 002: agent.model blocked, static code confirms precedence)

Experiments:
- EXP-008

Notes:
EVID-020 runtime-confirms the explicit `input.model` branch. EVID-021 runtime-confirms the fallback `currentModel(sessionID)` branch. The `agent.model` branch could not be dynamically confirmed due to testing blockages (EVID-022), but static code confirms its precedence in the `??` chain.

CLAIM-021
Statement:
`SessionPrompt.runLoop` resolves the persisted user-message model reference into a concrete provider model via `Provider.getModel` before creating the assistant message and invoking `SessionProcessor.process`.

State:
confirmed

Type:
observed

Confidence:
high

Evidence:
- EVID-019 (static)
- EVID-020 (runtime for explicit input.model branch)

Experiments:
- EXP-008

Notes:
Confirmed for the explicit `input.model` branch by EVID-020 (runtime observed: `runLoop.getModel`, `processor.process.input`, `llm.stream.input`, `TestLLMServer`). The `agent.model` and `currentModel` branches were not observed propagating through `runLoop`/`getModel`/`Processor`/`LLM.stream` in runtime. The mechanism itself (resolve persisted model via `getModel`) is confirmed for the observed branch.

CLAIM-022
Statement:
Subagent execution through `TaskTool` chooses `next.model` when the target subagent defines one, otherwise inherits the parent assistant message's `providerID/modelID`, and then calls `SessionPrompt.prompt` for the child session with that model.

State:
supported

Type:
observed

Confidence:
medium

Evidence:
- EVID-019

Experiments:
- pending

Notes:
This covers the inspected `TaskTool` subagent path only. Other subtask construction paths may exist and should not be ruled out prematurely.

CLAIM-023
Statement:
The final LLM execution point is `LLM.stream`: it obtains the provider language model through `Provider.getLanguage(input.model)` and either streams through the native runtime or calls AI SDK `streamText` with `wrapLanguageModel({ model: language, ... })`.

State:
confirmed

Type:
observed

Confidence:
high

Evidence:
- EVID-019
- EVID-020

Experiments:
- EXP-008

Notes:
Confirmed for the AI SDK runtime path by EVID-020. `LLM.stream` executes the already resolved `Provider.Model`; it selects runtime adapter (`native` or `ai-sdk`) but did not choose a different effective model in the inspected explicit-model run.
