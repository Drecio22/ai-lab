# Study-002 — Specs: AI Routing Architecture

## Reference Flow

From Study-001, OpenCode's model resolution and execution flow:

```
User prompt
  │
  ▼
SessionPrompt.prompt
  │
  ├── createUserMessage        ◄── MODEL RESOLUTION
  │     input.model ?? agent.model ?? currentModel(sessionID)
  │     stores {providerID, modelID} on user message
  │
  ├── runLoop
  │     ├── Provider.getModel  ◄── MATERIALIZATION
  │     ├── create assistant message with resolved model
  │     └── SessionProcessor.process
  │           └── LLM.stream
  │                 ├── Provider.getLanguage  ◄── LANGUAGE LOADING
  │                 ├── (native runtime) or (AI SDK streamText)
  │                 └── HTTP request to provider API  ◄── NETWORK
  │
  └── events → SSE → client store → UI
```

The resolution chain decomposes into distinct architectural layers, each a potential interception point:

| Layer | Component | What it does | Can route? |
|---|---|---|---|
| Configuration | `Config`, `model.json`, agent configs | Static model/provider definitions | Static only |
| Reference resolution | `createUserMessage` | `input.model ?? agent.model ?? currentModel(sessionID)` | **Before commitment** |
| Model materialization | `Provider.getModel` | Resolves `{providerID, modelID}` → `Provider.Model` | After commitment |
| Language model loading | `Provider.getLanguage` | `Provider.Model` → AI SDK `LanguageModelV3` | After commitment |
| Runtime adapter | `LLM.stream` | Selects native vs AI SDK path | After commitment |
| Network | HTTP request to provider API | Actual API call | External proxy only |

Additionally, Study-001 identified that the `TaskTool` subagent delegation path (CLAIM-022, EVID-019) selects `next.model` for child sessions. This is an existing form of model routing, but it is limited to subagent delegation, not primary prompt routing.

```
User prompt
  │
  ▼
SessionPrompt.prompt
  │
  ├── createUserMessage        ◄── MODEL RESOLUTION
  │     input.model ?? agent.model ?? currentModel(sessionID)
  │     stores {providerID, modelID} on user message
  │
  ├── runLoop
  │     ├── Provider.getModel  ◄── MATERIALIZATION
  │     ├── create assistant message with resolved model
  │     └── SessionProcessor.process
  │           └── LLM.stream
  │                 ├── Provider.getLanguage  ◄── LANGUAGE LOADING
  │                 ├── (native runtime) or (AI SDK streamText)
  │                 └── HTTP request to provider API  ◄── NETWORK
  │
  └── events → SSE → client store → UI
```

**Key architectural constraint**: Model is resolved once in `createUserMessage` and never re-evaluated. The routing layer must intervene **at or before** that point to dynamically control the model.

---

## Extension Points (EVID-002, EVID-003)

This section records the real extension surfaces found in OpenCode at commit `ae53163`. They determine which routing positions are reachable without a fork. Observed facts, inferences, and hypotheses are labeled.

### Two coexisting plugin systems

| System | Package | Status | Active in session execution? |
|---|---|---|---|
| V1 plugin | `@opencode-ai/plugin` (`packages/plugin/src/index.ts`) | Officially documented, published | **Yes** (hooks dispatched via `Plugin.Service.trigger`) |
| V2 plugin | `@opencode-ai/plugin/v2/effect` (`packages/plugin/src/v2/effect/*`) | "Implementation plan, not documentation for the current API" (`PLAN.md:5`) | Host wired (`packages/core/src/plugin/host.ts`, `plugin.ts:140-153`), feeds catalog/agent/skill, but its runtime hooks are **not** on the live LLM path |
| `@opencode-ai/llm` | `packages/llm` | Schema-first LLM core; `DESIGN.md` proposes replacing it with `@opencode-ai/ai` | Not on the active path analyzed in Study-001 |

[observed: EVID-002 §A, §D, §E]

### V1 hooks vs. routing intent

| Hook | Fires at (V1) | Can it change the primary model? | Notes |
|---|---|---|---|
| `chat.message` | new user message | **No** — `model` is read-only input; output mutates message/parts | `packages/plugin/src/index.ts:234-243` |
| `chat.params` | LLM request preparation (`llm/request.ts:114`) | **No** — `model` read-only input; output is generation params only | Decisive negative evidence for routing via this hook |
| `chat.headers` | LLM request preparation (`llm/request.ts:134`) | No | Headers only |
| `experimental.chat.system.transform` | LLM request preparation (`llm/request.ts:69`) | No | System prompt only |
| `experimental.chat.messages.transform` | prompt loop (`prompt.ts:1254`) | No | Message list only |
| `experimental.provider.small_model` | `Provider.getSmallModel` (`provider.ts:1858`) | **Only the small model** | Not the primary prompt model |
| `provider` (ProviderHook `.models`) | provider assembly (`provider.ts:1367`) | No (catalog-level) | Defines which models a provider offers; static per load |
| `tool.execute.before/after`, `tool.definition` | tool dispatch | No | Tool interception |
| `experimental.session.compacting`, `experimental.compaction.autocontinue`, `experimental.text.complete` | compaction / completion | No | Auxiliary flows |

[observed: EVID-002 §B, §C]

**Inference**: No V1 hook exposes the primary model as a writable value before or at commitment. The model is resolved in `createUserMessage` (Study-001, CLAIM-020) and is thereafter read-only input to every downstream hook.

### Provider registration and the model materialization layer

Providers are assembled in `packages/opencode/src/provider/provider.ts` from four sources [observed: EVID-002 §C]:

