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

**Active** — architectural mapping phase.

## Study Contents

- `questions.md`: the foundational question.
- `claims.md`: architectural claims about routing positions.
- `evidence.md`: formal evidence (imported from Study-001).
- `specs.md`: complete architectural map of routing positions with tradeoffs.
- `uncertainties.md`: open questions and unknowns.
- `next-steps.md`: proposal for the next investigation.
