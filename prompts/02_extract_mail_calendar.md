# PROMPT 02 — Mail + Calendar extractors (Tier 1)

> Paste into a fresh Claude Code session on the CEO's Mac. Build #2. Assumes Prompt 01 is
> done: `~/ceo-ai-executive-assistant/` exists, permissions granted, `reference/accounts.md` lists the
> real accounts/calendars.

> **Platform note.** This prompt is written in the **macOS** idiom (AppleScript + `osascript`,
> `icalBuddy`). If `reference/platform.md` says **Windows**, build the *identical behavior* in
> **PowerShell** instead: read email and calendar from **classic Outlook via COM**
> (`New-Object -ComObject Outlook.Application` → MAPI namespace; iterate the Inbox `MailItem`s
> by `ReceivedTime`, and `Calendar` `AppointmentItem`s by date window), write `.ps1` scripts
> under `bin/`, use `Move-Item -Force` for atomic writes, and emit the **same JSON shapes**.
> The New-Outlook / icalBuddy notes are macOS-only — ignore them on Windows.

## Context

Tier-1 extractors are **dumb, cheap, deterministic** scripts fired by launchd every ~5
minutes. They do NO reasoning. They just dump raw, recent data to JSON in
`~/ceo-ai-executive-assistant/state/` for the Tier-2 brain to read later. Keep them fast and robust —
they run constantly and must never hang or pop a dialog.

Read `~/ceo-ai-executive-assistant/reference/accounts.md` first so you iterate over the real account and
calendar names.

## Task A — `bin/extract_mail.applescript`

Write an AppleScript (run via `osascript`) that dumps recent mail across **all** accounts
to `~/ceo-ai-executive-assistant/state/mail.json`.

Requirements:
- Iterate `every account` → its `INBOX` mailbox → messages received in the **last 3 days**
  (a window wider than the 5-min cadence so nothing is missed across restarts; dedup is the
  brain's job via stable IDs).
- For each message capture: a **stable id** (`message id` header if available, else
  `id of message`), `date received` (ISO 8601), `sender` (name + address), `subject`,
  the **account name** it came from, read/unread flag, and the first ~1500 chars of
  `content` (plain text; strip nothing fancy, just truncate).
- Emit a **single JSON array**. Be careful with AppleScript→JSON: quote/escape strings
  properly (escape `"`, `\`, newlines). If you'd rather, build the JSON in a small helper
  using `text item delimiters`, or shell out to `python3 -c` for safe JSON encoding — your
  call, but the output must be valid JSON (verify with `python3 -m json.tool`).
- **Mail.app must be running.** Have the script `tell application "Mail" to launch` if not
  running, but do NOT bring it to the foreground (no `activate`).
- Honor `reference/accounts.md`: **skip any account flagged as New Outlook** (invisible to
  AppleScript) and leave a `"_note"` in the JSON saying which accounts were skipped, so the
  brain knows there's a blind spot. If a New-Outlook / Gmail account matters, tell me and
  we'll add an IMAP/Gmail-API fallback extractor as a follow-up.
- Wrap risky calls in `try` blocks so one bad message can't abort the whole dump.
- Write atomically: write to `state/mail.json.tmp` then `mv` over `state/mail.json`.

## Task B — `bin/extract_calendar.applescript` (prefer icalBuddy)

Dump events from **today through +7 days** across all calendars to
`~/ceo-ai-executive-assistant/state/calendar.json`.

Two paths — detect and choose at runtime:
- **Preferred: `icalBuddy`.** If `which icalBuddy` succeeds (or offer `brew install
  ical-buddy`), use it — it's fast and clean. Get events for the window and parse into JSON.
  Capture per event: title, start (ISO), end (ISO), location, attendees if available,
  calendar name, and notes/URL if present.
- **Fallback: Calendar.app AppleScript**, scoped tightly to the date window (whose start is
  today 00:00 and whose end is +7 days) — never an unbounded query (it will crawl). Same
  fields.

Why we capture upcoming events at all: the **brain** uses the calendar to (a) build the
brief and (b) **suppress already-moot commitments** — e.g. if an email asked "let's find
time to meet" and a meeting with that person already sits in the past on the calendar, no
task should be created. So **also include recently-past events** in this dump: widen the
window to **−5 days through +7 days** so the brain can see meetings that already happened.

Output valid JSON (verify it). Atomic write as in Task A.

## Task C — wrapper + logging

- Add `bin/extract_mail.sh` and `bin/extract_calendar.sh` thin wrappers that call
  `osascript`, append timestamped stdout/stderr to `logs/extract_mail.log` /
  `logs/extract_calendar.log`, and exit non-zero on failure (so launchd surfaces it later).
- Keep each run well under a few seconds where possible.

## Verify before finishing

Run both extractors manually and show me:
1. `python3 -m json.tool ~/ceo-ai-executive-assistant/state/mail.json | head -40` — valid JSON, real
   messages, sender/subject/date/account populated, content truncated.
2. `python3 -m json.tool ~/ceo-ai-executive-assistant/state/calendar.json | head -40` — events spanning
   roughly −5 to +7 days, with past events present.
3. Confirm which Calendar path was used (icalBuddy vs AppleScript) and which Mail accounts
   were skipped (if any).

Do not wire up launchd yet — that's Prompt 08. Leave the scripts runnable by hand.
