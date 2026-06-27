# OpenCode Provisional Specs

This file contains provisional technical specifications for the OpenCode study.

Specs must separate confirmed facts, hypotheses, inferences, and open questions. Do not invent conclusions.

## Architecture

State: empty

Confirmed facts:
- none yet

Hypotheses:
- pending investigation

Open questions:
- Q-001
- Q-003

## Prompt-to-response flow

State: draft

Confirmed facts:
- `SessionPrompt.prompt` creates a user message and publishes `message.updated` and `message.part.updated` events. [EVID-015]
- `SessionPrompt.loop` creates an assistant message, calls the LLM, publishes `message.part.delta` events during streaming, and finalizes with `message.updated`. [EVID-015]
- The event sequence for a normal prompt with LLM response is: `session.created → session.updated → message.updated → message.part.updated → session.status → message.updated → message.part.updated → message.part.delta → message.updated → session.status → session.idle`. [EVID-015]
- The SSE `/event` endpoint delivers instance events including `session.created` and `server.connected`. [httpapi-event.test.ts]
- The client-side session store correctly processes `message.updated`, `message.part.updated`, and `message.part.delta` events via `session.apply()`. [server-session.test.ts]
- Non-empty text parts passed to `<MessagePart>` are dispatched to `TextPartDisplay`, transformed through `readPartText`, rendered via `Markdown` (or `PacedMarkdown` during streaming), and written as sanitized HTML into a Markdown DOM container. [EVID-014 + EVID-017]
- A minimal happy-dom component test is sufficient to validate `MessagePart -> Markdown -> DOM` without Playwright. [EXP-007, confirmed: 2/2 pass, 12 assertions]
- A custom preload (`packages/session-ui/src/components/happydom.ts`) with `mock.module()` for the `?worker&url` import and a `React` shim enables `bun test` execution. [EVID-018]

Hypotheses:
- The app-facing prompt path starts in `packages/app/src/components/prompt-input/submit.ts` and calls `client.session.promptAsync`. [inferred from static trace + runtime on server side]
- The SSE delivery of `message.updated`, `message.part.updated`, and `message.part.delta` events follows the same mechanism as `session.created` (both go through EventV2Bridge → GlobalBus → SSE handler). [inferred from architecture]
- A full end-to-end execution (prompt submission → visible DOM update) follows the static trace order. [inferred from segment-level confirmation; no single test covers the full chain]

Open questions:
- Q-001
- Q-002
- Q-021
- Q-023
- Q-024
- Q-025
- Q-027

Tooling notes:
- Bun test runner cannot resolve Vite `?worker&url` imports natively, but `mock.module()` in a preload provides a clean workaround without modifying production code. [EVID-018, resolved]

Related claims:
- CLAIM-011
- CLAIM-012
- CLAIM-013
- CLAIM-014
- CLAIM-015
- CLAIM-016
- CLAIM-017
- CLAIM-018
- CLAIM-019

Evidence:
- EVID-004
- EVID-005
- EVID-006
- EVID-007
- EVID-008
- EVID-009
- EVID-010
- EVID-011
- EVID-012
- EVID-013
- EVID-014
- EVID-015
- EVID-016
- EVID-017
- EVID-018

## Agents

State: empty

Confirmed facts:
- none yet

Hypotheses:
- pending investigation

Open questions:
- Q-004
- Q-005
- Q-006
- Q-007

## Subagents

State: empty

Confirmed facts:
- none yet

Hypotheses:
- pending investigation

Open questions:
- Q-009
- Q-011
- Q-012

## Routing

State: empty

Confirmed facts:
- none yet

Hypotheses:
- pending investigation

Open questions:
- Q-013
- Q-020

## Model selection

State: empty

Confirmed facts:
- none yet

Hypotheses:
- pending investigation

Open questions:
- Q-007
- Q-008
- Q-014
- Q-016

## Providers

State: empty

Confirmed facts:
- none yet

Hypotheses:
- pending investigation

Open questions:
- Q-015
- Q-016

## LiteLLM

State: empty

Confirmed facts:
- none yet

Hypotheses:
- pending investigation

Open questions:
- Q-017
- Q-018

## Ollama

State: empty

Confirmed facts:
- none yet

Hypotheses:
- pending investigation

Open questions:
- Q-019

## Extension points

State: empty

Confirmed facts:
- none yet

Hypotheses:
- pending investigation

Open questions:
- Q-020

## Limitations

State: empty

Confirmed facts:
- none yet

Hypotheses:
- pending investigation

Open questions:
- pending investigation
