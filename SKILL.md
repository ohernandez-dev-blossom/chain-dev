---
name: chain-dev
description: Implement the Chain Dev Pattern for autonomous self-chaining AI development sprints. Use when setting up a new project with continuous autonomous dev agents, replacing fixed-interval cron sprints with event-driven chains, or when asked about "chain dev", "chained sprints", "autonomous dev loop", or "self-triggering agents". Covers dispatcher cron setup, chain file protocol, semaphore config, and agent task templates.
---

# Chain Dev Pattern

Autonomous, event-driven dev sprints where each agent triggers the next. No idle polling — only runs when there's real work.

## Core Concept

```
Dispatcher cron (every 5 min, near-zero cost)
  └── /tmp/chain-{project}.txt exists?
      ├── No  → exit silently
      └── Yes → delete file → spawn dev agent
                    └── Agent works → writes chain file → next sprint in ≤5 min
```

## Setup for a New Project

### 1. Create dispatcher cron

```json
{
  "name": "Chain Dispatcher — {Project}",
  "schedule": { "kind": "cron", "expr": "*/5 * * * *" },
  "sessionTarget": "main",
  "payload": {
    "kind": "systemEvent",
    "text": "Chain Dev Dispatcher for {Project}: check /tmp/chain-{project}.txt. If it exists, read its contents, delete the file, check semaphore (max 3 concurrent dev agents via `openclaw subagents list`), then spawn a dev agent with the chain message as context. If file does not exist, do nothing."
  }
}
```

### 2. Create chain-next.sh for the project

```bash
#!/bin/bash
# Usage: chain-next.sh "Sprint summary. Next: issue #N"
echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] ${1:-Sprint complete.}" > /tmp/chain-{project}.txt
```

### 3. Define the dev agent task template

Include in every spawned agent's task:
- Repo path and branch
- How to pick next issue (label: `po-ready`, sorted by priority)  
- Quality gates: `tsc --noEmit && npm run lint`
- **Mandatory final step**: write chain file or stop if backlog empty

See `references/agent-task-template.md` for the full template.

### 4. Kick off

```bash
echo "Initial kickoff." > /tmp/chain-{project}.txt
```

### 5. Stop

```bash
rm /tmp/chain-{project}.txt
# + disable dispatcher cron if full stop needed
```

## Chain File Protocol

- **Location:** `/tmp/chain-{project}.txt` (ephemeral — lost on reboot)
- **Written by:** dev agent at end of each sprint
- **Read by:** dispatcher cron
- **Content:** plain text summary — what was done, what's next
- **State vs context:** chain file is *context only* — real state lives in GitHub issues/DB

## Semaphore (concurrency guard)

```bash
MAX=3
ACTIVE=$(openclaw subagents list 2>/dev/null | grep -c running || echo 0)
[ "$ACTIVE" -ge "$MAX" ] && echo "Full ($ACTIVE/$MAX). Skip." && exit 0
```

## Operational Commands

```bash
# Start chain
echo "Sprint start." > /tmp/chain-myproject.txt

# Check active chains
ls /tmp/chain-*.txt 2>/dev/null

# Stop chain
rm /tmp/chain-myproject.txt

# Debug: last agent activity
tail -50 ~/clawd/logs/agent-activity.md
```

## When the Agent Should NOT Write the Chain File

- Backlog is empty
- Repeated failures on the same issue (>2 attempts)
- TSC errors that can't be fixed in one sprint
- Manual stop requested

In these cases, the chain stops naturally — no file, no next sprint.

## References

- `references/agent-task-template.md` — Full dev agent task template to copy/adapt per project
- `references/pattern-overview.md` — Detailed explanation with examples and comparison vs fixed crons
