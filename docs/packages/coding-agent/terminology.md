# Terminology — pi-coding-agent

**AgentSession**
The core abstraction shared by all pi run modes. Owns an `Agent`, manages session persistence, fires extension lifecycle hooks, and handles context compaction. Source: `src/core/agent-session.ts`.

**Branch (session branch)**
A named snapshot of the conversation's `AgentMessage[]` array at a point in time. Branches enable "what if" exploration — you can diverge and roll back. Analogous to a git branch for a conversation.

**Compaction**
The process of summarizing older messages in the context window into a condensed `CompactionEntry` when the token count approaches the model's limit. Preserves the full history on disk while keeping the live context small. Source: `src/core/compaction/`.

**Extension**
A TypeScript module that hooks into the agent lifecycle via named events (`onSessionStart`, `onTurnEnd`, `onToolExecutionEnd`, etc.) and can register custom tools, slash commands, and system prompt additions. Source: `src/core/extensions/`.

**Extension API**
The object passed to extension hooks. Provides access to `registerTool()`, `appendToSystemPrompt()`, `registerSlashCommand()`, `exec()`, `log()`, `agent`, and session state.

**Interactive mode**
The default run mode. Renders a full terminal UI using pi-tui with a multi-line editor, Markdown message display, and tool-execution banners.

**Pi Package**
An npm package or git repository that bundles Extensions, Skills, and Prompt Templates. Installed with `pi package install <package>`.

**Print mode**
A non-interactive run mode (`pi --print`) that renders events as plain text or JSON to stdout. Suitable for scripting and CI.

**Prompt Template**
A markdown file with `{{variable}}` placeholders that can be expanded into a system prompt fragment. Distributed via Pi Packages.

**RPC mode**
A run mode (`pi --rpc`) that exposes the agent via JSON-RPC over stdio. Used for IDE and editor integration.

**SDK mode**
Using `@mariozechner/pi-coding-agent` as a TypeScript library in another program, rather than via the CLI. Provides `createAgentSession()` as the entry point.

**Session**
A persisted conversation: a unique ID, a list of `AgentMessage[]`, and optional branch snapshots. Stored on disk in `~/.pi/sessions/`.

**Skill**
A markdown file that describes a CLI command the LLM can invoke via the bash tool. Skills are auto-discovered from `~/.pi/skills/` and from Pi Packages.

**Slash command**
A `/command` registered by an extension and available in the TUI's autocomplete. Slash commands run extension logic rather than sending a prompt to the LLM.

**Working directory**
The directory that `pi` treats as the root for all file operations. Defaults to the directory where `pi` was launched. Tools like `bash`, `read`, and `find` operate relative to this path.

---

**Back to:** [README](./README.md) | [Global Glossary](../../glossary.md)
