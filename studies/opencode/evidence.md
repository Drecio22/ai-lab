# OpenCode Evidence Register

This file records evidence used to support, weaken, or refute OpenCode claims.

Format:

```text
EVID-000
Type:
source-code / official-docs / test / release / PR / issue / discussion / community / experiment

Source:
...

Date:
YYYY-MM-DD

Summary:
...

Related claims:
- CLAIM-000

Limitations:
...
```

EVID-001
Type:
official-docs

Source:
pending

Date:
pending

Summary:
Placeholder for official OpenCode documentation about agents, configuration, models, and providers.

Related claims:
- CLAIM-001
- CLAIM-003
- CLAIM-007

Limitations:
Documentation must be checked against source code and experiments.

EVID-002
Type:
source-code

Source:
pending

Date:
pending

Summary:
Placeholder for source-code evidence about model selection, provider resolution, agent loading, and delegation.

Related claims:
- CLAIM-001
- CLAIM-002
- CLAIM-004
- CLAIM-009
- CLAIM-010

Limitations:
Source code describes implementation but still needs runtime validation for important behavior.

EVID-003
Type:
experiment

Source:
pending

Date:
pending

Summary:
Placeholder for runtime evidence produced by initial OpenCode experiments.

Related claims:
- CLAIM-001
- CLAIM-005
- CLAIM-007

Limitations:
Experiment results will depend on versions, configuration, providers, and available models.

EVID-004
Type:
source-code

Source:
anomalyco/opencode commit ae53163cad0048b2351e258699e815f4f2110807, `packages/app/src/components/prompt-input/submit.ts` lines 281-591; especially lines 565-573 and 155-162.

Date:
2026-06-27

Summary:
The app prompt submit handler captures the prompt, ensures or creates a session, builds a follow-up draft, creates an optimistic user message, and calls `sendFollowupDraft`; `sendFollowupDraft` calls `input.client.session.promptAsync` with `sessionID`, `agent`, `model`, `messageID`, `parts`, and `variant`.

Related claims:
- CLAIM-011

Limitations:
This confirms the app source path, not that a real runtime execution follows it in every OpenCode client.

EVID-005
Type:
source-code

Source:
anomalyco/opencode commit ae53163cad0048b2351e258699e815f4f2110807, `packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts` lines 309-327 and `groups/session.ts` lines 329-341.

Date:
2026-06-27

Summary:
The `promptAsync` endpoint requires the session, forks `promptSvc.prompt(...)` in a scope with `startImmediately: true`, and returns HTTP no content. The route description says it creates and sends a new message asynchronously, starts the session if needed, and returns immediately.

Related claims:
- CLAIM-012

Limitations:
Does not show how frontend receives later streamed updates.

EVID-006
Type:
source-code

Source:
anomalyco/opencode commit ae53163cad0048b2351e258699e815f4f2110807, `packages/protocol/src/groups/session.ts` lines 204-223.

Date:
2026-06-27

Summary:
The v2 protocol defines `POST /api/session/:sessionID/prompt` as `session.prompt`; its description states that it durably admits one session input and schedules agent-loop execution unless `resume` is false.

Related claims:
- CLAIM-013

Limitations:
This is protocol/schema evidence. It does not prove which user-facing clients use this endpoint.

EVID-007
Type:
source-code

Source:
anomalyco/opencode commit ae53163cad0048b2351e258699e815f4f2110807, `packages/core/src/session.ts` lines 360-386 and `packages/core/src/session/input.ts` lines 41-81, 216-266.

Date:
2026-06-27

Summary:
The v2 core `Session.prompt` resolves the prompt, creates or accepts a message ID, admits it through `SessionInput.admit`, checks equivalence, and wakes execution unless `resume === false`. `SessionInput.admit` publishes `SessionEvent.PromptAdmitted`; later promotion publishes `SessionEvent.Prompted`.

Related claims:
- CLAIM-013

Limitations:
This is v2/core behavior and may not be the same as the current app `promptAsync` flow.

EVID-008
Type:
source-code

Source:
anomalyco/opencode commit ae53163cad0048b2351e258699e815f4f2110807, `packages/opencode/src/session/prompt.ts` lines 1010-1071.

Date:
2026-06-27

