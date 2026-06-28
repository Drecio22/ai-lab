# OpenCode Lessons Learned

This document is a synthesis of the OpenCode investigation. It is not a chronological report and does not replace the detailed claims, evidence, experiment logs, or routing study artifacts. It records what an engineer should know before trying to understand or extend OpenCode.

Scope note: these conclusions are limited to the inspected OpenCode checkout and the experiments already recorded in the study. Where a conclusion depends on a known limitation, that limitation is stated explicitly.

## 1. General Architecture Discovered

OpenCode is not a single linear CLI around an LLM call. It is an application system with distinct layers for client interaction, session management, prompt processing, provider/model materialization, LLM streaming, event publication, and UI rendering.

The normal investigated app path is organized around a session. A user submits a prompt, OpenCode creates or updates session state, persists a user message, resolves the effective model reference, creates an assistant turn, streams LLM output, publishes events, and projects those events into UI state.

The architecture contains both current and emerging generations:

- The active prompt path observed in the study is the V1 session/provider path.
- A V2/core architecture exists with Effect services, a V2 catalog, V2 plugins, and V2 runner components.
- The V2 system is wired and active in some domains, but the runtime tracer confirmed that V2 `aisdk.sdk` and `aisdk.language` hooks do not fire during the active V1 live prompt path.

This matters because many tempting extension points exist in the repository but are not necessarily on the path that executes a normal prompt. An engineer extending OpenCode must first determine whether they are modifying an active execution path or a future/parallel architecture.

## 2. Prompt To Response Pipeline

The normal app prompt path was statically traced and validated by segment-level runtime evidence. The high-level flow is:

```text
user/client prompt
  -> promptAsync / session prompt endpoint
  -> SessionPrompt.prompt
  -> createUserMessage
  -> runLoop
  -> Provider.getModel
  -> SessionProcessor.process
  -> LLM.stream
  -> provider runtime / AI SDK stream
  -> internal LLM events
  -> message/session events
  -> SSE/client store
  -> timeline/message parts
  -> Markdown/DOM rendering
```

The investigation confirmed the full normal app path per segment. There is no single end-to-end test covering the entire chain in one execution, but each segment of the static trace has supporting runtime evidence in the study.

Important details:

- Prompt processing creates a user message before the assistant loop begins.
- The assistant response is streamed through internal LLM events and persisted as message parts/deltas.
- The observed streaming event system for the active path uses V1 events such as message part updates, not the V2 `session.next.*` path that appeared in earlier analysis.
- UI rendering for normal text output flows through message parts into markdown rendering and finally DOM output.

Known limitation: the native LLM runtime path was not tested. The confirmed runtime path covers the AI SDK path observed in the investigation.

## 3. Effective Model Resolution

The most important model-selection fact is that OpenCode resolves the effective model early. For the normal `SessionPrompt.prompt` path, `createUserMessage` chooses the model by this precedence:

```text
input.model ?? agent.model ?? currentModel(sessionID)
```

Once this reference is chosen and persisted on the user message, later layers materialize and execute that model. They do not re-route it based on the prompt.

The later steps are:

- `runLoop` reads the persisted user-message model reference.
- `Provider.getModel` resolves `{ providerID, modelID }` into a provider model object.
- `SessionProcessor.process` receives that already-resolved model.
- `LLM.stream` obtains the language model/runtime and executes the request.

The explicit `input.model` branch was runtime-confirmed with propagation through the later path. The `currentModel(sessionID)` fallback was runtime-confirmed for selection. The `agent.model` branch is supported by static evidence but could not be dynamically confirmed due to test harness limitations.

This has a direct engineering implication: dynamic primary model routing must happen at or before `createUserMessage`, or else it must use a deeper interception mechanism that effectively bypasses or wraps the committed model. Trying to change the primary model downstream through ordinary chat hooks is too late.

Subagent model selection is a special case. In the inspected `TaskTool` path, child sessions use the subagent's `next.model` when present; otherwise they inherit the parent assistant message model. That is a form of model selection during delegation, but it is not general primary-prompt routing.

## 4. Model Catalog

OpenCode already constructs a rich model catalog. It should not be treated as a simple static list hardcoded in the UI.

For the active V1 path, the effective catalog is built by `Provider.Service`. Its raw starting point is `ModelsDev.Service`, which can load models.dev data from cache, an embedded snapshot, an override path, or upstream `models.dev`. That raw data is then transformed, extended, activated, and filtered.

