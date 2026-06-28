# Insights

This file records lightweight cross-cutting observations that may later become claims, patterns, ADRs, or recommendations.

Insights are provisional. They are not confirmed facts unless linked to evidence.

## INSIGHT-001

Observation:
A lightweight lab structure is more valuable at the beginning than a complete knowledge-management system.

State:
provisional

Origin:
AI-LAB MVP design

## INSIGHT-002

Observation:
Separating observed truth from recommended truth helps prevent implementation details from becoming premature advice.

State:
provisional

Origin:
AI-LAB MVP design

## INSIGHT-003

Observation:
When evidence is insufficient, the next useful action is usually a minimal experiment rather than more documentation structure.

State:
provisional

Origin:
AI-LAB MVP design

## INSIGHT-004

Observation:
In a static model-resolution chain like OpenCode's (`input.model ?? agent.model ?? currentModel`), the best place for dynamic routing is often **before** the resolution, not at a deeper layer. This is because model identity is an input decision, not an execution detail — and reversing a committed model choice requires either forking or protocol-level interception.

State:
provisional

Origin:
Study-002: AI Routing Layer

## INSIGHT-005

Observation:
The invasiveness of a routing layer is not monotonic with respect to its position in the execution flow. A client-side preprocessor (before resolution) is less invasive than a provider-level interceptor (after resolution), even though it is "further" from the execution. The key factor is whether the interception uses a designed input parameter or requires modifying internal wiring.

State:
provisional

Origin:
Study-002: AI Routing Layer

## INSIGHT-006

Observation:
The presence of a rich plugin/hook system does not imply the presence of a routing hook. A tool can expose dozens of extension points (tools, events, params, headers, compaction, catalog) and still expose the model identity strictly as read-only input at every downstream hook. Whether a specific decision variable is *writable* at an extension point matters more than how many extension points exist. When evaluating extensibility for routing, the diagnostic question is: "is there any hook where the model is in the writable output?" — not "are there hooks?"

State:
provisional

Origin:
Study-002: AI Routing Layer (Q-002; OpenCode V1 exposes many hooks but none make the primary model writable — CLAIM-012)

## INSIGHT-007

Observation:
During a generational rewrite (V1 → V2), extension capability can temporarily live in the *next* system rather than the *current* one. A routing position may be "viable without fork" against the target architecture while being "not viable without core modification" against the running architecture. Treating either answer as final misrepresents a moving target; the honest verdict is conditional ("viable on V2 once active") and the decisive next step is a runtime check of which generation actually executes.

State:
provisional

Origin:
Study-002: AI Routing Layer (Position 4 split verdict; CLAIM-011, CLAIM-015, U-006)

## INSIGHT-008

Observation:
A product can have a rich internal inventory service without exposing a reusable public catalog boundary. For routing systems, "the catalog exists" and "a Decision Engine can safely consume the catalog" are separate questions. The first is about internal data construction; the second is about API stability, process boundary, and whether the exposed shape matches the active execution path.

State:
provisional

Origin:
Study-002: AI Routing Layer (Q-003; OpenCode has V1 `Provider.Service.list()` and V2 `Catalog.Service.model.available()`, but documented V1 plugins do not directly expose the complete active catalog; CLAIM-018, CLAIM-020, CLAIM-022)

## INSIGHT-009

Observation:
Adaptive model selection spans different information horizons: some inputs are known before execution, some are only estimates, some signals appear during execution, and some outcomes are only available after completion. Treating these horizons as one flat input set can hide the boundary between deterministic eligibility checks and probabilistic outcome prediction.

State:
provisional

Origin:
Study-003: Adaptive Model Selection

## INSIGHT-010

Observation:
In adaptive model selection, "objective variable" does not always mean "directly measurable value". Some decision-relevant variables are observed facts, some are declared constraints, some are catalog metadata, and some are inferred estimates that require confidence tracking. A useful decision model should preserve that distinction instead of flattening all variables into facts.

State:
provisional

Origin:
Study-003: Adaptive Model Selection (Q-015 variable inventory)

## INSIGHT-011

Observation:
For adaptive model selection, observability is a transformation layer rather than a passive data-collection step. Raw variables become useful only when converted into signals that preserve provenance, confidence, freshness, acquisition cost, and uncertainty. This keeps direct facts, user declarations, catalog metadata, inferred estimates, and unknowable outcomes from being treated as the same kind of input.

