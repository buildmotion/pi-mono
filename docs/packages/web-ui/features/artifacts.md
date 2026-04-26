# Feature: Artifact Rendering (Sandboxed iframe)

## Learning Objectives

- Understand what an "artifact" is in `pi-web-ui` and how it is produced.
- Explain the security model of the sandboxed iframe.
- Write a custom tool that returns an artifact for the panel to display.
- Know which content types are supported.

---

## Background

An **artifact** is a self-contained piece of content — HTML, SVG, Markdown, or a script output — that a tool returns and that the UI renders visually in a side panel. The most common source is the built-in `javascript_repl` tool, which can produce charts, tables, or any interactive HTML.

Source: `src/components/SandboxedIframe.ts`, `src/tools/artifacts/`

---

## How Artifacts Are Produced

1. The LLM calls the `javascript_repl` tool with a JavaScript snippet.
2. The tool executes the snippet in a worker and captures `console.log` output plus any returned HTML string.
3. The tool result's `details.html` field contains the HTML to render.
4. `ChatPanel` detects this field and posts the HTML to the `SandboxedIframe` as `srcdoc`.

```typescript
// src/tools/javascript-repl.ts (simplified)
execute: async (_id, { code }) => {
  const result = await runInWorker(code);
  return {
    content: [{ type: "text", text: result.stdout }],
    details: { html: result.html },
  };
},
```

---

## Supported Content Types

| Type | How it renders |
|------|---------------|
| `text/html` | Full HTML document or fragment in sandboxed iframe |
| `image/svg+xml` | SVG rendered inline or in iframe |
| `text/markdown` | Markdown string rendered via the Markdown component |
| Plain text | Shown in a `<pre>` block inside the artifact panel |

---

## Security Model

The `SandboxedIframe` element sets:

```html
<iframe sandbox="allow-scripts" srcdoc="..."></iframe>
```

`allow-scripts` is present — JavaScript inside the artifact runs. The following are deliberately **absent**:

| Missing permission | Effect |
|---|---|
| `allow-same-origin` | No access to parent page's DOM, cookies, or storage |
| `allow-forms` | Form submissions blocked |
| `allow-top-navigation` | Cannot navigate the parent page |

This is appropriate for AI-generated visualizations but means the artifact cannot make authenticated API calls or access the host page's state.

---

## Writing a Custom Tool That Produces an Artifact

```typescript
import { Type } from "typebox";

const chartTool = {
  name: "bar_chart",
  label: "Bar Chart",
  description: "Renders a bar chart from data.",
  parameters: Type.Object({
    labels: Type.Array(Type.String()),
    values: Type.Array(Type.Number()),
  }),
  execute: async (_id, { labels, values }) => {
    const html = `<!doctype html>
<html><body>
<canvas id="c"></canvas>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<script>
new Chart(document.getElementById('c'), {
  type: 'bar',
  data: { labels: ${JSON.stringify(labels)}, datasets: [{ data: ${JSON.stringify(values)} }] }
});
</script>
</body></html>`;

    return {
      content: [{ type: "text", text: "Chart rendered." }],
      details: { html },
    };
  },
};
```

Attach this tool to the agent and `ChatPanel` will automatically render the returned HTML in the artifacts panel.

---

## Check-Yourself Questions

1. What `sandbox` attribute value does `SandboxedIframe` use and what does it permit?
2. Which field in `AgentToolResult.details` triggers artifact rendering in `ChatPanel`?
3. Can an artifact make a `fetch()` request to the LLM API? Why or why not?
4. What is the difference between `content` and `details` in `AgentToolResult`?
5. Name two content types that the artifacts panel can render besides HTML.

---

**What's Next:** [Extension Points](../extension-points.md) | [Observability](../observability.md)
