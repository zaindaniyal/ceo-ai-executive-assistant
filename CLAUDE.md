# CEO Executive Assistant AI — project instructions

Local AI executive assistant for the CEO. Every session working in this project must
follow this file. The full design lives in `reference/ARCHITECTURE.md` — read it before
building anything.

## What this is

A local-only assistant that watches the CEO's channels (~4 email accounts, ~4 calendars,
WhatsApp, iMessage), produces a **Daily Brief** in Apple Notes, and turns **commitments**
("email me the deck", "I'll call you next week", "let's catch up") into **Apple Reminders**
+ Daily-Brief task lines — without letting anything fall through the cracks.

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

1. **Extractors (Tier 1)** — dumb, cheap AppleScript/shell fired by **launchd every ~5 min**.
   No LLM. Dump raw recent data to JSON in `state/`. Atomic writes, no dialogs, no foreground.
2. **Brain (Tier 2)** — headless `claude -p`, fired **less often** (~45 min; brief once each
   morning). Reads the JSON, reasons, acts. The only tier that costs tokens — keep it tight,
   feed it only unprocessed items.
3. **Writers (Tier 3)** — small AppleScript helpers the brain shells out to (Reminders, Notes).

Do not collapse these. Do not make an extractor call the LLM. Do not run the brain every 5 min.

## Hard rules

- **Local only.** No message or email content ever leaves this Mac.
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

## Destinations (decided)

- **Daily Brief** → Apple **Notes**, one dated note per day in a "Daily Briefs" folder.
- **Commitments/tasks** → **both** Apple **Reminders** (list "Exec Assistant") **and** the
  "Tasks / Follow-ups" section of that day's brief note.
- Future: migrate Notes → Obsidian behind a config flag (`prompts/09_obsidian_future.md`).
  Not yet.

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
~/ceo-executive-assistant-ai/
  bin/        extractor + writer scripts
  state/      JSON snapshots + processed.json ledger — PRIVATE, gitignored
  logs/       launchd stdout/stderr
  schedule/   launchd *.plist (mac) or Task Scheduler reg (win) + assistant-ctl
  .claude/skills/   commitment-detector, daily-brief, historical-backfill
  reference/  ARCHITECTURE.md, accounts.md, wacli.md, message_schema.md
  prompts/    the build prompts (01–09), pasted one per fresh session in order
```

## Conventions

- Absolute paths in launchd plists; set PATH to include `/opt/homebrew/bin`.
- Every script: correct shebang, `chmod +x`, lockfile in `state/` to prevent overlap.
- Validate every JSON write (`python3 -m json.tool`) before trusting it.
- Log suppression/dedup decisions so they can be audited.
- `state/` and `logs/` hold the CEO's private content — never commit them.

## Build order

`prompts/01 → 02 → 03 → 04 → 05 → 06 → 08 (scheduling) → 07 (one-time backfill) → 09 (later)`.
One fresh session per prompt; each ends with an explicit verify step — don't skip it.