Summary:
`SessionPrompt.prompt` loads the session, cleans revert state, creates a user message via `createUserMessage`, touches the session, optionally updates permissions, and calls `loop` unless `noReply === true`.

Related claims:
- CLAIM-014

Limitations:
The detailed implementation of `createUserMessage` was only partially inspected in this iteration.

EVID-009
Type:
source-code

Source:
anomalyco/opencode commit ae53163cad0048b2351e258699e815f4f2110807, `packages/opencode/src/session/prompt.ts` lines 1081-1346.

Date:
2026-06-27

Summary:
The current `SessionPrompt.runLoop` loads session messages, resolves the model and agent, creates an assistant message, resolves tools and system context, converts messages to model messages, and calls `handle.process(...)`; `loop` runs through `state.ensureRunning`.

Related claims:
- CLAIM-015

Limitations:
This confirms orchestration but not the UI display mechanism.

EVID-010
Type:
source-code

Source:
anomalyco/opencode commit ae53163cad0048b2351e258699e815f4f2110807, `packages/opencode/src/session/llm.ts` lines 240-383 and `packages/opencode/src/session/llm/ai-sdk.ts` lines 126-155.

Date:
2026-06-27

Summary:
OpenCode selects an LLM runtime (`native` or `ai-sdk`). The AI SDK path calls `streamText(...)`; `LLM.stream` exposes a normalized `LLMEvent` stream, converting AI SDK `fullStream` events into internal text start/delta/end events.

Related claims:
- CLAIM-015

Limitations:
This does not prove all providers use the same event shapes in practice.

EVID-011
Type:
source-code

Source:
anomalyco/opencode commit ae53163cad0048b2351e258699e815f4f2110807, `packages/opencode/src/session/processor.ts` lines 625-681 and lines 500-529; v2 corroboration in `packages/core/src/session/runner/publish-llm-event.ts` lines 239-267.

Date:
2026-06-27

Summary:
`SessionProcessor.process` calls `llm.stream(...)`, handles each event, and persists text deltas via `session.updatePartDelta`; text end updates the full part. The v2 runner publisher similarly maps `text-start`, `text-delta`, and `text-end` to session events.

Related claims:
- CLAIM-015

Limitations:
Persistence/streaming into session state is shown, but the final UI synchronization path remains unverified.

EVID-012
Type:
official-docs

Source:
https://opencode.ai/docs/ and https://github.com/anomalyco/opencode README, consulted 2026-06-27.

Date:
2026-06-27

Summary:
Official documentation describes OpenCode as an open source AI coding agent with terminal, desktop, and IDE interfaces. It explains user-level prompt usage, plan/build modes, `/init`, and that conversations can be shared, but does not document the internal prompt-to-screen execution path.

Related claims:
- CLAIM-011
- CLAIM-012
- CLAIM-013
- CLAIM-015

Limitations:
Useful contextual evidence only. It is not sufficient to answer Q-001 internally.

EVID-013
Type:
source-code

Source:
Static Call Trace Analysis over anomalyco/opencode commit ae53163cad0048b2351e258699e815f4f2110807.

Date:
2026-06-27

Summary:
Static trace for the normal app prompt-to-screen path:

1. `packages/app/src/pages/session.tsx` lines 1748-1786 renders `MessageTimeline` for the active session and passes `visibleUserMessages()`.
   Call performed: JSX renders `<MessageTimeline ... userMessages={visibleUserMessages()} />`.
   Next destination: `packages/app/src/pages/session/timeline/message-timeline.tsx` `MessageTimeline`.

2. `packages/app/src/pages/session.tsx` lines 263-271 creates `timeline = createTimelineModel(...)` and reads `timeline.visibleUserMessages`.
   Call performed: `createTimelineModel({ sessionID: () => params.id, revertMessageID })`.
   Next destination: `packages/app/src/pages/session/timeline/model.ts` `createTimelineModel`.

3. `packages/app/src/pages/session/timeline/model.ts` lines 19-40 syncs session messages for the active session.
   Call performed: `sync().session.sync(id)`.
   Next destination: `packages/app/src/context/directory-sync.ts` `session.sync` wrapper.

4. `packages/app/src/context/directory-sync.ts` lines 113-116 forwards session sync.
   Call performed: `serverSync.session.sync(sessionID, options)`.
   Next destination: `packages/app/src/context/server-session.ts` `sync` implementation.