1. **Config** (`cfg.provider`, opencode.json) — static definitions, `source: "config"`. A custom OpenAI-compatible endpoint is configurable without code (`provider.ts:1395-1449`, default npm `@ai-sdk/openai-compatible`). [CLAIM-013]
2. **Plugin `ProviderHook.models`** — returns a model catalog per provider (`provider.ts:1367-1392`). Catalog-level only; not per-prompt.
3. **`models.dev` catalog** — built-in data, `source: "custom"` (`provider.ts:1239`).
4. **V2 `catalog.transform`** — can `provider.update/remove`, `model.update/remove`, `model.default.set` (`packages/plugin/src/v2/effect/catalog.ts`). V2 only; domain-rebuild, not per-prompt.

The dynamic point — `Provider.getLanguage` (`provider.ts:1801`) turning a `Model` into an AI SDK `LanguageModelV3` — uses the internal `resolveSDK` (`provider.ts:1639`) and `s.modelLoaders[model.providerID]` (`provider.ts:1811`). The `modelLoaders` map is populated **only** from the closed built-in `custom(dep)` registry (`provider.ts:168`, applied at `1535-1551`). The V1 `ProviderHook` exposes `models`, not `getModel`, so a V1 plugin cannot install a routing model loader. [CLAIM-012]

`resolveSDK` also wraps the transport: a provider/model `fetch` option is wrapped in-place (`provider.ts:1703-1734`). This is an HTTP-level interception point, but because `fetch` is a function it requires code and behaves like Position 5 running in-process, not like a provider-abstraction interceptor.

### V2 interception points (emerging)

The V2 host exposes runtime hooks that ARE designed for provider-abstraction interception [observed: EVID-002 §D]:

- `aisdk.sdk` — intercepts AI SDK creation; can set `event.sdk`. Used by `DynamicProviderPlugin` to load any `@ai-sdk/*` package on demand (`packages/core/src/plugin/provider/dynamic.ts`).
- `aisdk.language` — intercepts `LanguageModelV3` creation; can set `event.language`. **This is the V2 equivalent of `Provider.getLanguage` and the natural home for Position 4.** [CLAIM-015, hypothesis]
- `catalog.transform` — provider/model/default manipulation at the catalog layer.

**Inference**: If V2 were the active execution path, Position 4 would be viable without a fork by returning a delegating `LanguageModelV3` from `aisdk.language`. Today the live path is V1 and does not reach these hooks (CLAIM-011).

### Effect-TS composition

OpenCode is built on Effect-TS (`Context.Service`, `Layer`, `Layer.provideMerge`, `Effect.fn`). The V2 host and all V2 domains are composed this way (`packages/core/src/plugin.ts:145-153`). This is a real dependency-injection/composition system, but it is an internal/V2 concern: the V1 active session path exposes extension through the `Hooks` trigger pattern, not through user-facing Effect layer replacement. [CLAIM-014]

### Extension tiers

| Tier | Examples | Routing utility |
|---|---|---|
| Officially supported/documented | V1 plugins (npm/local), documented hooks (`event`, `tool.execute.*`, `shell.env`, `experimental.session.compacting`, custom `tool`), config providers, agents, MCP | Static catalog (Position 2/3); tool/event interception (Position 7) |
| Public type, undocumented | `chat.params`, `chat.headers`, `provider` ProviderHook, `experimental.provider.small_model`, `experimental.chat.*.transform`, `tool.definition`, `permission.ask`, `command.execute.before` | Auxiliary; small-model only; catalog-level |
| Internal / not public API | V2 host + hooks (`aisdk.*`, `catalog.transform`), `modelLoaders`/`custom()`, `resolveSDK`, `BUNDLED_PROVIDERS` | Position 4 lives here; requires V2 activation or core mod |

[observed: EVID-002, EVID-003; CLAIM-016]

---

## Model Catalog Construction (EVID-005, EVID-006)

This section answers Q-003: how OpenCode builds the effective LLM model catalog available to the user, and whether a future Decision Engine can reuse it.

### Call Diagram

```text
ModelsDev.Service.get()
  ├─ load OPENCODE_MODELS_PATH/cache file
  ├─ else load embedded OPENCODE_MODELS_DEV snapshot
  └─ else fetch https://models.dev/api.json
       ↓
Provider.Service InstanceState initialization (V1 active path)
  ├─ catalog = mapValues(modelsDev, fromModelsDevProvider)
  ├─ database = public copy of catalog
  ├─ load plugins before config hook-sensitive provider read
  ├─ apply V1 plugin provider.models hooks to database provider models
  ├─ extend database from cfg.provider custom providers/models
  ├─ load env-backed providers into active providers
  ├─ load auth.json API-backed providers into active providers
  ├─ run plugin auth loaders
  ├─ run built-in custom(provider) customizers/autoload/model loaders
  ├─ re-apply config provider metadata/options
  ├─ append discovered provider models where a discovery loader exists
  └─ final filter pass: enabled/disabled providers, invalid aliases,
     alpha/deprecated status, whitelist/blacklist, variants, empty providers
       ↓
Provider.Service.list() -> Record<ProviderID, Provider.Info>
Provider.Service.getModel(providerID, modelID) -> Provider.Model
Provider.Service.defaultModel() -> { providerID, modelID }
       ↓
Consumers: /models picker, ACP directory, config option UI, SessionPrompt,
agents, compaction, tools, sharing, provider HTTP API
```

V2 has a parallel structure:

```text
V2 provider/catalog plugins + catalog.transform
       ↓
Catalog.Service state: Map<ProviderID, { provider, models: Map<ModelID, ModelV2.Info> }>
       ↓
catalog.provider.available() -> ProviderV2.Info[]
catalog.model.available() -> ModelV2.Info[]
       ↓
packages/server handlers and V2 runner model selection
```

[observed: EVID-005]

### Where the catalog is born

Observed:

- The raw provider/model database originates in `ModelsDev.Service`, backed by `https://models.dev/api.json`, a local cache file, an optional `OPENCODE_MODELS_PATH`, or an embedded `OPENCODE_MODELS_DEV` snapshot.
- `Provider.Service` transforms the raw database into the active V1 provider catalog using `fromModelsDevProvider` and `fromModelsDevModel`.
- V2 `Catalog.Service` is a separate stateful catalog with provider/model maps and transform support.

