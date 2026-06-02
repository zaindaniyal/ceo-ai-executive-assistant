# CEO Executive Assistant AI — Architecture Reference

> Read this once before pasting any prompt. It is the shared mental model that every
> prompt in `../prompts/` assumes. The prompts each repeat a short version of this, but
> this is the canonical description.

## The problem

The the CEO runs hot: ~4 calendars, ~4 email accounts, WhatsApp, iMessage, constant
travel and back-to-back meetings. Things fall through the cracks — specifically **verbal
and written commitments**:

- Someone tells him "email me the deck" / "send me that contract" → he must follow up.
- He tells someone "I'll call you next week" / "let's catch up" → he must follow up.

He needs (a) a **daily brief** and (b) **nothing to fall through the cracks**.

## Design principle: split "extract" from "brain"

Running a full Claude session every 5 minutes is expensive and unnecessary. So the system
has two tiers:

```
┌─────────────────────────────────────────────────────────────────────┐
│  TIER 1 — EXTRACTORS  (plain AppleScript / shell, fired by launchd)   │
│  Cheap, dumb, frequent (~every 5 min). No LLM. Just dump raw data.    │
│                                                                       │
│   extract_mail.applescript      → state/mail.json                     │
│   extract_calendar.applescript  → state/calendar.json                 │
│   extract_messages.sh (chat.db) → state/imessage.json                 │
│   extract_whatsapp.sh (wacli)   → state/whatsapp.json                 │
└─────────────────────────────────────────────────────────────────────┘
                                │  (raw JSON snapshots on disk)
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  TIER 2 — BRAIN  (headless `claude -p`, fired by launchd less often)  │
│  Reads the JSON snapshots, reasons, acts. Costs tokens, so runs       │
│  ~every 30–60 min + once at brief time, NOT every 5 min.              │
│                                                                       │
│   commitment-detector  → finds promises → Reminders + Notes           │
│   daily-brief          → builds the morning brief → Apple Notes        │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  TIER 3 — WRITERS  (AppleScript helpers the brain shells out to)     │
│   add_reminder.applescript   → Apple Reminders                        │
│   upsert_note.applescript    → Apple Notes (Daily Brief, dated)       │
└─────────────────────────────────────────────────────────────────────┘
```

**Why this split matters:** the extractors are deterministic and free to run constantly.
The brain is the only thing that costs money, so we control its cadence independently.

## Platform abstraction (macOS **or** Windows)

This assistant runs on whichever machine the executive uses. The **architecture is
identical** on both; only the *scripting stack* differs. **Detect the OS once** (Prompt 01,
recorded in `reference/platform.md`) and then commit fully:

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
| Reminders / Tasks | Apple Reminders via AppleScript | Microsoft To Do via Graph, or Outlook Tasks via COM |
| Notes (Daily Brief) | Apple Notes via AppleScript | OneNote via COM/Graph, **or** a Markdown file in a synced folder (simplest, recommended fallback) |
| Scheduler | launchd LaunchAgents (`StartInterval`) | Task Scheduler (`Register-ScheduledTask`, repetition interval) |
| Permissions model | TCC: Full Disk Access + Automation grants | Outlook "programmatic access" / Trust Center; Graph app registration + OAuth token |
| Brain invoke | `claude -p` | `claude -p` (same) |
| Atomic write | write `.tmp` then `mv` | write `.tmp` then `Move-Item -Force` |

Each prompt below states its macOS path in full and its Windows equivalent in a **Platform
note**. The session executes only the column matching the detected OS.

## Where things land (decided)

- **Daily Brief** → Apple **Notes**, one dated note per day in a "Daily Briefs" folder.
- **Commitments / tasks** → written to **BOTH**:
  - the Apple **Reminders** app (so they show on his phone with due dates), AND
  - a "Tasks / Follow-ups" section inside that day's Daily Brief **note**.
- Everything stays **local**. No message/email content leaves the Mac.
- (Future) migrate Notes → Obsidian. Not now. See `09_obsidian_future.md`.

## Directory layout (created on the CEO's Mac)

```
~/ceo-executive-assistant-ai/
  bin/                      # extractor + writer scripts
    extract_mail.applescript
    extract_calendar.applescript
    extract_messages.sh
    extract_whatsapp.sh
    add_reminder.applescript
    upsert_note.applescript
    run_brain.sh            # wrapper that invokes `claude -p` with the right skill
  state/                    # JSON snapshots + dedup ledgers (gitignored, private)
    mail.json
    calendar.json
    imessage.json
    whatsapp.json
    processed.json          # ledger of already-actioned message IDs (dedup)
  logs/                     # scheduler stdout/stderr
  schedule/                 # launchd *.plist (mac) / Task Scheduler reg (win) + control script
  .claude/
    skills/
      commitment-detector/
      daily-brief/
      historical-backfill/
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

## Build order (paste prompts in this sequence)

1. `01_project_init_and_permissions.md` — skeleton + permissions + verify `wacli`.
2. `02_extract_mail_calendar.md`
3. `03_extract_messages_whatsapp.md`
4. `04_writers_reminders_notes.md`
5. `05_commitment_detector_skill.md`
6. `06_daily_brief_skill.md`
7. `08_scheduling_launchd.md`  ← wire up launchd once the pieces exist
8. `07_historical_backfill_2weeks.md`  ← one-time 2-week sweep, run manually
9. `09_obsidian_future.md` — later, when migrating off Apple Notes.
