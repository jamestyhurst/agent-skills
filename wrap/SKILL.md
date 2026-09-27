---
name: wrap
description: Close out a session by writing one short, evidence-based log entry, then running whatever wrap-up duties the current project already asks for (a changelog note, a push, a handoff doc). Use when you say you're wrapping up, finishing, stopping for now, or ask to close out / log the session. Also fine to run unprompted at a natural end of substantial work.
argument-hint: "optional note about what the session was for"
license: MIT
---

# /wrap

A session that produces real learning or a real decision is easy to lose — it lives in
chat scrollback and is gone the moment the window closes. This skill's whole job is
turning that into one short, durable, append-only log entry, then handing off to
whatever else the project itself wants done at session end.

> **The log is an evidence base, not an activity record.** Its point is letting a later
> review argue "here's what I can actually do unaided" from cited fact instead of vibes.
> A padded entry is worse than a missing one — it inflates the evidence base with work
> that proves nothing. **A session that produced nothing worth citing gets no entry.**

## Configure once for your setup

This skill ships with sane defaults; edit the block below (or just tell the agent your
own paths the first time you use it) to match how you actually organize things.

```yaml
LOG:      "./docs/session-log/YYYY-MM.md"     # default: one Markdown file per month, in-repo
CONCEPTS: null                                 # optional: a folder of durable concept notes, if you keep one
DASHBOARD: null                                # optional: a "current skill level" summary, if you keep one — this skill never writes to it
```

If you already run a mastery-tracking system of your own (a skill-level dashboard, a
learning log with its own format), point `LOG` there instead and skip the default shape
below in favor of yours — the mechanism (decide → write → hand off → backup) is what
this skill actually provides.

## The loop

```pseudocode
wrap():
    1. decide whether this session earned an entry
    2. if yes: write it   (ask for the assistance level; never infer it)
    3. hand off to the project's own wrap-up duties
    4. offer a backup, if you have a backup routine configured
```

### 1. Does this session earn an entry?

```pseudocode
earns_entry(session):
    return any([
        a concept was taught, extended, or newly met,
        you made a design or architectural call worth citing later,
        you caught something — a drift, a wrong premise, a bad assumption,
        a skill area was exercised in a way the log cannot already show,
        work shipped that a future review would want as evidence,
    ])

# NOT sufficient on their own: files were edited, tests passed, a version was bumped,
# a long session happened. Activity is not evidence.
```

If nothing qualifies, say so in one line — *"nothing here the log would want as
evidence; skipping the entry"* — and go to step 3. That's a normal outcome, not a
failure.

### 2. Write the entry

Append to the current period's log file (creating it if this is its first entry).
**Append only** — never edit a past entry to revise a rating in hindsight; if a rating
was wrong, a later entry or a separate review process corrects it, on the record.

```markdown
## YYYY-MM-DD — <short title>

**Project:** <project>. **Session type:** interactive | autonomous.

**What happened:** <2-5 sentences. Concrete: name the files, the fork, the decision.
Write it for a reader six months out who has none of this context.>

**Concepts exercised:** `<slug>` (new | extended — <the delta>), ... | none.

**Assistance level:** <0-5> — <why that number>. Scale:

| Level | Name | Meaning |
|---|---|---|
| 0 | None | Book, docs, trial-and-error, your own reasoning — no AI involved. |
| 1 | Ask | Asked an AI a question, then did the work yourself. |
| 2 | Copy-paste | Took AI output (a command, a snippet, a paragraph) and adapted it. |
| 3 | IDE-assisted | Inline AI suggestions in the editor while you drove the file. |
| 4 | Agentic, supervised | Handed to an agent harness and watched it work. |
| 5 | Agentic, unattended | The harness ran while you were away. |

**Note for a future review:** <candidate evidence, named as candidate — never a rating
you're asserting as final. A separate review process, if you keep one, does the ruling.>
```

**Ask for the assistance level — don't infer it.** Polish is not evidence of
independence; a clean result can come from level 1 or level 5, and the whole scale is
worthless if a session just guesses. One question:

> *"What assistance level was this for you — 0 none, 1 asked, 2 copy-paste, 3
> IDE-assisted, 4 agentic supervised, 5 agentic unattended?"*

Work produced inside the current agent session is level 4 or 5 by definition (supervised
if you were watching, unattended if not) and needs no asking. Ask when the session
discussed or reviewed something you built elsewhere.

### 3. Hand off to the project's own wrap-up duties

This skill owns the session log and nothing else. Read the current project's own
`README`/`CLAUDE.md`/`AGENTS.md` (whichever it has) for anything else it asks for at
session end — a changelog entry, a version bump, a push, a handoff document for the next
session — and do that too. If the project says nothing, the log entry was the whole
duty.

### 4. The backup offer

```pseudocode
if you_have_a_backup_routine_configured():
    if anything_changed_this_session() or last_backup_older_than_your_own_threshold():
        offer a backup     # you run it; don't automate this step itself
else:
    say nothing     # no backup routine configured is a normal state, not a gap to nag about
```

Never wire this skill — or the backup it offers — to a scheduled task or cron job.
Automated system-level actions run while nobody's watching them fail; a triggered offer
that a human approves each time is the safer shape.

## Prohibitions

```pseudocode
NEVER write a rating into a dashboard/skill-tracker.       # a separate review step does that
NEVER edit a past log entry. Append.
NEVER infer the assistance level from how polished the work looks.
NEVER write an entry for a session that produced no citable evidence.
NEVER pad an optional field to look thorough — omit what wasn't actually observed.
NEVER automate this on a schedule.
```

## Adopting this

Copy `SKILL.md` into your agent's skills directory (Claude Code: `~/.claude/skills/wrap/`;
project-scoped: `.claude/skills/wrap/`), then fill in the **Configure once for your setup**
block. For agents that read a plain instructions file instead of a skills directory,
paste the loop and the entry format into that file directly.
