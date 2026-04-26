# `@mariozechner/pi-coding-agent` — Terminal Coding Harness (`pi` CLI)

## What is it?

`pi` is the primary end-user application in the pi-mono stack. It is a terminal-based AI coding assistant that wraps a full LLM agent (from `pi-agent-core`) in an interactive TUI (from `pi-tui`), adds file-system tools (`read`, `write`, `edit`, `bash`), and provides a rich extensibility model so teams can adapt it to their workflows without forking the source.

**npm package:** `@mariozechner/pi-coding-agent`  
**CLI binary:** `pi`  
**Version:** 0.70.2  
**Node requirement:** >= 20.6.0  
**License:** MIT

---

## The Problem It Solves

AI coding tools tend to be opinionated monoliths. Getting them to fit a team's specific workflow — custom context injection, specialized tools, project-specific prompts, bespoke UI — usually requires forking and patching the internals. `pi` inverts this: it ships with powerful defaults and exposes clean TypeScript extension points so you add capabilities without touching core code.

---

## Audience

- Individual developers who want a terminal AI pair-programmer
- Engineering teams who want a shared, customizable coding assistant
- Tool builders who want to embed a coding agent in their own Node.js applications (SDK mode)
- CI/CD pipelines that need a coding agent in non-interactive (print/JSON/RPC) mode

---

## Four Operating Modes

| Mode | How to trigger | Use case |
|------|---------------|----------|
| **Interactive** | `pi` (no arguments) | TUI with streaming responses and tool calls |
| **Print / JSON** | `pi "your prompt here"` or `pi --json "…"` | Scripting; single-shot prompt |
| **RPC** | `pi --rpc` | Process integration; IDE plugins receive structured JSON |
| **SDK** | `import { … } from '@mariozechner/pi-coding-agent'` | Embed in another Node.js app |

---

## Interactive Mode: The TUI

When you run `pi` in a terminal, you get a three-zone layout:

```
┌─────────────────────────────────────────────┐
│ Startup header                               │
│ (shortcuts, loaded AGENTS.md, templates,    │
│  skills, extensions)                         │
├─────────────────────────────────────────────┤
│ Message history                              │
│ - Your prompts                               │
│ - Assistant responses (streaming)            │
│ - Tool call cards (with results)             │
│ - Extension UI widgets                       │
├─────────────────────────────────────────────┤
│ Editor                                       │
│ (multi-line input, Emacs keybindings,        │
│  autocomplete, paste handling)               │
├─────────────────────────────────────────────┤
│ Footer: cwd / session / tokens / cost / model│
└─────────────────────────────────────────────┘
```

The editor uses `pi-tui`'s `Editor` component and supports all its keybindings. Extensions can replace the editor entirely, inject widgets above or below it, add a status line, or show overlays.

### Built-in Commands (slash commands)

| Command | Action |
|---------|--------|
| `/model` or `Ctrl+L` | Switch model |
| `/login` | Authenticate with a provider subscription |
| `/settings` | Open settings overlay |
| `/hotkeys` | Show all keyboard shortcuts |
| `/session` | Manage sessions (branch, compact, export) |
| `/compact` | Summarize old context to free token budget |

---

## Built-in Tools

Out of the box, the model receives four tools:

| Tool | What it does |
|------|-------------|
| `read` | Read a file's contents |
| `write` | Write or create a file |
| `edit` | Apply a targeted string replacement to a file |
| `bash` | Execute a shell command in the project directory |

These four tools are sufficient for most coding tasks: the model reads code, edits it, writes new files, and runs tests or linters to verify its work.

---

## Extensibility Model

`pi` is designed so you can add capabilities without modifying its source.

### Skills

Skills are small scripts (shell, Python, TypeScript) that the model can execute as additional tools. The model can also write new skills itself during a session, making it self-extending.

```
~/.pi/agent/skills/
  summarize-pr.sh     ← "summarize-pr" tool available to the model
  run-tests.ts
```

### Prompt Templates

Named prompts that appear in the startup header and can be invoked by name during a session. Useful for project-specific instructions.

```
~/.pi/agent/templates/
  refactor.md
  review.md
```

### Extensions

TypeScript modules loaded at startup that can:
- Register new tools
- Replace or augment the editor
- Add status lines, widgets, overlays
- Intercept events
- Provide custom UI for Q&A flows

```typescript
// .pi/extensions/my-extension.ts
export default function register(ctx: ExtensionContext) {
  ctx.addTool({ name: 'my_tool', … });
  ctx.addWidget(myWidget);
}
```

### Themes

JSON files that control colors and styling of all TUI components.

### Pi Packages

Bundle skills, templates, extensions, and themes into npm packages and share them:

```bash
npm install @team/pi-pkg-jira  # adds a Jira tool, prompts, etc.
```

---

## Sessions

A session is a saved conversation: all messages, tool results, metadata, and token usage.

- Sessions are stored in `~/.pi/sessions/` (or the configured path).
- Each session is associated with a working directory.
- You can **branch** a session (fork from any point in history) to try alternative approaches.
- **Compaction** summarizes old messages to stay within the model's context window without losing the gist of earlier work.
- Sessions can be exported to HTML for sharing.

