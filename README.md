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
| [fastapi-scaffold](skills/fastapi-scaffold/SKILL.md) | Scaffold a new FastAPI service with async SQLAlchemy 2.0, Alembic, Pydantic Settings, pytest, Docker multi-stage build, Makefile, and uv. |
| [terraform-module](skills/terraform-module/SKILL.md) | Generate GCP Terraform modules (gcs, cloud-run, cloud-sql, vpc, iam, full-stack) with provider config, variables, outputs, and tfvars examples. |
| [mcp-server](skills/mcp-server/SKILL.md) | Scaffold a new MCP server in Python (FastMCP) or Go (mcp-go) with tool definitions, transport config, Makefile, tests, and CI workflow. |
| [cobra-cli](skills/cobra-cli/SKILL.md) | Scaffold a Go CLI application or subcommand using Cobra with table-driven tests, ldflags versioning, Makefile, and CLAUDE.md. |

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
