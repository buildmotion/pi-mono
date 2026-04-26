# Feature: Extensions, Skills, and Pi Packages

## Learning Objectives

- Understand the three extension primitives: TypeScript Extensions, Skills, and Prompt Templates.
- Write a minimal TypeScript Extension that adds a custom tool.
- Define a Skill that the LLM can invoke as a CLI command.
- Package and distribute extensions as a Pi Package via npm or git.

---

## Background

`pi` is designed to be extended without forking. The extension system (`src/core/extensions/`) provides:

- **TypeScript Extensions** — modules that plug into the agent lifecycle.
- **Skills** — markdown-described CLI commands the LLM can call via the `bash` tool.
- **Prompt Templates** — reusable system prompt fragments with `{{variable}}` substitution.
- **Pi Packages** — npm packages or git repos that bundle any of the above.

---

## TypeScript Extensions

An extension is a TypeScript module that exports a factory function. It receives an `ExtensionAPI` object with access to the session, agent events, tool registration, and UI primitives.

```typescript
// my-extension/index.ts
import type { Extension } from "@mariozechner/pi-coding-agent";
import { Type } from "typebox";

export default {
  name: "my-extension",
  version: "1.0.0",

  async onSessionStart(api) {
    // Register a custom tool
    api.registerTool({
      name: "list_todos",
      label: "List TODOs",
      description: "Lists all TODO comments in the project.",
      parameters: Type.Object({}),
      execute: async () => {
        const result = await api.exec("grep -r 'TODO' src/ --include='*.ts'");
        return { content: [{ type: "text", text: result.stdout }], details: {} };
      },
    });
  },

  async onTurnEnd(api, event) {
    // Called after every assistant turn
    api.log("Turn complete. Tokens used:", event.usage?.totalTokens);
  },
} satisfies Extension;
```

Install by adding to `~/.pi/extensions.json` or via a Pi Package.

---

## Skills

A Skill is a markdown file that describes a CLI command the LLM can call. pi loads Skills from `~/.pi/skills/` and from Pi Packages.

```markdown
<!-- ~/.pi/skills/run-tests.md -->
---
name: run_tests
description: Runs the project test suite and returns the result.
command: npm test
---

Use this skill to run the project's test suite. It executes `npm test` in the working directory and returns the output.
```

The LLM sees the skill name and description. When it calls `bash` with the skill's command, the output is returned as a tool result.

---

## Prompt Templates

Templates are markdown files with `{{variable}}` placeholders:

```markdown
<!-- ~/.pi/templates/code-review.md -->
You are performing a code review on the following diff:

{{diff}}

Focus on: correctness, security, performance.
```

Use with: `pi --template code-review --var diff="$(git diff HEAD~1)"`

---

## Pi Packages

A Pi Package is an npm package (or git repo) with a `pi.json` manifest:

```json
{
  "name": "pi-package-my-tools",
  "version": "1.0.0",
  "extensions": ["./dist/my-extension/index.js"],
  "skills": ["./skills/"],
  "templates": ["./templates/"]
}
```

Install: `pi package install pi-package-my-tools`

This installs the npm package and registers its extensions, skills, and templates with pi.

---

## Check-Yourself Questions

1. What is the difference between an Extension and a Skill?
2. Which lifecycle hook fires when a new session starts?
3. How does `api.registerTool()` differ from adding a tool directly to `agent.state.tools`?
4. What file format is used for Skills?
5. How would you share a custom extension with your team?