---

## Context Files (AGENTS.md)

`pi` automatically loads any `AGENTS.md` file it finds in the current directory or its parents. These files inject project-specific context (architecture notes, coding conventions, forbidden patterns) into the system prompt automatically, without requiring a slash command.

---

## Provider and Model Support

`pi` inherits `pi-ai`'s full provider list. Authentication:

- **API keys:** Set the matching environment variable (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, etc.) or enter them via `/settings`.
- **Subscriptions (OAuth):** Run `/login` and follow the browser flow. Tokens are stored in `~/.pi/agent/auth.json`.

Model selection: `/model` (or `Ctrl+L`) opens a searchable model picker showing all tool-capable models for authenticated providers.

---

## Architecture: How It All Fits Together

```
pi CLI
├── cli.ts                  Parses CLI args, selects mode, bootstraps
├── main.ts                 Entry point
├── config.ts               ~/.pi/agent/ config loading
├── migrations.ts           Config schema migrations
├── core/
│   ├── model-resolver.ts   Picks default model per provider
│   ├── tools/              read, write, edit, bash tool definitions
│   ├── hooks/              SDK hooks API
│   ├── session/            Session save/load/branch/compact
│   ├── context/            AGENTS.md loading, context injection
│   └── export-html/        HTML export templates
├── modes/
│   ├── interactive/        TUI mode (pi-tui wiring, editor, renderers)
│   ├── print/              Non-interactive single-shot mode
│   ├── json/               JSONL event output mode
│   └── rpc/                JSON-RPC process interface
└── package-manager-cli.ts  Pi Package install/manage commands
```

`pi-agent-core`'s `Agent` class is the engine. `pi-tui` provides the rendering layer. `pi-ai` handles provider communication. `coding-agent` owns everything above that: session management, file tools, extensions, and the four operating modes.

---

## RPC Mode

RPC mode exposes `pi`'s full capabilities over stdin/stdout JSON-RPC. An IDE plugin or tool sends requests like:

```json
{ "id": 1, "method": "prompt", "params": { "text": "Refactor this function" } }
```

And receives back a stream of typed events. This is how IDE integrations can reuse the full `pi` stack without reimplementing it.

---

## SDK Mode

Import `pi-coding-agent` as a library and embed it in your own Node.js app:

```typescript
import { createAgent, createTools } from '@mariozechner/pi-coding-agent';

const agent = createAgent({ workingDir: '/my/project', model: '…' });
agent.on('message', (msg) => console.log(msg));
await agent.prompt('Add unit tests for src/utils.ts');
```

See [openclaw/openclaw](https://github.com/openclaw/openclaw) for a real-world example.

---

## Key Dependencies

| Dependency | Purpose |
|-----------|---------|
| `@mariozechner/pi-ai` | LLM provider access |
| `@mariozechner/pi-agent-core` | Agentic loop, event streaming |
| `@mariozechner/pi-tui` | Terminal UI rendering |
| `@mariozechner/jiti` | TypeScript extension loading at runtime |
| `diff` | Diff computation for `edit` tool display |
| `glob` | File-pattern matching for tools |
| `marked` | Markdown rendering in TUI |
| `yaml` | YAML config parsing |
| `chalk` | Terminal color output |

---

## Pros and Cons

**Pros**
- Works with 20+ providers, including subscription accounts
- Extensible without forking — skills, extensions, templates, themes, packages
- Session branching and compaction for long-running projects
- Four modes cover interactive, scripting, IDE integration, and library use
- AGENTS.md auto-context injection for project-aware behavior
- Cross-platform: Linux, macOS, Windows, Termux/Android

**Cons**
- Terminal-only interactive UI (browser users need `pi-web-ui`)
- Advanced features (sub-agents, plan mode) must be added as extensions — not shipped by default
- Extension TypeScript requires a build step or is loaded via `jiti` (dynamic compilation)
- Session storage is local filesystem only — no cloud sync out of the box

---

## How to Use It (Junior Developer Walkthrough)

Think of `pi` like a very capable pair programmer living in your terminal. You describe what you want; it reads your files, makes changes, and runs commands to verify the result.

1. **Install globally:**
   ```bash
   npm install -g @mariozechner/pi-coding-agent
   ```

2. **Set your API key:**
   ```bash
   export ANTHROPIC_API_KEY=sk-ant-...
   ```

3. **Start in your project:**
   ```bash
   cd my-project
   pi
   ```

4. **Type a task:**
   ```
   Add a unit test for the `parseDate` function in src/utils.ts
   ```
   `pi` reads `utils.ts`, generates a test, writes it to disk, then runs your test command to confirm it passes.

5. **Switch models mid-session:** `Ctrl+L` → search for any model → Enter.

6. **Branch to try something different:** `/session branch` — creates a fork from the current point.

7. **Install a community package:**
   ```bash
   pi install @team/pi-pkg-jira
   ```
   Now the model has a `jira` tool for looking up tickets.
