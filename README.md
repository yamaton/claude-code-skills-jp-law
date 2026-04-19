# claude-code-skills-jp-law

Japanese law domain skills for [Claude Code](https://claude.com/claude-code) and the Claude Agent SDK.

## Skills

| Skill | Description |
|-------|-------------|
| [egov-law-api](skills/egov-law-api/SKILL.md) | e-Gov Law API v2 — retrieve Japanese law data (articles, metadata, keyword search, attachments, revision history, point-in-time snapshots) |

## Installation

Copy the skill directory to either scope. No dependencies beyond `curl` (or an equivalent HTTP client).

**Project scope** — available in one project:

```bash
git clone https://github.com/yamaton/claude-code-skills-jp-law.git /tmp/skills-jp-law
cp -r /tmp/skills-jp-law/skills/egov-law-api .claude/skills/
```

**User scope** — available in all your Claude Code projects:

```bash
git clone https://github.com/yamaton/claude-code-skills-jp-law.git /tmp/skills-jp-law
cp -r /tmp/skills-jp-law/skills/egov-law-api ~/.claude/skills/
```

Claude Code auto-discovers skills placed under `.claude/skills/` or `~/.claude/skills/`.

## Contributing

Issues and pull requests welcome. New skills should:

- Target Japanese law / legal data domains
- Be self-contained (no project-specific local-tool dependencies)
- Follow the standard `SKILL.md` format (YAML frontmatter + markdown body)

## License

MIT — see [LICENSE](LICENSE).
