# PROMPT 04 — Writers: Apple Reminders + Apple Notes (Tier 3)

> Paste into a fresh Claude Code session on the CEO's Mac. Build #4. Assumes Prompts 01–03.

> **Platform note.** macOS writers are AppleScript (`add_reminder.applescript`,
> `upsert_note.applescript`). On **Windows**, build PowerShell equivalents (`Add-Reminder.ps1`,
> `Upsert-Note.ps1`) honoring the backends chosen in `reference/platform.md`: reminders →
> **Microsoft To Do / Outlook Tasks** (COM or Graph); daily note → **OneNote** *or* a synced
> **Markdown file** (recommended — simplest, fully idempotent). Keep the **same contract**:
> idempotent upsert, dedup by the `src:<id>` marker, and the same body fields (quote + name +
> handle + channel + datetime).

## Context

Tier-3 writers are small AppleScript helpers the **brain** shells out to. They take
structured arguments and create/update things in Apple Reminders and Apple Notes. They must
be **idempotent** — the brain runs repeatedly, so writing the same task twice must NOT
create duplicates. The writers handle dedup as a *second* safety net (the brain's
`processed.json` ledger is the first).

Decided destinations:
- **Commitments/tasks → BOTH** the Apple **Reminders** app *and* a "Tasks / Follow-ups"
  section in that day's Daily Brief **note**.
- **Daily Brief → Apple Notes**, one dated note per day in a "Daily Briefs" folder.

## Task A — `bin/add_reminder.applescript`

Create a reminder in Apple Reminders. Accept arguments (via `on run argv`):
`title`, `body`, `dueISO` (optional), `listName` (default e.g. "Exec Assistant").

Behavior:
- Ensure the target list exists; create it if missing.
- **Dedup:** before creating, check the list for an existing reminder with the same title
  whose body contains the same source-id marker (see below). If found, do nothing (or
  update the due date if changed). This makes re-runs safe.
- Set `due date` / `remind me date` from `dueISO` when provided (parse ISO → AppleScript
  `date`). If no due date, leave it as a plain reminder.
- The **body** must carry full context so the CEO can act without opening the thread:
  ```
  <one-line quote of the ask>
  — From: <name> <handle>
  — Channel: <whatsapp|imessage|email|calendar>
  — When: <original message datetime>
  — src:<stable-source-id>      ← machine marker used for dedup
  ```
- Print the created/updated reminder id so the brain can log it.

## Task B — `bin/upsert_note.applescript` (Daily Brief note)

Idempotently maintain **one note per day** in a Notes folder called "Daily Briefs".

Accept: `dateYYYYMMDD`, `section` (e.g. "Brief", "Tasks / Follow-ups", "Inbox Highlights"),
and `htmlContent`.

Behavior:
- Ensure the "Daily Briefs" folder exists.
- Note title = e.g. `Daily Brief — 2026-06-02`. If it doesn't exist, create it with a
  skeleton (H1 title + the standard sections as H2s).
- **Upsert by section:** replace the contents under the named `<h2>section</h2>` with
  `htmlContent`, leaving other sections intact. (Apple Notes bodies are HTML; manipulate the
  HTML between section headers.) This lets the brain re-run and refresh "Tasks /
  Follow-ups" without duplicating or clobbering the narrative brief.
- Within "Tasks / Follow-ups", dedup line items by the `src:<id>` marker so re-runs don't
  pile up duplicates.
- Do not `activate` Notes (no foreground).

> Apple Notes HTML manipulation via AppleScript is fiddly. If clean section-replace proves
> too brittle, fall back to: keep a canonical Markdown file per day at
> `state/briefs/<date>.md` (the brain owns it, easy to dedup/upsert in plain text), and have
> this script **render that whole Markdown to HTML and overwrite the note body each run**.
> That's simpler and fully idempotent. Pick whichever you can make reliable; tell me which.

## Task C — (optional helper) `bin/send_imessage.applescript`

Not used for the brief (Notes is the destination), but useful later for nudges. A small
helper that sends an iMessage to a given handle. Build it but leave it unused for now.
Messages.app **can** send via AppleScript (`send "..." to buddy ...`). Note it requires the
Messages Automation grant from Prompt 01.

## Verify before finishing

1. Call `add_reminder.applescript` twice with identical args including the same `src:` id →
   confirm only **one** reminder exists in the "Exec Assistant" list, with full context body and
   correct due date.
2. Call `upsert_note.applescript` twice for today's date, same section, same `src:` line →
   confirm the Daily Brief note exists in "Daily Briefs", the section updated in place, and
   the task line appears **once**.
3. Show me the resulting reminder body and the note's HTML/Markdown so I can eyeball the
   formatting.

Idempotency is the whole point of this prompt — prove it with the double-call tests.