Inference:

- For the current active execution path, the catalog "birth" relevant to prompts is not the V2 `Catalog.Service`; it is the V1 `Provider.Service` state initialized from `ModelsDev.Service` plus local config/auth/plugin overlays.

### Provider registration

Observed V1 registration sources:

| Source | Mechanism | Effect |
|---|---|---|
| models.dev | `fromModelsDevProvider` | Creates baseline provider/model database. |
| Config | `cfg.provider` | Adds custom providers, overrides existing providers/models, sets `npm`, API URL, env, options, model fields, whitelist/blacklist. |
| V1 plugin provider hook | `hook.provider.models` | Replaces/returns a provider's model map for providers already known in `database`. |
| Environment | provider `env` vars | Activates a provider with `source: "env"` and optional key. |
| Stored auth | `auth.all()` | Activates a provider with `source: "api"`. |
| Built-in customizers | internal `custom(dep)` | Autoloads some providers, mutates options, installs model loaders/vars/discovery loaders. |
| Discovery loader | internal `discoverModels` | Appends missing dynamically discovered models for providers with such a loader. |

Observed docs:

- Official docs describe Models.dev + AI SDK, `/connect`, `provider`, `model`, `small_model`, custom providers, `disabled_providers`, `enabled_providers`, `blacklist`, `whitelist`, variants, and `/models` selection.

### Model list acquisition per provider

Observed:

- Baseline model lists come from each models.dev provider's `models` map.
- Config can add or override provider model entries under `provider.<providerID>.models`.
- V1 `ProviderHook.models` can return a model map for a provider.
- Internal discovery loaders can append models after provider activation.
- Some models.dev `experimental.modes` become additional synthetic model IDs.

Inference:

- OpenCode does not generally call each live provider's `/models` endpoint during normal catalog construction. Dynamic live discovery exists only through explicit provider-specific discovery loaders and selected plugin/provider implementations.

### Catalog data structures

V1 active structure:

```ts
type ProviderCatalog = Record<ProviderID, {
  id: ProviderID
  name: string
  source: "env" | "config" | "custom" | "api"
  env: string[]
  key?: string
  options: Record<string, any>
  models: Record<ModelID, ProviderModel>
}>
```

V1 model structure (normalized from code):

```ts
type ProviderModel = {
  id: ModelID
  providerID: ProviderID
  api: { id: string; url: string; npm: string }
  name: string
  family?: string
  capabilities: {
    temperature: boolean
    reasoning: boolean
    attachment: boolean
    toolcall: boolean
    input: { text: boolean; audio: boolean; image: boolean; video: boolean; pdf: boolean }
    output: { text: boolean; audio: boolean; image: boolean; video: boolean; pdf: boolean }
    interleaved: boolean | { field: "reasoning" | "reasoning_content" | "reasoning_details" }
  }
  cost: {
    input: number
    output: number
    cache: { read: number; write: number }
    tiers?: Array<{ input: number; output: number; cache: { read: number; write: number }; tier: { type: "context"; size: number } }>
    experimentalOver200K?: { input: number; output: number; cache: { read: number; write: number } }
  }
  limit: { context: number; input?: number; output: number }
  status: "alpha" | "beta" | "deprecated" | "active"
  options: Record<string, any>
  headers: Record<string, string>
  release_date: string
  variants?: Record<string, Record<string, any>>
}
```

V2 available model structure:

```ts
type ModelV2Info = {
  id: ModelID
  providerID: ProviderID
  family?: string
  name: string
  api: Api
  capabilities: { tools: boolean; input: string[]; output: string[] }
  request: { headers: Record<string, string>; body: Record<string, Json>; variant?: string }
  variants: Array<{ id: VariantID; headers: Record<string, string>; body: Record<string, Json> }>
  time: { released: number }
  cost: Array<{ tier?: { type: "context"; size: number }; input: number; output: number; cache: { read: number; write: number } }>
  status: "alpha" | "beta" | "deprecated" | "active"
  enabled: boolean
  limit: { context: number; input?: number; output: number }
}
```

Observed answer to `AvailableModel[]`:

- V1: no canonical `AvailableModel[]`; canonical service shape is provider-indexed. ACP derives `modelOptions` as a flattened list for UI/config consumers.
- V2: `catalog.model.available()` is the closest actual equivalent to `AvailableModel[]`, returning `ModelV2.Info[]`.

### Merge, filtering, duplicates, aliases

Observed:

- Merge is mostly provider-key and model-key based. Same provider/model keys merge or overwrite depending on phase; different providers are distinct identities.
- Config model aliases are represented by separating `modelID` (OpenCode-facing key) from `api.id` (provider-facing ID).
- There is no global dedupe across providers. `providerID/modelID` is the identity.
- Hardcoded alias removal exists only for a narrow set of invalid chat aliases.
- Alpha models are hidden unless experimental model runtime flags allow them; deprecated models are removed.
- `disabled_providers` and `enabled_providers` gate provider inclusion.
- Provider `blacklist` and `whitelist` gate model inclusion.
- Variants are generated and then config-merged; disabled variants are removed.

Inference:

- A Decision Engine should treat `providerID/modelID[/variant]` as the stable identity and should not assume model names are globally unique.

### Consumers

Observed V1 consumers:

| Consumer | Catalog API used | Purpose |
|---|---|---|
| Active prompt path / `SessionPrompt` | `Provider.getModel`, `Provider.defaultModel` | Resolve selected/default model before LLM execution. |
| Agents | `Provider.defaultModel`, `Provider.getModel` | Validate/resolve agent model. |
| ACP directory | `Provider.list`, `Provider.defaultModel` | Build provider snapshot, flattened `modelOptions`, variants, default model. |
| ACP config options | flattened providers/models | Build model and variant selector values. |
| HTTP instance provider handler | `ModelsDev.get`, `Provider.list` | Return `all`, `default`, `connected` providers. |
| Compaction / plan / sharing | `Provider.getModel`, `Provider.defaultModel` | Resolve model metadata for secondary flows. |

