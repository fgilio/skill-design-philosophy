# skill-design-philosophy

Design philosophy and principles for creating outstanding [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills.

## What's Inside

| File | Description |
|------|-------------|
| [SKILL.md](SKILL.md) | Core principles, anti-patterns, script template, and checklists for building skills |
| [skill-project-structure-setup.md](skill-project-structure-setup.md) | Directory structure, global access patterns, and SKILL.md frontmatter reference |

## Installation

Clone into your Claude Code skills directory:

```bash
git clone https://github.com/fgilio/skill-design-philosophy.git ~/.claude/skills/skill-design-philosophy
```

Claude Code will automatically discover the skill and use it when creating or improving skills.

## Core Principles

1. **Zero-Error by Design** - hide tool quirks from users
2. **Intuitive Interfaces** - commands that read naturally
3. **Predictable Failure Modes** - catchable, parseable, actionable errors
4. **Validate Early, Fail Fast** - check inputs before doing work
5. **Sensible Defaults** - don't require specifying the obvious
6. **Consistent Output** - JSON for data, plain text for logs, stderr for errors
7. **Scripts Are Self-Contained** - no external dependencies or build steps

## License

[MIT](LICENSE)
