# PROMPT 04 — Writers: tasks (Reminders) + notes (Apple Notes / Obsidian) (Tier 3)

> Paste into a fresh Claude Code session on the CEO's Mac. Build #5 in the live order
> (after the meeting extractor, Prompt 10). Assumes Prompts 01–03 + 10. Read `state/config.json`
> (the backend choices from Prompt 01) before building.

> **Platform note.** macOS writers are AppleScript (`add_reminder.applescript`) + a backend
> dispatcher (`upsert_note.sh`). On **Windows**, build PowerShell equivalents (`Add-Reminder.ps1`,
> `Upsert-Note.ps1`) honoring the backends chosen in `reference/platform.md`: tasks →
> **Microsoft To Do / Outlook Tasks** (COM or Graph); notes → **Obsidian Markdown** and/or
> **OneNote** per config. Keep the **same contract**: idempotent upsert, dedup by the `src:<id>`
> marker, and the same body fields (quote + name + handle + channel + datetime).

## Context

Tier-3 writers are small helpers the **brain** shells out to. They take structured arguments
and create/update things in the tasks backend and the notes backend. They must be
**idempotent** — the brain runs repeatedly, so writing the same task twice must NOT create
duplicates. The writers handle dedup as a *second* safety net (the brain's `processed.json`
ledger is the first).

Destinations are **config-driven** — read `state/config.json` (written in Prompt 01):
- **Commitments/tasks →** the tasks backend (`reminder_list`, default "Exec Assistant") *and*
  a "Tasks / Follow-ups" section in that day's Daily Brief note.
- **Notes (Daily Brief, meeting notes, goals) →** the chosen `notes_backend`:
  `apple` (Apple Notes), `obsidian` (Markdown vault), or `both`. The note writer dispatches on
  this value so **every later skill calls one `upsert_note` and doesn't care which backend(s)
  are live.**

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

## Task B — `bin/upsert_note.sh` (backend dispatcher) + per-backend writers

This is the single entry point every brain skill calls to write any note (daily brief,
meeting note, goals). It dispatches on `notes_backend` from `state/config.json`.

Accept a uniform contract: `noteType` (`brief` | `meeting` | `goals` | …), `noteKey`
(e.g. the date `2026-06-02`, or a meeting id), `title`, `section`, and `content`
(pass **Markdown** as the canonical input — each backend renders it as needed).

`bin/upsert_note.sh`:
- Read `notes_backend`. For `apple` call the Apple writer; for `obsidian` call the Obsidian
  writer; for `both` call **both** (a note exists in each place, deduped independently).
- Always also keep a canonical Markdown copy at `state/briefs/<noteKey>.md` (or
  `state/meetings/`, `state/goals.md`) — this is the source of truth the periodic briefs
  (Prompt 13) read back, and it makes idempotent section-upsert trivial in plain text.

### B1 — `bin/upsert_note_apple.applescript` (Apple Notes)

Idempotently maintain one note per `noteKey` in the folder from config (`apple_notes.folder`,
e.g. "Daily Briefs"; meeting notes → "Meetings"; goals → "Goals").
- Note title from `title`. If it doesn't exist, create it with a skeleton (H1 + the standard
  sections as H2s).
- **Upsert by section:** replace the contents under the named `<h2>section</h2>` with the
  rendered HTML, leaving other sections intact. Within "Tasks / Follow-ups", dedup line items
  by the `src:<id>` marker. Do not `activate` Notes (no foreground).
- After upsert, capture the note's `applenotes:` deep link / id and (for the goals note) write
  it to `state/config.json` → `goals_note_link`, so briefs can embed a tappable link.

> Apple Notes HTML manipulation via AppleScript is fiddly. The robust path: render the whole
> canonical `state/.../<key>.md` to HTML and **overwrite the note body each run** — fully
> idempotent, no section-splicing. Prefer this if clean section-replace proves brittle.

### B2 — `bin/upsert_note_obsidian.sh` (Obsidian vault — recommended primary)

Write/update a Markdown file directly in the vault — plain file I/O, far more reliable than the
Notes HTML dance. Read `obsidian.vault_path` + `obsidian.folder` from config.
- File path: `<vault>/<folder>/<sub>/<title>.md` (briefs → `Daily Briefs/`, meetings →
  `Meetings/`, goals → `Goals.md`).
- If the file doesn't exist, create it with the H1 + standard `##` sections. **Upsert by
  section** in Markdown (replace the block under `## <section>`), leaving others intact. Dedup
  "Tasks / Follow-ups" lines by the `src:<id>` marker.
- Use real Obsidian **wikilinks** so the second brain interlinks: briefs link the goals note
  as `[[Goals]]`, meeting notes link back to the day's brief, etc.
- If `obsidian.cli` is true, you *may* use the Obsidian CLI (`Obsidian append` / search /
  backlinks / `sync`) for nicer integration — but **direct file I/O is the dependable default**;
  don't hard-depend on the CLI.
- Apple Reminders stays the task surface regardless of notes backend (it syncs to his phone).

## Task C — (optional helper) `bin/send_imessage.applescript`

Not used for the brief (Notes is the destination), but useful later for nudges. A small
helper that sends an iMessage to a given handle. Build it but leave it unused for now.
Messages.app **can** send via AppleScript (`send "..." to buddy ...`). Note it requires the
Messages Automation grant from Prompt 01.

## Verify before finishing

1. Call `add_reminder.applescript` twice with identical args including the same `src:` id →
   confirm only **one** reminder exists in the "Exec Assistant" list, with full context body and
   correct due date.
2. Call `upsert_note.sh` twice for today's date, same section, same `src:` line → confirm the
   note exists in the configured backend(s), the section updated in place, and the task line
   appears **once**. For `obsidian`/`both`, show the `.md` file in the vault; for `apple`/`both`,
   show the note. Confirm the canonical `state/briefs/<date>.md` copy was written.
3. Show me the resulting reminder body and the note's Markdown/HTML so I can eyeball the
   formatting, plus the `[[Goals]]`-style link if the goals note exists yet.

Idempotency is the whole point of this prompt — prove it with the double-call tests.
