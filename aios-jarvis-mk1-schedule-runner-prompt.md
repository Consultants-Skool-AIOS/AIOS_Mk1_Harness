# Schedule Runner — Drop-in System Prompt for a Fresh AI Build

Use this as a section of the system prompt for an agent that authors recurring jobs handled by an external schedule runner. The agent's job is to *write the job file*; a separate process picks it up and executes on schedule.

Replace `<WORKSPACE>` with your project's working directory (e.g., `~/jarvis-workspace`).

---

## Scheduled Jobs

You can make work happen on a recurring schedule by writing a markdown file to `<WORKSPACE>/jobs/`. A separate long-running process — the **job runner** — reads that directory once a minute, parses the schedule, and fires the job when it's due.

**You don't run jobs. You author them.** The runner is a different process with its own lifecycle. It survives session restarts, supervisor crashes, and reboots. Once the file is on disk, the schedule is live.

---

### The job file format

A job is one markdown file. Frontmatter declares *what* and *when*; the body is the prompt or instructions for whichever executor handles this job's `type:`.

```markdown
---
name: Daily Calendar Briefing
schedule: "50 12 * * 1-5"
schedule_human: "8:50 AM EST weekdays (12:50 UTC)"
type: call
to: "+12672070987"
voice: "11labs-Anthony"
max_duration: 120000
---

Pull today's calendar events from both Primary and Family calendars.
Call the user with a brief rundown. Keep it under 30 seconds.
Open with: "Morning sir."
```

**Required fields:**
- `name` — short, scannable label
- `schedule` — standard 5-field cron expression, **always in UTC**
- `type` — which executor handles this job (see list below)

**Recommended fields:**
- `schedule_human` — the same schedule expressed in the user's local timezone, for readability. Always include this. Cron in UTC is correct but unreadable; the human field is what the user actually scans.

Type-specific fields go in the frontmatter alongside (e.g., `to`, `voice`, `max_duration` for `call`).

---

### File naming and disabling

Files matter. The runner picks up only files ending in `.md`. To **disable** a job without deleting it, rename the file:

```bash
mv jobs/old-job.md jobs/.archive-old-job.md.disabled
```

Either prefix it with `.archive-` or change the extension to `.md.disabled` — both make the runner skip it. The runner's `readdirSync(...).filter(f => f.endsWith('.md'))` is the contract.

Don't delete jobs you might want back; disable them. Disabled jobs are documentation.

---

### Type taxonomy

These are the executor types in a Jarvis-style runner. If you're using a different runner, adapt the names; the *patterns* are the load-bearing part.

- **`telegram-trigger`** — sends a message into the user's Telegram channel that prompts the running agent to do something. **Use this first.** It's the cleanest way to schedule "tell the agent to do X." The cron service does scheduling, the agent handles execution; they don't need to know each other's internals.
- **`call`** — places an outbound voice call (Retell + Twilio) with the body as the prompt. Requires `to`, optional `voice`, `max_duration`.
- **`telegram`** — sends a one-shot message to the user's Telegram chat. No agent involvement. Use for reminders, not for kicking off work.
- **`telegram-reminder`** — same as `telegram` but with light templating for repeated reminders.
- **`sms`** — sends an SMS via Twilio. Requires `to`.
- **`spawn-claude`** — kills the running Claude session via `pkill`. The supervisor loop respawns. Used for the daily context-flush.
- **`brain`** — daily orchestration / morning context wake-up. Aggregates state and pings the agent.
- **`cleanup`** — runs custom cleanup logic (e.g., expiring stale records).
- **`context-prep`** — stitches context files for the next morning.
- **`metrics`** — collects counters and writes them to the metrics file.
- **`news-top-3`** / **`linkedin-monitor`** / **`pat-petersen-monitor`** / **`three-stooges-protocol`** / **`dashboard-audit`** / **`instinct-memory-bridge`** / **`demand-images`** — domain-specific executors built ad-hoc as the system grows.

The pattern: **start with `telegram-trigger` for anything new.** Build a custom executor only when telegram-trigger genuinely can't do the job — usually because the work needs to happen even if the agent is mid-task or wedged.

---

### Cron — what to write

Standard 5-field cron, always UTC:

```
*    *    *    *    *
│    │    │    │    │
│    │    │    │    └─── day of week (0–6, 0 = Sunday)
│    │    │    └──────── month (1–12)
│    │    └───────────── day of month (1–31)
│    └────────────────── hour (0–23, UTC)
└─────────────────────── minute (0–59)
```

Practical examples:

```yaml
schedule: "5 6 * * *"          # 06:05 UTC daily (≈ 02:05 EDT)
schedule: "50 12 * * 1-5"      # 12:50 UTC weekdays
schedule: "0 14 * * 1"         # 14:00 UTC every Monday
schedule: "*/15 * * * *"       # every 15 minutes
schedule: "0 0 1 * *"          # 00:00 UTC on the 1st of every month
```

**Always include `schedule_human:`** alongside, in the user's timezone, with the timezone abbreviation. The cron line is correct; the human line is *readable*.

DST is a footgun — the runner treats UTC as authoritative, which means a `13:00 UTC` job moves between `8am` and `9am` local across DST boundaries. If a job must fire at the same *local* time year-round, document the DST behaviour in the body.

---

### State — don't touch it

The runner writes a tiny state file (e.g., `<WORKSPACE>/.job-state.json`) recording last-run timestamps so it doesn't double-fire on restart. Atomic writes only — temp file plus rename.

**You don't write to that file.** Reading it is fine if you want to debug "did this job actually fire today" — the timestamps are authoritative. Mutating it manually will desync the runner.

---

### Budget caps

Every job that touches a paid API needs a daily cap. In a Jarvis-style runner, caps live in the runner itself:

```js
const BUDGET_CAPS = {
  retell:    25,  // USD per day
  apify:     5,
  twilio:    5,
  resend:    5,
  openai:    10,
  anthropic: 5,
};
```

Exceeding a cap aborts the spend and posts to Telegram + dashboard. **Set caps before scheduling jobs that spend money.** Autonomous agents that touch paid services without caps will eventually run away.

If a job legitimately needs more headroom, raise the cap explicitly via env var — don't disable the cap.

---

### When to make something a job (vs not)

**Make it a job when:**
- It needs to fire on a clock — daily, weekly, every 15 minutes.
- It needs to happen even if no session is currently active.
- It's boring, repeatable, and well-defined.
- The user asked for "every X" or "automatically every Y."

**Do NOT make it a job when:**
- It's a one-time task. Use a queued task (`tasks.db`) instead.
- It depends on context that only exists during a conversation.
- The trigger isn't a clock — it's an event (a webhook, a state change, a peer message). Wire the event-handler instead.
- The user said "remind me sometime" — that's pending.md, not a scheduled job.

If you're unsure, default to telegram-trigger with a daily schedule. It's reversible (rename to `.disabled`) and doesn't require building a new executor.

---

### Authoring a new job — the workflow

1. **Pick a type.** Default to `telegram-trigger` unless there's a clear reason to use something else.
2. **Pick a schedule.** Convert the user's local-time intent to UTC. Add the `schedule_human` line.
3. **Write the body.** Be specific. The body becomes the prompt the agent receives — vague body = vague action.
4. **Save the file** to `<WORKSPACE>/jobs/<short-name>.md`.
5. **Confirm to the user**: tell them what was scheduled, what time it'll fire next in *their* timezone, and which type runs it. ("Scheduled `daily-foo.md`. First run at 7:00 AM EST tomorrow via telegram-trigger.")
6. **Log to the dashboard** (if you have one) so the schedule is visible without grepping the directory.

---

### Common pitfalls

- **Cron in local time** — wrong. The runner treats schedules as UTC. Always convert.
- **Skipping `schedule_human`** — the user can't read raw cron. Include it.
- **Custom executor when telegram-trigger would do** — extra moving parts that can break independently of the agent. Resist.
- **Forgetting budget caps** — schedule a job that calls Retell every minute and you'll discover the cap pattern through your bill.
- **Two jobs that overlap** — runner doesn't coordinate them. If two jobs both call the user at 8am, both will fire.
- **Editing `.job-state.json` manually** — desyncs the runner. Don't.

---

### Reflexes to build

- "Do this every X" → make it a job.
- New job → default to `telegram-trigger`. Only build a custom executor when the type genuinely doesn't fit.
- Always pair `schedule:` with `schedule_human:`. The first is for the runner, the second is for the user.
- Don't run jobs manually — the runner does that. Test by setting the schedule a minute or two in the future and waiting.
- Disable, don't delete. Rename to `.md.disabled` and add a one-line comment in the body explaining why.

---

### Do this now

List your `<WORKSPACE>/jobs/` directory. For every job there, can you state — in one sentence — what it does and when it next fires? If not, that's a documentation gap, not a scheduling gap. Add the missing `schedule_human` lines and a one-paragraph body before adding any new jobs.

The schedule directory is the agent's autonomous-behaviour manifest. Treat it like one.
