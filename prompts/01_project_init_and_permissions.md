# PROMPT 01 — Project skeleton, platform detection, permissions, and `wacli` check

> Paste this into a fresh Claude Code session **on the executive's machine** (macOS or
> Windows). This is the foundation for an AI Executive Assistant called **CEO Executive
> Assistant AI**. Build #1 of a series. Do this one fully and verify before the next prompt.

## Context (read this — the whole series depends on it)

I'm building a local AI executive assistant for a very busy CEO. **Split architecture:**

- **Tier 1 — Extractors:** dumb, cheap scripts fired by the OS scheduler every ~5 minutes
  that dump raw data to JSON on disk (email, calendar, iMessage, WhatsApp). No LLM.
- **Tier 2 — Brain:** a headless `claude -p` run, fired less often (~30–60 min), that reads
  those JSON snapshots, finds **commitments** ("email me…", "I'll call you next week…"), and
  writes tasks to the OS reminders app + the daily note.
- **Tier 3 — Writers:** small helpers that create reminders and update the daily note.

Everything stays **local**. Nothing leaves the machine.

### ⚠️ Platform rule — decide once, commit fully

This runs on **macOS or Windows**. Same architecture, different scripting stack:

- **macOS → every script is AppleScript/shell**, scheduled by **launchd**.
- **Windows → every script is PowerShell (`.ps1`)**, scheduled by **Task Scheduler**.

**Never mix stacks.** Only `claude -p` (the brain) and `wacli` (WhatsApp) are cross-platform.
**iMessage is macOS-only** — on Windows it's a permanent blind spot, skip it.

## Your tasks

### 0. Detect the OS and record it

Detect whether this is macOS or Windows. Write the result + the chosen stack to
`reference/platform.md` (e.g. `platform: macos | stack: applescript+launchd`, or
`platform: windows | stack: powershell+taskscheduler`). **Everything below has two
columns — execute only the one matching this machine.** If you're unsure which OS, stop and
ask me.

### 0b. Choose the second-brain + task backends, and record the connections

The person setting this up gets to choose **where the assistant stores everything** (the
"second brain") and **where tasks go**. Ask me, then record the answers in **two files**:
`state/config.json` (machine-readable, read by every later script) and `reference/config.md`
(human-readable). Ask:

- **Notes backend** — where briefs, meeting notes, and goals are written:
  `obsidian` | `apple` | `both`. **Obsidian is the recommended primary** (Markdown vault,
  greppable, `[[backlinks]]`, syncable). `both` writes to Apple Notes *and* the Obsidian vault.
  - If `obsidian` or `both`: capture the **vault path** and a target subfolder (e.g.
    `CEO Assistant/`). Check whether the **Obsidian CLI** is installed (`which obsidian` /
    the user may have the built-in `Obsidian` binary on PATH) — record it; the note writer
    (Prompt 04) prefers direct Markdown file I/O but can use the CLI for append/search/sync.
  - If `apple` or `both`: confirm the Apple Notes Automation grant (below).
- **Tasks backend** — `apple_reminders` (macOS) / `microsoft_todo` (Windows) is the default
  to-do list. Record the list name (default "Exec Assistant").

Write `state/config.json`, e.g.:
```json
{
  "notes_backend": "obsidian",
  "obsidian": { "vault_path": "/Users/<me>/Vault", "folder": "CEO Assistant", "cli": true },
  "apple_notes": { "folder": "Daily Briefs" },
  "tasks_backend": "apple_reminders",
  "reminder_list": "Exec Assistant",
  "email_brief": { "enabled": false, "to": "", "from_account": "", "periods": ["daily"] },
  "goals_note_link": ""
}
```

### 0c. Discover meeting-transcript + email-send connections, and write `reference/integrations.md`

This assistant also ingests **meeting transcripts** (Otter / Fireflies / etc.) and can
**email the briefs**. Find out what's actually connected and record it — "keep a note of
whatever we're connected to" is a standing requirement.

- **Meeting tools** — check for a **Fireflies** API key (env / `.env` / Keychain /
  `state/config.json`); check whether **Otter** delivers transcripts by email or to a synced
  folder; ask if there's a local transcript folder. Record provider, auth method, where the
  secret lives, and what's reachable. Don't assume — verify a connected provider with a tiny
  call. (Full extractor is Prompt 10.)
- **Email send** — confirm a Mail.app account (macOS) / Outlook (Windows) can **send** a
  message silently (no foreground window). Note which account would send the briefs. (Wiring
  is Prompt 14.)

Write all of this to `reference/integrations.md` as the canonical connection log, and keep it
current whenever a connection changes.

### 1. Create the project skeleton

Create this layout (use `~` on macOS, `%USERPROFILE%` on Windows) at the project root
`ceo-ai-executive-assistant/`:

