# Research Protocol

This protocol defines the standard operating procedure for AI-LAB studies.

It is practical by design. Use the smallest process that can answer the question reliably.

## Core Rule

The methodology must fit the size of the question.

A simple question may only need one evidence entry and one claim update. A complex architectural investigation may need questions, claims, evidence, experiments, runs, specs, ADRs, and eventually patterns.

The goal is useful knowledge, not completed paperwork.

## Standard Flow

```text
Question
↓
Claim / Hypothesis
↓
Evidence / Experiment
↓
Run if needed
↓
Claim update
↓
Spec / ADR / Pattern only if justified
```

## 1. Starting An Investigation

Start an investigation when there is a concrete technical uncertainty.

Good starting points:

- A behavior is unclear.
- Documentation and observed behavior may differ.
- A design decision depends on tool behavior.
- A capability needs validation.
- A limitation needs confirmation.
- A comparison between tools, models, or providers is needed.

Do not start by writing final documentation. Start by writing a question.

## 2. Creating A Question

A question belongs in the study's `questions.md` when it identifies something the lab needs to know.

Create a question when:

- The answer is not already known.
- The answer may affect a claim, spec, experiment, or architecture decision.
- The question can be investigated through evidence or experiment.

Do not create questions for vague curiosity. Rewrite them until they are technically answerable.

## 3. Turning A Question Into A Claim

A question becomes a claim when it can be expressed as a verifiable statement.

Example:

```text
Question:
Can an agent define its own model?

Claim:
OpenCode allows assigning a specific model to an agent.
```

Create a claim when:

- The statement can be supported or refuted.
- The result matters for the study.
- The statement may later appear in a spec or decision.

Do not create claims for every tiny observation. Small facts can stay inside evidence notes unless they affect a broader conclusion.

## 4. Registering Evidence

Evidence is enough when it directly answers the claim with acceptable confidence for the size of the question.

Evidence may come from:

- Source code.
- Official documentation.
- Official tests.
- Releases.
- Pull requests.
- Issues.
- Discussions.
- Community examples.
- Experiments.

Prefer stronger evidence when the claim is important.

For low-risk claims, one clear source may be enough.

For architectural claims, prefer source code, tests, or experiment results.

For recommendations, evidence alone is rarely enough; consider experiments or comparison.

## 5. Designing An Experiment

Design an experiment when:

- Evidence is missing.
- Evidence is contradictory.
- Documentation is unclear or too high-level.
- Source code suggests behavior that needs runtime confirmation.
- The claim depends on configuration, model, provider, or environment.
- The answer affects architecture, routing, delegation, model selection, or tool behavior.

The experiment should be minimal.

It must define:

- Question.
- Hypothesis.
- System and version.
- Configuration.
- Procedure.
- Expected result.
- Related claims.

Avoid experiments that try to answer too many things at once.

## 6. Creating A Run

A run is one execution of an experiment.

One run may be sufficient when:

- The behavior is deterministic.
- The question is narrow.
- The result is directly observable.
- The claim is low risk.
- The run confirms source code or official documentation.

More than one run is needed when:

- Results vary.
- Models or providers may behave differently.
- The claim affects a recommendation.
- The experiment compares configurations.
- The result depends on timing, cost, context, delegation, or tool use.
- The first run produces an unexpected result.

Every run should record minimum observability: system, version, environment, configuration, model, provider, task, commands, logs, result, metrics when available, and limitations.

## 7. Updating A Claim

Update a claim whenever new evidence or a run changes its status, confidence, or limitations.

Allowed MVP states:

```text
unknown
hypothesis
supported
confirmed
refuted
contradictory
deprecated
```

Use states conservatively:

- `unknown`: not enough evidence.
- `hypothesis`: plausible, but not yet supported.
- `supported`: evidence points in favor, but validation is incomplete.
- `confirmed`: strong enough evidence for the current scope.
- `refuted`: evidence shows the claim is false.
- `contradictory`: credible evidence conflicts.
- `deprecated`: no longer relevant or superseded.

Do not mark a claim as confirmed just because documentation says so. Important claims require stronger evidence.

## 8. Moving A Claim Into A Spec

A claim deserves to enter a spec when:

- It explains meaningful behavior of the system.
- It has at least supported status.
- Its limitations are understood.
- It helps future readers understand architecture, flow, agents, routing, providers, or extension points.

Specs may include:

- Confirmed facts.
- Hypotheses.
- Inferences.
- Open questions.

But they must label each clearly.

Do not use specs as dumping grounds for raw notes.

## 9. Creating A Pattern

Create a pattern only when a learning appears transferable beyond one tool or one isolated case.

A pattern should usually require:

- More than one supporting claim, or
- One strong claim plus a clear reason it generalizes, or
- Repeated observation across tools, models, providers, or experiments.

Early patterns must be marked as candidates.

Do not create patterns from attractive ideas. Patterns must be earned.

## 10. Registering An ADR

Create an ADR only for decisions with meaningful architectural consequences.

Use an ADR when deciding:

- A major direction for the lab.
- A study architecture.
- A recommendation that affects future work.
- A routing, provider, benchmark, or experimentation strategy.
- A tradeoff where alternatives matter.

Do not create ADRs for routine notes, small edits, or temporary choices.

An ADR should include context, decision, alternatives, consequences, and related evidence or claims when available.

## 11. Handling Contradictions

When evidence conflicts:

1. Mark related claims as `contradictory` if the conflict is material.
2. Record the conflicting evidence.
3. Check whether the difference depends on version, configuration, provider, or model.
4. Prefer source code and reproducible experiments over undocumented assumptions.
5. Design a minimal experiment if the contradiction matters.

Do not hide contradictions to make specs look cleaner.

## 12. Handling Unknowns

Unknown is an explicit state.

When something remains unknown, record:

- What is unknown.
- Why current evidence is insufficient.
- What evidence or experiment could resolve it.
- Whether the unknown blocks any decision.

Unknowns are not failures. They are future work made visible.

## 13. Scaling The Process

Use the light path for simple questions:

```text
Question → Evidence → Claim update
```

Use the full path for complex questions:

```text
Question → Claim → Evidence → Experiment → Runs → Claim update → Spec → ADR / Pattern if justified
```

If the process feels heavier than the question, simplify the process.

If the conclusion would influence architecture or recommendations, strengthen the evidence.

## Final Rule

AI-LAB exists to produce useful, traceable knowledge about AI engineering architectures.

Do not optimize for document completeness. Optimize for validated understanding.
