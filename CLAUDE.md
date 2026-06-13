# CEO AI Executive Assistant — project instructions

Local AI executive assistant for the CEO. Every session working in this project must
follow this file. The full design lives in `reference/ARCHITECTURE.md` — read it before
building anything.

## What this is

A local-first assistant that watches the CEO's channels (~4 email accounts, ~4 calendars,
WhatsApp, iMessage, **meeting transcripts from Otter/Fireflies**), and:

- produces a **Daily Brief** (and **weekly / monthly / quarterly** briefs) into the CEO's
  **second brain** — **Obsidian, Apple Notes, or both** (chosen at setup);
- turns **commitments** ("email me the deck", "I'll call you next week", spoken asks in
  meetings) into **Apple Reminders** + Daily-Brief task lines;
- reads **full meeting transcripts** and writes its **own** summary/decisions/action-items —
  never just trusting the vendor's auto-summary;
- runs a **daily goal-detection** pass that infers the CEO's goals across every signal and
  maintains a living **Goals note** (weekly/monthly/yearly × spiritual / knowledge & growth /
  personal / family / professional / other), diffing daily and checking in weekly/monthly;
- **emails the briefs** to him (opt-in) so they land as a phone notification —
  without letting anything fall through the cracks.

## Platform (decide once, then commit)

Runs on macOS **or** Windows — same architecture, different scripting stack. Detect the OS
in Prompt 01 and record it in `reference/platform.md`, then:

- **macOS → all scripts are AppleScript/shell**, scheduled by **launchd**.
- **Windows → all scripts are PowerShell (`.ps1`)**, scheduled by **Task Scheduler**.

**Never mix stacks.** Only the brain (`claude -p`) and `wacli` are cross-platform. iMessage
is **macOS-only** (no Windows equivalent — record as a blind spot there). See
`reference/ARCHITECTURE.md` → "Platform abstraction" for the full capability mapping
(Outlook COM, Microsoft To Do, OneNote/Markdown, etc.). On Windows, also read
`reference/WINDOWS-NOTES.md` (prerequisites, Graph fallback, Task Scheduler trap).

## Architecture (non-negotiable)

Three tiers — keep them separate:

1. **Extractors (Tier 1)** — dumb, cheap AppleScript/shell fired by **launchd every ~5 min**
   (meeting extractor every ~30 min). No LLM. Dump raw recent data to JSON in `state/`
   (mail, calendar, iMessage, WhatsApp, **meetings**). Atomic writes, no dialogs, no foreground.
2. **Brain (Tier 2)** — headless `claude -p`, fired **less often**: commitment-detector ~45
   min, meeting-assistant ~hourly, daily brief once each morning, **goal-detector once daily**,
   weekly/monthly/quarterly briefs on their cadence. Reads the JSON, reasons, acts. The only
   tier that costs tokens — keep it tight, feed it only unprocessed items.
3. **Writers (Tier 3)** — small helpers the brain shells out to: the reminder writer, the
   **backend-dispatching note writer** (`upsert_note.sh` → Apple Notes / Obsidian / both per
   `state/config.json`), and the **email sender**.

Do not collapse these. Do not make an extractor call the LLM. Do not run the brain every 5 min.

## Hard rules

- **Local only — one deliberate exception.** No message, email, or transcript content ever
  leaves this Mac, **except** the daily/periodic brief email, which the CEO opted into and which
  only ever sends **to himself** from **his own** mail account. Everything else stays local.
- **Read the transcript, not just the vendor summary.** For meetings, the brain forms its own
  summary/decisions/action-items from the full transcript; the Otter/Fireflies auto-summary is
  a cross-check, never the source of truth.
- **Idempotent.** Everything is keyed by a stable source id and recorded in
  `state/processed.json`. Extractors run every 5 min and the brain re-runs — a given message
  may produce **at most one** reminder. Prove dedup with a double-run on any change.
- **Reminders carry context:** person name + contact handle (email/phone) + channel + a
  one-line verbatim quote of the ask. Body also embeds `src:<id>` for dedup.
