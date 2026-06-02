# CEO AI Executive Assistant

> A cross-platform (macOS / Windows), fully-local AI executive assistant for a busy CEO —
> a build-prompt set that stands up a daily brief and multi-channel commitment capture
> (email, calendar, WhatsApp, iMessage) into reminders and notes, with nothing leaving the
> machine.

A generic, local AI executive assistant for a busy CEO. This folder is the **authoring
workspace** — it contains a set of **prompts** you paste, one at a time, into fresh Claude
Code sessions **on the executive's machine**. Each prompt makes that session build a piece
of the assistant (skills + scripts + scheduled jobs).

You are not running the assistant from here. You are shipping the prompts that build it
over there.

## Cross-platform: macOS **or** Windows

Same architecture on both; only the scripting stack differs — **AppleScript + launchd** on
macOS, **PowerShell + Task Scheduler** on Windows. Prompt 01 detects the OS and records it
in `reference/platform.md`; every later prompt has a **Platform note** telling the session
which stack to use. The only cross-platform pieces are the brain (`claude -p`) and the
WhatsApp CLI (`wacli`). **iMessage is macOS-only** (no Windows equivalent). Full capability
mapping in `reference/ARCHITECTURE.md` → "Platform abstraction".

## What the assistant does

- Pulls the CEO's world together: email (Mail.app / Outlook), calendars, WhatsApp (`wacli`),
  and iMessage (macOS only).
- Produces a **Daily Brief** every morning (Apple Notes on macOS; OneNote/Markdown on Windows).
- Watches every channel for **commitments** — "email me the deck", "I'll call you next
  week", "let's catch up" — and turns each into a **reminder** (Apple Reminders / Microsoft
  To Do) *and* a line in that day's Daily Brief, tagged with the person's name + contact handle.
- **Temporal reconciliation:** suppresses commitments that are already moot (e.g. a "let's
  meet" ask whose meeting already shows as a past calendar event).
- Runs continuously and cheaply via a split **extract → brain** architecture.

## How to use this folder

1. **Read `reference/ARCHITECTURE.md` first.** It's the shared model every prompt assumes.
2. Copy the project to the executive's machine, then paste **`prompts/00_kickoff.md`** to
   orient the session and drive the build with a stop after each step.
3. Open a **fresh Claude Code session** for **each** prompt, in order. Fresh sessions keep
   context clean and let each build step be reviewed independently.
4. Paste the contents of the prompt file. Let the session build + verify that piece.
5. Move to the next prompt.

## Paste order

| # | Prompt file | What it builds | Cadence once live |
|---|-------------|----------------|-------------------|
| 1 | `prompts/01_project_init_and_permissions.md` | Skeleton, **OS detection**, permissions (TCC on macOS / Outlook+execution-policy on Windows), confirm `wacli` | one-time |
| 2 | `prompts/02_extract_mail_calendar.md` | Email + Calendar extractors → JSON (Mail.app/icalBuddy **or** Outlook COM) | every 5 min |
| 3 | `prompts/03_extract_messages_whatsapp.md` | iMessage (macOS only) + WhatsApp (`wacli`) extractors → JSON | every 5 min |
| 4 | `prompts/04_writers_reminders_notes.md` | Reminders + daily-note writer scripts, idempotent | on demand |
| 5 | `prompts/05_commitment_detector_skill.md` | The "brain" skill that finds commitments → reminders + note | every 30–60 min |
| 6 | `prompts/06_daily_brief_skill.md` | Daily Brief generator → note | once each morning |
| 7 | `prompts/08_scheduling_launchd.md` | All scheduled jobs + a control script (launchd **or** Task Scheduler) | one-time |
| 8 | `prompts/07_historical_backfill_2weeks.md` | One-time 2-week sweep of WhatsApp + email (+ iMessage on macOS) for missed commitments | run once, manually |
| 9 | `prompts/09_obsidian_future.md` | (Later) migrate the daily note → Obsidian | future |

> Build #7 (scheduling) **before** #8 (backfill) is intentional: get the live system
> working and verified, then run the historical sweep once it's all proven.

## Ground rules baked into every prompt

- **Local only.** No message/email content leaves the machine.
- **One stack per OS.** AppleScript on macOS, PowerShell on Windows — never mixed.
- **Idempotent.** A given message can only ever produce one reminder (dedup ledger in
  `state/processed.json`). Extractors run every 5 minutes — nothing may double-fire.
- **Reminders carry context.** Every task says *who* (name + email/phone), *which channel*,
  and *a one-line quote* of the ask.
- **Don't create moot tasks.** Temporal reconciliation against the calendar + later replies.
- **`wacli` interface is discovered, not assumed.** WhatsApp prompts run `wacli --help` first.
- **Verify each piece** before moving on (each prompt ends with an explicit test step).

## License

[MIT](LICENSE) © 2026 Zain Daniyal
