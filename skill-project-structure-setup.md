# Skill Project Structure & Setup

How to structure a skill project and make scripts globally accessible.

---

## Directory Structure

### Bash skills

```
skill-name/
├── SPEC.md               # Optional: collaborative spec (kept after implementation)
├── SKILL.md              # Required: primary reference for Claude Code and humans
├── reference.md          # Optional: deep documentation (or references/ directory)
└── scripts/
    ├── setup.md              # Recommended: human installation guide
    ├── skill-resource-action # e.g., gitlab-mr-create, browser-screenshot
    └── skill-action-resource # Or verb-noun: gitlab-job-trace
```

### TypeScript (Bun) skills

```
skill-name/
├── SPEC.md               # Optional: collaborative spec (kept after implementation)
├── SKILL.md              # Required: primary reference for Claude Code and humans
├── reference.md          # Optional: deep documentation (or references/ directory)
├── package.json          # bin field for global access via `bun link`
└── scripts/
    ├── setup.md              # Recommended: human installation guide
    ├── lib.ts                # Shared infra: auth, HTTP client, validation
    ├── skill-resource-action.ts
    └── skill-action-resource.ts
```

**Naming convention**: Scripts should be prefixed with skill name for global access: `{skill-name}-{resource}-{action}` (e.g., `gitlab-mr-create`, `browser-screenshot`). This prevents namespace collisions when scripts are in PATH.

---

## Global Access Pattern

Skills with scripts should be installable for global CLI access. Two approaches based on runtime:

### For bun/TypeScript skills

Add `bin` field to `package.json`:

```json
{
  "bin": {
    "browser-start": "./scripts/browser-start.ts",
    "browser-screenshot": "./scripts/browser-screenshot.ts"
  }
}
```

Install globally:
```bash
cd ~/.claude/skills/skill-name && bun link
```

Commands go to `~/.bun/bin/` (must be in PATH).

### For bash skills

Create symlinks to `~/.local/bin/`:

```bash
for script in ~/.claude/skills/skill-name/scripts/skill-*; do
  ln -sf "$script" ~/.local/bin/$(basename "$script")
done
```

Requires `~/.local/bin` in PATH.

---

## setup.md

Recommended for local skills. Each skill with scripts should have `scripts/setup.md` containing:
- Installation command(s)
- PATH requirements
- List of available commands after setup
- Uninstall instructions

Published GitHub repos typically use `README.md` instead.

**Important:** `setup.md` is for human CLI installation. It should NOT be referenced in `SKILL.md` (which is for Claude Code usage).

---

## SKILL.md Frontmatter

```yaml
---
name: skill-name          # lowercase, hyphens only
description: >
  [What it does].
  [When to use it].
  Trigger words: [comma-separated keywords].
---
```
