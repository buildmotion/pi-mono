# Feature: Docker Sandbox

## Learning Objectives

- Configure mom to run bash commands inside a Docker container.
- Understand what the Docker sandbox isolates and what it does not.
- Know the tradeoffs between `--sandbox=host` and `--sandbox=docker`.

---

## Background

By default, mom executes bash commands directly on the host machine (`--sandbox=host`). This is convenient for development but risky in production: the agent has the same file-system and network permissions as the user running mom.

The `--sandbox=docker` mode routes all bash execution through a Docker container, providing OS-level isolation. Source: `src/sandbox.ts`.

---

## Enabling Docker Sandbox

```bash
# Requires Docker to be running and the user to have access to the Docker socket
npx @mariozechner/pi-mom /workspace --sandbox=docker
```

mom creates (or reuses) a Docker container on startup. All subsequent `bash` tool calls are routed to `docker exec` on that container.

---

## What Is Isolated

| Resource | `--sandbox=host` | `--sandbox=docker` |
|----------|------------------|--------------------|
| File system (workspace) | Shared | Volume-mounted (`/workspace`) |
| File system (rest of host) | Full access | Not accessible |
| Network | Full access | Container networking (configurable) |
| Installed packages | Host packages | Container image packages |
| Process namespace | Host PIDs | Container PIDs |

---

## How `SandboxExecutor` Works (`src/sandbox.ts`)

```typescript
// src/sandbox.ts (simplified)
export async function createExecutor(config: SandboxConfig) {
  if (config.type === "docker") {
    const containerId = await ensureContainer(config);
    return {
      exec: (command: string) => dockerExec(containerId, command),
    };
  }
  return {
    exec: (command: string) => hostExec(command),
  };
}
```

`hostExec` uses `child_process.exec`; `dockerExec` uses `docker exec -i <containerId> sh -c <command>`.

---

## Volume Mounts

The workspace directory is mounted into the container so mom's files are accessible:

```
docker run -v /your/workspace:/workspace ...
```

Files written inside the container at `/workspace` appear immediately on the host and survive container restarts.

---

## Security Tradeoffs

- `--sandbox=docker` prevents the agent from accessing host files outside the workspace, but does NOT prevent network access by default.
- The Docker socket itself is a root-equivalent capability. Ensure the user running mom does not have unrestricted Docker access in production.
- For highest security, combine `--sandbox=docker` with a restricted container image and a network policy that blocks outbound connections except to the LLM API.

---

## Check-Yourself Questions

1. Which bash execution backend does `--sandbox=host` use?
2. How does `dockerExec` pass a command to the container?
3. What is volume-mounted into the container so mom's workspace is accessible?
4. Does `--sandbox=docker` prevent network access? Why or why not?
5. When would you prefer `--sandbox=host` over `--sandbox=docker`?