5. `packages/app/src/components/prompt-input/submit.ts` lines 281-407 handles prompt submit, creates/locates a session, collects current model and agent, and constructs a `FollowupDraft`.
   Call performed: `handleSubmit(...)` builds `draft`.
   Next destination: same file, `sendFollowupDraft`.

6. `packages/app/src/components/prompt-input/submit.ts` lines 565-573 calls `sendFollowupDraft(...)` with the draft and message ID.
   Call performed: `sendFollowupDraft({ client, sync, serverSync, draft, messageID, ... })`.
   Next destination: same file, `sendFollowupDraft`.

7. `packages/app/src/components/prompt-input/submit.ts` lines 106-144 builds request parts, creates an optimistic user message, and adds it to session optimistic state.
   Call performed: `buildRequestParts(...)` and `input.sync.session.optimistic.add(...)`.
   Next destination for persisted prompt: same function lines 155-162.

8. `packages/app/src/components/prompt-input/submit.ts` lines 155-162 sends the prompt through the generated SDK.
   Call performed: `input.client.session.promptAsync({ sessionID, agent, model, messageID, parts, variant })`.
   Next destination: `packages/sdk/js/src/v2/gen/sdk.gen.ts` `promptAsync`.

8a. `packages/sdk/js/src/v2/gen/sdk.gen.ts` lines 4095-4147 maps `promptAsync` parameters to an HTTP request.
    Call performed: `(options?.client ?? this.client).post({ url: "/session/{sessionID}/prompt_async", ... })`.
    Next destination: HTTP endpoint `session.prompt_async`.

9. `packages/opencode/src/server/routes/instance/httpapi/groups/session.ts` lines 329-341 defines `promptAsync` as `POST /:sessionID/prompt_async` and describes async prompt acceptance.
   Call performed by route definition: `HttpApiEndpoint.post("promptAsync", SessionPaths.promptAsync, ...)`.
   Next destination: handler registered in `handlers/session.ts`.

10. `packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts` lines 309-327 handles `promptAsync`.
    Call performed: `promptSvc.prompt({ ...ctx.payload, sessionID }).pipe(... Effect.forkIn(scope, { startImmediately: true }))`.
    Next destination: `packages/opencode/src/session/prompt.ts` `SessionPrompt.prompt`.

11. `packages/opencode/src/session/prompt.ts` lines 1052-1071 implements `SessionPrompt.prompt`.
    Call performed: `createUserMessage(input)`, `sessions.touch(input.sessionID)`, optional permissions update, then `loop({ sessionID })` unless `noReply === true`.
    Next destination: same file, `loop` / `runLoop`.

12. `packages/opencode/src/session/prompt.ts` lines 1010-1049 show `createUserMessage` persisting the user message and parts.
    Call performed: `sessions.updateMessage(info)` and `sessions.updatePart(part)`.
    Next destination: `packages/opencode/src/session/session.ts` event publication for message/part updates.

13. `packages/opencode/src/session/session.ts` lines 634-648 publish `message.updated` and `message.part.updated` events.
    Call performed: `events.publish(SessionV1.Event.MessageUpdated, ...)` and `events.publish(SessionV1.Event.PartUpdated, ...)`.
    Next destination: `EventV2Bridge` event stream consumed by `/event` handler.

14. `packages/opencode/src/session/prompt.ts` lines 1342-1346 implements `loop` via `state.ensureRunning(..., runLoop(sessionID))`.
    Call performed: `state.ensureRunning(input.sessionID, lastAssistant(input.sessionID), runLoop(input.sessionID))`.
    Next destination: same file, `runLoop`.

15. `packages/opencode/src/session/prompt.ts` lines 1081-1285 implements `runLoop` provider turn setup.
    Call performed: load messages, resolve latest user/assistant state, resolve model and agent, create assistant message, resolve tools/system/model messages, then `handle.process(...)`.
    Next destination: `packages/opencode/src/session/processor.ts` `SessionProcessor.process`.

16. `packages/opencode/src/session/processor.ts` lines 98-115 creates a processor handle for the assistant message.
    Call performed: `SessionProcessor.create({ assistantMessage, sessionID, model })`.
    Next destination: same file, handle `process`.

