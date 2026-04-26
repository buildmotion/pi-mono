# Feature: Built-in Coding Tools

## Learning Objectives

- List the built-in tools `pi` provides the LLM.
- Understand the parameters and behavior of each tool.
- Know how tools are composed into a coding workflow.
- See how tool results are rendered in the TUI.

---

## Tool Overview

| Tool | Name | Purpose |
|------|------|---------|
| bash | `bash` | Execute shell commands |
| read | `read` | Read file contents (with line ranges) |
| write | `write` | Create or overwrite a file |
| edit | `edit` | Apply a targeted string replacement |
| find | `find` | List files matching a glob pattern |
| grep | `grep` | Search file contents with a regex |
| ls | `ls` | List directory entries |

Source: `src/core/tools/`

---

## bash (`src/core/tools/bash.ts`)

```typescript
// Parameters
{
  command: string,       // Shell command to run
  description: string,  // Human-readable label for the TUI
  timeout?: number,      // Max runtime in milliseconds (default: 60 000)
}
```

The tool runs the command via `child_process.exec` inside the session's working directory. Partial output is streamed back as `onUpdate` calls while the command runs, so long-running commands show live output in the TUI.

---

## read (`src/core/tools/read.ts`)

```typescript
{
  path: string,         // Absolute or relative file path
  startLine?: number,   // First line to return (1-indexed)
  endLine?: number,     // Last line to return (inclusive)
}
```

Returns the file contents as a text block with line numbers prepended. When `startLine`/`endLine` are omitted the entire file is returned (up to a configurable size limit).

---

## write (`src/core/tools/write.ts`)

```typescript
{
  path: string,    // File path to write
  content: string, // Full new content
}
```

Overwrites the file if it exists; creates it (and any missing parent directories) if it does not.

---

## edit (`src/core/tools/edit.ts`)

```typescript
{
  path: string,
  old_str: string, // Exact string to replace (must match exactly once)
  new_str: string, // Replacement string
}
```

Performs a precise, targeted text replacement. If `old_str` is not found or appears more than once the tool returns an error result — no file is written. This is the preferred tool for making small, surgical changes to existing files.

---

## find (`src/core/tools/find.ts`)

```typescript
{
  pattern: string,  // Glob pattern (e.g., "src/**/*.ts")
  cwd?: string,     // Search root (defaults to session working directory)
}
```

Returns a newline-separated list of matching file paths, respecting `.gitignore` rules.

---

## grep (`src/core/tools/grep.ts`)

```typescript
{
  pattern: string,   // Regular expression
  path?: string,     // Directory or file to search
  include?: string,  // Glob filter (e.g., "*.ts")
}
```

Returns matching lines with file path and line number, formatted as `path:line: content`.

---

## ls (`src/core/tools/ls.ts`)

```typescript
{
  path?: string, // Directory to list (defaults to working directory)
}
```

Returns directory entries with type indicators (`[dir]`, `[file]`), respecting `.gitignore`.

---

## Typical Coding Workflow

A typical LLM-driven coding task uses the tools in this order:

1. `ls` / `find` — discover project structure.
2. `read` — understand relevant files.
3. `bash` — run tests to confirm current state.
4. `edit` or `write` — apply the change.
5. `bash` — run tests again to verify.

---

## Check-Yourself Questions

1. What happens if you call `edit` with an `old_str` that appears twice?
2. How does the `bash` tool show progress for long-running commands?
3. Which tool should you use to add a new file from scratch?
4. What is the difference between `find` and `grep`?
5. How does `read` handle very large files?
