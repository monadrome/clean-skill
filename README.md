# clean-skill

A skill that audits which agent skills you actually use, then tells you which ones to hide, turn off, or keep.

It judges skills by how often they are used, not by how many tokens they cost. The audit is read-only.

## Install

```bash
npx skills add monadrome/clean-skill -g
```

Or clone this repo into your skills folder, for example `~/.claude/skills/clean-skill`.

## Use

Run it yourself: `/clean-skill` in Claude Code, `$clean-skill` in Codex. It is marked user-invoked only, so these agents won't trigger it on their own.

It ends with a report. Read it and decide what to change; the agent can apply changes after you confirm.

The instructions are in Chinese. For an English report, run `/clean-skill reply in English`.

No skill support? Paste the body of [SKILL.md](SKILL.md), everything below the frontmatter, to any coding agent that can read files and run commands.