17. `packages/opencode/src/session/processor.ts` lines 625-681 processes the model stream.
    Call performed: `const stream = llm.stream(streamInput)` then `Stream.tap((event) => handleEvent(event))`.
    Next destination: `packages/opencode/src/session/llm.ts` `LLM.stream`.

18. `packages/opencode/src/session/llm.ts` lines 357-383 implements `LLM.stream`.
    Call performed: `run({ ...input, abort: ctrl.signal })`; native runtime returns a stream directly, AI SDK runtime maps `result.result.fullStream` through `LLMAISDK.toLLMEvents(...)`.
    Next destination: native stream or AI SDK stream conversion.

19. `packages/opencode/src/session/llm.ts` lines 280-339 shows the AI SDK path calling `streamText(...)`.
    Call performed: `streamText({ ..., messages: prepared.messages, model: wrapLanguageModel(...) })`.
    Next destination: AI SDK `fullStream`, then `LLMAISDK.toLLMEvents`.

20. `packages/opencode/src/session/llm/ai-sdk.ts` lines 126-155 converts AI SDK text events.
    Call performed: maps `text-start`, `text-delta`, and `text-end` to internal `LLMEvent.textStart`, `LLMEvent.textDelta`, and `LLMEvent.textEnd`.
    Next destination: `SessionProcessor.handleEvent` through the stream tap.

21. `packages/opencode/src/session/processor.ts` lines 500-529 persists streamed text updates.
    Call performed: on text delta, `session.updatePartDelta({ field: "text", delta: value.text })`; on text end, `session.updatePart(ctx.currentText)`.
    Next destination: `packages/opencode/src/session/session.ts` `updatePartDelta` / `updatePart`.

22. `packages/opencode/src/session/session.ts` lines 882-890 publishes text delta events.
    Call performed: `events.publish(MessageV2.Event.PartDelta, input)`.
    Next destination: `/event` SSE stream through `EventV2Bridge`.

23. `packages/opencode/src/server/routes/instance/httpapi/handlers/event.ts` lines 25-85 creates the SSE event response.
    Call performed: `events.listen(...)`, filters events by directory/workspace, maps to `{ id, type, properties }`, encodes with `Sse.encode()`, and returns `text/event-stream`.
    Next destination: app SDK event stream.

24. `packages/opencode/src/server/routes/instance/httpapi/groups/event.ts` lines 7-22 defines `GET /event` as `event.subscribe`.
    Call performed by route definition: `HttpApiEndpoint.get("subscribe", EventPaths.event, ...)`.
    Next destination: generated SDK `global.event(...)` used by the app.

25. `packages/app/src/context/server-sdk.tsx` lines 161-230 starts the event stream.
    Call performed: `eventSdk.global.event(...)`, iterates `events.stream`, calls `enqueueServerEvent(queue, { directory, payload })`, then `schedule()`.
    Next destination: same file, `flush`.

26. `packages/app/src/context/server-sdk.tsx` lines 113-129 flushes queued events.
    Call performed: `coalesceServerEvents(events)` and `emitter.emit(event.directory, event.payload)`.
    Next destination: listeners registered through `serverSDK.event.listen`.

27. `packages/app/src/context/server-sync.tsx` lines 355-400 listens to emitted server events.
    Call performed: `serverSDK.event.listen((e) => { ... session.apply(event); ... applyDirectoryEvent(...) })`.
    Next destination: `packages/app/src/context/server-session.ts` `session.apply`.

28. `packages/app/src/context/server-session.ts` lines 659-903 applies session events to reactive store.
    Call performed: for `message.updated`, update `data.message`; for `message.part.updated`, update `data.part`; for `message.part.delta`, append delta to `data.part_text_accum_delta` and mutate the matching part field.
    Next destination: Solid reactive consumers of `sync().data.message` and `sync().data.part`.

29. `packages/app/src/context/directory-sync.ts` lines 30-44 exposes server session fields through the directory sync proxy.
    Call performed: proxy returns `serverSync.session.data.message`, `part`, and related session fields for `sync().data`.
    Next destination: session page/timeline consumers.

30. `packages/app/src/pages/session/timeline/model.ts` lines 42-57 derives messages and visible user messages from `sync().data.message[id]`.
    Call performed: `messages = createMemo(() => sync().data.message[id] ?? [])`, then `selectUserMessages(...)` and `selectVisibleUserMessages(...)`.
    Next destination: `MessageTimeline` props in `pages/session.tsx`.

