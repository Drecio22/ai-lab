# OpenCode Study

OpenCode is the first case study of AI-LAB.

## Objective

Understand OpenCode as an AI engineering system: its architecture, runtime behavior, agent model, model selection, providers, delegation mechanisms, and extension points.

## Initial Scope

This study will focus on evidence-based investigation and practical experiments. It will not produce final documentation until conclusions are sufficiently supported.

## Initial Topics

- Architecture.
- Prompt-to-response flow.
- Agents.
- Subagents.
- Delegation.
- Routing.
- Model selection.
- Providers.
- LiteLLM.
- Ollama.
- Extension points.

## Current Status

Status: active.

Phase 1 completed (OpenCode Core Architecture investigation).

This study is no longer just initialized.

- It contains recorded questions, claims, evidence, experiments, and run records.
- It includes both static and runtime investigation artifacts.
- Some claims are now confirmed for the current study scope.
- The study remains provisional and evidence-driven; it is not final documentation.

## Executive Summary (Phase 1)

### Questions answered

- **Q-027**: The OpenCode monorepo at commit `ae53163` had no existing automated DOM rendering test for `MessagePart -> TextPartDisplay -> Markdown -> DOM`. **Answered**: confirmed by test inventory (EVID-016).
- **Q-024, Q-025, Q-001 (partial)**: The prompt-to-screen path was statically traced (EVID-013, EVID-014) and server-side runtime-confirmed via event capture (EVID-015, EXP-006). The SSE delivery and client store processing were validated by existing tests. The final UI rendering step was validated via a minimal happy-dom test (EXP-007, EVID-017).
- **Q-028 (partial)**: `SessionPrompt.createUserMessage` is the component that decides the effective model for a normal prompt, applying `input.model ?? agent.model ?? currentModel(sessionID)`. The `input.model` branch was runtime-confirmed with full propagation (EXP-008-run-001). The `currentModel` fallback was runtime-confirmed for selection (EXP-008-run-003). The `agent.model` branch is confirmed by static code only (EXP-008-run-002 blocked).

### Key findings

1. The normal app prompt path (`promptAsync -> SessionPrompt.prompt -> LLM stream -> events -> SSE -> client store -> MessagePart -> Markdown -> DOM`) is fully traced statically and validated per segment at runtime.
2. Model resolution follows a strict precedence: `input.model ?? agent.model ?? currentModel(sessionID)`. Once chosen by `createUserMessage`, the same model propagates unchanged through `Provider.getModel`, `SessionProcessor.process`, and `LLM.stream` to the final runtime request.
3. A minimal happy-dom test (2 cases, 12 assertions) is sufficient to validate the UI rendering segment without Playwright or a full e2e setup.
4. The runtime uses v1 events (`message.part.delta`, `message.updated`, `message.part.updated`) for streaming, not the v2 `session.next.*` events referenced in earlier analysis.

### Known limitations

- The `agent.model` branch could not be dynamically validated due to test harness limitations (EXP-008-run-002 blocked).
- The `currentModel` fallback was validated for selection only; propagation is inferred from the `input.model` branch.
- No single end-to-end test covers the full prompt-to-screen chain in one execution — only segment-level validation exists.
- The native LLM runtime path (`experimentalNativeLlm`) was not tested.
- Subagent model selection (TaskTool path) was inspected statically only (CLAIM-022).

### Open questions for future phases

- Q-007: Can an agent define its own model? (Requires investigation of agent configuration loading)
- Q-008: What precedence exists between global and agent-specific model config?
- Q-014: Where in the execution flow is the final model selected? (Partially answered)
- Q-009, Q-011, Q-012: Subagent context and tool inheritance
- Q-013, Q-020: Internal routing and extension points
- Q-017, Q-018: LiteLLM support
- Q-019: Ollama support

## Study Contents

- `questions.md`: research questions tracked for the study.
- `claims.md`: verifiable statements and current states.
- `evidence.md`: source-code, documentation, and runtime evidence.
- `experiments.md`: planned and completed experiments.
- `runs/`: concrete experiment executions.
- `specs.md`: current technical understanding, separated into facts, hypotheses, and open questions.
- `architecture-decisions.md`: study ADRs when architectural decisions are justified.

## Scope Note

Conclusions in this study are scoped to the investigated versions, configurations, and environments recorded in the evidence and run artifacts.
