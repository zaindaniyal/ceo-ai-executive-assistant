# Windows notes — prerequisites, backends, and fallbacks

Read this if `reference/platform.md` resolves to **Windows**. It's the Windows companion to
`ARCHITECTURE.md` → "Platform abstraction". The architecture is identical to macOS; this
file spells out the Windows-specific prerequisites, the Outlook-COM-vs-Graph decision, and
the things that simply don't exist on Windows.

The whole stack on Windows is **PowerShell (`.ps1`)** scheduled by **Task Scheduler**. Never
emit AppleScript here. The only cross-platform pieces are the brain (`claude -p`) and `wacli`.

---

## 1. Prerequisites — confirm these before building (Prompt 01)

| Need | Why | Check |
|------|-----|-------|
| **Classic Outlook desktop** (Win32, part of Microsoft 365 / Office) | Our default email + calendar reader uses the **Outlook COM object model**, which only the classic desktop app exposes. | `New-Object -ComObject Outlook.Application` succeeds and `GetNamespace("MAPI").Folders` lists the accounts. |
| **NOT "New Outlook" / Outlook-on-the-web only** | New Outlook (the toggle-able web-wrapper) has **no COM automation**. If that's all that's installed, COM fails → use the Graph fallback (§3). | In Outlook, the "New Outlook" toggle should be **off**, or classic Outlook should be separately installed. |
| **PowerShell 5.1+ or PowerShell 7** | Runtime for every script. | `$PSVersionTable.PSVersion` |
| **Execution policy allows our scripts** | Scheduled `.ps1` must run unattended. | `Get-ExecutionPolicy -List`; set `RemoteSigned` for `CurrentUser`, or sign the scripts. |
| **`wacli` on PATH** | WhatsApp extractor. Same CLI as macOS. | `wacli --help` (and confirm login / QR state). |
| **`claude` on PATH** | The brain. | `claude -p "Reply with exactly: ASSISTANT-OK"` |
| **Machine stays logged in / awake** | Task Scheduler tasks that touch the Outlook profile must run as the logged-in user (see §5). Sleep kills the cadence. | Set power plan to never sleep on AC; confirm the user account auto-logs-in if unattended. |

Record the outcome (Outlook COM available? which backend chosen for tasks + notes?) in
`reference/platform.md`.

---

## 2. Email + Calendar — Outlook COM (default path)

PowerShell drives classic Outlook directly. Sketch for the extractors (Prompt 02):

```powershell
$ol = New-Object -ComObject Outlook.Application
$ns = $ol.GetNamespace("MAPI")

# Email: iterate each account's Inbox, last 3 days, newest first
foreach ($store in $ns.Stores) {
  $inbox = $store.GetDefaultFolder(6)   # 6 = olFolderInbox
  $items = $inbox.Items
  $items.Sort("[ReceivedTime]", $true)
  $items.Restrict("[ReceivedTime] >= '" + (Get-Date).AddDays(-3).ToString("g") + "'") |
    ForEach-Object { <# capture EntryID, ReceivedTime, SenderName, SenderEmailAddress,
                         Subject, Body (truncate ~1500), store/account name #> }
}

# Calendar: AppointmentItems in a window. IncludeRecurrences must be set BEFORE Restrict,
# and you must Sort ascending for recurrence expansion to work.
$cal = $ns.GetDefaultFolder(9)          # 9 = olFolderCalendar
$appts = $cal.Items
$appts.IncludeRecurrences = $true
$appts.Sort("[Start]")
$appts.Restrict("[Start] >= '" + (Get-Date).AddDays(-14).ToString("g") +
                "' AND [Start] <= '" + (Get-Date).AddDays(7).ToString("g") + "'")
```

Notes / gotchas:
- **Stable id** = `EntryID` (use as the dedup `src:` key, analogous to the email Message-ID).
- **Capture past events too** (−14 days here) — the brain needs them for temporal
  reconciliation (suppress already-happened meetings).
- COM dates are locale-sensitive; build the `Restrict` filter string with an invariant/`"g"`
  format and test it actually filters.
- Emit the **same JSON shapes** the macOS extractors produce, so the brain is OS-agnostic.
- Atomic write = temp file then `Move-Item -Force`.
- A first COM call may raise an Outlook security/"a program is trying to access" prompt on
  older/locked-down configs — note it; on modern Microsoft 365 with the machine domain-joined
  it's usually silent.

