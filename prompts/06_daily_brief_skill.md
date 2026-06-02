# PROMPT 06 — Daily Brief skill (Tier 2 brain)

> Paste into a fresh Claude Code session on the CEO's Mac. Build #6. Assumes Prompts 01–05.

> **Platform note.** Writes via the Tier-3 note writer chosen in Prompt 04 (Apple Notes on
> macOS; OneNote or a synced Markdown file on Windows). Entrypoint `bin/run_brief.sh` (macOS)
> or `bin/run_brief.ps1` (Windows). On Windows there's no iMessage source to summarize.

## Context

Each morning, the brain assembles a one-screen **Daily Brief** into Apple Notes so the CEO —
who's usually mid-travel and back-to-back — gets his whole day in one place. Same Tier-2
rules: cheap, reads the Tier-1 JSON snapshots, writes via the Tier-3 `upsert_note`
helper. Build it as a skill at `~/ceo-ai-executive-assistant/.claude/skills/daily-brief/` plus a
`bin/run_brief.sh` launchd entrypoint.

## Inputs

- `state/calendar.json` (today's events — and note conflicts/double-bookings).
- `state/mail.json`, `state/imessage.json`, `state/whatsapp.json` (overnight + recent).
- `state/processed.json` and the day's "Tasks / Follow-ups" already created by the
  commitment-detector — so the brief reflects open follow-ups, not a re-detection.

## Output — one note, today's date, in "Daily Briefs"

Use `bin/upsert_note.applescript` to write/refresh today's `Daily Brief — YYYY-MM-DD` note.
Sections (H2s), kept tight — this is a glance, not a report:

1. **Today's Schedule** — chronological timeline of today's events across all calendars,
   with times, titles, locations, and **flagged conflicts / double-bookings**. Note the
   first hard commitment of the day and any travel/buffer risk.
2. **Top 3 Priorities** — the 3 things that actually matter today, inferred from
   calendar + open follow-ups + urgent inbound. Opinionated, short.
3. **Tasks / Follow-ups** — the open commitments for today/overdue (read these from the
   section the commitment-detector maintains; don't recompute). Each with name + handle.
4. **Needs a Decision / Reply** — inbound emails or messages that look like they're waiting
   on *him* specifically (questions, approvals, scheduling) — with who + one-line summary.
5. **Heads-up** — flights/travel, anything time-sensitive in the next 24–48h.

Keep the whole thing to roughly one screen. Prefer bullets over prose. If a section is
empty, show it with "— none —" rather than dropping it (consistency helps him scan).

## Discipline

- **Don't duplicate the commitment-detector's job.** The brief *surfaces* existing
  follow-ups; it doesn't create reminders. (If it genuinely spots a brand-new commitment the
  detector missed, it may add it — but route through the same `add_reminder` +
  `processed.json` path so dedup holds.)
- Be **honest about blind spots**: if `mail.json` notes a skipped New-Outlook account, add a
  one-line "⚠ <account> not visible to the assistant" so he knows the brief isn't omniscient.
- Cheap: summarize, don't dump full message bodies into your own context unnecessarily.
- Local only.

## Verify

1. Run `bin/run_brief.sh` once. Open the resulting Apple Note and show me its content.
2. Confirm it reads today's real calendar + open follow-ups and renders all five sections,
   conflicts flagged, blind spots noted.
3. Run again → confirm it **updates in place** (idempotent upsert), not a second note.

Do not wire launchd yet (Prompt 08).
