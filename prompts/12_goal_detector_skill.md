# PROMPT 12 — Goal-detection skill (Tier 2 brain: daily deep analysis + diffing)

> Paste into a fresh Claude Code session on the CEO's machine. Build #9 in the live order.
> Assumes Prompts 01–06 + 10–11 done (all extractors producing JSON; note writer working;
> meeting transcripts available).

> **Platform note.** Pure reasoning + the configured note writer (Apple Notes / Obsidian /
> both per `state/config.json`). Entrypoint `bin/run_goal_detector.sh` (macOS) / `.ps1`
> (Windows). Inputs differ only in that Windows has no iMessage source.

## Context

Once a day, the brain steps back from individual tasks and asks a bigger question: **what is
the CEO actually trying to achieve?** It performs a **deep analysis across every available
signal**, infers his goals, organizes them, and maintains a single living **Goals** note in
the second brain. Each subsequent day it re-reads everything, **diffs** against the existing
Goals note, and folds in anything new — without clobbering history or duplicating goals.

This is a Tier-2 brain skill at
`~/ceo-ai-executive-assistant/.claude/skills/goal-detector/` with a `SKILL.md` and
`bin/run_goal_detector.sh`. It runs **once a day** (scheduled in Prompt 08), separate from the
commitment loop — goal inference is a slower, more reflective pass.

## Inputs — read everything, deeply

- `state/mail.json` — email across all accounts (intent, projects, recurring themes).
- `state/whatsapp.json`, `state/imessage.json` — personal + family + informal signals.
- `state/meetings.json` — **full meeting transcripts** (often where real ambitions surface).
- `state/calendar.json` — where his time actually goes (revealed priorities ≠ stated ones).
- The current **Apple Notes / Obsidian** notes the assistant maintains (briefs, meeting
  notes) and, if reachable, the CEO's own notes — read what's there to ground the inference.
- `state/reminders_snapshot.json` — a dump of the current Apple Reminders / To Do (have the
  reminder writer expose a read mode, or add a tiny `bin/dump_reminders.applescript`). His
  open tasks are strong goal signal.

> "Author transcriptions" in the original spec = the **meeting transcripts** above. If the CEO
> also dictates voice memos, point this skill at that folder too (record it in
> `reference/integrations.md`).

## The Goals note — structure

Maintain **one** living note, `Goals — <CEO name>`, in a "Goals" location (Apple Notes folder
"Goals" / Obsidian `CEO Assistant/Goals.md`). Two axes:

**Horizon** × **Category**.

- Horizons: **Weekly**, **Monthly**, **Yearly**.
- Categories (use these exact buckets; add "Other" only when something genuinely fits nowhere):
  - **Spiritual**
  - **Knowledge & Growth** *(the "knowledge motion" bucket — learning, skills, craft, mastery)*
  - **Personal** *(health, habits, finances, self)*
  - **Family**
  - **Professional / Business**
  - **Other**

Layout (Markdown for Obsidian; the Apple Notes writer renders the same to HTML):

```markdown
# Goals — <CEO name>
_Last deep analysis: 2026-06-13 · maintained by the Executive Assistant_

## Yearly
### Spiritual
- [ ] <goal>  · _inferred 2026-06-13 · evidence: <source/quote>_
### Knowledge & Growth
- [ ] …
### Personal
### Family
### Professional / Business
### Other

## Monthly
### Spiritual
…

## Weekly
### Spiritual
…

## Changelog
- 2026-06-13 — initial deep analysis: N goals inferred.
- 2026-06-14 — +2 new (Family: …, Knowledge: …); 1 marked done.
```

Every goal line carries: a checkbox, the goal, the date inferred, and a **one-line evidence
pointer** (which source + a short quote) so he can see *why* the assistant thinks this is a
goal. Never invent goals with no evidence — when a signal is weak, mark it `(tentative)`
rather than asserting it.

## Two modes: first run vs daily diff

**First run (no Goals note exists):**
- Do the **deep analysis** across all inputs. Cluster recurring themes into goals, sort into
  horizon × category, write the full Goals note. Seed the Changelog with "initial deep
  analysis".

**Every subsequent daily run (Goals note exists):**
- Re-read all inputs. Re-derive the candidate goal set.
- **Diff** against the existing note:
  - **New goals** not already present → add them under the right horizon/category, dated, with
    evidence. Append a Changelog line.
  - **Apparent progress / completion** (a goal clearly achieved per recent signal) → check it
    off (`[x]`) and note it in the Changelog. Don't delete — history stays.
  - **Stale / abandoned** (no signal for a long time, or explicitly dropped) → mark
    `~~struck~~ (stale, no signal since <date>)`, don't silently remove.
  - **Refined** (a vague goal now has specifics) → update it in place, note the refinement.
- **Idempotent & non-destructive:** running twice in a day must not duplicate goals or
  re-write the Changelog twice. Key goals by a normalized text hash to detect "already present".

## Linking — so the briefs can point here

The Daily/Weekly/Monthly/Quarterly briefs (Prompts 06, 13) **link to this Goals note** when
they talk about progress. Make the note linkable both ways:
- **Obsidian:** the note is `Goals.md`; briefs reference it as `[[Goals]]`. Trivial.
- **Apple Notes:** expose the note's link. Best path: after upsert, capture the note's
  `applenotes:` deep link (or its id) and write it to `state/config.json` →
  `"goals_note_link"`, so the brief writer can embed a real tappable link. If the deep link
  proves unreliable, fall back to referencing it by exact title (`see “Goals — …” note`).

## Verify before finishing

1. **First run:** delete/rename any test Goals note, run `bin/run_goal_detector.sh`. Show me
   the created Goals note — all three horizons, all categories, each goal with an evidence
   pointer. Confirm categories match the spec (Spiritual / Knowledge & Growth / Personal /
   Family / Professional / Other).
2. **Diff run:** run it again the same day → **no duplicates**, Changelog not double-written.
3. Simulate a new goal (e.g. add a message/meeting line implying one) and re-run → it appears
   under the right bucket with a Changelog entry, nothing else disturbed.
4. Confirm `state/config.json` now has `goals_note_link` (or the agreed fallback) so briefs
   can link here.

Do not wire the scheduler yet (Prompt 08). This runs once daily there.
