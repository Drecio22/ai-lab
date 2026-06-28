# Study-002 — AI Routing Layer

## Objective

Answer a single architectural question:

**Where should an intelligent routing layer be placed to control which LLM model OpenCode uses, without modifying OpenCode?**

## Context

Study-001 (OpenCode) confirmed that OpenCode resolves the effective model through a static precedence chain (`input.model ?? agent.model ?? currentModel(sessionID)`) in `SessionPrompt.createUserMessage`, *before* LLM execution begins (CLAIM-020, CLAIM-021, CLAIM-023, EVID-019).

The hypothesis is that intelligent routing must occur **before** that resolution.

## Scope

- Architectural analysis only.
- No investigation of concrete products (LiteLLM, OpenRouter, etc.).
- No experiments.
- No implementation designs.
- Stops when a sufficiently complete architectural map exists.

## Status

**Active** — extension-points investigation (Q-002) and runtime confirmation (U-006) completed; architectural mapping continues.

## Latest results (runtime confirmation iteration)

Q-002 ("Does OpenCode have real extension points that allow inserting a routing layer without modifying core?") is answered by EVID-002/EVID-003 with runtime confirmation from EVID-004:

- Formal plugin + hook systems exist (V1 active/documented; V2 Effect-based, migration target).
- **U-006 resolved**: Runtime tracer (EVID-004) confirmed the V1 path is the exclusive execution path for live prompts. V2 `aisdk.sdk`/`aisdk.language` hooks do NOT fire during prompts.
- No V1 hook can redirect the primary model per-prompt (CLAIM-012 ⇒ confirmed). Position 4 is **not** viable on the active V1 path without core modification.
- Position 4 is **conditionally viable** on the V2 path via `aisdk.language` (CLAIM-015 ⇒ supported; standalone dispatch test confirmed interception works). Blocked on V2 becoming the active execution path.
- Extension surfaces classified into three tiers (officially supported / public-but-undocumented / internal).

### Remaining open questions

- U-007: Does a delegating `LanguageModelV3` preserve streaming, tool-call, and usage semantics?
- U-003/U-005: Protocol transparency for Position 5; cross-client portability for Position 1.

## Study Contents

- `questions.md`: the foundational question.
- `claims.md`: architectural claims about routing positions.
- `evidence.md`: formal evidence (imported from Study-001).
- `specs.md`: complete architectural map of routing positions with tradeoffs.
- `uncertainties.md`: open questions and unknowns.
- `next-steps.md`: proposal for the next investigation.
