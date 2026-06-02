# PROMPT 03 — iMessage + WhatsApp extractors (Tier 1)

> Paste into a fresh Claude Code session on the CEO's Mac. Build #3. Assumes Prompts 01–02
> done. Read `~/ceo-executive-assistant-ai/reference/wacli.md` first.

> **Platform note.** macOS idiom below. On **Windows**: **iMessage does not exist — SKIP
> Task A entirely** and record the blind spot in `reference/platform.md`. Task B (`wacli`) is
> the same cross-platform CLI — implement the WhatsApp extractor as a `.ps1` writing the same
> JSON schema, with `Move-Item -Force` atomic writes.

## Context

Same Tier-1 rules: dumb, cheap, deterministic, launchd-fired every ~5 min, output JSON to
`~/ceo-executive-assistant-ai/state/`, atomic writes, no dialogs, no foreground apps. The brain reads
these later to find commitments.

## Task A — `bin/extract_messages.sh` (iMessage via chat.db)

**Read iMessage from the SQLite database, NOT via Messages.app AppleScript** (Messages can
send but cannot reliably read). Requires Full Disk Access (confirmed in Prompt 01).

Write a shell script that queries `~/Library/Messages/chat.db` for messages from the **last
3 days** and dumps `~/ceo-executive-assistant-ai/state/imessage.json`.

Key facts about chat.db to get right:
- `message.date` is **nanoseconds since 2001-01-01** (Apple epoch). Convert to ISO:
  `datetime(message.date/1000000000 + 978307200, 'unixepoch', 'localtime')`.
- Join `message` ↔ `chat_message_join` ↔ `chat` to get the chat/handle; join `handle` for
  the counterpart's phone/email (`handle.id`).
- `message.is_from_me` (0/1) tells direction — **critical**: inbound asks ("email me") vs
  outbound promises ("I'll call you") are detected differently by the brain.
- `message.text` may be NULL when the body is in `attributedBody` (newer macOS). Capture
  `text`; if you can decode `attributedBody` cheaply do so, otherwise emit text and set a
  `"body_in_attributedbody": true` flag so the brain knows some bodies are missing.
- Group chats: include `chat.display_name` / `chat.chat_identifier`.

Per message emit: stable `guid` (`message.guid`), ISO datetime, `is_from_me`, counterpart
handle (phone/email), counterpart display name if resolvable, chat id/name, and text.
Use `sqlite3 -json` if available (cleanest), else build JSON with `python3`. Verify it
parses. Atomic write.

> Contact-name resolution: handles are often just phone numbers. If easy, enrich names from
> Contacts via `osascript`/AddressBook; if not, leave the handle and let the brain note
> "unknown contact" — don't block the dump on perfect names.

## Task B — `bin/extract_whatsapp.sh` (via `wacli`)

Use the **real** `wacli` interface documented in `reference/wacli.md`. If anything there is
unverified, re-run `wacli --help` / `wacli <sub> --help` now and confirm before coding.

Goal: dump recent WhatsApp messages from the **last 3 days** across active chats to
`~/ceo-executive-assistant-ai/state/whatsapp.json`.

- List recent chats, then pull recent messages per chat (respect whatever count/time-window
  flags `wacli` actually exposes).
- Per message emit, normalized to match the iMessage shape as closely as possible: stable
  message id, ISO datetime, direction (from-me vs from-them — find how `wacli` signals
  this), counterpart name + phone/jid, chat name (for groups), and text.
- If `wacli` only emits human text, parse it deterministically (don't LLM it here — Tier 1
  has no brain). If it emits JSON, normalize that.
- Be gentle: if `wacli` spins up a headless WhatsApp Web session per call, add a short
  guard so we don't hammer it every 5 min if a previous run is still going (a simple
  lockfile in `state/`). Note any such caveat in the log.
- Atomic write. Verify JSON parses.

## Task C — unify the shape

The brain wants one consistent record shape across iMessage + WhatsApp. Document the agreed
schema in `~/ceo-executive-assistant-ai/reference/message_schema.md`, e.g.:

```json
{
  "source": "imessage | whatsapp",
  "id": "stable-unique-id",
  "datetime": "2026-06-02T14:03:00+01:00",
  "from_me": false,
  "counterpart_name": "Jane Doe",
  "counterpart_handle": "+15551234567",
  "chat_name": "Investors (group)",
  "text": "Can you email me the deck after the call?"
}
```

Both extractors should emit arrays of this shape (plus source-specific extras are fine).

## Verify before finishing

Show me:
1. `python3 -m json.tool ~/ceo-executive-assistant-ai/state/imessage.json | head -40` — real messages,
   correct dates (not 2001!), `from_me` populated, handles present.
2. `python3 -m json.tool ~/ceo-executive-assistant-ai/state/whatsapp.json | head -40` — real messages,
   direction + counterpart populated.
3. Confirm both conform to `message_schema.md`.

Do not wire launchd yet (Prompt 08).
