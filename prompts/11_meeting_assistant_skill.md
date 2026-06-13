# PROMPT 11 — Meeting assistant skill (Tier 2 brain: own summaries + action items)

> Paste into a fresh Claude Code session on the CEO's machine. Build #7 in the live order
> (after the writers in Prompt 04 and the commitment-detector in Prompt 05 — it reuses both).
> Assumes Prompts 01–05 + 10 done (`state/meetings.json` populated; note + reminder writers
> working).

> **Platform note.** Pure reasoning that shells out to the Tier-3 writers chosen in Prompt 04
> (note writer → Apple Notes / Obsidian / both per `state/config.json`; reminder writer). Same
> entrypoint pattern: `bin/run_meeting_assistant.sh` (macOS) / `.ps1` (Windows).

## Context

This is the **meeting brain**. For every meeting in `state/meetings.json`, it reads the
**full transcript** and forms its **own** summary, decisions, and action items — and only then
glances at the vendor summary to catch anything it missed. It writes a clean **meeting note**
to the second brain and routes any commitments through the **same** reminder + dedup pipeline
as the commitment-detector.

> ⚠️ **The core instruction the CEO gave: do NOT just trust the Fireflies/Otter summary.**
> Read the transcript yourself, from his perspective as his executive assistant. The vendor
> summary is a *cross-check*, not the source of truth. If the vendor missed an ask or a
> decision that matters to him, you catch it.

Build it as a skill at `~/ceo-ai-executive-assistant/.claude/skills/meeting-assistant/` with a
`SKILL.md`, plus `bin/run_meeting_assistant.sh` for the scheduler to call headlessly.

## Inputs

- `state/meetings.json` — full transcripts + vendor summaries (Prompt 10).
- `state/calendar.json` — to tie a transcript to its calendar event (attendees, project) and
  for temporal reconciliation of any commitments.
- `state/processed.json` — dedup ledger. Skip meeting ids already summarized; skip commitment
  ids already actioned. Append new ids after acting.
- `state/config.json` — `notes_backend` (apple | obsidian | both), reminder list name.

## What it produces per meeting

For each **unprocessed** meeting, generate — **from the transcript, in your own words**:

1. **TL;DR** — 3–5 bullets: what this meeting was actually about and what changed.
2. **Decisions made** — concrete decisions + who owns each. (Vendors routinely miss these.)
3. **Action items**, split by owner:
   - **The CEO's** — these become commitments (see below).
   - **Others'** — things he's waiting on from someone (track as "awaiting X from <name>").
4. **Open questions / risks** — anything unresolved he should know about.
5. **Notable quotes** — 1–2 verbatim lines that carry weight (commitments, numbers, dates).
6. **Vendor-summary delta** — a short note of anything the Fireflies/Otter summary got wrong
   or omitted vs your read. (Proof you didn't just copy theirs.)

Write this as a meeting note via the configured note writer:
- **Title:** `Meeting — <title> — YYYY-MM-DD`.
- **Location:** a "Meetings" folder (Apple Notes) / `CEO Assistant/Meetings/` (Obsidian vault).
- Link it from that day's Daily Brief (Prompt 06 surfaces "Meetings today" with links).
- Idempotent: re-running must update the same note, not create a second.

## Commitments from meetings → same pipeline as Prompt 05

Spoken commitments count exactly like typed ones:
- **Outbound (he promised):** "I'll send you the deck", "I'll get back to you Friday",
  "let me loop in legal" → `Follow up with <name> (<handle>) re <thing>`.
- **Inbound (they asked him):** "can you share the numbers", "send me the contract" →
  `Send <thing> to <name> (<handle>)`.

For each: extract person + best contact handle (from the attendee list / calendar), a
**verbatim quote** from the transcript, the channel (`meeting`), and an inferred due date.
Then run the **same temporal reconciliation** as Prompt 05 (don't create a task already made
moot by a later event/reply) and act:
1. `add_reminder` with the context body (quote + name + handle + `channel: meeting` +
   meeting title/date + `src:<id>`).
2. Append to today's Daily Brief "Tasks / Follow-ups" via the note writer.
3. Append the source id to `state/processed.json`.

Use a stable per-commitment id derived from the meeting id + a hash of the quote so re-runs
dedup cleanly.

## Discipline

- **Precision over recall** — same as the rest of the system. A meeting can throw off dozens
  of lines; only the genuine, owner-attributed asks become reminders.
- Don't fragment: one person making three related asks in one meeting → group sensibly.
- Cheap: summarize from the transcript; don't echo the whole transcript back into context.
- Local only. The transcript stays on the machine.

## Verify before finishing

1. Run `bin/run_meeting_assistant.sh` against current `state/meetings.json`. Show me one
   generated meeting note — TL;DR, decisions, action items by owner, and the **vendor-summary
   delta** (proving you read the transcript, not just their summary).
2. Show the reminders it created from spoken commitments (name + handle + verbatim quote + due).
3. Run it **again** → zero new notes, zero new reminders (dedup via `processed.json`).
4. Show one suppression in `logs/meeting_assistant.log` if temporal reconciliation dropped a
   moot ask.

Do not wire the scheduler yet (Prompt 08).
