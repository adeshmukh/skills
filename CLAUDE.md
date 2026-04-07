# Skills Repository

This repo hosts custom agent skills for Claude Code, structured as an installable plugin.

## Layout

```
skills/
└── <skill-name>/
    └── SKILL.md        # Required: frontmatter + instructions
```

All skills live under `skills/`. Each skill is a directory containing at minimum a `SKILL.md`.

## Adding a skill

1. Create a new directory under `skills/` named after the skill (lowercase, hyphens).
2. Add `SKILL.md` with YAML frontmatter and Markdown instructions.
3. Update the Skills table in `README.md`.

### SKILL.md template

```markdown
---
name: skill-name
description: Short, front-loaded description of when to use this skill (max ~250 chars).
---

Detailed instructions for Claude here.
```

## Conventions

- Keep `SKILL.md` under 500 lines; use supplementary `.md` files for reference material.
- The `description` field is the primary signal Claude uses to decide when to invoke a skill — make it specific and action-oriented.
- Prefer `allowed-tools` to restrict tool access to only what the skill needs.
- Test each skill locally before committing by invoking it via `/skill-name` in Claude Code.
