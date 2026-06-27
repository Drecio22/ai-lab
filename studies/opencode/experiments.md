# OpenCode Experiments

This file indexes planned and completed experiments for the OpenCode study.

Format:

```text
EXP-000
Question:
...

Hypothesis:
...

State:
planned / running / completed / inconclusive

Run records:
- pending

Related claims:
- CLAIM-000
```

EXP-001
Question:
Does an OpenCode agent use the model defined in its own configuration?

Hypothesis:
An agent configured with a specific model uses that model instead of the global default model.

State:
planned

Run records:
- pending

Related claims:
- CLAIM-001
- CLAIM-002

EXP-002
Question:
Does an OpenCode subagent inherit context from the parent agent?

Hypothesis:
A subagent receives a bounded subset of context from the parent agent rather than the full session context.

State:
planned

Run records:
- pending

Related claims:
- CLAIM-004
- CLAIM-005

EXP-003
Question:
Can OpenCode work with LiteLLM as an OpenAI-compatible endpoint?

Hypothesis:
OpenCode can use LiteLLM through an OpenAI-compatible provider configuration if LiteLLM exposes the expected chat completion interface.

State:
planned

Run records:
- pending

Related claims:
- CLAIM-007

EXP-004
Question:
Does a user-facing app prompt produce the observed sequence `promptAsync -> SessionPrompt.prompt -> user message -> loop -> LLM stream -> persisted text parts`?

Hypothesis:
Submitting a normal prompt from the OpenCode app follows the `promptAsync` path and produces observable session message/part updates corresponding to streamed model output.

State:
planned

Run records:
- pending

Related claims:
- CLAIM-011
- CLAIM-012
- CLAIM-014
- CLAIM-015

EXP-005
Question:
How does persisted streamed output become visible in the UI after a prompt is submitted?

Hypothesis:
The UI subscribes to session synchronization or event updates, and rendered response text appears as session parts are updated by the processor.

State:
planned

Run records:
- pending

Related claims:
- CLAIM-016

EXP-006
Question:
Does the static prompt-to-DOM trace observed in EVID-013 and EVID-014 occur in order during a real OpenCode app prompt submission?

Hypothesis:
A normal app prompt produces the runtime sequence `promptAsync -> SessionPrompt.prompt -> message.updated/part.updated -> LLM stream -> message.part.delta -> SSE /event -> serverSDK.event.listen -> session.apply -> MessageTimeline -> MessagePart -> Markdown DOM update`.

State:
completed

Result:
The hypothesis is partially confirmed. The server-side portion of the sequence (`promptAsync -> SessionPrompt.prompt -> message.updated/part.updated -> LLM stream -> message.part.delta`) is **runtime-confirmed** via EventV2Bridge event capture.

The client-side portions (`SSE /event -> serverSDK.event.listen -> session.apply -> MessageTimeline -> MessagePart -> Markdown DOM update`) are:
- `SSE /event`: confirmed by httpapi-event.test.ts
- `session.apply`: confirmed by server-session.test.ts
- `MessagePart/Markdown DOM`: static trace only (EVID-014)

Key divergence from static trace: the runtime uses v1 events (`message.part.delta`, `message.updated`, `message.part.updated`) rather than the v2 `session.next.*` events referenced in earlier analysis. The v1 event path is the active system at the tested commit.

Run records:
- EXP-006-run-001

Related claims:
- CLAIM-017
- CLAIM-016

EXP-007
Question:
Can the remaining runtime UI rendering gap be closed with a minimal component-level DOM test instead of a full Playwright e2e?

Hypothesis:
A minimal happy-dom component test can mount the exported `Part` component with a completed assistant text part, wrap it with the same essential providers used by Storybook (`DataProvider`, `MarkedProvider`, `I18nProvider`/fallback i18n as required, and any lightweight UI providers needed by child components), then assert that `[data-component="text-part"]` contains `[data-component="markdown"]` and markdown-derived DOM nodes such as `h2`, `ul/li`, `code`, or sanitized links.

State:
completed

Result:
The hypothesis is **confirmed**. A minimal happy-dom component test (2 test cases, real `MarkedProvider` + `DataProvider`, no Playwright, no server) is sufficient to validate the `MessagePart -> TextPartDisplay -> Markdown -> DOM` rendering segment.

**Final execution command:**
```
cd packages/session-ui
bun test --conditions=browser --preload ./src/components/happydom.ts src/components/message-part-dom.test.tsx
```

**Result:** 2 pass, 0 fail, 12 expect() calls.

**Runtime-confirmed DOM:**
- `[data-component="text-part"]` and `[data-component="markdown"]` exist
- `h2` text === "Runtime heading"
- 2 `li` items
- `code` contains "inlineCode"
- `a` href === "https://opencode.ai"

**Environment blockers resolved:**
1. Bun not in PATH — Bun existed at `%APPDATA%\npm\bun.cmd`; PATH prepended.
2. Vite `?worker&url` import — mocked via `mock.module("./markdown-worker", ...)` in a custom preload.
3. SolidJS server build — resolved via `--conditions=browser`.
4. JSX transpilation — resolved via `globalThis.React = { createElement: h }` from `solid-js/h`.

**Test infrastructure files:**
- Created: `packages/session-ui/src/components/happydom.ts` (preload with worker mock + React shim)
- Modified: `packages/session-ui/src/components/message-part-dom.test.tsx` (removed explicit happy-dom import)

No production code was modified.

Run records:
- EXP-007-run-001 (completed)

Related claims:
- CLAIM-017
- CLAIM-019