```
ceo-ai-executive-assistant/
  bin/          # extractor + writer scripts (built by later prompts)
  state/        # JSON snapshots + dedup ledgers — PRIVATE
  logs/         # scheduler stdout/stderr
  schedule/     # launchd *.plist (mac) OR Task Scheduler *.xml/registration (win) + control script
  .claude/skills/
  reference/
  README.md     # short local readme describing the layout
```

Add a `.gitignore` excluding `state/` and `logs/` — they hold the CEO's private message and
email content and must never be committed.

### 2. Establish the permission model — and verify each grant

**▶ macOS path.** Two separate permission systems:

- **(a) Full Disk Access (FDA)** — required to read `~/Library/Messages/chat.db` (iMessage)
  and, on some macOS versions, Mail data. Add the **Terminal/iTerm** app *and* the **`claude`**
  binary's host (and `/usr/bin/osascript` if invoked directly) in **System Settings → Privacy
  & Security → Full Disk Access**, then probe:
  ```bash
  sqlite3 "$HOME/Library/Messages/chat.db" "SELECT count(*) FROM message;" 2>&1
  ```
  Should print a count, not "operation not permitted".
- **(b) Automation (Apple Events / TCC)** — to control Mail, Calendar, Notes, Reminders,
  Messages. Trigger the consent prompts NOW (launchd can't show a prompt — it'd silently
  fail). Run each, report pass/fail, and tell me which toggle to flip in **Privacy & Security
  → Automation** on any failure:
  ```bash
  osascript -e 'tell application "Mail" to get name of every account'
  osascript -e 'tell application "Calendar" to get name of every calendar'
  osascript -e 'tell application "Notes" to get name of every folder'
  osascript -e 'tell application "Reminders" to get name of every list'
  osascript -e 'tell application "Messages" to get name'
  ```

**▶ Windows path.** No TCC, but there are equivalents:

- **Outlook programmatic access** — confirm classic **Outlook desktop** is installed (not
  just "New Outlook"/web; the COM automation we rely on needs classic Outlook). Probe COM:
  ```powershell
  $ol = New-Object -ComObject Outlook.Application
  $ns = $ol.GetNamespace("MAPI")
  $ns.Folders | ForEach-Object { $_.Name }   # should list your accounts
  ```
  If a security/"a program is trying to access" prompt appears, note it. If Outlook isn't
  available, we'll fall back to **Microsoft Graph** (app registration + OAuth) — flag that to me.
- **Execution policy** — confirm scripts can run: `Get-ExecutionPolicy -List`. If needed,
  we'll sign the scripts or set `RemoteSigned` for CurrentUser. Tell me what you find.
- **Tasks/Notes backends** — confirm what's available: **Microsoft To Do** / Outlook Tasks
  (via COM or Graph) for reminders, and **OneNote** *or* a synced Markdown folder for the
  daily note. Record the choice in `reference/platform.md`.
- iMessage: **skip** — record as a permanent blind spot in `reference/platform.md`.

### 3. Discover the account / calendar landscape

Capture real names so later extractors iterate over the actual accounts. Write findings to
`reference/accounts.md`.

- **macOS:** list every Mail account + INBOX; flag any **New Outlook** account (invisible to
  AppleScript — needs Gmail/IMAP fallback); list every Calendar and its owner (iCloud/Google/
  Exchange).
- **Windows:** enumerate Outlook accounts/stores and calendars via the COM probe above; note
  which are Exchange/IMAP/POP.

### 4. Confirm `wacli` (WhatsApp CLI) works and learn its interface

`wacli` is the WhatsApp CLI (cross-platform — same on both OSes). **Do not assume flags.**
```
wacli --help
wacli <subcommand> --help      # explore list-chats / read-messages style subcommands
```
Record in `reference/wacli.md`: login/auth state (surface any QR login needed), the command
to **list recent chats**, the command to **read recent messages** (time-window or count?),
whether it can emit **JSON** (preferred) or only text, and any "spins up a headless WhatsApp
Web session" / rate-limit caveats.

### 5. Verify `claude` headless works for Tier 2 (both OSes)

```
claude -p "Reply with exactly: ASSISTANT-OK"
```
Report the output.

## Done when

- `reference/platform.md` records the OS + chosen stack (+ Windows backend choices / blind spots).
- `state/config.json` + `reference/config.md` record the chosen **notes backend**
  (obsidian/apple/both, vault path if relevant) and **tasks backend**.
- `reference/integrations.md` records the **meeting-tool** connections (Fireflies/Otter/folder)
  and the **email-send** account — or notes each as "not connected".
- The directory tree exists; `.gitignore` protects `state/` and `logs/`.
- **macOS:** chat.db probe returns a count; all five Automation probes pass.
  **Windows:** Outlook COM lists accounts; execution policy allows our scripts; To Do/Notes
  backends chosen.
- `reference/accounts.md` lists real accounts + calendars with caveats.
- `reference/wacli.md` documents the real `wacli` interface + login state.
- `claude -p` returns `ASSISTANT-OK`.

Give me a short PASS/FAIL checklist at the end. Do not start building extractors — that's
the next prompt.
