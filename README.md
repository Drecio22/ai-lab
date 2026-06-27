# AI-LAB

AI-LAB is a lightweight engineering laboratory for researching AI architectures applied to software development.

The lab studies how tools, models, agents, subagents, providers, routers, gateways, and runtimes behave in real development workflows. Its goal is not to collect documentation, but to produce traceable technical knowledge through evidence and experiments.

## Project Status

AI-LAB is an experimental MVP.

- The repository is intentionally lightweight.
- The methodology is active and usable.
- The first study (`studies/opencode/`) contains real evidence, runs, and confirmed claims.
- The structure is expected to evolve as more studies are added.

This is not a polished framework or benchmark suite. It is a working research repository.

## Mission

Understand and evaluate architectures for working with multiple AI models, agents, tools, routers, and providers in software engineering environments.

## Repository Map

- `methodology/`: lab rules, workflow, glossary, and research protocol.
- `templates/`: minimal templates for claims, experiments, runs, and ADRs.
- `studies/`: concrete case studies, each with its own evidence, claims, questions, specs, and runs.
- `knowledge/`: cross-study insights and patterns when they become justified.

## How To Navigate A Study

Each study is self-contained under `studies/<name>/`.

Typical reading order:

1. `README.md`: scope and current status of the study.
2. `questions.md`: what the study is trying to answer.
3. `claims.md`: verifiable statements and their current state.
4. `evidence.md`: supporting source-code, docs, and runtime evidence.
5. `experiments.md`: planned or completed experiments.
6. `runs/`: concrete executions of experiments.
7. `specs.md`: provisional technical understanding.
8. `architecture-decisions.md`: ADRs only when needed.

## Core Principles

- Evidence before opinion.
- Experiments before conclusions.
- Reasonable reproducibility before perfection.
- Observed facts must stay separate from recommendations.
- Uncertainty must be explicit.
- Methodology must stay lightweight until practice requires more structure.

## First Case Study

The first study is OpenCode. OpenCode is only the first case study, not the center of the lab.

## MVP Constraints

This repository intentionally avoids schemas, automation, dashboards, CI, formal ontology, a real knowledge graph, and deep folder structures.

The goal of the MVP is to prove that a small set of disciplined artifacts is enough to produce useful, traceable technical knowledge.

## Anti-Overengineering Rule

Before adding any new process, folder, artifact, or tool, answer:

- What real problem does it solve?
- How often have we felt the lack of it?
- Can the current structure handle the need?
- Does it add more value than maintenance cost?
