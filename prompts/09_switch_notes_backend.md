# PROMPT 09 — (Optional) Switch / migrate the notes backend + backfill history

> Paste only if you want to **change** the notes backend after the fact, or **convert existing
> history** between Apple Notes and Obsidian. The backend is now chosen up front in Prompt 01
> (`notes_backend`: `obsidian` | `apple` | `both`) and the writers in Prompt 04 already support
> all three — so a fresh install does **not** need this. This is the move/convert helper.

## Context

The note writer (`bin/upsert_note.sh`, Prompt 04) dispatches on `state/config.json` →
`notes_backend`. Switching backend is therefore mostly a **config flip** — the value of this
prompt is doing it *safely* and **migrating the existing notes** so history is continuous and
nothing is lost in the switch. Apple **Reminders** stays the task surface regardless; only the
*note* backend moves.

## Tasks

1. **Confirm source + target.** Read the current `notes_backend`. Decide the target
   (`obsidian` ← → `apple`, or `both`). For Obsidian, confirm the vault path + folder from
   `config.json` (or capture them now and write them).
2. **Migrate existing notes** so history carries over:
   - **Apple Notes → Obsidian:** export each Daily Brief / Meeting / Goals note's body to a
     Markdown file in `<vault>/<folder>/…` with the same titles and `##` sections. Convert
     Apple Notes links to `[[wikilinks]]` where resolvable. The canonical
     `state/briefs/*.md` / `state/meetings/*.md` / `state/goals.md` copies (Prompt 04) make
     this mostly a file-copy.
   - **Obsidian → Apple Notes:** render each vault Markdown file to HTML and upsert it into the
     matching Apple Notes folder via `bin/upsert_note_apple.applescript`.
3. **Flip the flag.** Set `notes_backend` to the target (or `both`). Because every skill calls
   the single `upsert_note.sh` dispatcher, no skill code changes.
4. **Re-point the goals link.** Re-run the goal-detector (Prompt 12) once so `goals_note_link`
   is refreshed for the new backend (Obsidian uses `[[Goals]]`; Apple Notes uses the deep link).
5. **Keep a rollback.** Don't delete the old notes until a few days of clean runs confirm the
   new backend works. `both` is the safest transition state — run it for a week, then narrow.

## Verify

1. After migration, the target backend holds all historical Daily Briefs / Meeting notes /
   Goals note with sections + links intact.
2. Run the daily brief + goal-detector → they write to the new backend, idempotently
   (re-run = update in place, no duplicates), with working goal links.
3. Flip back to the original backend → still works (clean rollback), proving the dispatcher is
   the only thing that changed.

Keep it a config flip + a one-time history copy, not a rip-and-replace — so the CEO never
loses a day of briefs, meeting notes, or goals during the switch.
