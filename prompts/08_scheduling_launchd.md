# PROMPT 08 — Scheduling everything with launchd

> Paste into a fresh Claude Code session on the CEO's Mac. Build #12 in the live order (paste
> this BEFORE the historical backfill). Assumes Prompts 01–06 + 10–14 produced working,
> hand-verified scripts.

## Context

The split architecture means two cadences:
- **Tier-1 extractors** (Mail, Calendar, iMessage, WhatsApp) — cheap, run **every 5 min**.
- **Tier-2 brain** — costs tokens, runs **less often**: commitment-detector ~**every 45
  min**, daily-brief **once each morning** (and a refresh, optional).

We use **launchd** (LaunchAgents), not cron — cron is deprecated on macOS and runs in a TCC
sandbox that can't see the Automation grants. This machine runs all the time, so plain
`StartInterval` agents are fine.

> **Platform note.** This prompt is written for **launchd (macOS)**. On **Windows**, deliver
> the *identical cadence* with **Task Scheduler** instead:
> - One `Register-ScheduledTask` per job, pointing at the `.ps1` wrappers in `bin/`:
>   the extractors every **5 min** (meeting extractor every **30 min**), the commitment brain
>   every **45 min**, the meeting-assistant **hourly**, the **goal-detector daily at 05:30**,
>   the daily brief **daily at 06:30**, and the weekly/monthly/quarterly briefs on their
>   first-of-period days. (No iMessage task on Windows.)
> - Build `schedule/assistant-ctl.ps1` with the same subcommands —
>   `load`/`unload`/`reload`/`status`/`kick` mapping to
>   `Register-ScheduledTask` / `Unregister-ScheduledTask` / `Get-ScheduledTask` /
>   `Start-ScheduledTask`.
> - **Run each task as the logged-in user with "Run only when user is logged on"** — a task
>   running as SYSTEM or in session 0 won't have the user's Outlook profile/credentials. This
>   is the Windows analogue of the macOS "launchd context ≠ interactive shell" TCC trap below.
> - Same lockfile + atomic-write guards. Use the **same JSON output paths** and `state/`.
>
> Everything below describes the macOS launchd implementation; the `schedule/` directory holds
> launchd `*.plist` files on macOS and the Task Scheduler registration on Windows.

## Tasks

### 1. Write the LaunchAgent plists into `~/ceo-ai-executive-assistant/schedule/`

Create one plist per job. Use reverse-DNS labels like `com.exec-assistant.extract-mail`. Each:
- `ProgramArguments` → the script's wrapper (e.g. `bin/extract_mail.sh`). Use **absolute
  paths** (launchd has a minimal env). Set `WorkingDirectory` to `~/ceo-ai-executive-assistant`.
- Redirect `StandardOutPath` / `StandardErrorPath` into `logs/<job>.out` / `.err`.
- Set a sane `EnvironmentVariables` `PATH` that includes Homebrew (`/opt/homebrew/bin`),
  `/usr/local/bin`, `/usr/bin`, `/bin` — so `wacli`, `icalBuddy`, `python3`, `sqlite3`,
  `claude` resolve.

Jobs and cadence:

| Label | Runs | Cadence |
|-------|------|---------|
| `com.exec-assistant.extract-mail` | `bin/extract_mail.sh` | `StartInterval` 300 |
| `com.exec-assistant.extract-calendar` | `bin/extract_calendar.sh` | `StartInterval` 300 |
| `com.exec-assistant.extract-messages` | `bin/extract_messages.sh` | `StartInterval` 300 |
| `com.exec-assistant.extract-whatsapp` | `bin/extract_whatsapp.sh` | `StartInterval` 300 |
| `com.exec-assistant.extract-meetings` | `bin/extract_meetings.sh` | `StartInterval` 1800 (30 min) |
| `com.exec-assistant.brain-commitments` | `bin/run_brain.sh` | `StartInterval` 2700 (45 min) |
| `com.exec-assistant.meeting-assistant` | `bin/run_meeting_assistant.sh` | `StartInterval` 3600 (60 min) |
| `com.exec-assistant.daily-brief` | `bin/run_brief.sh` | `StartCalendarInterval` 06:30 daily |
| `com.exec-assistant.goal-detector` | `bin/run_goal_detector.sh` | `StartCalendarInterval` 05:30 daily |
| `com.exec-assistant.weekly-brief` | `bin/run_weekly_brief.sh` | `StartCalendarInterval` Mon 06:45 |
| `com.exec-assistant.monthly-checkin` | `bin/run_monthly_checkin.sh` | `StartCalendarInterval` day 1, 07:00 |
| `com.exec-assistant.quarterly-brief` | `bin/run_quarterly_brief.sh` | `StartCalendarInterval` Jan/Apr/Jul/Oct 1, 07:15 |

Stagger the extractors so they don't all fire on the same second (e.g. offset by using
slightly different intervals, or `ThrottleInterval`) — mostly to avoid `wacli` contention.

Cadence notes: the **meeting extractor** runs every 30 min (meetings don't finish every 5
min); the **meeting-assistant** brain hourly. The **goal-detector** runs once daily at 05:30
so the Goals note is fresh **before** the 06:30 daily brief reads it. Weekly/monthly/quarterly
fire just before the daily brief on their respective first-of-period days. `launchd` does not
have a native "first Monday of the quarter" — use a `StartCalendarInterval` on the 1st and let
the script itself no-op if it's not the right month/weekday (record last-run in `state/`).
Email delivery has **no job of its own** — it piggybacks on the brief jobs (Prompt 14).

### 2. Control script `schedule/assistant-ctl.sh`

A small helper with subcommands:
- `load` — `launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/<each>.plist` after
  symlinking/copying the plists from `schedule/` into `~/Library/LaunchAgents/`.
- `unload` — `launchctl bootout gui/$(id -u)/<label>` for each.
- `reload` — unload + load.
- `status` — `launchctl list | grep com.exec-assistant` plus the last line of each log.
- `kick <label>` — `launchctl kickstart -k gui/$(id -u)/<label>` to force a run now.

### 3. Sanity guards

- The brain wrapper (`run_brain.sh`) should **lock** (flock/lockfile in `state/`) so an
  overlong run can't overlap the next 45-min trigger.
- Extractors already write atomically; make sure a hung extractor can't pile up — add a
  per-job lockfile too.
- All scripts must be `chmod +x` and start with a correct shebang.

## Verify

1. `assistant-ctl.sh load`, then `assistant-ctl.sh status` → all labels present, no crash codes.
2. `assistant-ctl.sh kick com.exec-assistant.extract-mail` → fresh `state/mail.json` timestamp + clean
   log.
3. Kick `brain-commitments` → confirm it runs end-to-end under launchd (this is where TCC
   problems surface: if Reminders/Notes writes fail silently here but worked by hand, the
   Automation grant isn't attached to the launchd context — fix by ensuring the
   grant covers the `claude`/`osascript` host, then re-kick).
4. Confirm `daily-brief` is scheduled for tomorrow 06:30 (`launchctl list` shows it).
5. Watch logs for ~15 min (or kick a couple of cycles) and confirm no duplicate reminders
   appear — dedup holds under real scheduling.

Report a PASS/FAIL per job. **This is the step most likely to expose TCC/permission gaps**
(launchd context ≠ your interactive shell), so test the brain under launchd specifically,
not just by hand.
