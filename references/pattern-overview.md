# Chain Dev Pattern

> Autonomous, self-chaining development sprints for AI agents.  
> No wasted compute. No idle polling. Just work when there's work.

---

## The Problem with Fixed Dev Sprints

Running a dev agent on a fixed timer (e.g., every 15 min) has a fundamental flaw: **it runs even when there's nothing to do**. You burn tokens, spin up containers, and generate noise — just to find an empty backlog.

Chain Dev solves this.

---

## How It Works

Instead of a clock triggering agents, **agents trigger each other**.

Each agent, when it finishes its work, writes a small file signaling "I'm done, here's what's next." A lightweight dispatcher cron checks for that file every 5 minutes. If it exists → consume it → spawn the next agent immediately.

```
Dispatcher (cron, every 5 min)
  └── Check /tmp/chain-{project}.txt
      ├── Not found → exit silently (near-zero cost)
      └── Found → delete file → spawn dev agent
                      └── Agent works:
                          1. Merge pending PRs
                          2. Post-merge QA
                          3. Pick next issue from backlog
                          4. Implement
                          5. Type-check
                          6. Open PR
                          └── Write /tmp/chain-{project}.txt
                              └── Dispatcher picks it up in ≤5 min
```

---

## Components

### 1. Dispatcher Cron
A minimal cron job running every 5 minutes. Its only job:

```bash
CHAIN_FILE="/tmp/chain-myproject.txt"

if [ -f "$CHAIN_FILE" ]; then
  MESSAGE=$(cat "$CHAIN_FILE")
  rm "$CHAIN_FILE"
  # spawn dev agent with $MESSAGE as context
  openclaw agent spawn dev --task "Continue sprint. Context: $MESSAGE"
fi
```

Cost when idle: essentially zero — just a file check.

### 2. Chain File
`/tmp/chain-{project}.txt`

A plain text file written by the agent at the end of each sprint. Contains a summary of what was done and what's next — this becomes the context for the next agent.

Example content:
```
[2026-03-23T14:30Z] Merged PR #45 (fix auth flow), PR #46 (add map markers).
Next: issue #47 (SEO metadata for wiki pages), issue #48 (fix mobile nav).
TSC: 0 errors. Build: passing.
```

### 3. chain-next.sh
A simple script agents call at the end of their task:

```bash
#!/bin/bash
# Usage: ./chain-next.sh "Sprint summary and what's next"
MESSAGE="${1:-Sprint complete.}"
echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] $MESSAGE" > "/tmp/chain-${PROJECT}.txt"
echo "✅ Chain queued. Next sprint will fire in ≤5 min."
```

Every agent task **must end** with this call. If the agent decides there's nothing left to do, it simply doesn't write the file — the chain stops naturally.

### 4. Concurrency Semaphore (optional)
To prevent runaway parallel sprints:

```bash
# Check before spawning
MAX_AGENTS=3
ACTIVE=$(openclaw subagents list | grep -c running)

if [ "$ACTIVE" -ge "$MAX_AGENTS" ]; then
  echo "Semaphore full ($ACTIVE/$MAX_AGENTS). Skipping spawn."
  exit 0
fi
```

---

## vs. Fixed Interval Crons

| | Fixed Cron (every 15 min) | Chain Dev |
|---|---|---|
| Runs when backlog is empty | ✅ Always | ❌ Never |
| Latency between sprints | Up to 15 min | ≤5 min |
| Token cost at idle | High | ~0 |
| Context from previous sprint | ❌ None | ✅ Via chain file |
| Projects interfere with each other | Sometimes | Never |
| Stops automatically when done | ❌ No | ✅ Yes |

---

## Project Setup

### 1. Create the dispatcher cron

Using OpenClaw (or any cron system):

```json
{
  "name": "Chain Dispatcher — MyProject",
  "schedule": { "kind": "cron", "expr": "*/5 * * * *" },
  "payload": {
    "kind": "systemEvent",
    "text": "Check /tmp/chain-myproject.txt. If it exists, read it, delete it, and spawn a dev agent with its contents as context."
  },
  "sessionTarget": "main"
}
```

### 2. Define the agent task template

The task passed to each dev agent should include:
- Repo path and branch
- How to pick the next issue (label, milestone, etc.)
- Quality gates (tsc, lint, tests)
- **The mandatory final step:** call `chain-next.sh` with a summary

Example task template:
```
You are a dev agent for MyProject.

Repo: ~/dev/myproject (branch: main)
Backlog: GitHub issues with label "po-ready", sorted by priority

Your sprint:
1. Merge any open PRs that are approved and green
2. Pick the top unassigned issue
3. Implement it following the patterns in src/
4. Run: tsc --noEmit && npm run lint
5. Open a PR

When done (success or backlog empty), run:
  ~/scripts/chain-next.sh "Merged: #N. Implemented: #M. Next: #O, #P. TSC: 0 errors."

If backlog is empty:
  ~/scripts/chain-next.sh "Backlog empty. No more issues to pick."
  (The dispatcher will stop the chain when it sees this message — or you can simply not write the file.)
```

### 3. Kick off the chain

```bash
echo "Initial kickoff." > /tmp/chain-myproject.txt
```

The dispatcher will pick it up within 5 minutes and the first sprint fires.

### 4. Stop the chain

```bash
rm /tmp/chain-myproject.txt
# Also disable the dispatcher cron if you want a full stop
```

Or: instruct your agent to not write the chain file when the backlog is empty.

---

## Operational Notes

**Resuming after a pause:**
```bash
echo "Resuming after pause. Check backlog from scratch." > /tmp/chain-myproject.txt
```

**Multiple projects run fully independently** — each has its own chain file, its own dispatcher cron. No cross-project interference.

**The chain file is ephemeral** — it lives in `/tmp`. On server restart it's gone. Design your agents to work from backlog state (GitHub issues, a DB queue, etc.), not from the chain file alone. The chain file is *context*, not *state*.

**Debugging a stuck chain:**
```bash
# Is there a file waiting?
ls /tmp/chain-*.txt

# Is the dispatcher cron enabled?
openclaw cron list

# What did the last agent do?
cat ~/logs/agent-activity.md | tail -50
```

---

## Example: Full Flow

```
Monday 09:00 — Omar kicks off chain:
  echo "Sprint start. Focus: issues #34, #35, #36" > /tmp/chain-casave.txt

09:05 — Dispatcher fires. Reads file, deletes it, spawns dev agent.

09:05–09:35 — Dev agent:
  - Merges PR #31 (approved, green CI)
  - Picks issue #34 (Add property filters)
  - Implements, opens PR #32
  - Writes: "Merged #31. Implemented #34 → PR #32. Next: #35, #36. TSC: 0 errors."

09:40 — Dispatcher fires again. Reads file, spawns next dev agent.

09:40–10:05 — Dev agent:
  - Merges PR #32 (just opened, auto-approved)
  - Picks issue #35
  - ...

(Continues autonomously until backlog is empty or chain is stopped)
```

---

## Summary

Chain Dev is the pattern you want when:
- You have a persistent issue backlog
- You want autonomous dev agents that don't burn resources at idle
- You want sprints to chain immediately without waiting for a fixed timer
- You want each sprint to have context from the previous one

The invariant is simple: **if there's work, the chain runs. If there isn't, it stops.**

---

*Pattern by TARS / HernandezBastos — 2026*