31. `packages/app/src/pages/session/timeline/message-timeline.tsx` lines 270-323 consumes session status, messages, and parts.
    Call performed: `sessionMessages = createMemo(...)`, `getMsgParts = (msgId) => sync().data.part[msgId] ?? []`, and `createTimelineProjection({ messages, parts: getMsgParts, ... })`.
    Next destination: timeline projection.

32. `packages/app/src/pages/session/timeline/projection.ts` lines 15-69 groups assistant messages by parent user message and constructs timeline rows.
    Call performed: `Timeline.constructMessageRows(userMessage, input.parts, assistantMessagesByParent().get(userMessage.id) ?? [], ...)`.
    Next destination: `packages/app/src/pages/session/timeline/rows.ts`.

33. `packages/app/src/pages/session/timeline/rows.ts` lines 107-195 converts messages and parts into renderable `TimelineRow.AssistantPart` rows.
    Call performed: `getMessageParts(message.id).filter(renderable).map(...)`, then `groupParts(...)`, then push `new TimelineRow.AssistantPart(...)`.
    Next destination: `MessageTimeline` row renderer.

34. `packages/app/src/pages/session/timeline/message-timeline.tsx` lines 916-974 renders assistant part groups.
    Call performed: resolve `messageByID().get(...)`, resolve `getMsgPart(...)`, then render `<MessagePart part={part()} message={message()} ... />`.
    Next destination: `@opencode-ai/session-ui/message-part` component.

35. `packages/app/src/pages/session/timeline/message-timeline.tsx` lines 1157-1218 renders virtual rows.
    Call performed: `VirtualTimelineRow` calls `<TimelineRowView row={row()} />`, which calls `renderTimelineRow(...)` and ultimately `renderAssistantPartGroup(...)` for assistant rows.
    Next destination: DOM output through Solid rendering.

36. `packages/app/src/pages/session/timeline/message-timeline.tsx` lines 1558-1579 places virtual rows in the timeline container.
    Call performed: `<For each={virtualRowKeys()}>{(rowKey) => <VirtualTimelineRow rowKey={rowKey} />}</For>`.
    Next destination: visible timeline DOM.

Point where static tracing stops:
The trace stops at the external UI component `<MessagePart>` imported from `@opencode-ai/session-ui/message-part` and Solid's DOM rendering. The app code proves that updated `Part` data is passed to `MessagePart`, but this iteration did not inspect the implementation of that external package/component or execute the UI.

Missing evidence:
- Runtime confirmation that the traced path is exercised by a real prompt submission.
- Runtime event sequence showing `promptAsync`, message updates, part deltas, SSE receipt, store mutation, and visible DOM update in order.
- Static continuation from `<MessagePart>` to DOM is recorded separately in EVID-014.

Related claims:
- CLAIM-011
- CLAIM-012
- CLAIM-014
- CLAIM-015
- CLAIM-016
- CLAIM-017

Limitations:
This is static source-code evidence only. It is scoped to the normal app prompt path at commit `ae53163`; it does not prove behavior for every client, every prompt mode, shell mode, command mode, TUI, CLI, SDK, desktop packaging differences, or runtime provider behavior.

EVID-014
Type:
source-code

Source:
Static Call Trace Analysis continuation over anomalyco/opencode commit ae53163cad0048b2351e258699e815f4f2110807, `packages/session-ui/src/components/message-part.tsx`, `packages/session-ui/src/components/message-part-text.ts`, `packages/session-ui/src/components/markdown.tsx`, and `packages/session-ui/src/components/markdown-cache.tsx`.

Date:
2026-06-27

Summary:
Static trace continuation for `MessagePart -> render visible`:

1. `packages/app/src/pages/session/timeline/message-timeline.tsx` lines 958-969 passes the selected assistant part to the external package component.
   Prop received by next component: `part={part()}`, `message={message()}`.
   Transformation applied: none at this boundary; the already selected `Part` and `Message` are forwarded.
   Next destination: `@opencode-ai/session-ui/message-part` export `Part`.

2. `packages/session-ui/src/components/message-part.tsx` lines 173-185 defines `MessagePartProps`.
   Prop received: `part: PartType`, `message: MessageType`, plus rendering options.
   Transformation applied: type contract for part rendering.
   Next destination: same file, exported `Part` component.

