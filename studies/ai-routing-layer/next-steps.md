# Study-002 - Next Investigation Proposal

## Current State

The study has shifted from generic model routing to practical OpenCode-native routing through fixed-model agents/subagents.

Current verdict:

1. **Agent/subagent fixed-model routing** is now the strongest near-term candidate for daily use.
2. **Client-side preprocessing** remains theoretically useful but is less necessary if the user's recurring work maps well to agent lanes.
3. **External HTTP proxy routing** is deprioritized for this use case because it adds protocol complexity without solving the user's main manual-decision burden better than explicit lanes.

Resolved facts:

- OpenCode supports primary agents and subagents officially.
- Each agent/subagent can have a fixed model.
- Source code confirms configured subagent model overrides the parent assistant message model.
- Subagents can be invoked automatically through Task tool descriptions or manually with `@`.
- Child sessions and messages record enough metadata to audit model choice in principle.

## Recommended Next Investigation

### Priority 1: Minimal runtime verification of fixed-model subagents (U-009)

**Why**: Source evidence is strong, but daily use needs an operational check for "which model actually ran".

**Approach**:

- Create one temporary non-definitive test subagent with a distinctive cheap/local model.
- Run a harmless `@test-subagent` prompt.
- Inspect child session `parentID`, `agent`, `model`, task metadata, and user/assistant message model fields.
- Document the simplest verification workflow.

**Expected output**: A short runtime note proving model precedence and showing how to audit model usage.

### Priority 2: Prototype conceptual lane set without committing final config

**Why**: The app de oposiciones has clear recurring lanes: CTE/legal corpus, JSON audits, question linking, Next.js refactors, senior review, validation, commits.

**Approach**:

- Draft agent names, descriptions, model classes, and permissions in a design note or temporary scratch file.
- Do not install final config yet.
- Validate that the lane count stays small enough to reduce, not increase, cognitive load.

**Expected output**: Candidate `opos-*` agent matrix ready for later implementation.

### Priority 3: Automatic subagent selection scenario test (U-010)

**Why**: Manual `@` invocation is enough for control, but automatic routing is the convenience win.

**Approach**:

- Use representative prompts for legal audit, JSON audit, linking, refactor, review, validation.
- Observe whether an orchestrator chooses the intended subagent from descriptions alone.
- Tighten descriptions and `permission.task` boundaries if selection is ambiguous.

**Expected output**: Small routing-quality matrix: prompt -> expected subagent -> actual subagent -> notes.

### Priority 4: Child-session context template (U-011)

**Why**: Subagents are only useful if they receive enough context without excessive prompt boilerplate.

**Approach**:

- Define a minimal delegation template: objective, files, constraints, acceptance criteria, output format, relevant prior findings.
- Test it on one real CTE/legal audit and one real Next.js refactor review.

**Expected output**: Reusable delegation convention for orchestrator/subagent prompts.

## Deferred

- LiteLLM, OpenRouter, and other external routers.
- HTTP proxy protocol tracing unless fixed-model agents prove insufficient.
- V2 `aisdk.language` delegating model prototypes until V2 is active for live prompts.
- Final OpenCode configuration implementation.
- Commits.
