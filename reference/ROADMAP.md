# Roadmap — ideas to make the second brain better

> Not build prompts yet — a backlog of high-leverage additions, roughly ordered by
> value-for-effort. Each could become a numbered prompt (`15+`) when you decide to build it.
> The system stays local-first; anything that reaches out (email, APIs) is opt-in and logged
> in `reference/integrations.md`.

## Near-term, high leverage

1. **Relationship / people memory.** A per-person note (auto-maintained) that aggregates every
   thread, meeting, and commitment with each contact: "last spoke 9 days ago, you owe them the
   deck, they owe you nothing, recurring topic = pricing." Surfaces "you've gone quiet with X"
   in the brief. This is where a second brain compounds.

2. **Daily shutdown / journal capture.** A short evening pass (or a prompt the CEO can fire)
   that asks 2–3 questions, logs what actually happened vs the plan, and feeds the goal-detector
   better signal than passive channels alone. Closes the loop between intention and outcome.

3. **Two-way reminder sync.** Right now tasks flow one way (brain → Reminders). Read the
   Reminders/To-Do **back** so the brain knows what's done, what's overdue, and stops nagging
   about closed items — and so the briefs reflect real task state.

4. **Smarter due-date + nudge escalation.** When a commitment's inferred due date passes with no
   sign it was handled, escalate it in the brief ("⚠ 3 days overdue: send deck to Jane") rather
   than letting it sit as one of N flat reminders.

5. **Confidence + feedback loop.** Let the CEO mark a reminder/goal as wrong ("not a real task")
   — capture that signal in `state/feedback.json` and feed it back so precision improves over
   time. The fastest way to keep him from switching the system off.

## Capture & channels

6. **Voice-memo / dictation ingestion.** Point a Tier-1 extractor at a voice-memo folder (or
   a transcription tool) so spoken thoughts on the move become goal/commitment signal — the
   "author transcriptions" idea, generalized beyond meetings.

7. **More meeting sources.** Granola, Zoom AI, Google Meet, Teams, Otter folder export — the
   `extract_meetings.sh` schema already supports a `folder` source; add adapters as needed.

8. **Document / file awareness.** Watch a Downloads / "to-read" folder; surface "you saved this
   contract 5 days ago and haven't opened it" and link decks referenced in commitments.

## Intelligence

9. **Weekly "what I'm avoiding" pass.** Detect commitments and goals that keep getting deferred
   — the things he routinely dodges — and name them. High signal, slightly uncomfortable, very
   useful.

10. **Calendar/load defense.** Flag overload, back-to-backs with no prep buffer, meetings with
    no agenda, and droppable/async-able meetings (respect any "locked" clients). Feed the
    weekly brief's "where the time went".

11. **Goal ↔ calendar alignment.** Each week, compare stated goals against where time actually
    went; surface the gap ("0 hours on the 'knowledge & growth' goal this week").

12. **Draft replies, never auto-send.** For inbound asks awaiting a reply, draft a one-liner he
    can approve — surfaced in the brief, sent only on explicit approval. (Keep the no-auto-send
    rule; drafting is safe, sending is not.)

## Reliability & trust

13. **Health check + heartbeat.** A tiny job that verifies each extractor produced fresh JSON
    and each scheduled job ran; if mail.json is 12h stale, say so in the brief instead of
    silently going blind.

14. **Encrypt `state/` at rest** (or store secrets only in Keychain) — transcripts and message
    bodies are sensitive even though they never leave the machine.

15. **Backup the second brain.** If Obsidian, the vault is likely already synced/versioned; if
    Apple Notes, add a periodic Markdown export so history is portable and greppable.

## Reach (opt-in, beyond the brief email)

16. **Push to phone via a second channel** (e.g. a Telegram/Signal bot or ntfy) for time-
    sensitive heads-ups, in addition to the email brief.

17. **Morning brief as audio.** TTS the daily brief into a short clip he can listen to while
    getting ready / commuting.
