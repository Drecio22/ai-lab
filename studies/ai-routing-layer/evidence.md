# Study-002 Evidence

This study's formal evidence is limited to imported results from Study-001. Architectural analysis and reasoning are documented in `specs.md`.

EVID-001
Type:
source-code / experiment

Source:
OpenCode Study-001, EVID-019, EVID-020, EVID-021

Date:
2026-06-28

Summary:
Study-001 confirmed the static model resolution chain in OpenCode:
- `SessionPrompt.createUserMessage` resolves model as `input.model ?? agent.model ?? currentModel(sessionID)`.
- `currentModel(sessionID)` falls back through: session current model → recent user message model → `provider.defaultModel()` → `cfg.model` → `model.json` → first available provider/model.
- `runLoop` calls `Provider.getModel` to materialize the resolved reference.
- `LLM.stream` executes the already-resolved model through native or AI SDK runtime.
- No dynamic routing or model re-selection occurs after `createUserMessage`.

This establishes the architectural constraint: any routing must intercept **before** or **at** `createUserMessage`.

Related claims:
- CLAIM-001
