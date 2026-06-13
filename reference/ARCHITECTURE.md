# CEO AI Executive Assistant — Architecture Reference

> Read this once before pasting any prompt. It is the shared mental model that every
> prompt in `../prompts/` assumes. The prompts each repeat a short version of this, but
> this is the canonical description.

## The problem

The CEO runs hot: ~4 calendars, ~4 email accounts, WhatsApp, iMessage, **back-to-back meetings
that get transcribed** (Otter/Fireflies), constant travel. Things fall through the cracks —
specifically **verbal and written commitments**:

- Someone tells him "email me the deck" / "send me that contract" → he must follow up.
- He tells someone "I'll call you next week" / "let's catch up" → he must follow up.
- A decision or action item is **spoken in a meeting** and the vendor's auto-summary misses it.

He also loses sight of the **bigger picture** — what he's actually trying to achieve across
spiritual, knowledge/growth, personal, family, and professional life.

He needs: (a) a **daily brief** (plus weekly/monthly/quarterly zoom-outs), (b) **nothing to
fall through the cracks**, (c) his **meetings read properly** (own summary, not just the
vendor's), (d) his **goals tracked** and progressed, and (e) it all in a **second brain**
(Obsidian / Apple Notes) with the briefs **emailed** to him.

## Design principle: split "extract" from "brain"

Running a full Claude session every 5 minutes is expensive and unnecessary. So the system
has two tiers:

```
┌─────────────────────────────────────────────────────────────────────┐
│  TIER 1 — EXTRACTORS  (plain AppleScript / shell, fired by launchd)   │
│  Cheap, dumb, frequent (~5 min; meetings ~30 min). No LLM. Dump raw.  │
│                                                                       │
│   extract_mail.applescript      → state/mail.json                     │
│   extract_calendar.applescript  → state/calendar.json                 │
│   extract_messages.sh (chat.db) → state/imessage.json                 │
│   extract_whatsapp.sh (wacli)   → state/whatsapp.json                 │
│   extract_meetings.sh (Otter/   → state/meetings.json  (full          │
│      Fireflies/folder)             transcript + vendor summary, kept   │
│                                    separate)                           │
└─────────────────────────────────────────────────────────────────────┘
                                │  (raw JSON snapshots on disk)
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  TIER 2 — BRAIN  (headless `claude -p`, fired by launchd less often)  │
│  Reads the JSON snapshots, reasons, acts. Costs tokens, so runs       │
│  on modest cadences, NOT every 5 min.                                 │
│                                                                       │
│   commitment-detector  ~45m → promises → Reminders + Notes            │
│   meeting-assistant    ~60m → OWN summary from transcript →           │
│                               meeting note + commitments              │
│   daily-brief          06:30 → morning brief (+ goals link) → Notes   │
│   goal-detector        05:30 → deep goal analysis + daily diff →      │
│                               living Goals note                        │
│   weekly/monthly/quarterly briefs + goal check-ins → Notes           │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  TIER 3 — WRITERS  (helpers the brain shells out to)                 │
│   add_reminder.applescript   → Apple Reminders / Microsoft To Do      │
│   upsert_note.sh  (dispatch) → Apple Notes AND/OR Obsidian vault      │
│        ├ upsert_note_apple.applescript   → Apple Notes (HTML)         │
│        └ upsert_note_obsidian.sh         → Markdown in the vault      │
│   send_email.applescript     → emails the brief (opt-in) to the CEO   │
└─────────────────────────────────────────────────────────────────────┘
```

**Why this split matters:** the extractors are deterministic and free to run constantly.
The brain is the only thing that costs money, so we control its cadence independently.

## Platform abstraction (macOS **or** Windows)

This assistant runs on whichever machine the executive uses. The **architecture is
identical** on both; only the *scripting stack* differs. **Detect the OS once** (Prompt 01,
recorded in `reference/platform.md`) and then commit fully:

On Windows, also read `reference/WINDOWS-NOTES.md` — prerequisites (classic Outlook vs the
Graph fallback), reminder/note backend choices, and the Task Scheduler "run as logged-in
user" trap.

- **On macOS → every script is AppleScript / shell**, driven by `osascript`, scheduled by
  **launchd**.
- **On Windows → every script is PowerShell (`.ps1`)**, scheduled by **Task Scheduler**.

**Never mix stacks.** Don't write AppleScript on Windows or PowerShell on macOS. The only
cross-platform pieces are the **brain** (`claude -p`) and the **WhatsApp CLI** (`wacli`) —
those are the same command on both.

Capability mapping (Tier 1 extractors + Tier 3 writers resolve to the right column):

| Capability | macOS (AppleScript stack) | Windows (PowerShell stack) |
|------------|---------------------------|----------------------------|
| Email read | Mail.app via `osascript` | Outlook (classic) via COM `New-Object -ComObject Outlook.Application`; or Microsoft Graph |
| Calendar read | `icalBuddy` / Calendar.app | Outlook Calendar via COM; or Microsoft Graph |
| iMessage | `~/Library/Messages/chat.db` (SQLite) | **N/A — iMessage doesn't exist on Windows.** Skip the extractor; record it as a permanent blind spot. (Phone Link's DB is unreliable; don't depend on it.) |
| WhatsApp | `wacli` | `wacli` (same cross-platform CLI) |
| Meeting transcripts | Fireflies GraphQL API / Otter email-or-folder / local transcript folder (cloud, OS-agnostic) | same — HTTP/email/folder, not OS-specific |
| Reminders / Tasks | Apple Reminders via AppleScript | Microsoft To Do via Graph, or Outlook Tasks via COM |
| Notes (second brain) | **Obsidian** (Markdown vault, recommended) and/or **Apple Notes** via AppleScript — `notes_backend` config | **Obsidian** (Markdown vault) and/or OneNote via COM/Graph — same config flag |
| Email the brief | Mail.app via AppleScript (send only, no foreground) | Outlook COM `MailItem.Send()` |
| Scheduler | launchd LaunchAgents (`StartInterval`) | Task Scheduler (`Register-ScheduledTask`, repetition interval) |
| Permissions model | TCC: Full Disk Access + Automation grants | Outlook "programmatic access" / Trust Center; Graph app registration + OAuth token |
| Brain invoke | `claude -p` | `claude -p` (same) |
| Atomic write | write `.tmp` then `mv` | write `.tmp` then `Move-Item -Force` |

Each prompt below states its macOS path in full and its Windows equivalent in a **Platform
note**. The session executes only the column matching the detected OS.

## Where things land (config-driven — `state/config.json`, chosen in Prompt 01)

- **Notes (the second brain)** → `notes_backend`: **`obsidian`** (Markdown vault, recommended
  primary — greppable, `[[backlinks]]`, syncable), **`apple`** (Apple Notes), or **`both`**.
  Holds the Daily/Weekly/Monthly/Quarterly briefs, the per-meeting notes, and the Goals note.
  Every skill calls the single `bin/upsert_note.sh` dispatcher and is backend-agnostic.
- **Commitments / tasks** → written to **BOTH**:
  - the Apple **Reminders** / Microsoft To Do app (so they show on his phone with due dates), AND
  - a "Tasks / Follow-ups" section inside that day's Daily Brief **note**.
- **Goals** → one living **Goals note** in the notes backend; every brief links to it.
- **Email delivery** → `email_brief.enabled` — the rendered brief is also emailed to the CEO.
- Everything stays **local** except that opt-in brief email (to himself, from his own account).
- Switching/migrating the backend later: optional helper `09_switch_notes_backend.md`.

## Directory layout (created on the CEO's Mac)

```
~/ceo-ai-executive-assistant/
  bin/                      # extractor + writer scripts
    extract_mail.applescript
    extract_calendar.applescript
    extract_messages.sh
    extract_whatsapp.sh
    extract_meetings.sh           # Otter/Fireflies/folder transcripts
    add_reminder.applescript
    upsert_note.sh                # backend dispatcher (apple | obsidian | both)
    upsert_note_apple.applescript
    upsert_note_obsidian.sh
    send_email.applescript        # opt-in brief delivery
    run_brain.sh                  # commitment-detector wrapper
    run_meeting_assistant.sh
    run_brief.sh                  # daily
    run_goal_detector.sh
    run_weekly_brief.sh / run_monthly_checkin.sh / run_quarterly_brief.sh
  state/                    # JSON snapshots + ledgers + config (gitignored, private)
    config.json             # notes_backend, vault path, reminder list, email_brief, goals_note_link
    mail.json / calendar.json / imessage.json / whatsapp.json / meetings.json
    processed.json          # ledger of already-actioned message/meeting/commitment IDs (dedup)
    briefs/<date>.md        # canonical Markdown copies of briefs
    meetings/<id>.md        # canonical meeting notes
    goals.md                # canonical Goals note
  logs/                     # scheduler stdout/stderr
  schedule/                 # launchd *.plist (mac) / Task Scheduler reg (win) + control script
  .claude/
    skills/
      commitment-detector/
      meeting-assistant/
      daily-brief/
      goal-detector/
      periodic-briefs/
      historical-backfill/
  reference/                # ARCHITECTURE.md, platform.md, config.md, accounts.md,
                            # wacli.md, integrations.md, message_schema.md, ROADMAP.md
  README.md
```

## The hard technical facts each prompt must respect

1. **Full Disk Access is mandatory.** Reading `~/Library/Messages/chat.db` (iMessage) and,
   in some macOS versions, Mail data, requires the *invoking binary* (Terminal / the
   `claude` binary / `osascript` host) to have **Full Disk Access** in System Settings →
   Privacy & Security. Automation permission (the per-app "allow control" prompts) is
   separate and also required for Mail/Calendar/Notes/Reminders/Messages.

2. **iMessage: read from `chat.db`, do NOT script Messages.app for reading.** Messages.app
   AppleScript can *send* but reliably *cannot read* history. The reliable read path is a
   SQLite query against `~/Library/Messages/chat.db`. (Needs Full Disk Access — see #1.)

3. **Mail.app must be running** for its AppleScript to work, and you iterate over
   `every account` → `mailbox "INBOX"` → `messages`. If the CEO uses **New Outlook** for
   any account, that account is invisible to AppleScript (New Outlook stripped the
   scripting dictionary) — note it and fall back to a Gmail/IMAP path or "Legacy Outlook".

4. **Calendar.app AppleScript is slow and flaky** over large date ranges. Prefer
   [`icalBuddy`](https://hasseg.org/icalBuddy/) if installed (`brew install ical-buddy`) —
   it's fast and emits clean text. Fall back to Calendar AppleScript scoped to a tight
   date window (today + 7 days) if icalBuddy isn't available.

5. **WhatsApp = `wacli`.** The exact flag surface is unknown to the author of these
   prompts. Every WhatsApp prompt must begin by running `wacli --help` (and
   `wacli <subcommand> --help`) to learn the real interface, then adapt. Assume it can
   list recent chats/messages and emit something parseable (ideally JSON; otherwise parse
   text). Do **not** hardcode flags without confirming them.

6. **Idempotency is non-negotiable.** Extractors run every ~5 min and the brain runs
   repeatedly. Every commitment must be keyed by a stable ID (message GUID / email
   Message-ID / WhatsApp message id) and recorded in `state/processed.json` so the same
   "email me the deck" never becomes three Reminders. Reminders creation also dedups by
   title+source as a second safety net.

7. **launchd, not cron.** Use LaunchAgents in `~/Library/LaunchAgents/` with
   `StartInterval`. Load with `launchctl bootstrap gui/$(id -u) <plist>`. cron is
   deprecated on macOS and runs in a sandbox that can't reach the Automation TCC grants.

8. **Token cost lives entirely in Tier 2.** Keep the brain prompt tight, feed it only the
   *new* (unprocessed) items, and keep its cadence modest.

## Commitment taxonomy (what the brain is hunting for)

| Direction | Trigger examples | Reminder created |
|-----------|------------------|------------------|
| **Inbound — they want something from him** | "email me…", "send me…", "can you share…", "ping me", "follow up with me", "message me tomorrow" | "Send <thing> to <name> (<contact>)" due per their timeframe |
| **Outbound — he promised something** | "I'll email you", "I'll call you next week", "let's catch up", "I'll send it over", "let me get back to you" | "Follow up with <name> (<contact>) re <thing>" due per his timeframe |

Each Reminder body must carry: **person name + contact handle (email / phone) + the channel
it came from + a one-line quote of the ask** so he knows exactly what to do without
re-opening the thread.

### Temporal reconciliation — never create an already-moot task

The brain must reason about the **timeline**, not just pattern-match phrases. A commitment
can already be fulfilled or obsolete by the time it's processed. Suppress (do NOT create a
reminder) when:

- **The meeting already happened.** Canonical case: an email 4 days ago said "let's meet",
  and the calendar shows a meeting with that person 2 days ago. The catch-up occurred →
  no task. (Match calendar events to the person by attendee email/name or title; the event
  must be dated *after* the ask and *before* now.)
- **He already replied / acted.** A later outbound message or sent email to the same person
  that plausibly satisfies the ask (sent the deck, answered the question) → handled.
- **A future event already covers it.** "Let's find time to meet" that already has a
  scheduled future calendar event with that person → scheduling done; at most a light
  "prep for <event>".

This is why the calendar extractor captures **past** events (−5 to −14 days), not just
upcoming ones. Every suppression is logged with its reason for audit. When genuinely unsure,
prefer flagging low-confidence over silently dropping real work.

## Meetings & goals — what the two newer brains do

- **Meeting-assistant** (`11`): for each meeting in `meetings.json`, reads the **full
  transcript** and writes its **own** TL;DR / decisions / action-items-by-owner / open
  questions / vendor-summary delta to a meeting note, and routes spoken commitments through the
  same Reminder + dedup pipeline. The vendor (Otter/Fireflies) summary is a cross-check only.
- **Goal-detector** (`12`): once daily, deep-analyses every signal (mail, WhatsApp, iMessage,
  **meeting transcripts**, calendar, notes, reminders) to infer goals, organized by
  **horizon** (weekly/monthly/yearly) × **category** (spiritual / knowledge & growth / personal
  / family / professional / other) into a single living **Goals note**. First run = full
  analysis; every later run **diffs** for new/progressed/stale goals, non-destructively. The
  daily brief links it; weekly/monthly/quarterly briefs (`13`) score progress against it.

## Build order (paste prompts in this sequence)

1. `01_project_init_and_permissions.md` — skeleton + **backend & integrations choices** +
   permissions + verify `wacli`.
2. `02_extract_mail_calendar.md`
3. `03_extract_messages_whatsapp.md`
4. `10_extract_meetings.md`  ← meeting transcripts (Tier 1)
5. `04_writers_reminders_notes.md`  ← reminder + backend-dispatching note writer
6. `05_commitment_detector_skill.md`
7. `11_meeting_assistant_skill.md`  ← own summaries from transcripts
8. `06_daily_brief_skill.md`
9. `12_goal_detector_skill.md`  ← daily deep goal analysis + diff
10. `13_periodic_briefs_and_checkins.md`  ← weekly/monthly/quarterly + goal check-ins
11. `14_email_delivery.md`  ← email the briefs
12. `08_scheduling_launchd.md`  ← wire up launchd once the pieces exist
13. `07_historical_backfill_2weeks.md`  ← one-time 2-week sweep, run manually
14. `09_switch_notes_backend.md` — optional, only to change/migrate the notes backend later.
