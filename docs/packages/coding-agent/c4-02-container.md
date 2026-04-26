## Learning Objectives

- Identify the major runtime sub-systems inside the `pi` Node.js process.
- Understand how the four modes (interactive, print, RPC, SDK) share the same `AgentSession` core.
- See how extensions and skills are loaded at startup.

---

## C4 Container Diagram

```mermaid
C4Container
  title Container View — @mariozechner/pi-coding-agent

  Person(dev, "Developer")

  Container_Boundary(pi, "pi Node.js process") {
    Container(cli, "CLI Entry (src/cli.ts)", "Node.js", "Parses arguments, selects run mode, bootstraps configuration")
    Container(session, "AgentSession (src/core/agent-session.ts)", "TypeScript class", "Owns the Agent; manages session persistence, compaction, extensions")
    Container(agent, "Agent (pi-agent-core)", "TypeScript class", "Drives the LLM → tool-call → result loop")
    Container(tools, "Built-in Tools (src/core/tools/)", "AgentTool[]", "bash, read, write, edit, find, grep, ls")
    Container(extensions, "Extension Runner (src/core/extensions/)", "Plugin host", "Loads and runs TypeScript extensions; provides UI and session APIs")
    Container(tui, "Interactive Mode (src/modes/interactive/)", "TUI layer", "pi-tui-based UI for the interactive terminal session")
    Container(print, "Print Mode (src/modes/print-mode.ts)", "Stdout renderer", "Renders events as plain text or JSON to stdout")
    Container(rpc, "RPC Mode (src/modes/rpc/)", "JSON-RPC server", "Exposes agent over JSON-RPC on stdio for IDE integration")
    Container(sdk, "SDK (src/core/sdk.ts)", "Programmatic API", "Use pi-coding-agent as a library from other TypeScript code")
  }

  System_Ext(llm, "LLM Provider API")
  System_Ext(fs, "File System")

  Rel(dev, cli, "Launches pi with arguments")
  Rel(cli, session, "Creates AgentSession for the chosen mode")
  Rel(session, agent, "Delegates prompt/tool execution to Agent")
  Rel(session, tools, "Registers tools with Agent")
  Rel(session, extensions, "Loads extension modules, fires lifecycle events")
  Rel(agent, llm, "Calls LLM provider", "HTTPS")
  Rel(tools, fs, "Reads/writes files, runs bash")
  Rel(session, tui, "Interactive mode only")
  Rel(session, print, "Print mode only")
  Rel(session, rpc, "RPC mode only")
```

---

## Mode Comparison

| Mode | Entry | I/O | Use Case |
|------|-------|-----|----------|
| Interactive | `pi` (no flags) | pi-tui terminal UI | Daily coding assistant |
| Print/JSON | `pi --print` | stdout text or JSON | Shell scripts, CI |
| RPC | `pi --rpc` | JSON-RPC over stdio | IDE/editor integration |
| SDK | `import { AgentSession }` | Programmatic TypeScript | Embed in other tools |

---

**← [Context](./c4-01-context.md)** | **[Component View →](./c4-03-component.md)**