State:
provisional

Origin:
Study-003: Adaptive Model Selection (Q-016 observability model)

## INSIGHT-012

Observation:
In adaptive model selection, the step from signals to decision may need an explicit representation of the current problem state. A `Situation` concept is useful if it captures relationships among signals — consistency, completeness, contradictions, missing information, staleness, and temporal validity — that would otherwise be hidden inside each decision mechanism.

State:
provisional

Origin:
Study-003: Adaptive Model Selection (Q-017 Situation layer hypothesis)

## INSIGHT-013

Observation:
A candidate architectural concept should survive a redundancy test before becoming fundamental. In Study-003, `Situation` currently survives only as an optional abstraction for relationships among Signals; it is not yet justified as a mandatory layer because sufficiently rich Signals may preserve the same information for simple or single-engine decisions.

State:
provisional

Origin:
Study-003: Adaptive Model Selection (Q-018 Situation refutation attempt)

## INSIGHT-014

Observation:
Adaptive model selection appears less like a new standalone theory and more like a synthesis problem across mature fields: algorithm selection, meta-reasoning, information fusion, context modeling, MCDM, utility theory, and resource-aware AI. The originality, if any, is likely in integrating these under LLM-specific prompt/context/catalog/privacy constraints, not in the basic decision concepts.

State:
provisional

Origin:
Study-003: Adaptive Model Selection (Q-019 bibliographic map)

## INSIGHT-015

Observation:
Algorithm Selection is the strongest structural ancestor of Adaptive Model Selection, but only after extension. The direct mapping is problem instance -> prompt/context, instance features -> Signals, algorithms -> candidate models, and performance -> quality/cost/latency/reliability. The extension pressure comes from LLM-specific dynamics: catalog volatility, privacy eligibility, multi-turn prompts, tool-mediated context changes, and subjective output quality.

State:
provisional

Origin:
Study-003: Adaptive Model Selection (Q-020 Algorithm Selection foundation analysis)

## INSIGHT-016

Observation:
The minimum LLM-specific gap over classical Algorithm Selection is not selecting among solvers; it is establishing which solvers are currently eligible and observable under dynamic catalog, cost, quota, latency, privacy, policy, context, and environment constraints. In Adaptive Model Selection, eligibility and observability are part of the core problem, not preprocessing details.

State:
provisional

Origin:
Study-003: Adaptive Model Selection (Q-021 minimum extensions over Algorithm Selection)

## INSIGHT-017

Observation:
In adaptive model selection, the objective function cannot be defined until constraints, objectives, preferences, and policies are separated. The phrase "best model" is misleading unless it means "best eligible candidate under uncertainty for this task and context," not globally strongest, cheapest, fastest, or most preferred in isolation.

State:
provisional

Origin:
Study-003: Adaptive Model Selection (Q-022 objective function model)

## INSIGHT-018

Observation:
Adaptive model selection is better understood as a decision lifecycle than as a single choice point. The system has different knowledge before execution, during execution, and after execution; feedback only becomes meaningful when tied back to the lifecycle phase that produced the relevant uncertainty or failure.

State:
provisional

Origin:
Study-003: Adaptive Model Selection (Q-023 decision lifecycle model)

## INSIGHT-019

Observation:
Preferences should be treated as scoped, authoritative, and uncertain tradeoff inputs rather than as a flat user profile. The key distinction is not only explicit versus inferred, but whether a preference is permanent, temporary, task-level, conversation-level, organizational, inherited, stale, conflicting, or low-confidence.

State:
provisional

Origin:
Study-003: Adaptive Model Selection (Q-024 preference model)

## INSIGHT-020

Observation:
For daily LLM-assisted software work, the most useful "routing" layer may be a small set of named work lanes rather than an opaque per-prompt decision engine. If tasks recur as recognizable categories, a fixed-model agent/subagent can bundle model choice, prompt, permissions, context expectations, and auditability into one selectable unit. This reduces manual model selection without introducing the protocol and observability costs of an external router.

State:
provisional

Origin:
Study-002: AI Routing Layer (Q-004; OpenCode agents/subagents with fixed models as practical routing)