Observed V2 consumers:

| Consumer | Catalog API used | Purpose |
|---|---|---|
| `packages/server` model/provider handlers | `catalog.model.available`, `catalog.provider.available` | Server API for available V2 models/providers. |
| V2 runner model selection | `catalog.model.available` | Pick active V2 model. |
| V2 plugins | `catalog.transform` draft | Mutate provider/model/default catalog. |

### Dynamic behavior

Observed:

- `ModelsDev.Service` can refresh from upstream, but `get()` is cached indefinitely until invalidation.
- `Provider.Service` computes an `InstanceState` snapshot of effective providers; prompt execution uses this service state and does not rediscover providers per call.
- `/connect`, config reloads, process restart, or service reload boundaries may alter the effective catalog, but this was not runtime-tested in this iteration.

Hypothesis:

- In the current product, model picker changes after `/connect` likely depend on service/app reload or explicit state invalidation paths outside the inspected snippets. This is not necessary to answer the static construction mechanism.

### Decision Engine verdict

Confirmed:

- OpenCode already builds a rich internal model inventory with IDs, capabilities, modalities, limits, prices, variants, status, and provider materialization details.
- The active/V1 reusable inventory is `Provider.Service.list()` plus `getModel()`/`defaultModel()`.
- The V2 reusable inventory is cleaner for decisioning: `Catalog.Service.model.available()` returns a flattened `ModelV2.Info[]`.

Inference:

- An **in-process** Decision Engine integrated into OpenCode core could reuse the existing catalog and should not reconstruct models.dev/config/auth/provider merging independently.
- A **standalone external** Decision Engine can reuse the catalog only through whatever OpenCode server/client endpoint is available; otherwise it must reconstruct equivalent logic or be given a new API.
- A **V1 plugin** today cannot directly and officially reuse the complete active catalog through the documented plugin context. It would need a client call to OpenCode APIs or a new supported hook/service exposure.

Verdict:

> A future Decision Engine can reuse OpenCode's catalog infrastructure if it runs inside OpenCode's service graph or consumes a catalog API. It should not rebuild the catalog by scraping providers. However, under today's documented V1 plugin surface, the complete active catalog is internal; plugin-level reuse would require an API call or a new OpenCode-supported catalog service/hook.

---

## Practical Routing Through Agents/Subagents (EVID-007, EVID-008)

This section reorients Study-002 from a generic external router toward a practical OpenCode-native architecture: specialized agents/subagents with fixed models, prompts, and permissions.

### Documented officially

| Feature | Official status | Practical meaning |
|---|---|---|
| Primary agents | Documented | Main assistants selected directly by user/TUI/CLI/default agent. |
| Subagents | Documented | Specialized assistants callable by primary agents or user `@` mention. |
| Agent `model` | Documented | Each agent can pin a model optimized for that task. |
| Subagent inheritance | Documented | If a subagent has no model, it uses the invoking primary agent's model. |
| Manual invocation | Documented | User can invoke `@subagent` explicitly. |
| Automatic invocation | Documented | Primary model can call subagents through Task tool based on descriptions. |
| Permissions | Documented | Each agent can be read-only, edit-capable, shell-restricted, etc. |
| `permission.task` | Documented | A primary/orchestrator agent can expose only selected subagents to the model. |
| `hidden` | Documented | Hide subagents from autocomplete while keeping them callable by Task tool. |
| `default_agent` | Documented | Choose the default primary agent globally/project-wide. |

### Implemented in code

The implementation gives a concrete routing flow:

```text
Parent assistant turn
  │
  ├─ Task tool selected automatically by model
  │       OR
  ├─ user writes @subagent, converted to agent part
  │
  ▼
TaskTool.execute({ subagent_type, prompt })
  │
  ├─ Agent.get(subagent_type)
  ├─ create/resume child session:
  │     parentID = parent session
  │     agent = subagent name
  │     permission = derived child permissions
  │
  ├─ model = subagent.model ?? parent assistant message model
  │
  └─ prompt child session with:
        sessionID = child session
        agent = subagent name
        model = selected model
        parts = generated/resolved prompt parts
```

Observed source behavior:

- `Agent.Info` supports `mode`, `model`, `prompt`, `permission`, `options`, `temperature`, `topP`, `variant`, `steps`, `hidden`, and `description`.
- Primary prompt model resolution remains `input.model ?? ag.model ?? currentModel(sessionID)`.
- Task-tool subagent model resolution is `next.model ?? parent assistant message model`.
- Therefore a configured subagent model prevails over the main session model for that child session.
- If the subagent lacks a model, it inherits the actual model used by the invoking assistant message.
- User `@agent` mentions are represented as `agent` parts and converted into a synthetic instruction to call Task tool with that subagent.
- The Task tool result is returned to the parent as a structured tool output containing task session ID and text result.

### Proven in runtime

Runtime evidence already available in this study:

- EVID-004 proved the active live prompt path is V1, not V2, and that V1 hooks fire while V2 `aisdk.language` does not.
- This matters because the agent/subagent approach works with the active V1 path; it does not depend on V2 provider-interception hooks.

Runtime evidence not yet collected:

- No new prompt was run in this iteration to create a custom fixed-model subagent and inspect the resulting child session record.
- No local log/API recipe was validated for extracting the exact model used by each child session.

### Inferred

