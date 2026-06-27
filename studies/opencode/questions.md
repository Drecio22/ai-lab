# OpenCode Questions

This file tracks research questions for the OpenCode study.

Format:

```text
Q-000
Question:
...

State:
open

Related to:
...
```

Q-001
Question:
What exact path does a prompt follow from user submission until the response appears on screen?

State:
open

Related to:
architecture, prompt-flow

Q-002
Question:
How is a user prompt converted into internal messages?

State:
open

Related to:
prompt-flow, messages

Q-003
Question:
How does OpenCode load and resolve configuration?

State:
open

Related to:
architecture, configuration

Q-004
Question:
What is an agent in OpenCode's implementation?

State:
open

Related to:
agents

Q-005
Question:
How are built-in agents defined and loaded?

State:
open

Related to:
agents, configuration

Q-006
Question:
Can user-defined agents override or extend built-in agents?

State:
open

Related to:
agents, configuration

Q-007
Question:
Can an agent define its own model?

State:
open

Related to:
agents, model-selection

Q-008
Question:
What precedence exists between global model configuration and agent-specific model configuration?

State:
open

Related to:
model-selection, configuration

Q-009
Question:
What is a subagent in OpenCode: a distinct concept or an invocation pattern?

State:
open

Related to:
subagents, delegation

Q-010
Question:
How does delegation between agents actually occur?

State:
open

Related to:
delegation, subagents

Q-011
Question:
Does a subagent inherit context from the parent agent?

State:
open

Related to:
subagents, context

Q-012
Question:
Does a subagent inherit tools and permissions from the parent agent?

State:
open

Related to:
subagents, tools, permissions

Q-013
Question:
Does OpenCode have an internal router, or only static configuration resolution?

State:
open

Related to:
routing

Q-014
Question:
Where in the execution flow is the final model selected?

State:
open

Related to:
model-selection, providers

Q-015
Question:
How does OpenCode represent providers internally?

State:
open

Related to:
providers, architecture

Q-016
Question:
How does OpenCode determine whether a model supports tools, streaming, or other capabilities?

State:
open

Related to:
models, providers, tools

Q-017
Question:
Can OpenCode use LiteLLM as an OpenAI-compatible endpoint?

State:
open

Related to:
LiteLLM, providers

Q-018
Question:
Does LiteLLM preserve tool calls correctly when used with OpenCode?

State:
open

Related to:
LiteLLM, tools

Q-019
Question:
Can OpenCode use Ollama directly or through an OpenAI-compatible API?

State:
open

Related to:
Ollama, providers

Q-020
Question:
What extension points could support an intelligent router?

State:
open

Related to:
extension-points, routing

Q-021
Question:
How are tool calls represented, executed, and persisted?

State:
open

Related to:
tools, prompt-flow

Q-022
Question:
How are sessions persisted across interactions?

State:
open

Related to:
sessions, persistence

Q-023
Question:
Which prompt submission path is authoritative for the current user-facing OpenCode app: the v1 `promptAsync` flow, the v2 `session.prompt` flow, or both in different clients?

State:
open

Related to:
prompt-flow, architecture, clients

Q-024
Question:
How do streamed model events become visible UI updates in the OpenCode app?

State:
open

Related to:
prompt-flow, streaming, UI

Q-025
Question:
Which event stream or synchronization mechanism does the UI subscribe to after submitting a prompt?

State:
open

Related to:
prompt-flow, events, UI

Q-026
Question:
Does the static app prompt-to-screen call trace observed at commit `ae53163` execute unchanged in a real installed OpenCode runtime?

State:
open

Related to:
prompt-flow, static-trace, runtime-validation

Q-027
Question:
Does the OpenCode monorepo already contain an automated test, story, fixture, or harness that validates runtime DOM rendering for `MessagePart -> TextPartDisplay -> Markdown / PacedMarkdown -> DOM`?

State:
answered

Related to:
runtime-ui-rendering, MessagePart, Markdown, DOM
