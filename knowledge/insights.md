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
