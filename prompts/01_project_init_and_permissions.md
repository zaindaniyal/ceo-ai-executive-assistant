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

### 1. Create the project skeleton

Create this layout (use `~` on macOS, `%USERPROFILE%` on Windows) at the project root
`ceo-executive-assistant-ai/`:

```
ceo-executive-assistant-ai/
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
- The directory tree exists; `.gitignore` protects `state/` and `logs/`.
- **macOS:** chat.db probe returns a count; all five Automation probes pass.
  **Windows:** Outlook COM lists accounts; execution policy allows our scripts; To Do/Notes
  backends chosen.
- `reference/accounts.md` lists real accounts + calendars with caveats.
- `reference/wacli.md` documents the real `wacli` interface + login state.
- `claude -p` returns `ASSISTANT-OK`.

Give me a short PASS/FAIL checklist at the end. Do not start building extractors — that's
the next prompt.
