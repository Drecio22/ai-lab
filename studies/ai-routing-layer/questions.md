# Study-002 Questions

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
Where should an intelligent routing layer be placed to control which LLM model OpenCode uses, without modifying OpenCode source code?

State:
open

Related to:
routing, architecture, model-selection, extension-points, proxy

---

## Derivation from Study-001

Study-001 confirmed that OpenCode's effective model resolution occurs in `SessionPrompt.createUserMessage` via:

```
input.model ?? agent.model ?? currentModel(sessionID)
```

This resolution happens **before** `runLoop`, `Provider.getModel`, `SessionProcessor.process`, and `LLM.stream` execute. After this point, no further model selection occurs — only materialization and execution.

Therefore, Q-001 asks: at what architectural layer can we inject routing logic **before** or **instead of** this static resolution?
