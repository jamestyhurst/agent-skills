---
name: pair-programmer
description: >
  Dictate-mode pairing: instead of editing files or running commands itself, the
  agent narrates what to do and gives the exact text to type, one step at a time,
  while you type it yourself — for muscle memory, independence, and an honest
  record of what you actually built unaided. Use by saying "pair with me",
  "dictate mode", "let me type it myself", "I want to type this myself", or by
  invoking /pair-programmer. Works well as the default in a folder you've marked
  as a learning/practice project.
license: MIT
---

# Pair-Programmer

Narrate and dictate; you execute. The quality bar doesn't change — only who types does.

## Why

Most agent defaults optimize for the task getting done. This skill optimizes for a
different goal: you get better at doing it yourself. Typing dictated code, hitting your
own errors, and fixing them from a hint rather than a diff builds a kind of fluency that
watching a diff land never does.

## Trigger

- **Manual, any time:** "pair with me" / "dictate mode" / "let me type it myself" /
  "I want to type this myself" / `/pair-programmer`.
- **Default in a marked project:** if you keep a convention for tagging a repo or folder
  as "learning" / "practice" (a frontmatter field, a README note, whatever you already
  use), have the agent treat that tag as an automatic trigger for this mode. This skill
  doesn't invent that tagging system for you — it just says how to *react* to it if you
  have one.
- **Exit:** "stop pairing" / "just do it" / "autopilot", or the task naturally ends.

## The loop

```pseudocode
for step in task.steps:
    explain(what_and_why)          # one or two sentences
    dictate(exact_text_to_type)    # fenced code block, copy-typeable, ONE step
    wait_for(you_report_done_or_result_or_error)
    if error:
        diagnose(error)            # reading files/output to figure out the fix is fine
        # continue: dictate the fix as the next step
```

## Rules

- **No file edits on the task's own files.** Reading them to figure out what to dictate is
  fine — that's the agent's prep, not doing the work.
- **No running the task's own commands.** A read-only check *after* you report a step
  done — a test run, `git status`, `git diff` — is verification, not driving, and stays
  allowed.
- **One step per turn.** Small enough that a wrong keystroke is easy to catch before the
  next step compounds it. Never a five-step wall of instructions at once.
- **Dictate the literal text, not a description of it.** "Add a type hint" is not
  dictation; `def foo(x: int) -> str:` is.
- **A single named exception doesn't end the mode.** "Just add that import for me" — do
  that one thing, then resume dictating. Don't quietly fall back to doing everything.

## What this isn't

- Not a stricter or looser code-review bar — same standards either way.
- Not silence on a bad approach — the agent still says when something's wrong; it just
  doesn't fix it by editing.
- Not a teaching/note-writing skill — pair with whatever your own concept-note or
  learning-log system is (see the companion `wrap` skill in this repo for one shape of
  that) rather than expecting this skill to write one itself.

## Adopting this

Copy `SKILL.md` into your agent's skills directory (Claude Code: `~/.claude/skills/pair-
programmer/`; project-scoped: `.claude/skills/pair-programmer/`). For agents that read a
plain instructions file instead of a skills directory (Warp, Copilot via `AGENTS.md`,
etc.), paste the **Trigger**, **The loop**, and **Rules** sections above into that file
directly — the mechanism has no Claude-specific dependency.
