# Feature: Self-Extending Tools

## Learning Objectives

- Understand how mom installs new CLI tools autonomously at runtime.
- Trace the tool installation flow from LLM decision to first use.
- Know the security model and isolation of installed tools.

---

## Background

mom ships with a minimal fixed toolset (bash, read, write, edit, attach, truncate). When a task requires a tool that does not exist on the system, mom can install it herself — using `bash` to call `npm install`, `apt-get`, or `apk add` — without any operator intervention.

This is not a formal plugin API: mom simply has permission to run arbitrary bash commands in her workspace, and she autonomously chooses to install tools when she needs them.

---

## Installation Flow

1. **User asks mom to do something that requires `jq`.**
2. **LLM calls `bash` with `which jq` to check existence.** Returns empty / error.
3. **LLM calls `bash` with `npm install -g jq` or `apt-get install -y jq`.**
4. **LLM calls `bash` again with `jq --version` to confirm installation.**
5. **Subsequent calls use `jq` as part of commands.**
6. **mom writes a note to `memory.md`:** "Installed: jq (apt)."

---

## npm-Based Tool Installation

For Node.js tools, mom installs into the channel workspace:

```bash
# LLM-generated bash command
cd /workspace/C08ABCDEF/tools && npm install node-fetch
```

Installed packages live in `workspace/{channelId}/tools/node_modules/`.

---

## Security Considerations

- With `--sandbox=host`, mom runs as the host user and can install system packages. This is appropriate for single-user development machines.
- With `--sandbox=docker`, installations happen inside the Docker container and do not affect the host system. The container is rebuilt or reset when you restart mom.
- **Audit `memory.md`** — mom records what she installs. Review this file periodically.

---

## Persisting Installed Tools

Because `workspace/{channelId}/tools/` is on the host file system, npm packages installed in one session are available in all subsequent sessions for the same channel. System packages installed with `apt` survive only if you are using a persistent Docker volume or the host system.

---

## Check-Yourself Questions

1. What mechanism does mom use to install new tools — is there a formal plugin API?
2. Where does mom record what tools she has installed?
3. What is the difference in tool persistence between `--sandbox=host` and `--sandbox=docker`?
4. How could you pre-install tools so mom never needs to install them at runtime?
5. What is the risk of letting mom run `apt-get install` on a shared server?
