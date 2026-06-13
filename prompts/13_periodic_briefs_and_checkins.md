# PROMPT 13 — Weekly / monthly / quarterly briefs + goal check-ins (Tier 2 brain)

> Paste into a fresh Claude Code session on the CEO's machine. Build #10 in the live order.
> Assumes Prompts 01–06 + 10–12 done (daily brief working; Goals note maintained by Prompt 12).

> **Platform note.** Pure reasoning + the configured note writer (Apple Notes / Obsidian /
> both) and, optionally, the email sender from Prompt 14. Entrypoints `bin/run_weekly_brief.sh`,
> `bin/run_monthly_checkin.sh`, `bin/run_quarterly_brief.sh` (`.ps1` on Windows).

## Context

The daily brief (Prompt 06) is a glance at *today*. This prompt adds the **zoom-out**
cadence — week, month, quarter — so the CEO sees trajectory, not just the next 12 hours, and
so his **goals get a recurring check-in** instead of quietly drifting. Each brief is a dated
note in the same second brain, and each **links to the Goals note** (Prompt 12) when it
reports progress.

Build three small skills (or one parameterized skill with a `--period` arg) under
`~/ceo-ai-executive-assistant/.claude/skills/periodic-briefs/`, plus the three `bin/` wrappers.

## Shared inputs

- The week's/month's/quarter's **daily briefs** (read them from the notes backend or from
  `state/briefs/*.md` if the writer keeps Markdown copies — see Prompt 04).
- `state/meetings.json` history + meeting notes (what actually happened).
- `state/calendar.json` (widen the pull for these runs — see cadence below).
- The **Goals note** (Prompt 12) — the spine of every check-in.
- `state/processed.json` and the tasks created over the period (completed vs still-open).

> These longer windows need more history than the live 3-day extractors hold. Have each
> periodic run either widen its extractor window for that run, or read accumulated
> `state/briefs/*.md` + meeting notes. Don't try to reconstruct a quarter from a 3-day snapshot.

## A — Weekly brief (`run_weekly_brief.sh`)

One note, `Weekly Brief — <ISO week, e.g. 2026-W24>`, in a "Weekly Briefs" location. Sections:

1. **The week in one paragraph** — what actually moved.
2. **Goal progress** — for each **Weekly** goal (from the Goals note): on track / slipped /
   done, with the evidence. Link the Goals note (`[[Goals]]` / Apple Notes link).
3. **Wins & decisions** — pulled from the week's meetings + briefs.
4. **Slipping / overdue** — follow-ups created this week and not closed; meetings with no
   follow-through.
5. **Where the time went** — a rough calendar breakdown (meeting hours vs focus blocks),
   flagging overload.
6. **Set up next week** — 3–5 priorities for the coming week, opinionated.

## B — Monthly check-in (`run_monthly_checkin.sh`)

One note, `Monthly Check-in — YYYY-MM`. Focused on the **Monthly** (and rolled-up Weekly)
goals from the Goals note:

1. **Goal scorecard** — every Monthly goal × category (Spiritual / Knowledge & Growth /
   Personal / Family / Professional / Other): status + a one-line "how it's going". Link
   `[[Goals]]`.
2. **Trajectory** — themes across the four weekly briefs; what's compounding, what's stalling.
3. **Course corrections** — concrete suggestions where a goal is slipping.
4. **Surfacing drift** — goals stated but unworked; time spent on things tied to no goal.

It may also **write back** to the Goals note's Changelog (via Prompt 12's writer/logic) noting
the month's progress — but it must not duplicate goals; it only annotates.

## C — Quarterly brief (`run_quarterly_brief.sh`)

One note, `Quarterly Brief — YYYY-Qn`. The big zoom-out:

1. **The quarter in review** — what was achieved against **Yearly** goals (proportional
   progress), category by category. Link `[[Goals]]`.
2. **Highlight reel** — the handful of things that genuinely mattered (meetings, decisions,
   wins).
3. **What didn't happen** — yearly goals with little/no movement, honestly.
4. **Next quarter's bets** — 3–5 focus areas to carry the yearly goals forward.

## Cross-cutting rules

- **Idempotent:** re-running a given week/month/quarter updates that one note, never spawns a
  duplicate.
- **Link, don't copy, the goals.** The Goals note (Prompt 12) is the single source of truth;
  briefs reference and report against it.
- **Optional email delivery.** If `state/config.json` → `email_brief.enabled` is true and
  `email_brief.periods` includes this period, hand the rendered brief to the Prompt 14 sender
  so it lands as a notification. Daily stays the default; weekly/monthly/quarterly are opt-in.
- Cheap and honest — summarize, flag blind spots (skipped accounts, missing transcripts),
  don't pad.

## Verify before finishing

1. Run `run_weekly_brief.sh` → a Weekly Brief note with all six sections, goal progress
   reported against the Goals note, and a working link to it.
2. Run `run_monthly_checkin.sh` → goal scorecard by category, links to `[[Goals]]`, no
   duplicate goals introduced.
3. Run `run_quarterly_brief.sh` → quarter-in-review against Yearly goals.
4. Re-run each → updates in place, no duplicate notes.

Scheduling (Prompt 08): weekly Sun/Mon morning, monthly on the 1st, quarterly on the 1st of
Jan/Apr/Jul/Oct.
