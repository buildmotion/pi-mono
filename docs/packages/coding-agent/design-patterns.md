# Design Patterns — pi-coding-agent

## 1. Facade (AgentSession)

**Pattern:** A single class provides a simplified interface to a complex subsystem.

`AgentSession` wraps `Agent` (pi-agent-core), `SessionManager` (disk persistence), `ExtensionRunner` (plugin lifecycle), and context compaction into one coherent object that modes interact with.

**Why:** All four run modes (interactive, print, RPC, SDK) need identical agent behavior with different I/O. The facade isolates that shared behavior.

---

## 2. Plugin / Extension System

**Pattern:** A host defines lifecycle hooks; plugins register handlers without modifying host code.

`ExtensionRunner` (`src/core/extensions/runner.ts`) fires named events (`session_start`, `turn_end`, `tool_execution_end`, …). Extensions subscribe to the events they care about. New capabilities — custom tools, slash commands, system prompt additions — are added without touching core code.

**Why:** Allows teams to customize `pi` for their workflows while staying on the upstream release track.

---

## 3. Decorator (ToolDefinitionWrapper)

**Pattern:** Wrap an object to add behavior before and after its operations.

`ToolDefinitionWrapper` (`src/core/tools/tool-definition-wrapper.ts`) wraps each `AgentTool`'s `execute()` method. Before execution it calls extension `beforeToolCall` hooks; after execution it calls `afterToolCall` hooks. The wrapped tool looks identical to the Agent.

**Why:** Extension hooks are transparent to both the tool implementation and the Agent.

---

## 4. Strategy (Run Modes)

**Pattern:** A family of interchangeable algorithms selected at runtime.

Interactive mode, print mode, and RPC mode are all strategies for the same core operation: "run the agent and present output." `AgentSession` does not know which mode is in use; modes consume the same event stream.

**Why:** Adding a new mode (e.g., a web server mode) requires only a new I/O strategy, not changes to `AgentSession`.

---

## 5. Memento (Session Branching)

**Pattern:** Capture and restore an object's state.

`SessionManager` saves named branch snapshots of `AgentMessage[]` arrays. `switchToBranch()` restores a previous state. This is the session-level analogue of git branches.

**Why:** Lets users experiment with risky LLM operations and roll back without losing prior context.

---

## 6. Chain of Responsibility (Context Compaction)

**Pattern:** Pass a request along a chain of handlers; the first capable handler processes it.

`shouldCompact()` checks whether the context has grown too large. If so, `compact()` invokes the LLM to summarize old messages. Extensions can intercept via `onBeforeCompact` to customize what gets summarized.

**Why:** Keeps the context window manageable for long-running sessions without manual intervention.

---

**What's Next:** [Terminology](./terminology.md) | [Extension Points](./extension-points.md)