- Agents/subagents are not a general dynamic router. They do not inspect every prompt and choose among arbitrary models unless a primary/orchestrator model decides to call a subagent.
- They are a strong practical router when work categories are stable and recognizable: legal audit, JSON audit, question linking, refactor, senior review, validation.
- The routing decision moves from "choose a model every time" to "choose or let OpenCode choose a specialist lane". The lane owns model, prompt, tools, and permissions.
- Manual `@` invocation is the reliable override; automatic Task-tool invocation is convenient but depends on descriptions, permissions, and parent model judgment.

### Context, tools, and return path

Observed context behavior:

- The subagent receives a child session, not just an inline function call.
- The child user message is built from the Task-tool `prompt` after `resolvePromptParts`; file references can be expanded/resolved.
- The child session records `parentID`, agent, model, permissions, metadata, tokens and cost.
- The parent should include required context in the Task-tool prompt; the child should not be assumed to see the full parent transcript unless it is summarized or referenced explicitly.

Observed tool behavior:

- Available tools are resolved using the subagent's `permission` and the selected model.
- Parent session deny rules and external-directory constraints are inherited into child session permissions.
- `permission.task` on the parent filters which subagents appear in the Task tool description.
- `todowrite` and `task` are denied by default for subagents unless explicitly allowed.

Observed return behavior:

```xml
<task id="ses_child" state="completed">
<task_result>
...
</task_result>
</task>
```

- Background subagents require the experimental background flag and can inject a later synthetic result into the parent session.
- Parent agent sees the returned result text; deeper inspection requires navigating into or querying the child session.

### Verifying the model used

Implemented/observable from source:

- Child session has `parentID`, `agent`, and `model` fields.
- User messages store `model: { providerID, modelID, variant }` and `agent`.
- Assistant messages/logs include provider/model/agent/mode.
- Task-tool metadata contains `parentSessionId`, `sessionId`, and selected `model`.
- The system prompt visible to the model includes the exact model ID.

Not yet runtime-tested here:

- Best local operational command/API for extracting the child session record and message model after a real run.

### Limitations as practical routing

| Limitation | Consequence | Mitigation |
|---|---|---|
| Static lane model | No per-prompt optimization inside a lane | Keep lanes coarse and adjust manually when needed. |
| Automatic invocation is model judgment | Parent may choose wrong subagent or not call one | Use explicit `@subagent` for high-value tasks. |
| Context is not magically shared | Subagent may miss parent assumptions | Require parent to pass compact task brief, files, constraints, expected output. |
| More agents add cognitive overhead | Too many lanes recreate model-selection burden | Start with 6-8 stable lanes, not dozens. |
| Model IDs can go stale | Config may break after provider/catalog changes | Add a validation check to list agents and resolve models. |
| Tool permissions can surprise | Subagent may be too weak or too powerful | Make permissions part of each lane design. |
| Not a generic cost/latency optimizer | Does not choose cheapest model dynamically | Use cheaper fixed models for known cheap lanes. |
| Runtime audit workflow still unvalidated | Harder to prove actual usage operationally | Run minimal child-session test before adopting. |

### Conceptual architecture for the oposiciones app

Primary agents:

| Agent | Mode | Role | Model class | Permissions |
|---|---|---|---|---|
| `opos-orchestrator` | primary | Default daily coordinator; decides whether to answer directly or delegate | Balanced coding/reasoning model | Task allowed to selected subagents; edits ask/deny depending preference |
| `opos-build` | primary | Direct implementation lane for normal Next.js work | Strong coding model | Edit/bash allowed or ask |
| `opos-plan` | primary | Planning and analysis without writes | Cheaper/fast reasoning model | Edit deny; bash ask/deny |

Subagents:

| Subagent | Purpose | Model class | Permissions | Invocation |
|---|---|---|---|---|
| `cte-legal-auditor` | Check CTE/legal corpus interpretation, trace citations, identify normative risks | High-accuracy long-context reasoning model | Read/grep/glob/webfetch allowed; edit denied | Manual for audits; automatic from orchestrator for legal questions |
| `json-audit` | Validate audit JSON, schemas, invariants, idempotency | Deterministic cheaper model or strong structured-output model | Read/grep/bash test commands ask/allow; edit denied by default | Manual before importing or after generation |
| `question-linker` | Link questions to corpus/legal references, detect weak links and duplicates | Long-context semantic model | Read/grep/glob allowed; edit ask if it writes candidate links | Automatic/manual depending batch size |
| `next-refactor` | Implement focused Next.js/TypeScript refactors | Strong coding model | Edit/bash allowed or ask | Manual when implementation is desired |
| `senior-reviewer` | Review diffs for regressions, architecture, security, tests | Strong reasoning/coding review model | Read/grep/git diff/log allowed; edit denied | Manual before commit/merge |
| `validation-runner` | Run tests, typecheck, lint, migrations, summarize failures | Fast model with shell competence | Bash allow for safe test commands; edit denied | Manual after implementation |
| `commit-preparer` | Inspect status/diff/log, propose commit message, optionally commit only on explicit user request | Reliable concise coding model | Git status/diff/log allow; commit ask; push deny/ask | Manual at end of completed work |

Recommended operating pattern:

1. Use `opos-orchestrator` as default for broad requests.
2. Use explicit `@cte-legal-auditor`, `@json-audit`, `@question-linker`, or `@senior-reviewer` when correctness matters more than speed.
3. Use `opos-build`/`next-refactor` only for write-intended tasks.
4. Keep validation and commit preparation separate from implementation.
5. Start with fixed models per lane; do not add dynamic routing until there is evidence that lane-level routing is insufficient.

Practical verdict:

> Yes. For the app de oposiciones, fixed-model OpenCode agents/subagents are a better near-term solution than a generic external router. They match the real recurring work types, use documented OpenCode mechanisms, preserve manual override, give permission boundaries, and are auditable through sessions/messages/logs. They do not replace a future adaptive router, but they likely solve the daily model-selection burden with much less complexity.

---

## Position 1: Client-Side Preprocessor