---

## 3. Fallback — Microsoft Graph (no classic Outlook, or Exchange Online only)

If COM isn't available (New-Outlook-only, or you'd rather not depend on the desktop app),
read mail + calendar via **Microsoft Graph**:

- Register an app in Entra ID (Azure AD) → grant **delegated** `Mail.Read`, `Calendars.Read`
  (and `Tasks.ReadWrite` if using To Do via Graph).
- Auth with the device-code or auth-code flow; **cache the refresh token locally** so the
  scheduled task runs unattended. Store it outside the repo (e.g. `state/`, gitignored).
- Endpoints: `GET /me/messages?$filter=receivedDateTime ge ...`,
  `GET /me/calendarView?startDateTime=...&endDateTime=...`.
- Same JSON output shapes; same `src:` id from the Graph `id` field.
- **Trade-off:** Graph needs an app registration + an admin who can consent, and it reads the
  *Exchange Online* mailbox (not local PSTs or non-Exchange IMAP accounts). COM sees whatever
  the desktop profile sees. If the CEO has non-Exchange accounts in Outlook, COM is the more
  complete reader. Flag the choice to the operator.

---

## 4. Reminders + Daily Note backends

**Reminders / tasks** (the `Add-Reminder.ps1` writer from Prompt 04) — pick one, record it:
- **Microsoft To Do via Graph** (`POST /me/todo/lists/{id}/tasks`) — syncs to phone, the
  closest analogue to Apple Reminders. Needs the Graph app + `Tasks.ReadWrite`.
- **Outlook Tasks via COM** (`$ns.GetDefaultFolder(13)` = `olFolderTasks`,
  `CreateItem(3)` = `olTaskItem`) — no Graph needed, but Outlook Tasks are less visible on
  mobile than To Do.

Keep the **same contract** either way: idempotent (dedup by the `src:<id>` marker in the
task body), and the body carries quote + name + handle + channel + original datetime.

**Daily Brief note** (the `Upsert-Note.ps1` writer):
- **Recommended: a synced Markdown file** (e.g. under OneDrive) — `Daily Brief — YYYY-MM-DD.md`.
  Plain file I/O is the most reliable and fully idempotent option, and it's a clean on-ramp to
  the future Obsidian migration (Prompt 09). Section-upsert in Markdown; dedup task lines by
  `src:`.
- **OneNote via COM/Graph** is possible but the OneNote object model is fiddly — only choose
  it if the CEO specifically lives in OneNote.

---

## 5. Task Scheduler (replaces launchd) — the one trap that bites

Prompt 08 covers this; the critical Windows-specific point:

> **Run each task as the logged-in user with "Run only when user is logged on".**

A task set to "Run whether user is logged on or not" executes in **session 0 / as a service
context** and will **not** have access to the user's Outlook MAPI profile or cached
credentials — Outlook COM calls then fail or hang silently. This is the exact Windows
analogue of the macOS "launchd context ≠ interactive shell / missing TCC grant" failure.
Test the brain task **under Task Scheduler**, not just from your interactive PowerShell prompt.

Other specifics:
- Register with `Register-ScheduledTask`; extractors get a **5-min repetition interval**
  (`New-ScheduledTaskTrigger -Once -RepetitionInterval (New-TimeSpan -Minutes 5)`), the brain
  45 min, the brief a daily trigger at 06:30.
- Point the action at `powershell.exe -NoProfile -ExecutionPolicy Bypass -File <wrapper>.ps1`.
- `schedule/assistant-ctl.ps1` provides load/unload/status/kick via
  `Register-/Unregister-/Get-/Start-ScheduledTask`.
- Use a per-job **lockfile in `state/`** so a long run can't overlap the next trigger
  (`Start-ScheduledTask` + repetition can otherwise stack).

---

## 6. Permanent blind spots on Windows

- **iMessage does not exist.** Skip the iMessage extractor entirely; record it in
  `platform.md`. (Phone Link stores some SMS/RCS in a local DB, but it's undocumented,
  unstable across updates, and excludes iMessage — do **not** depend on it.)
- **Non-Exchange Outlook accounts under the Graph fallback** are invisible (Graph only reads
  the Exchange Online mailbox). COM avoids this.
- As on macOS, surface any blind spot in the Daily Brief (a one-line "⚠ … not visible to the
  assistant") so the CEO knows the brief isn't omniscient.
