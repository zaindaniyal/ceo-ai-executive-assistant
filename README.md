# CEO AI Executive Assistant

> A cross-platform (macOS / Windows), local-first AI executive assistant for a busy CEO —
> a build-prompt set that stands up daily / weekly / monthly / quarterly briefs, multi-channel
> commitment capture (email, calendar, WhatsApp, iMessage, **meeting transcripts**), a daily
> **goal-detection** pass, and emailed briefs — writing everything into a **second brain**
> (Obsidian, Apple Notes, or both) with reminders on the side, and nothing leaving the machine
> except the briefs you choose to email yourself.

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
  iMessage (macOS only), and **meeting transcripts** (Otter / Fireflies / a transcript folder).
- Produces a **Daily Brief** every morning, plus **weekly / monthly / quarterly** briefs — into
  the chosen **second brain**: **Obsidian, Apple Notes, or both** (a setup-time choice).
- Watches every channel for **commitments** — "email me the deck", "I'll call you next
  week", "let's catch up", *and spoken asks in meetings* — and turns each into a **reminder**
  (Apple Reminders / Microsoft To Do) *and* a line in that day's Daily Brief, tagged with the
  person's name + contact handle.
- **Reads meeting transcripts itself** and writes its own summary / decisions / action-items —
  it does **not** just trust the Otter/Fireflies auto-summary.
- **Detects goals daily:** a deep pass over every signal infers the CEO's goals, organizes them
  by horizon (weekly/monthly/yearly) × category (spiritual / knowledge & growth / personal /
  family / professional / other) into a living **Goals note**, diffs for new goals each day, and
  reports progress in the briefs (with links) plus weekly/monthly/quarterly check-ins.
- **Emails the briefs** to the CEO (opt-in) so they arrive as a phone notification.
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

Paste in this order (the file numbers aren't strictly sequential — newer pieces were appended
as 10–14 to avoid renumbering, but the **order below is the build order**):

| # | Prompt file | What it builds | Cadence once live |
|---|-------------|----------------|-------------------|
| 1 | `prompts/01_project_init_and_permissions.md` | Skeleton, **OS detection**, **backend choice** (Obsidian/Apple/both), **integrations discovery** (Otter/Fireflies, email-send), permissions, confirm `wacli` | one-time |
| 2 | `prompts/02_extract_mail_calendar.md` | Email + Calendar extractors → JSON | every 5 min |
| 3 | `prompts/03_extract_messages_whatsapp.md` | iMessage (macOS only) + WhatsApp (`wacli`) extractors → JSON | every 5 min |
| 4 | `prompts/10_extract_meetings.md` | Meeting transcript extractor (Otter / Fireflies / folder) → JSON | every ~30 min |
| 5 | `prompts/04_writers_reminders_notes.md` | Reminder writer + **backend-dispatching note writer** (Apple Notes / Obsidian / both) | on demand |
| 6 | `prompts/05_commitment_detector_skill.md` | The "brain" skill that finds commitments → reminders + note | every 30–60 min |
| 7 | `prompts/11_meeting_assistant_skill.md` | Reads full transcripts → **own** summary/decisions/actions → meeting notes + commitments | every ~60 min |
| 8 | `prompts/06_daily_brief_skill.md` | Daily Brief generator (incl. Meetings + Goals link) → note | once each morning |
| 9 | `prompts/12_goal_detector_skill.md` | Daily deep goal analysis + diffing → living **Goals note** | once daily |
| 10 | `prompts/13_periodic_briefs_and_checkins.md` | Weekly / monthly / quarterly briefs + goal check-ins | weekly/monthly/quarterly |
| 11 | `prompts/14_email_delivery.md` | Email the briefs so they arrive as a notification | piggybacks on briefs |
| 12 | `prompts/08_scheduling_launchd.md` | All scheduled jobs + a control script (launchd **or** Task Scheduler) | one-time |
| 13 | `prompts/07_historical_backfill_2weeks.md` | One-time 2-week sweep for missed commitments | run once, manually |
| 14 | `prompts/09_switch_notes_backend.md` | (Optional) switch/migrate the notes backend + backfill history | as needed |

> Scheduling (#12) **before** backfill (#13) is intentional: get the live system working and
> verified, then run the historical sweep once it's all proven.

## Ground rules baked into every prompt

- **Local-first.** No message / email / transcript content leaves the machine — **except** the
  brief email, which you opt into and which only sends to **yourself** from **your own** account.
- **One stack per OS.** AppleScript on macOS, PowerShell on Windows — never mixed.
- **Idempotent.** A given message/meeting can only ever produce one reminder (dedup ledger in
  `state/processed.json`); notes upsert in place. Nothing may double-fire.
- **Reminders carry context.** Every task says *who* (name + email/phone), *which channel*,
  and *a one-line quote* of the ask.
- **Read the transcript, not just the vendor summary.** The meeting brain forms its own view.
- **Don't create moot tasks.** Temporal reconciliation against the calendar + later replies.
- **`wacli` interface is discovered, not assumed.** WhatsApp prompts run `wacli --help` first.
- **Keep a note of what we're connected to** in `reference/integrations.md`.
- **Verify each piece** before moving on (each prompt ends with an explicit test step).
- **Suggestions for going further** live in `reference/ROADMAP.md`.

## License

[MIT](LICENSE) © 2026 Zain Daniyal
