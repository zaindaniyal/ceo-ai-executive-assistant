# PROMPT 10 — Meeting transcript extractor (Tier 1: Otter / Fireflies)

> Paste into a fresh Claude Code session on the CEO's machine. Build #4 in the live order
> (right after the message extractors, before the writers). Assumes Prompts 01–03 done.
> Read `~/ceo-ai-executive-assistant/reference/integrations.md` first (created in Prompt 01).

> **Platform note.** This source is **cloud-based and OS-agnostic** — Otter and Fireflies are
> reached over HTTP or via email/synced files, not via AppleScript or COM. Build it as a
> shell script (`bin/extract_meetings.sh`) on macOS and a `.ps1` on Windows that produce the
> **same `state/meetings.json` shape**. Atomic write either way (`mv` / `Move-Item -Force`).

## Context

The CEO is in back-to-back meetings, and the most valuable commitments + decisions are spoken
aloud, not typed. We capture every meeting's **full transcript** so the brain can read it and
form its **own** view — never relying solely on a vendor's auto-summary, which routinely
misses the asks that matter to him.

Tier-1 rules apply: this extractor is **dumb and deterministic** — it pulls raw transcripts
and any vendor-provided summary to JSON. It does **no** reasoning, no summarizing (that's the
meeting-assistant brain in Prompt 11). Fired by the scheduler on a modest cadence (meetings
don't finish every 5 min — every ~30 min is plenty; set in Prompt 08).

## Step 0 — discover what we're actually connected to, and record it

**Do not assume a provider is connected.** Determine, in this order, which transcript sources
are actually available on this machine, and write the result to
`~/ceo-ai-executive-assistant/reference/integrations.md`:

- **Fireflies.ai** — has a GraphQL API at `https://api.fireflies.ai/graphql` with a Bearer
  API key. Check for a key in the environment / a `.env` / the macOS Keychain / `state/config.json`.
  If present, confirm it works with a tiny query (`{ user { name } }`). If not, record
  "Fireflies: not connected — needs API key" and move on.
- **Otter.ai** — no clean public API. Realistic ingestion paths, in order of preference:
  (a) Otter's email-delivered transcript/summary (the CEO can set Otter to email completed
  notes → we read those from the mail extractor's account), (b) a synced export folder, or
  (c) browser automation as a last resort. Detect which, if any, is set up.
- **Email-delivered summaries (both vendors).** Fireflies and Otter both email a notes/summary
  link after each meeting. Even with no API, we can catch these from `state/mail.json`
  (Prompt 02 already pulls mail). Record the sender addresses to look for.
- **A local transcript folder.** Many setups just drop `.txt` / `.vtt` / `.srt` / `.md`
  transcript files into a folder (Zoom/Granola/manual). Ask the CEO if such a folder exists;
  if so, record its path and treat it as a source.

Write `reference/integrations.md` as the **canonical record of every connection** — provider,
auth method, where the secret lives, what's reachable, and any blind spots. The brain and
later prompts read this to know what exists. Keep it current whenever a connection changes.

## Task — `bin/extract_meetings.sh`

Pull meetings from **whatever Step 0 found is connected**, for the **last 3 days** (window
wider than the cadence so nothing is missed across restarts; dedup is the brain's job).

- **Fireflies (if connected):** query recent transcripts via GraphQL. For each, capture the
  meeting id, title, date/time (ISO), duration, attendee list (names + emails if present),
  the **full transcript** (speaker-labelled sentences if available), and — kept **separately**
  — the **vendor's own summary / action items** (so the brain can compare its own read against
  theirs, never inherit theirs blindly).
- **Otter / email-delivered / folder (if that's what's connected):** pull the transcript text
  and any included summary the same way, normalized to the shape below.
- Skip recordings still processing; only emit finished transcripts.

Emit a JSON array to `state/meetings.json`, one record per meeting:

```json
{
  "source": "fireflies | otter | email | folder",
  "id": "stable-unique-id",
  "title": "Voxy weekly sync",
  "datetime": "2026-06-12T15:00:00+01:00",
  "duration_min": 47,
  "attendees": [{ "name": "Jane Doe", "email": "jane@acme.com" }],
  "transcript": "Full speaker-labelled transcript text…",
  "vendor_summary": "Whatever Fireflies/Otter auto-generated (may be empty)",
  "vendor_action_items": ["…"],
  "source_url": "https://app.fireflies.ai/view/…"
}
```

- Keep `transcript` and `vendor_summary` in **separate fields** — never overwrite one with the
  other. The brain (Prompt 11) reads the transcript itself.
- Validate the JSON (`python3 -m json.tool`). Atomic write (`.tmp` then `mv`).
- Add `bin/extract_meetings.sh` log wrapper → `logs/extract_meetings.log`, exit non-zero on
  failure. Lockfile in `state/` so a slow API pull can't overlap the next run.

> **Privacy note.** Transcripts are sensitive. They live in `state/` (gitignored, local). The
> API key never gets logged. The full transcript never leaves the machine except, optionally,
> back to the vendor you already gave it to. This is consistent with the system's local-only
> rule.

## Verify before finishing

1. `python3 -m json.tool ~/ceo-ai-executive-assistant/state/meetings.json | head -60` — real
   meetings, transcript populated, `vendor_summary` separate from `transcript`, attendees present.
2. Show `reference/integrations.md` — which providers are connected, auth method, blind spots.
3. Confirm which source(s) actually fired (Fireflies API vs email vs folder).

Do not wire the scheduler yet (Prompt 08). The brain that summarizes these is Prompt 11.
