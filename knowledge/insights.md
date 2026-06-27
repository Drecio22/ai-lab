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