The active V1 catalog is provider-indexed, not a first-class flat `AvailableModel[]`:

```ts
Record<ProviderID, Provider.Info>
```

Each provider contains a nested model map:

```ts
models: Record<ModelID, Model>
```

Model entries carry enough information for capability-aware decisions: provider/model identity, display name, provider-facing API ID, SDK package/API metadata, capabilities, modalities, tool-call support, reasoning/temperature/attachment/interleaving flags, context/input/output limits, cost and cache pricing, status, family, release date, headers/options, and variants.

The effective catalog is not just models.dev. It is affected by:

- models.dev baseline provider/model data.
- `provider` configuration.
- custom provider definitions.
- plugin provider model hooks.
- environment credentials.
- stored auth credentials.
- built-in provider customizers.
- selected provider-specific discovery loaders.
- enabled/disabled provider filters.
- whitelist/blacklist model filters.
- alpha/deprecated model filtering.
- variant generation and config overrides.

There is no global model deduplication across providers. The stable identity is `providerID/modelID`, optionally with a variant. Provider-facing API IDs can differ from OpenCode-facing model IDs, which allows aliases and custom deployments.

V2 has a cleaner catalog shape for future use. `Catalog.Service` exposes `catalog.model.available()`, which returns a flat list of available `ModelV2.Info` entries. However, the V2 catalog is not the active V1 prompt catalog in the confirmed runtime path.

Engineering implication: a future in-process Decision Engine should reuse OpenCode's catalog infrastructure rather than reconstruct provider discovery independently. But a V1 plugin does not currently have a documented direct service boundary to read the complete active V1 catalog. It would need an API/client call or a new supported catalog hook/service.

## 5. Plugin System

OpenCode has a real plugin system, and this matters. The investigation found two plugin generations.

The V1 plugin system is documented and active for the current session execution path. It exposes hooks for events, tools, shell environment, permissions, chat parameters/headers, chat message/system/message transforms, provider model catalogs, small model selection, compaction, and other behaviors.

The V2 plugin system is Effect-based and includes transform domains and runtime hooks such as `aisdk.sdk`, `aisdk.language`, and `catalog.transform`. It is wired into the V2/core architecture and can affect some domains, including catalog transformation. But runtime evidence showed that V2 `aisdk.sdk` and `aisdk.language` are not invoked during the active V1 live prompt path.

The important lesson is that a hook existing in the type system is not enough. The hook must be both:

- on the active execution path; and
- writable at the decision point you care about.

For primary model routing, the decisive negative result is that V1 hooks expose the primary model as read-only input downstream. `chat.params` fires, but it cannot rewrite the selected primary model because its mutable output is generation parameters, not model identity. `chat.message` and related chat transforms are also not primary model rewrite hooks. The small-model hook is scoped to auxiliary small-model selection, not normal prompt model selection.

## 6. V1 And V2 Differences

OpenCode is in a transitional architecture. This is one of the most important conclusions for anyone extending it.

V1 characteristics:

- Active normal prompt path.
- `SessionPrompt.prompt` and `createUserMessage` determine the effective prompt model.
- `Provider.Service` owns the active provider/model catalog.
- `Provider.getLanguage` uses the V1 provider materialization path.
- V1 plugin hooks fire on the active path.
- Model routing via V1 plugins is limited because primary model identity is not writable at the downstream hooks.

V2 characteristics:

- Effect-service based architecture.
- `Catalog.Service` exposes structured provider/model availability.
- V2 plugins can transform catalog data.
- V2 `aisdk.language` is a genuine provider-abstraction interception point when called.
- Standalone dispatch showed V2 language interception works when invoked.
- Runtime prompt tracing showed V2 AISDK hooks are not invoked by the active V1 prompt path.

The split verdict is therefore precise: provider-level interception is viable in the V2 design, but not currently reachable from normal V1 prompt execution without core changes or V2 activation.

## 7. What Can Be Extended Today

The current documented and observed extension surface supports several useful extension categories.

Today, without modifying core, an engineer can reasonably extend:

- tool behavior via tool hooks and custom tools;
- event observation and side effects;
- shell environment injection;
- compaction context/prompt behavior;
- permission handling;
- provider/model catalog entries through configuration;
- custom OpenAI-compatible providers through config;
- static model selection via explicit model input, config defaults, and agent model configuration;
- small-model selection through the scoped small-model hook;
- model lists for known providers through the V1 provider model hook, within its limitations.

