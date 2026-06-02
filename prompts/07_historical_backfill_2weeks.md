# PROMPT 07 — One-time historical backfill (last 2 weeks)

> Paste into a fresh Claude Code session on the CEO's Mac. Build #8 — run AFTER the live
> system (Prompts 01–06 + 08) is verified. This is a **one-time** manual sweep, not a
> scheduled job.

> **Platform note.** Reuses the OS-appropriate extractors and writers from earlier prompts.
> On **Windows** the email/calendar pull is via Outlook COM and there is **no iMessage
> source** — sweep email + calendar + WhatsApp only.

## Context

Before the live assistant existed, two weeks of commitments piled up unactioned. Sweep the
**last 14 days** of WhatsApp + all email + iMessage and create Apple Reminders (+ Daily
Brief "Tasks / Follow-ups" lines) for every still-open commitment — using the **same
detection, temporal-reconciliation, and dedup logic** as the live commitment-detector
(Prompt 05). This is exactly that brain, run once over a wider window.

## Steps

### 1. Pull a 14-day window from every channel

Reuse the existing extractors but widen the window to 14 days for this run (pass a date
arg, or write `state/backfill_*.json` snapshots so you don't disturb the live 3-day files):
- **WhatsApp** via `wacli` — 14 days, all active chats. (Mind `wacli`'s windowing flags;
  page through if needed.)
- **Email** — all accounts' INBOX **and Sent** for 14 days (Sent matters for outbound
  promises and for detecting he already replied).
- **iMessage** — chat.db, 14 days, both directions.
- Also pull `calendar.json` spanning **−14 to +14 days** — you need past events (to suppress
  meetings that already happened) and future events (to suppress already-scheduled catch-ups).

### 2. Detect commitments (same taxonomy as Prompt 05)

- **Inbound:** "email me", "send me…", "share…", "ping me", "message me tomorrow" →
  he owes them something.
- **Outbound:** "I'll email/call you", "let's catch up", "I'll send it", "let me get back to
  you" → he promised something.
- Capture person name + contact handle + channel + verbatim quote + inferred due date.

### 3. ⚠️ Temporal reconciliation — critical for a backfill

Over a 14-day window, **most** "let's meet" asks have already resolved. Suppress aggressively
but correctly. For each candidate, walk the timeline and DROP it if:
- A **past calendar event** with that person exists dated **after** the ask → the meeting
  happened. (The canonical case: email 4 days ago "let's meet", event 2 days ago on the
  calendar → no task.)
- A **later reply / sent email / outbound message** to that person plausibly fulfilled the
  ask (e.g. he already sent the deck) → handled.
- A **future scheduled event** already covers a "let's find time" ask → scheduling done.
- The thread shows it was explicitly closed/cancelled.

Keep a task only if, as of **today**, it's genuinely still open. Log every suppression with
its reason to `logs/backfill.log` — I want to review what was dropped and why.

### 4. Respect the live dedup ledger

Load `state/processed.json`. Skip anything the live system already actioned. Append every
new id you act on, so the live brain won't re-create them later. (The 3-day live window and
14-day backfill overlap — dedup prevents doubles.)

### 5. Create the tasks — but show me first

Because this is a bulk one-time action, **don't auto-create silently**. First produce a
review table:

| # | Direction | Person (handle) | Channel | The ask (quote) | Due | Keep? | Suppressed-reason (if dropped) |

Show me the full list — kept and suppressed. Let me approve / strike rows. **Then** create
the approved reminders via `add_reminder.applescript` and append them to today's Daily Brief
note, and update `processed.json`.

## Verify

1. Show the review table (kept vs suppressed with reasons).
2. After approval, confirm N reminders created in "Exec Assistant", each with name + handle +
   quote + channel in the body.
3. Confirm `processed.json` now contains all backfilled ids (so the live brain won't dupe).
4. Spot-check 2–3 suppressions in `logs/backfill.log` to confirm the timeline logic is sound
   (e.g. show the "meeting already happened" drop).

Precision matters more here than in the live loop — a 14-day dump of false tasks would bury
the real ones. When unsure whether something's still open, flag it as **low-confidence** in
the table rather than dropping or blindly creating.