### Description
Insert routing logic between the user interface (TUI, CLI, desktop, IDE) and the `promptAsync` API call. The preprocessor examines the prompt, session context, or task characteristics and decides which `input.model` to pass.

### Ubicación
```
User → [Router CLI/TUI layer] → promptAsync({ model: decided })
                                  │
                                  ▼
                                OpenCode (unchanged)
```

### Ventajas
- **Zero invasiveness**: OpenCode source is never touched.
- **Full control**: can examine prompt, parts, agent, session, and any external signals.
- **Supports any strategy**: cost-based, capability-based, latency-based, user-preference-based.
- **Independent lifecycle**: the router can evolve, fail, or be replaced without affecting OpenCode.
- **Works with any client**: can wrap CLI, TUI, desktop, or SDK calls.

### Limitaciones
- **Duplicate context**: the router may need to replicate some OpenCode state (session info, agent config, available providers) to make good routing decisions.
- **Race conditions**: if the router and OpenCode both try to decide the model, there is no single source of truth.
- **Not transparent to the user**: the router's decision happens "before" OpenCode, so the user sees a potentially unexpected model in the UI.
- **Client-dependent**: each client (TUI, CLI, IDE plugin) may need its own preprocessor integration; not a single universal solution.

### Grado de invasividad
**Ninguno** — no modification to OpenCode.

### Compatibilidad con OpenCode
- Fully compatible. Sets `input.model` which OpenCode already supports.
- Works with both v1 `promptAsync` and v2 `session.prompt` paths (both accept `model`).
- No risk of breaking streaming, tool calls, or SSE events.

### Impacto en mantenimiento
- Low for OpenCode (zero changes).
- Medium for the router itself: must stay compatible with the `model` parameter shape across OpenCode versions.

---

## Position 2: Agent Configuration Routing

### Description
Leverage OpenCode's existing agent→model mapping. Define multiple agents, each configured with a specific model. The routing decision becomes: **which agent to invoke** for a given prompt. The router selects the agent, and OpenCode picks up `agent.model` from the configuration.

### Ubicación
```
User → [Router] → promptAsync({ agent: "code-fast" })
                    │
                    ▼
                  OpenCode
                    │
                    ├── agent.model = "claude-sonnet-4"   (from config)
                    └── createUserMessage → fallback → agent.model
```

### Ventajas
- **Uses existing OpenCode feature** — agent→model mapping is already supported, confirmed by Study-001.
- **Configuration-driven**: no code changes in OpenCode; only config files.
- **Natural separation**: different agents can have different instructions, tools, and permissions, not just different models.
- **Granularity per task type**: "code generation agent uses model X, debugging agent uses model Y".

### Limitaciones
- **Static mapping**: agent→model is fixed in configuration. Dynamic routing requires changing which agent is selected, not the model per prompt.
- **Configuration overhead**: each (routing decision, model) combination needs a distinct agent definition. For N routing criteria × M models, you may need N×M agents.
- **Agent semantics overload**: using agents purely for model routing conflates agent identity (role, tools, instructions) with model selection.
- **Routing intelligence is external**: the router decides which agent; OpenCode just follows the static mapping.

### Grado de invasividad
**Bajo** — requires creating agent config files; no source modification.

### Compatibilidad con OpenCode
- Fully compatible. Agents with `model` are a documented, tested feature.
- No risk of breaking streaming, tools, or events.

### Impacto en mantenimiento
- Low for OpenCode.
- Medium-high for the agent definitions: each change in routing strategy requires updating agent configs.

---

## Position 3: Configuration / Fallback Injection

### Description
Influence the `currentModel(sessionID)` fallback chain by modifying `model.json`, `opencode.json`, environment variables, or provider configuration. This changes the **default** model that OpenCode uses when neither `input.model` nor `agent.model` is specified.

### Ubicación
```
OpenCode startup
  │
  ├── model.json         ◄── set default model
  ├── opencode.json      ◄── set default provider/model
  └── env vars           ◄── set provider defaults
        │
        ▼
      createUserMessage → currentModel(sessionID)
                           ├── session.currentModel (set by config)
                           ├── recent user message model
                           └── provider.defaultModel()
                                 ├── cfg.model
                                 ├── model.json
                                 └── first available provider/model
```

### Ventajas
- **Minimal invasiveness**: configuration-only; no code or proxy.
- **Quick to change**: update a JSON file or env var.
- **Works globally**: affects all prompts that don't specify an explicit model.

### Limitaciones
- **No per-prompt granularity**: the configuration applies globally; dynamic routing per task/prompt is impossible.
- **Static**: changing the default requires a config reload or restart.
- **Limited expressiveness**: can only set one default; cannot route based on prompt content, cost, latency, or capability.
- **Cannot override explicit `input.model`**: if a caller sets `input.model`, the fallback is never reached.
- **Pollution of state**: setting `session.currentModel` affects subsequent prompts in the same session.

### Grado de invasividad
**Mínimo** — configuration only.

### Compatibilidad con OpenCode
- Fully compatible, designed to work this way.
- No risk to streaming, tools, or events.

### Impacto en mantenimiento
- Very low. Configuration schema is stable.
- Risk of invisible config drift: multiple config files can override each other in non-obvious ways.

---

## Position 4: Provider-Level Interceptor

### Description
Implement a custom OpenCode provider (or wrap an existing one) that intercepts `getModel` and `getLanguage` calls. The custom provider can inspect the request context and delegate to different backend providers/models dynamically.

### Ubicación
```
createUserMessage → { providerID: "router", modelID: "dynamic" }
                            │
                            ▼
                      Provider.getModel("router", "dynamic")
                            │
                            ▼
                      [Custom Router Provider]
                            │
                            ├── analyze request context
                            ├── decide: "claude-sonnet" or "gpt-4o" or "gemini-2.5"
                            └── delegate to real provider
                                  │
                                  ▼
                            Provider.getLanguage(realModel)
                                  │
                                  ▼
                            LLM.stream → real provider API
```

