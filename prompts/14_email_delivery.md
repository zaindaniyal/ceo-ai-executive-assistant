# PROMPT 14 — Email the briefs (Tier 3 sender + wiring)

> Paste into a fresh Claude Code session on the CEO's machine. Build #11 in the live order.
> Assumes Prompts 01–06 done (daily brief working) and ideally 12–13 (goals + periodic briefs).

> **Platform note.** macOS → send via **Mail.app AppleScript** (`bin/send_email.applescript`),
> reusing an existing Mail account. Windows → send via **Outlook COM** (`Send-Email.ps1`,
> `$ol.CreateItem(0)` → `.Send()`). Same contract and same `state/config.json` settings on both.

## Context

The CEO wants the brief to **reach him as a notification**, not sit in a note he has to
remember to open. Email is the universal push — it hits his phone, watch, and laptop. So after
the daily brief (and optionally the weekly/monthly/quarterly briefs) is written to the second
brain, we **also email it to him**.

> **On the "local only" rule.** The whole system is local by design. Emailing the brief is the
> **one deliberate, opt-in exception the CEO asked for** — and it only ever sends *to himself*,
> using *his own* mail account already on the machine. No third-party service, no content to
> anyone else. Gate it behind `state/config.json` → `email_brief.enabled` so it's explicit.

## Step 1 — confirm the send path (done in Prompt 01, re-verify here)

- **macOS:** confirm a Mail.app account can send via AppleScript without popping a window
  (`make new outgoing message` → `send`). Pick which account to send **from** (his primary).
- **Windows:** confirm Outlook COM can create + send a `MailItem` silently.
- Record in `state/config.json`:
  ```json
  "email_brief": {
    "enabled": true,
    "to": "ceo@example.com",
    "from_account": "Personal",
    "periods": ["daily"],
    "format": "html"
  }
  ```
  `periods` controls which briefs get emailed (`daily` default; add `weekly`/`monthly`/`quarterly`).

## Step 2 — `bin/send_email.applescript` (Tier-3 sender)

A small, dumb writer the brief skills shell out to. Arguments: `to`, `subject`,
`htmlBody` (or a path to an HTML/Markdown file), `fromAccount` (optional).

- Build an outgoing message, set sender to `from_account`, set the recipient, set the
  HTML content, **send without activating Mail** (no foreground window).
- Wrap in `try`; exit non-zero + log to `logs/email.log` on failure (so a send failure never
  silently swallows the brief — the note is still written regardless).
- Idempotency: emailing the *same* brief twice in one day is annoying but not corrupting. Guard
  with a per-day sentinel in `state/` (`state/emailed/<period>-<date>.sent`) so a re-run /
  brief refresh doesn't re-send unless content materially changed. Re-send only on `--force`.

## Step 3 — render the brief for email

The brief skills already produce the note content. For email:
- Convert the brief (Markdown/HTML) into a clean, **mobile-readable HTML email**: the
  schedule, top priorities, follow-ups, decisions-needed, heads-up, and — when present — a
  **Goals & Progress** line.
- **Subject:** `☀️ Daily Brief — Fri 13 Jun` (period + date). Weekly/monthly/quarterly get
  their own subjects.
- Include a **link back to the full note** in the second brain (the Apple Notes deep link or
  the Obsidian URI / file path) so a tap opens the canonical version. Reuse the
  `goals_note_link` pattern from Prompt 12 for the Goals link inside the email.

## Step 4 — wire it into the brief skills

- In the **daily-brief** skill (Prompt 06): after the note upsert, if
  `email_brief.enabled && periods includes "daily"`, call `send_email` with the rendered brief.
- In **periodic-briefs** (Prompt 13): same, gated on the matching period.
- The note write is the source of truth; email is a **secondary delivery** — if email fails,
  log it and move on, never block or duplicate the note.

## Verify before finishing

1. Set `email_brief.enabled = true`, run the daily brief → confirm an email arrives on the
   CEO's phone with a readable schedule + priorities + follow-ups + goals line + a working
   link back to the note.
2. Re-run the brief same day → confirm it does **not** re-send (sentinel works), and that
   `--force` does re-send.
3. Confirm Mail/Outlook never came to the foreground during send.
4. Set `enabled = false`, run → note still written, **no** email sent (clean opt-out).

Scheduling: no new job — email piggybacks on the existing brief jobs (Prompt 08).
