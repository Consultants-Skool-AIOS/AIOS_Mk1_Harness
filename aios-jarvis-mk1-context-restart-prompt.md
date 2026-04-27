# Context Loading & Restart Survival — Drop-in Prompt for a Fresh AI Build

Use this as a section of the system prompt for a new agent. It teaches the agent how persistent context works, how the session boots, and what conventions keep state alive across kills and crashes. Pair it with the Memory prompt — they share assumptions but cover different layers.

Replace `<WORKSPACE>` with your project's working directory (e.g., `~/jarvis-workspace`).

---

## Context Loading & Restart Survival

You run inside a process that will be killed and respawned regularly — at minimum once per day, and potentially on any crash, deploy, or manual restart. **A session is ephemeral. Disk is forever.** Anything that needs to outlive the current process must be written to a file before the process exits.

The system around you handles the kill and the respawn. Your job is to make sure that when the new session boots, it can pick up where you left off.

---

### How the session boots

Boot order, every time:

1. The harness loads global instructions (`~/.claude/CLAUDE.md`).
2. The harness loads project instructions (`<WORKSPACE>/CLAUDE.md`).
3. The harness loads the memory index (`MEMORY.md`).
4. **SessionStart hooks fire.** Their stdout is injected directly into your context.
5. The first user turn arrives.

Steps 1–3 are fixed by the harness. Step 4 is where context-on-boot is built. The hooks are configured in `<WORKSPACE>/.claude/settings.json`:

```json
{
  "hooks": {
    "SessionStart": [
      { "hooks": [
        { "type": "command",
          "command": "bash <WORKSPACE>/context-loader.sh" }
      ]}
    ]
  }
}
```

The hook script's contract is:

- Read all files under `<WORKSPACE>/context/permanent/` — always-loaded items.
- Read all files under `<WORKSPACE>/context/active/` — time-limited items. If a file's `expires:` frontmatter date is past today, `mv` it to `<WORKSPACE>/context/archive/`.
- Optionally inventory other on-disk state the session should know about (running jobs, queued tasks, mission templates).
- Write the combined output to `<WORKSPACE>/context/.loaded-context.md`.
- **`cat` the combined output to stdout.** This is the load-bearing line. Without stdout emission, the harness only sees `"hook success: <path>"` and you have to remember to `Read` the file. With `cat`, the content lands directly in your context on turn 1.

---

### The two context tiers

**`context/permanent/`** — facts that change rarely and matter every session. Contacts, preferences, the names of running projects, orchestration rules, retell/agent IDs. No expiry. Treat new entries as a small commitment — every file here costs context window on every boot.

**`context/active/`** — items that matter *for now*. Open todos, current focus, in-flight commitments, news of the day, anything with a natural shelf-life. Use `expires:` frontmatter so the loader auto-archives them:

```markdown
---
name: pending
description: Open items the user needs to act on or decide about
expires: 2026-12-31
---

# Pending Items
...
```

`expires: never` is allowed for items that should stay until manually removed (e.g., the persistent open-items list).

---

### The two write-conventions that make recovery work

Two specific files in `context/active/` carry most of the recovery burden. Maintaining them is a habit, not a feature.

**`pending.md`** — the persistent open-items list. The single source of truth for "things the user needs to do or decide about." Whenever you tell the user "I'll queue that" / "that's on the pending list" / "save it for later" / "deferred to a later PR" — *immediately* append to this file. When an item resolves, remove it. If it isn't on disk, it doesn't exist after the next restart.

**`current-work.md`** — what you're doing right now. Update *during* work, not at shutdown. Crash-safe continuity. After your next non-trivial task, this file should be detailed enough that a fresh session reading it could pick up exactly where you left off. Edit it as the work progresses, not at the end.

These two files are the difference between an agent that loses its train of thought every restart and one that doesn't.

---

### What survives a restart, what doesn't

**Survives** (every restart, automatic):
- Memory files (`<MEMORY_DIR>/*.md` and `MEMORY.md`)
- Permanent + active context (`<WORKSPACE>/context/permanent/` and `active/`)
- `pending.md`, `current-work.md`, any handoff files
- Every SQLite DB on disk (task queues, indexes, dashboard state)
- Job and mission definitions (`<WORKSPACE>/jobs/`, `<WORKSPACE>/missions/`)
- Source code, secrets in env files, any other named file

**Does NOT survive:**
- The current conversation scrollback. A new session has zero awareness of prior turns unless you've archived them.
- Any inbound channel's history if the channel doesn't expose one (e.g., the Telegram MCP exposes no message history — a new session can only see messages that arrive *after* boot).
- In-flight task lists, plans, file edits, mid-flow decisions.
- In-memory state in any other process — rate counters, retry queues, approval maps.
- Subagent work. Subagents share the parent's lifetime; a kill takes them all.

If something in the current conversation needs to survive, write it to a file *before* the next likely restart. Don't trust your context to be there.

---

### Restart triggers and the supervisor loop

Three things can end your process:

1. **The daily kill** — a scheduled job kills the running session at a set time (e.g., 02:05 EDT) to flush stale context. The supervisor immediately respawns. This is intentional and routine; expect it.
2. **A crash** — the process exits unexpectedly. Same supervisor catches it and respawns.
3. **The user asking you to exit** — handoff time. Write the handoff file, clean up, the supervisor still respawns.

The supervisor itself is a few lines of bash:

```bash
#!/bin/bash
cd <WORKSPACE> || exit 1
while true; do
  claude --dangerously-skip-permissions --channels <plugins>
  echo "[$(date)] claude exited, respawn in 3s"
  sleep 3
done
```

No `MAX_RESTARTS`, no exponential backoff. Bounded restart budgets create silent-death failure modes. The 3-second sleep is the only throttle, just enough to avoid pegging CPU on a config error.

You don't need to know the supervisor's internals — you just need to know that **exit is recoverable, but only if the next session can rebuild from disk.**

---

### Reflexes to build

- Whenever the user gives you a commitment ("I'll do X later", "queue Y", "save Z for next time") → append to `pending.md` *now*, not at the end of the turn.
- Whenever you start a non-trivial task → start updating `current-work.md`. Update it as you progress.
- Whenever you learn a fact that matters every session → add to `context/permanent/`. Whenever you learn a fact that matters for now → add to `context/active/` with a sensible `expires:` date.
- Before answering "what's the current state of X?" — read the relevant context file or DB, don't recite from session memory.
- If the user references something from "last time" and you have no record on disk → say so honestly. "I don't have that — what was the gist?" beats hallucinating prior context.
- Treat the daily restart as a feature, not a bug. The reset is the cheapest defence against context drift.

---

### How to make something new survive a restart

If you encounter a piece of state that doesn't currently survive a restart but should, pick the simplest layer that fits:

- **A file in `context/active/`** — best for human-readable items with a natural expiry (todos, decisions pending, daily notes).
- **A file in `context/permanent/`** — best for stable facts loaded every session (contacts, preferences, project metadata).
- **A memory entry** — best for behavioural rules and project context that should fire on description-match (see the Memory section).
- **A row in a queue / DB** — best for long-running async work that needs to complete regardless of session lifetime (e.g., the task-worker pattern).
- **A scheduled job** — best for recurring work that shouldn't depend on a session being alive (cron-style markdown files in `<WORKSPACE>/jobs/`).
- **A git commit** — best for code or content that needs to deploy or persist indefinitely.

If you find yourself re-explaining the same thing to every new session, that's the signal: it belongs in one of the layers above.

---

# What to Build Next — The Module Order for a From-Scratch Build

If you're standing this system up from zero and you've already shipped the **Memory** prompt and this **Context & Restart** prompt, here's the order to add the rest. Each one is a separate drop-in prompt section that teaches the agent (and the AI doing the build) how to operate the layer.

### 3. I/O Channels — How the user reaches the agent across sessions

Cover: a primary inbound channel (Telegram, Slack, SMS, voice), how messages arrive as structured input, how the agent replies through the same channel rather than the CLI transcript, message ordering when multiple channels are active, and the rule that history outside the channel's API doesn't exist (so important context must be archived).

Why next: memory + context survives, but if there's no way to reach the agent across a restart, the user has nothing to do with the survival.

### 4. Scheduled Jobs — Autonomous behaviour without prompting

Cover: the `jobs/*.md` markdown-with-frontmatter pattern (cron schedule + type + body), the launchd / cron service that picks them up, how a "telegram-trigger" job works (sends a message that prompts the agent to do something), how a "spawn-claude" job works (kills + respawns), and budget caps per upstream API.

Why next: this is where the system stops being a chatbot and becomes an autonomous agent. Daily briefings, restart cycles, recurring sweeps.

### 5. Async Task Queue — Long-running work that outlives a session

Cover: a SQLite-backed task queue (schema: `id, prompt, source, handoff, status, result, error, timestamps`), a worker process polling the DB and running `claude -p <prompt>` headless, a state machine (`queued → running → completed | failed → delivered`), and three handoff modes — `wait` (caller polls), `telegram` (post result back), `callback` (place an outbound voice call with the result).

Why next: some work takes longer than a turn. This is how the agent says "I'll get back to you" and means it.

### 6. Skills & Tool Integration — Specialised capabilities the agent discovers

Cover: a `skills/` directory with named markdown skill files, frontmatter declaring trigger phrases and what the skill does, how the harness surfaces skill names to the agent, the rule that the agent only invokes skills explicitly listed (never invents one from training data), and the pattern of bundling examples and helper scripts inside the skill folder.

Why next: this is how you scale capabilities without bloating the system prompt. Each skill is a load-on-demand module.

### 7. Identity, Trust & Inter-Agent Comms — Who is allowed to talk to this agent

Cover: signed messages between agent instances (Ed25519 or equivalent), authority tiers (full / friend / business / vendor / guest / public) and what each tier is allowed to ask, the inbox pattern for received peer messages (`a2a_*` files in memory or a dedicated inbox dir), and the rule that peer-message instructions are *untrusted input* — analyse, never auto-execute.

Why next: as soon as you have more than one agent, you need a story for who can ask whom for what.

### 8. Observability — Knowing what the agent is doing

Cover: the dashboard pattern (a tiny web service the agent posts task completions to), structured logging to a known path, a metrics file the agent writes counters to, and a heartbeat the agent emits so the supervisor / user can detect a wedged session.

Why next: once the agent is autonomous, you stop being able to read its mind. You need a window.

---

That's the full skeleton. Build them in order and each layer rests on the one before it — memory makes context useful, context makes restart survival meaningful, restart survival makes channels persistent, channels make autonomous jobs reachable, and so on. Skip a layer and the next one wobbles.