3. `packages/session-ui/src/components/message-part.tsx` lines 1272-1291 implements exported `Part`.
   Prop received: `props.part`.
   Transformation applied: `const component = createMemo(() => PART_MAPPING[props.part.type])`; renders `<Dynamic component={component()} part={props.part} message={props.message} ... />`.
   Next destination for normal assistant text: `PART_MAPPING["text"]`.

4. `packages/session-ui/src/components/message-part.tsx` lines 1482-1592 implements `PART_MAPPING["text"]` as `TextPartDisplay`.
   Prop received: `props.part as TextPart`, `props.message`.
   Transformation applied: computes `part = () => props.part as TextPart`; computes `streaming` from assistant completion state; computes `text = () => readPartText(data.store.part_text_accum_delta, part())`.
   Next destination: `readPartText` helper.

5. `packages/session-ui/src/components/message-part-text.ts` lines 1-3 implements `readPartText`.
   Prop received: `accum` and `part` with `id` and optional `text`.
   Transformation applied: returns `(accum?.[part.id] ?? part.text ?? "").trim()`.
   Next destination: `TextPartDisplay` conditional rendering.

6. `packages/session-ui/src/components/message-part.tsx` lines 1559-1566 renders non-empty text.
   Prop received: computed `text()`.
   Transformation applied: `<Show when={text()}>`; creates `<div data-component="text-part" data-timeline-part-id={part().id}>`; renders either `<Markdown text={text()} cacheKey={part().id} streaming={false} />` or `<PacedMarkdown text={text()} cacheKey={part().id} streaming={streaming()} />`.
   Next destination: `PacedMarkdown` or `Markdown`.

7. `packages/session-ui/src/components/message-part.tsx` lines 275-286 implements `PacedMarkdown`.
   Prop received: `text`, `cacheKey`, `streaming`.
   Transformation applied: `createPacedValue` may progressively expose the text while streaming; renders `<Markdown text={value()} cacheKey={props.cacheKey} streaming={props.streaming} />` when value exists.
   Next destination: `Markdown` component.

8. `packages/session-ui/src/components/markdown.tsx` lines 323-341 implements `Markdown` input setup.
   Prop received: `text: string`, optional `cacheKey`, `streaming`.
   Transformation applied: creates projection from `local.text` and `local.streaming`.
   Next destination: markdown resource and DOM effect.

9. `packages/session-ui/src/components/markdown.tsx` lines 350-421 computes render blocks.
   Prop received: current markdown text and projection.
   Transformation applied: for non-code blocks, parses markdown with `marked.parse(block.src)`, sanitizes via `sanitizeMarkdown(...)`, caches result when possible; fallback path uses escaped text from `fallback(src.text)`.
   Next destination: `createEffect` that writes blocks to the container.

10. `packages/session-ui/src/components/markdown-cache.tsx` lines 35-38 implements `sanitizeMarkdown`.
    Prop received: HTML string from markdown parser.
    Transformation applied: `DOMPurify.sanitize(html, config)` when supported.
    Next destination: sanitized block HTML returned to `Markdown`.

11. `packages/session-ui/src/components/markdown.tsx` lines 426-458 performs DOM update.
    Prop received: rendered blocks from `html.latest ?? html()` and projected pending blocks.
    Transformation applied: gets the root container, computes pending blocks, calls `updateBlock(container, index, block, labels)` for each block, and removes stale children.
    Next destination: `updateBlock`.

12. `packages/session-ui/src/components/markdown.tsx` lines 512-552 implements `updateBlock` for non-code markdown blocks.
    Prop received: `container: HTMLDivElement`, block with sanitized `html`.
    Transformation applied: creates a `div`, sets `data-markdown-block`, `data-markdown-key`, `data-markdown-hash`, `style.display = "contents"`, assigns `next.innerHTML = block.html`, decorates it, appends it if no current block exists, otherwise updates via `morphdom(current, next, ...)`.
    Next destination: browser DOM inside the Markdown root.

13. `packages/session-ui/src/components/markdown.tsx` lines 466-476 returns the root container.
    Prop received: component props.
    Transformation applied: renders `<div data-component="markdown" ref={setRoot} ... />`; the effect above mutates this root's children.
    Next destination: visible DOM managed by Solid/browser.

