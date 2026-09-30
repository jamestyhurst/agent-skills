# agent-skills

Small, single-purpose skills for coding agents (Claude Code, and anything else that reads
either a skills directory or a plain `AGENTS.md`-style instructions file).

## Skills

| Skill | What it does |
|---|---|
| [`pair-programmer`](pair-programmer/SKILL.md) | Dictate-mode pairing: the agent tells you exactly what to type, one step at a time, instead of editing/running things itself — for practice and independence, not speed. |
| [`wrap`](wrap/SKILL.md) | Close out a session with one short, evidence-based log entry, then hand off to whatever your project already wants done at session end. |
| [`review-pr`](review-pr/SKILL.md) | Merge-readiness triage for a queue of open pull requests: one at a time, oldest first, weighing CI, conflicts, blast radius, tests, and design fit — then merges only when you say so. |

## Installing a skill

Copy the skill's folder into your agent's skills directory:

```bash
# Claude Code, personal (all projects on this machine)
cp -r pair-programmer ~/.claude/skills/

# Claude Code, one project only
cp -r pair-programmer .claude/skills/
```

For an agent that reads a plain instructions file instead of a skills directory (Warp,
GitHub Copilot via `AGENTS.md`, etc.), paste the relevant sections of the `SKILL.md`
directly into that file — neither skill here depends on a Claude-specific mechanism.

## License

MIT — see [LICENSE](LICENSE).
