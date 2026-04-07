# skills

Custom agent skills for Claude Code.

## Installation

```bash
claude plugin install https://github.com/adeshmukh/skills
```

## Skills

| Skill | Description |
|-------|-------------|
| [cheat-sheet](skills/cheat-sheet/SKILL.md) | Generate a dense, print-ready cheat sheet for exam memorization (AP Physics, AP Calculus, AP Chemistry, GRE, SAT, etc.). Outputs HTML, LaTeX, or Markdown with formulas, key facts, and color-coded sections. |

## Development

Each skill lives in its own directory under `skills/`:

```
skills/
└── <skill-name>/
    └── SKILL.md
```

`SKILL.md` uses YAML frontmatter followed by the skill instructions in Markdown.

```markdown
---
name: skill-name
description: One-line description of what this skill does and when Claude should use it.
---

Skill instructions here...
```

### Frontmatter fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | No | Display name (defaults to directory name) |
| `description` | Recommended | What the skill does; front-load the key trigger phrase |
| `argument-hint` | No | Autocomplete hint shown in the `/` menu (e.g. `[issue-number]`) |
| `allowed-tools` | No | Space-separated list of tools usable without prompting |
| `model` | No | Override the default model for this skill |
| `user-invocable` | No | Set to `false` to hide from the `/` slash-command menu |
