# PROMPT 00 — Kickoff (run this first, once the folder is pasted in)

> Paste this into the Claude Code session on the executive's machine **after** you've copied
> the whole project folder in. It orients the session, then walks the build in order with a
> hard stop after each step so a human stays in control at the checkpoints that need one
> (permissions, and the historical backfill review).
>
> Prefer one fresh session per build prompt (01–09). This kickoff is for driving a single
> long session through them, or for re-orienting a fresh session that's continuing the build.

---

```
You are building the CEO AI Executive Assistant on this machine (the executive's computer).
The whole project has been pasted into this folder. Work through it carefully and
incrementally — do NOT try to do everything at once.

START BY READING, IN THIS ORDER:
1. CLAUDE.md                    (the rules — these override everything)
2. reference/ARCHITECTURE.md    (the design you must follow)
3. README.md                    (the build order)
4. reference/WINDOWS-NOTES.md   (ONLY if this is a Windows machine)

Then build by executing the prompts in prompts/ IN ORDER, one at a time:
  01 → 02 → 03 → 04 → 05 → 06 → 08 (scheduling) → 07 (one-time backfill) → 09 (later/skip)

PLATFORM:
- Prompt 01 detects whether this is macOS or Windows and records it in reference/platform.md.
- macOS  → every script is AppleScript/shell, scheduled by launchd.
- Windows → every script is PowerShell (.ps1), scheduled by Task Scheduler.
- Never mix stacks. iMessage is macOS-only (skip it on Windows).

RULES FOR HOW YOU WORK:
- Do ONE prompt fully, run its built-in verify step, and SHOW ME the results.
- Then STOP and wait for me to say "continue" before starting the next prompt.
- Prompt 01 needs me to grant OS permissions and confirm `wacli` is logged in:
    * macOS  → Full Disk Access + Automation toggles. Tell me exactly what to click;
               don't guess past a permission failure.
    * Windows → confirm classic Outlook COM works (or fall back to Graph), and the
               PowerShell execution policy allows our scripts.
- Prompt 07 (historical backfill) must show me the review table of kept-vs-suppressed
  tasks and get my approval BEFORE creating any reminders.
- Everything stays local. Be conservative — precision over recall.
- After each prompt, give me a short PASS/FAIL checklist of what you verified.

Begin now: read the docs above, then give me a one-paragraph summary of the system, the
detected OS + chosen stack, and the exact list of steps you're about to take. Do not start
building until I reply "go". Then start with prompt 01.
```

---

## Driving it between steps

- After each prompt it stops and shows a PASS/FAIL. Reply `continue` (or `go`) to advance.
- If a verify step fails, tell it what you saw and let it fix that piece before moving on —
  don't let it push past a red check, especially on permissions (01) and dedup (04/05).

## Two checkpoints where you must be hands-on

- **Prompt 01** — grant the OS permissions (macOS: FDA + Automation; Windows: Outlook COM /
  execution policy) and do any `wacli` QR re-login.
- **Prompt 07** — review the kept-vs-suppressed table before it writes ~2 weeks of reminders.
  This is where you sanity-check the temporal-reconciliation logic on real data.

## If the session gets long

The prompts are written to run one-per-fresh-session. Running them all in one long session
works, but context fills up. If it gets sluggish or vague around prompt 05–06, start a fresh
session and tell it: "read CLAUDE.md and reference/ARCHITECTURE.md, prompts 01–04 are already
built and verified, continue from prompt 05."
