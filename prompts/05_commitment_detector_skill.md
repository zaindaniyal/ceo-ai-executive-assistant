# PROMPT 05 — Commitment-detector skill (Tier 2 brain)

> Paste into a fresh Claude Code session on the CEO's Mac. Build #5 — the heart of the
> system. Assumes Prompts 01–04 done (extractors producing JSON; writers working).

> **Platform note.** The brain is platform-agnostic reasoning, but it shells out to the
> Tier-3 writers from Prompt 04 — call whichever exist for this OS (`*.applescript` on macOS,
> `*.ps1` on Windows). Entrypoint is `bin/run_brain.sh` (macOS) or `bin/run_brain.ps1`
> (Windows). On Windows there is **no iMessage input** — just email + calendar + WhatsApp.

## Context

This is the **brain**. It is invoked by launchd (~every 30–60 min) as a headless run, reads
the Tier-1 JSON snapshots, finds **commitments**, and — for the ones that are still
actionable — creates Apple Reminders **and** appends them to today's Daily Brief note via
the Tier-3 writers. It must be **cheap, idempotent, and conservative** (a missed task is
recoverable; a flood of junk reminders gets the whole system turned off).

Build this as a Claude Code **skill** at `~/ceo-executive-assistant-ai/.claude/skills/commitment-detector/`
with a `SKILL.md` and any helper scripts. Also create `bin/run_brain.sh` that invokes it
headlessly (`claude -p` with the skill) for launchd to call.

## Inputs it reads

- `state/mail.json`, `state/imessage.json`, `state/whatsapp.json` — recent messages.
- `state/calendar.json` — events from −5 to +7 days (past events included on purpose).
- `state/processed.json` — the **dedup ledger**: source-ids already actioned. Load it,
  skip anything already in it, and append new ids after acting.

## What counts as a commitment

| Direction | Detected from | Examples | Reminder |
|-----------|---------------|----------|----------|
| **Inbound** (they want something from him) | messages/emails where `from_me=false` | "email me the deck", "send me that", "can you share…", "ping me Monday", "message me tomorrow" | "Send <thing> to <name> (<handle>)" |
| **Outbound** (he promised something) | where `from_me=true` (and his sent mail) | "I'll email you", "I'll call you next week", "let's catch up", "I'll send it over", "let me get back to you" | "Follow up with <name> (<handle>) re <thing>" |

For each detected commitment, extract: the **person** (name + best contact handle), the
**channel**, a **one-line verbatim quote** of the ask, and a **due date** inferred from the
language ("tomorrow", "next week", "Monday", "after the call" → best-effort ISO date; if
none, leave undated).

## ⚠️ Temporal reconciliation — DO NOT create stale/moot tasks

Be clever about time. A commitment can already be **fulfilled or obsolete** before this run.
Suppress it (don't create a reminder) when any of these hold:

1. **The meeting already happened.** If the commitment is about meeting/catching up with a
   person, and `calendar.json` shows an event **in the past** with that person (match by
   attendee email/name, or by title) dated **after** the message, the catch-up already
   occurred → **no task**. (This is the explicit case: an email 4 days ago said "let's
   meet", the meeting happened 2 days ago and is on the calendar → skip it.)
2. **He already replied / acted.** If a *later* outbound message or sent email to the same
   person plausibly satisfies the ask (e.g. inbound "send me the deck" on Mon, and an
   outbound message/email to them on Tue mentioning the deck / with an attachment), treat it
   as handled → **no task** (or a low-priority "confirm received" at most).
3. **A future event already covers it.** If "let's find time to meet" already has a
   **scheduled future** calendar event with that person, the scheduling is done → **no
   task** (optionally a light "prep for <event>" instead).
4. **The deadline has already passed harmlessly** and context shows it's dead — use
   judgment; when unsure, prefer creating the task (err toward not dropping real work), but
   add a note that it may be stale.

Always reason about the **timeline**: message timestamp vs. calendar event times vs. any
later reply. Cross-reference before writing. When you suppress something, log *why* (to
`logs/brain.log`) so we can audit the suppression decisions.

## Acting

For each **surviving** commitment not already in `processed.json`:
1. Call `bin/add_reminder.applescript` with title, context body (quote + name + handle +
   channel + original datetime + `src:<id>`), `dueISO`, list "Exec Assistant".
2. Call `bin/upsert_note.applescript` to add the same line to today's Daily Brief note,
   "Tasks / Follow-ups" section.
3. Append the source-id to `state/processed.json`.

Group multiple asks from the same person/thread sensibly; don't fragment one conversation
into five near-identical reminders.

## Cost & safety discipline

- Only feed yourself the **unprocessed** slice (filter out ids already in `processed.json`
  *before* reasoning) to keep token use low.
- Be **conservative**: only fire on a genuine, specific ask. Vague pleasantries ("we should
  hang out sometime!" with no actionable intent) → skip or mark low-confidence. False
  positives erode trust fast.
- Never send anything to the person. This system only creates reminders/notes for the CEO.
- Everything local.

## Verify before finishing

1. Run `bin/run_brain.sh` once against the current real snapshots. Show me the reminders it
   created and the Daily Brief "Tasks / Follow-ups" section.
2. **Prove temporal reconciliation:** construct or point to a case where a "let's meet"
   message predates a past calendar event with that person, and show in `logs/brain.log`
   that the brain **suppressed** it with a reason.
3. Run it a **second time** immediately → confirm it creates **zero** new reminders (dedup
   via `processed.json` works).
4. Show me 2–3 example reminder bodies so I can confirm they carry name + handle + quote.

Tune for precision over recall. Do not wire launchd yet (Prompt 08).