### Ventajas
- **Centralized routing logic**: all model decisions go through one provider.
- **Access to full context**: the provider has access to the model reference, and through the surrounding system, potentially to prompt content and session state.
- **Clean abstraction**: OpenCode's provider interface is designed for this kind of extension.
- **Dynamic at materialization time**: can route per-call, not per-configuration.

### Limitaciones
- **After model resolution**: the model reference (`providerID/modelID`) is already chosen by `createUserMessage`. The provider intercepts during materialization, not selection. The routing decision is constrained by the original reference.
- **Provider registration may require source changes**: it is unknown whether OpenCode supports registering custom providers through configuration alone (CLAIM-006).
- **Effect-TS dependency injection**: OpenCode uses Effect-TS services; overriding a provider may require understanding its service layer.
- **Tool call compatibility**: the routed-to provider must support the same tool-call interface that OpenCode expects.
- **Streaming must work through the delegation**: the custom provider must correctly forward or reconstruct streaming responses.

### Grado de invasividad
**Medio** — requires implementing a provider; may need configuration changes; may need service-layer understanding.

### Compatibilidad con OpenCode
- Functionally compatible if the provider interface is respected.
- Risk area: tool call format differences between upstream providers.
- Risk area: streaming event format differences.

### Impacto en mantenimiento
- Medium: custom provider must track OpenCode's provider interface changes.
- Medium: must handle provider-specific error modes, rate limits, and auth.

### Veredicto actualizado (EVID-002, CLAIM-007, CLAIM-011, CLAIM-012, CLAIM-015)

This position was the principal target of the Q-002 investigation. The refined verdict, split by generation:

- **Active V1 path (today, commit `ae53163`): NOT viable without core modification.** The provider abstraction's only dynamic point (`Provider.getLanguage` via `modelLoaders`) is a closed built-in registry (`provider.ts:168`); the V1 plugin API exposes only `models`, not `getModel`. No V1 hook can redirect the primary model per-prompt (CLAIM-012). The sole code-reachable per-call interception on V1 is the `customFetch` wrapper in `resolveSDK` (`provider.ts:1703-1734`), which is HTTP-level interception (i.e. Position 5 in-process), not a provider-abstraction interceptor.
- **V2 path: viable without fork, conditional on activation.** The V2 `aisdk.language` runtime hook can set `event.language` to a delegating `LanguageModelV3`, which is exactly Position 4 (CLAIM-015, hypothesis). But V2's runtime hooks are not reached by the live prompt path today (CLAIM-011, supported; residual runtime uncertainty in U-006).

**Net**: Position 4 graduates from pure hypothesis to *conditionally viable*. Today it requires either (a) a core modification to open `modelLoaders`/add a V1 model-redirect hook, or (b) waiting for/migrating to the V2 execution path. It is not a no-fork option on the current active path.

---

## Position 5: External HTTP Proxy

### Description
Place an HTTP proxy between OpenCode and the LLM provider APIs. The proxy intercepts outgoing requests, inspects the request body, rewrites the `model` field and possibly the endpoint URL, and forwards to the target provider. The response (including streams) is passed back transparently.

### Ubicación
```
OpenCode → HTTP request (e.g., POST https://api.openai.com/v1/chat/completions)
              │
              ▼
          [External Proxy]
              │
              ├── inspect request body
              ├── rewrite model: "gpt-4o" → "claude-sonnet-4-20250514"
              ├── rewrite endpoint: openai.com → anthropic.com (or use unified API)
              ├── handle auth translation
              └── forward to selected provider
                    │
                    ▼
              [Real Provider API]
                    │
                    ▼
              Response → Proxy → OpenCode
```

### Ventajas
- **Zero modification to OpenCode**: works at the network level.
- **Transparent**: OpenCode is unaware of the proxy; models appear as the configured provider.
- **Can route on any HTTP-visible signal**: prompt content, token count, user agent, session cookie.
- **Can implement any strategy**: cost, latency, capability, fallback, load balancing.
- **Independent lifecycle**: deploy, scale, and maintain separately from OpenCode.

### Limitaciones
- **Protocol compatibility risk**: OpenCode's provider integration may use non-standard request/response shapes, streaming SSE formats, or custom headers. The proxy must preserve these perfectly.
- **Tool call serialization**: different providers have different tool call schemas. A proxy routing from OpenAI-formatted requests to Anthropic must also translate the tool call format — or require that OpenCode already uses a normalized format.
- **Streaming must be preserved**: SSE streams with custom events must pass through without corruption.
- **Authentication complexity**: the proxy must either forward OpenCode's API keys or manage its own credentials per backend provider.
- **Latency overhead**: additional network hop.
- **Stateful routing decisions**: if the proxy needs session context (beyond a single HTTP request), it must maintain or reconstruct state.
- **Not portable**: requires infrastructure (proxy server) to deploy and operate.

### Grado de invasividad
**Ninguno** — no modification to OpenCode. However, it may require changes to OpenCode's provider configuration (change the base URL to point to the proxy).

### Compatibilidad con OpenCode
- Potentially compatible, but **unvalidated**.
- Streaming, tool calls, event formats, and authentication are all risk areas.
- OpenCode's provider configuration must be pointed at the proxy.

### Impacto en mantenimiento
- High for the proxy: must track provider API changes, tool call format evolution, OpenCode-specific protocol behaviors.
- Low for OpenCode: no changes needed.

---

## Position 6: LLM Execution Wrapper

### Description
Wrap or replace the `LLM.stream` function, the AI SDK's `streamText`, or the native runtime to intercept the already-resolved model and redirect execution to a different provider/model.

### Ubicación
```
runLoop → SessionProcessor.process
              │
              ▼
          LLM.stream(input)
              │
              ├── (intercepted) change input.model
              ├── native path or AI SDK streamText
              └── HTTP request
```