Point where static tracing stops:
The trace stops at browser DOM rendering after `Markdown` writes sanitized HTML into the Markdown root container. This is sufficient static evidence that a non-empty text `part` is transformed into DOM nodes under `<div data-component="markdown">`.

Missing evidence:
- Runtime confirmation that a real prompt produces the full observed sequence and visible DOM update.
- Runtime confirmation that target environments support the sanitization/rendering path as expected.

Related claims:
- CLAIM-017

Limitations:
This evidence covers normal text parts rendered through `@opencode-ai/session-ui/message-part`. It does not cover every part type, every tool-specific renderer, non-app clients, or runtime timing/visibility behavior.

EVID-015
Type:
test-execution

Source:
Integration test execution at commit `ae53163cad0048b2351e258699e815f4f2110807`, `packages/opencode/test/session/exp006-runtime-trace.test.ts`.

Date:
2026-06-27

Summary:
Runtime validation of the server-side prompt-to-event chain via EventV2Bridge capture during a real Effect-layer prompt execution with TestLLMServer streaming.

Observed event sequence (key events, with plugin/catalog noise filtered):

1. `session.created` — Session created
2. `session.updated` — Session metadata updated
3. `message.updated` — User message persisted (via `SessionPrompt.prompt`)
4. `message.part.updated` — User text part persisted
5. `session.updated` — Model/agent update
6. `session.status` — Status transitions to busy
7. `message.updated` — Assistant message created (loop starts)
8. `session.status` — Status update during loop
9. `message.part.updated` — Assistant text part created
10. `message.part.updated` — Assistant text part updated with text
11. `message.part.delta` — Streaming delta received from LLM
12. `message.part.updated` — Part updated with final text
13. `message.updated` — Assistant message finalized
14. `session.status` / `session.idle` — Status transitions to idle

Runtime confirmation of:
- ✅ `SessionPrompt.prompt` → `updateMessage` → `message.updated` event
- ✅ `SessionPrompt.prompt` → `updatePart` → `message.part.updated` event
- ✅ `SessionPrompt.loop` → assistant message creation → `message.updated` event
- ✅ LLM streaming → `updatePartDelta` → `message.part.delta` event
- ✅ Session status transitions during loop (busy/idle)
- ✅ Assistant text part contains LLM output ("world")

Not observed in this test (covered by other tests):
- SSE delivery of these events (validated separately by `httpapi-event.test.ts`)
- Client-side session store processing (validated by `server-session.test.ts`)
- Browser DOM rendering (static trace only)
- v2 `session.next.*` events (the v1 event path is the active system)

Related claims:
- CLAIM-017
- CLAIM-016

EVID-016
Type:
source-code

Source:
Repository test/story/harness inventory over `<REPO_ROOT>`, focused only on `MessagePart -> TextPartDisplay -> Markdown / PacedMarkdown -> DOM`.

Date:
2026-06-27

Summary:
No existing automated unit/component test was found that mounts `MessagePart`, `TextPartDisplay`, `Markdown`, or `PacedMarkdown` into a DOM and asserts rendered markdown nodes. `packages/session-ui/src/components/message-part.test.ts` only tests `readPartText()`. Markdown-related tests in `packages/session-ui/src/components/markdown-*.test.ts` cover stream projection, cache preload, code-state, worker protocol/queue/transport, and inline-code classification, but do not render Solid components or query DOM. `packages/app/test-browser/*` has a happy-dom preload and uses `createRoot` for Solid reactive primitives, but does not render JSX components into DOM. `packages/app/src` tests only import `@opencode-ai/session-ui/context` as types and do not render session-ui components.

Stories and fixtures exist but are not automated DOM assertions. `packages/session-ui/src/components/markdown.stories.tsx` renders `Markdown` through Storybook scaffold with markdown fixture text. `packages/session-ui/src/components/message-part.stories.tsx` exposes a `MessagePart` story via generic scaffold, but without default args visible in the story file. `packages/session-ui/src/components/timeline-playground.stories.tsx` is the richest manual harness: it creates text/reasoning/tool parts and renders `SessionTurn` under `DataProvider` and `FileComponentProvider`. Storybook preview wraps stories in `MetaProvider`, `ThemeProvider`, `DialogProvider`, and `MarkedProvider`, which is the working provider setup for manual rendering.

