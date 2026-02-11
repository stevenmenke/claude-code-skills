# Claude Code Skills

Reusable skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Each folder is a self-contained skill — copy it into `~/.claude/skills/` (global) or `.claude/skills/` (project-level).

## Skills

| Skill | What it does |
|-------|-------------|
| [understand-repo](./understand-repo/) | Deeply understand any public GitHub repo using DeepWiki MCP + GitHub CLI. Understand implementations, debug dependency behavior, evaluate libraries, or extract self-contained code. |
| [remember](./remember/) | Session continuity — document decisions, context, and learnings in CLAUDE.md and CHANGELOG.md so the next session can continue without re-deriving understanding. |

## Install

Copy any skill folder to your skills directory:

```bash
# Global (available in all projects)
cp -r understand-repo ~/.claude/skills/

# Project-level (shared with team via git)
cp -r understand-repo .claude/skills/
```

Some skills require MCP servers or CLI tools — check the **Setup** section at the bottom of each `SKILL.md`.

## What are Claude Code skills?

Skills are markdown files that give Claude specialized knowledge and instructions. They're loaded on-demand (not always in context), so they're token-efficient. Claude decides when to activate a skill based on its `description` field matching what you ask.

## Contributing

PRs welcome. Each skill should be a folder with at minimum a `SKILL.md` containing YAML frontmatter (`name`, `description`) and clear instructions.