### Ventajas
- **Full control at the last responsible moment**: can override the model just before the API call.
- **Access to the full stream API**: can transform, log, or augment the response.
- **Can implement fallback logic**: if the primary model fails, retry with a different model.

### Limitaciones
- **After model resolution**: the model is already chosen by `createUserMessage` and materialized by `Provider.getModel`. Wrapping at `LLM.stream` means routing against an already-committed decision.
- **High invasiveness**: requires hooking into OpenCode's internal module/service — likely requires source modification or deep monkey-patching.
- **Effect-TS dependency**: `LLM.stream` is likely an Effect-TS function or service; wrapping it requires understanding the Effect runtime.
- **Two runtime adapters**: native and AI SDK have different code paths; a wrapper must handle both.
- **Maintenance burden**: internal APIs change more frequently than external ones.

### Grado de invasividad
**Alto** — requires modifying or monkey-patching OpenCode internals.

### Compatibilidad con OpenCode
- Technically possible but fragile.
- Risk of breaking with any OpenCode update that changes `LLM.stream` internals.

### Impacto en mantenimiento
- High: tracks OpenCode's internal changes.
- High: must handle both native and AI SDK runtime paths.

---

## Position 7: MCP Server / Tool-Based Intercept

### Description
Use OpenCode's MCP server or tool system to influence model selection indirectly. A tool could analyze the task and return a model recommendation, or an MCP server could expose model information that affects agent selection.

### Ubicación
```
User → prompt
  │
  ├── router tool executes before main LLM call
  │     └── tool returns "recommend model: claude-sonnet-4"
  │
  └── (agent may or may not act on tool output)
```

### Ventajas
- **Non-invasive**: tools and MCP servers are supported extension mechanisms.
- **Can use model information as context**: the agent can see model recommendations and act on them.

### Limitaciones
- **Indirect and non-deterministic**: the agent/tool has no binding authority over model selection. It can only suggest.
- **Not suitable as primary router**: cannot guarantee a specific model is used.
- **Adds latency and token cost**: the tool runs within the LLM loop, consuming context and generation time.
- **Complex interaction**: the routing logic competes with the agent's own decisions.
- **MCP servers are not designed for this**: MCP is for tools and context, not model routing.

### Grado de invasividad
**Bajo** — uses supported extension mechanisms.

### Compatibilidad con OpenCode
- Compatible as an advisory mechanism.
- Not suitable for deterministic routing.

### Impacto en mantenimiento
- Low: uses standard MCP/tool interfaces.
- Adds operational complexity: must deploy and maintain an MCP server.

---

## Position 8: Fork / Source Modification

### Description
Fork the OpenCode repository and add a routing hook directly into the source code at the desired interception point (e.g., inside `createUserMessage`, `runLoop`, or `LLM.stream`).

### Ubicación
```
[Forked OpenCode]
  │
  ├── createUserMessage: [NEW] call router before model resolution
  ├── runLoop: [NEW] allow model re-selection
  └── LLM.stream: [NEW] routing-aware execution
```

### Ventajas
- **Full control**: can route at any point, with full access to all internal state.
- **Clean integration**: the routing logic is part of the system, not an external bolt-on.

### Limitaciones
- **Highest maintenance burden**: must rebase on every OpenCode release.
- **Diverges from upstream**: security fixes, features, and bug patches require manual merging.
- **Not a realistic option for most teams** unless they are OpenCode maintainers.
- **Requires deep understanding of OpenCode internals**.

### Grado de invasividad
**Total** — modifies OpenCode source.

### Compatibilidad con OpenCode
- N/A — it is a modified OpenCode.

### Impacto en mantenimiento
- Very high: fork maintenance, rebase burden, divergence risk.

---

## Summary Comparison

| Position | Invasiveness | Dynamic | Before resolution | Streaming safe | Tool call safe | Config only |
|---|---|---|---|---|---|---|
| 1. Client-side preprocessor | None | Yes | **Yes** | Yes | Yes | No |
| 2. Agent config routing | Low | Static mapping | **At resolution** | Yes | Yes | Yes |
| 3. Configuration injection | Minimal | No | **At resolution** | Yes | Yes | Yes |
| 4. Provider-level interceptor | Medium | Yes | No (at materialization) | Unknown | Unknown | Unknown |
| 5. External HTTP proxy | None | Yes | No (at network) | Unknown | Unknown | No |
| 6. LLM execution wrapper | High | Yes | No (at execution) | Unknown | Unknown | No |
| 7. MCP/tool-based | Low | Advisory | No | Yes | Yes | No |
| 8. Fork/modify source | Total | Yes | **Yes** | Yes | Yes | No |

**Key insight**: Only Positions 1 (client-side), 2 (agent config), and 3 (config injection) intercept **before** model resolution. Position 8 (fork) also can but at maximum cost.

For **dynamic, non-invasive routing**, Position 1 (client-side preprocessor) is the strongest candidate because it operates before model resolution with zero invasiveness and full expressiveness.

For **configuration-driven static routing**, Position 2 (agent config) is the most OpenCode-native approach.

For **transparent, protocol-level routing** with potential full dynamic capability, Position 5 (external proxy) is the strongest candidate but carries significant protocol compatibility risk.

**Updated insight after Q-002 (EVID-002/003)**: Position 4 (provider-level interceptor) is *conditionally viable* — not on the active V1 path without core modification (the `modelLoaders` registry is closed and no V1 hook can redirect the primary model; CLAIM-012), but viable on the V2 path via `aisdk.language` once V2 is the active execution path (CLAIM-015). Position 6 (LLM execution wrapper) remains high-invasiveness: the only code-reachable per-call interception on V1 is `resolveSDK`'s `customFetch` wrapper (HTTP-level, i.e. Position 5 in-process), not a stable public hook.