Existing Playwright e2e files and fixtures can exercise the full app timeline, but the inspected assertions focus on timeline presence/order/scroll/resize/collapse through selectors such as `[data-timeline-part-id]`, not on markdown DOM structure under `[data-component="markdown"]`. `packages/app/e2e/smoke/session-timeline.fixture.ts` provides reusable text/reasoning parts and realistic timeline data, but its text is generated lorem prose rather than targeted markdown expected-output cases.

Related claims:
- CLAIM-017
- CLAIM-019

Limitations:
This is inventory/static evidence, not an executed test run. It is scoped to the inspected checkout and to the `MessagePart -> DOM` segment only. It does not evaluate agents, routing, providers, LiteLLM, Ollama, tools, or subagents.

EVID-017
Type:
test-design

Source:
`packages/session-ui/src/components/message-part-dom.test.tsx` created during iteration.

Date:
2026-06-27

Summary:
Test design evidence for the missing `MessagePart -> TextPartDisplay -> Markdown -> DOM` segment. The test mounts the exported `Part` component from `message-part.tsx` with real `MarkedProvider` and `DataProvider` only (no `I18nProvider` or `DialogProvider` needed), using `solid-js/web` render().

Two test cases:
1. **Completed assistant text part**: `message.time.completed` is set → goes through `TextPartDisplay -> Markdown -> DOM`. Verifies that markdown-derived DOM nodes appear under `[data-component="markdown"]`.
2. **Incomplete (streaming) assistant text part**: `message.time.completed` is absent → goes through `TextPartDisplay -> PacedMarkdown -> Markdown -> DOM`. Same assertions, verifying the streaming wrapper does not block markdown rendering.

Target markdown text:
```text
## Runtime heading

- first item
- second item with `inlineCode`

[OpenCode](https://opencode.ai)
```

DOM assertions:
- `[data-component="text-part"]` exists
- `[data-component="markdown"]` exists
- `h2` textContent === "Runtime heading"
- `li` count === 2
- `code` textContent === "inlineCode"
- `a` href === "https://opencode.ai"

Executed on 2026-06-27: All 2 tests pass (12 expect() calls), after resolving four environment/tooling blockers:
1. Bun not in PATH (resolved by prepending `%APPDATA%\npm`)
2. Vite `?worker&url` import incompatibility (resolved via `mock.module()` in preload)
3. SolidJS server build resolution (resolved via `--conditions=browser`)
4. JSX transpilation to React.createElement (resolved via `globalThis.React = { createElement: h }`)

Full command:
```text
cd packages/session-ui
bun test --conditions=browser --preload ./src/components/happydom.ts src/components/message-part-dom.test.tsx
```

Runtime-confirmed DOM:
- `[data-component="text-part"]` ✅
- `[data-component="markdown"]` ✅
- `h2` text === "Runtime heading" ✅
- `li` count === 2 ✅
- `code` text === "inlineCode" ✅
- `a` href === "https://opencode.ai" ✅

Related claims:
- CLAIM-017
- CLAIM-019

Limitations:
Worker module is mocked; code-block syntax highlighting is not exercised. Test uses `--conditions=browser` and a `React` global shim. Validates inline markdown rendering only.

EVID-018
Type:
test-blocker (resolved)

Source:
Actual execution attempt: `bun test src/components/message-part-dom.test.tsx` after making Bun available via PATH.

Date:
2026-06-27

Summary:
Runtime tooling blocker — resolved. Bun (v1.3.14) initially failed with `SyntaxError: Missing 'default' export in module './markdown-shiki.worker.ts?worker&url'` because `?worker&url` is Vite-specific syntax. The solution was a test preload (`packages/session-ui/src/components/happydom.ts`) that:
1. Calls `mock.module("./markdown-worker", ...)` to provide a mock implementation before any import reaches the real worker module.
2. Provides `globalThis.React = { createElement: h }` from `solid-js/h` so Bun's JSX transpiler produces Solid-compatible calls.

Additional blockers resolved during the same attempt:
- `--conditions=browser` needed to select Solid's browser build over the server build.
- `@happy-dom/global-registrator` dependency resolution via importing `../../../app/happydom` from the preload.

Related claims:
- CLAIM-017

Limitations:
Worker module is fully mocked; code-block syntax highlighting is not exercised. The mock and React shim are test-only and do not affect production code.
