# PROMPT 09 — (Future) Migrate Apple Notes → Obsidian

> Paste later, once the Apple Notes flow is proven and the CEO wants notes in Obsidian.
> Not part of the initial build. Apple Notes stays the destination until this is run.

## Context

The assistant currently writes the Daily Brief and "Tasks / Follow-ups" into **Apple
Notes** via `bin/upsert_note.applescript`. The eventual home is **Obsidian** (Markdown
vault, backlinks, greppable, syncable). This prompt swaps the destination with minimal
disruption.

## Tasks

1. **Confirm the vault** — get the Obsidian vault path on this machine and a target folder
   (e.g. `CEO Assistant/Daily Briefs/`). Decide a filename convention (e.g.
   `Daily Brief — YYYY-MM-DD.md`).
2. **New writer `bin/upsert_note_obsidian.sh`** — writes/updates a Markdown file per day
   directly in the vault (plain file I/O — far simpler and more reliable than the Notes
   AppleScript HTML dance). Idempotent section upsert in Markdown; dedup "Tasks /
   Follow-ups" by the `src:<id>` marker, exactly as before.
3. **Point the brain at it** — switch the `daily-brief` and `commitment-detector` skills
   from `upsert_note.applescript` to `upsert_note_obsidian.sh` behind a single config flag
   (e.g. `state/config.json` → `"notes_backend": "obsidian"`), so we can flip back if needed.
4. **Backfill (optional)** — convert existing Apple Notes Daily Briefs to Markdown files in
   the vault so history is continuous.
5. **Reminders unchanged** — Apple Reminders stays the task surface (syncs to his phone).
   Only the *note* backend moves.

## Verify

1. Run the brain with the Obsidian backend → a dated `.md` appears in the vault with the
   same five sections, conflicts flagged, tasks deduped.
2. Re-run → file updates in place, no duplicate task lines.
3. Confirm flipping `notes_backend` back to `apple` still works (clean rollback).

Keep it a config flip, not a rip-and-replace — so the CEO never loses a day of briefs during
the switch.