An engineer can also build external routing around OpenCode without modifying core by deciding the `input.model` before prompt submission. This aligns with the actual model resolution point.

For static or semi-static routing, agent-to-model mapping is also a natural OpenCode-native mechanism. It routes by choosing the agent rather than rewriting the model later.

## 8. What Requires Modifying Core

Some goals require core changes because the existing public hooks do not expose the right writable decision point.

Primary per-prompt model routing inside the active V1 prompt path requires core modification unless it is done before `createUserMessage` through an existing input surface. Possible core-level changes would include adding a model-decision hook before persistence, exposing a supported provider/model router service, or opening the model-loader/materialization layer in a stable way.

Installing a true provider-level primary model interceptor in V1 also requires core changes. The dynamic language-model materialization registry is internal and not exposed by the V1 plugin provider hook.

Direct plugin access to the complete active model catalog also requires either a new public service/hook/API or relying on an existing client/server API shape. The internal catalog exists, but internal existence is not the same as a stable plugin boundary.

Using V2 `aisdk.language` for prompt routing today would require V2 to become the active prompt execution path or require core bridging from V1 provider materialization into the V2 AISDK service.

## 9. What Not To Attempt

Do not try to implement primary model routing in V1 by mutating `chat.params`. That hook fires, but the model is read-only input and the output does not include a replacement model.

Do not assume every promising V2 hook affects the current product path. The runtime tracer specifically showed that V2 AISDK runtime hooks are registered but not called during live V1 prompts.

Do not treat models.dev as the whole effective catalog. The actual user-visible and executable catalog is filtered and overlaid by config, credentials, auth, plugin hooks, provider customizers, runtime flags, and variants.

Do not treat model names as globally unique. Use `providerID/modelID`, and include variants when relevant.

Do not build routing logic after the model has already been persisted unless you are deliberately implementing a lower-level proxy/interceptor. At that point OpenCode is materializing and executing an already chosen model.

Do not assume one client path covers every OpenCode surface. The normal app path was traced in detail, but all clients and all entry points were not proven equivalent.

## 10. Lessons Learned

The most important architecture lesson is that model identity is an input decision in OpenCode's active path, not a late execution detail. The cleanest routing point is before the effective model is persisted.

The second lesson is that extension-point quantity is not extension-point suitability. OpenCode has many hooks, but the primary model is not writable at the downstream V1 hooks investigated.

The third lesson is that a generational rewrite can make conclusions conditional. V2 contains better abstractions for catalog and provider/language-model interception, but the active prompt path remains V1 for the investigated runtime.

The fourth lesson is that an internal catalog and a public catalog API are different things. OpenCode already constructs a useful model inventory, but an external Decision Engine still needs a stable way to consume it.

The fifth lesson is that segment-level validation can be enough to understand a large UI/server/LLM pipeline. The study did not produce one full end-to-end test, but it did validate each segment of the normal prompt-to-screen chain.

## 11. Questions Still Open

The investigation leaves several questions open or partially open:

- Whether all OpenCode clients share the same prompt submission and model selection path. The normal app path is confirmed; universal client equivalence is not.
- Whether the `agent.model` branch behaves dynamically exactly as the static code indicates. Static precedence is clear, but the runtime branch was blocked in the test harness.
- Whether a V2 delegating `LanguageModelV3` preserves all streaming, tool-call, error, usage, and accounting semantics expected by OpenCode.
- Which stable public API should an external Decision Engine use to read the complete effective model catalog.
- Whether an external HTTP proxy can transparently preserve all provider-specific streaming and tool-call semantics for OpenCode.
- How and when the active provider/model catalog is refreshed after credential or config changes in every runtime mode.

These are not blockers for understanding the current active architecture. They are boundaries around what was proven.

## Final Takeaway

An engineer extending OpenCode should start from three facts.

First, the active prompt path resolves the model early through `input.model ?? agent.model ?? currentModel(sessionID)`.

Second, OpenCode already builds a rich provider/model catalog, but the active V1 catalog is provider-indexed and internally exposed through `Provider.Service`, while the cleaner flat V2 catalog is not the active prompt catalog yet.

Third, OpenCode's plugin system is real and useful, but it does not currently provide a no-core-change hook for primary per-prompt model rewriting on the active V1 path.

The safest extension strategy is therefore to either operate before prompt submission, use existing static configuration surfaces, or explicitly modify core to introduce a supported routing/catalog boundary.