- **Temporal reconciliation:** never create an already-moot task. If the calendar shows the
  meeting already happened, or a later reply/sent email fulfilled the ask, or a future event
  already covers it → **suppress** and log why. The calendar extractor deliberately captures
  past events for this. See ARCHITECTURE.md → "Temporal reconciliation".
- **Precision over recall.** A flood of junk reminders gets the system switched off. When
  unsure, mark low-confidence rather than spamming.
- **`wacli` is the WhatsApp CLI — discover its interface, don't assume flags.** Run
  `wacli --help` first; record findings in `reference/wacli.md`.

## Destinations (config-driven — chosen in Prompt 01, in `state/config.json`)

- **Notes (second brain)** → `notes_backend`: **`obsidian`** (recommended primary, Markdown
  vault + `[[backlinks]]`), **`apple`** (Apple Notes), or **`both`**. Covers Daily/periodic
  Briefs, Meeting notes, and the Goals note. Every skill calls one `bin/upsert_note.sh`
  dispatcher and is backend-agnostic.
- **Commitments/tasks** → **both** the tasks backend (Apple **Reminders** / Microsoft To Do,
  list "Exec Assistant") **and** the "Tasks / Follow-ups" section of that day's brief note.
- **Goals** → a single living **Goals note** in the same notes backend; briefs **link** to it.
- **Email delivery** → optional (`email_brief.enabled`); the brief is also emailed to the CEO.
- Switching/migrating the notes backend later is an optional helper (`prompts/09_switch_notes_backend.md`).

## Platform facts (get these right)

- **iMessage:** read from `~/Library/Messages/chat.db` (SQLite), NOT Messages.app AppleScript
  (it can send but not read). `message.date` is ns since 2001-01-01 — convert epoch correctly.
- **Mail.app** must be running to script it; iterate over real accounts. **New Outlook is
  invisible to AppleScript** — flag any such account as a blind spot.
- **Calendar:** prefer `icalBuddy`; fall back to Calendar.app AppleScript scoped to a tight
  date window.
- **Permissions:** Full Disk Access (for chat.db) **and** Automation grants (Mail/Calendar/
  Notes/Reminders/Messages) are separate and both required. The **launchd context differs
  from the interactive shell** — verify the brain works *under launchd*, not just by hand.
- **launchd, not cron** (cron can't see the TCC Automation grants).

## Layout

```
~/ceo-ai-executive-assistant/
  bin/        extractor + writer scripts (incl. extract_meetings, upsert_note dispatcher,
              upsert_note_apple / upsert_note_obsidian, send_email)
  state/      JSON snapshots + processed.json ledger + config.json + briefs/ meetings/ goals.md
              — PRIVATE, gitignored
  logs/       launchd stdout/stderr
  schedule/   launchd *.plist (mac) or Task Scheduler reg (win) + assistant-ctl
  .claude/skills/   commitment-detector, daily-brief, meeting-assistant, goal-detector,
                    periodic-briefs, historical-backfill
  reference/  ARCHITECTURE.md, platform.md, config.md, accounts.md, wacli.md,
              integrations.md, message_schema.md
  prompts/    the build prompts (01–14), pasted one per fresh session in order
```

## Conventions

- Absolute paths in launchd plists; set PATH to include `/opt/homebrew/bin`.
- Every script: correct shebang, `chmod +x`, lockfile in `state/` to prevent overlap.
- Validate every JSON write (`python3 -m json.tool`) before trusting it.
- Log suppression/dedup decisions so they can be audited.
- `state/` and `logs/` hold the CEO's private content — never commit them.

## Build order

`prompts/01 → 02 → 03 → 10 (meeting extract) → 04 (writers) → 05 (commitments) →
11 (meeting-assistant) → 06 (daily brief) → 12 (goal-detector) → 13 (weekly/monthly/quarterly) →
14 (email delivery) → 08 (scheduling) → 07 (one-time backfill) → 09 (optional backend switch)`.
One fresh session per prompt; each ends with an explicit verify step — don't skip it.
