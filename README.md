# Agent Skills

A collection of reusable [Agent Skills](https://agentskills.io/specification) — markdown-based instructions that teach AI coding agents specialized workflows, domain knowledge, and best practices.

## Available Skills

| Skill | Description |
|-------|-------------|
| [starrocks-expert](skills/starrocks-expert/) | Expert guidance for StarRocks database operations including table management, data loading, query optimization, materialized views, partitioning strategies, and cluster configuration. |

## Installation

Copy skills into respective agent skill locations, e.g. `~/.agent/skills`.

## Skill Structure

```
skills/<skill-name>/
├── SKILL.md              # Required — main instructions
├── reference.md          # Optional — detailed documentation
├── examples.md           # Optional — usage examples
└── scripts/              # Optional — utility scripts
    └── helper.py
```

### SKILL.md Format

Every skill requires YAML frontmatter with two fields:

```yaml
---
name: my-skill-name          # lowercase, hyphens, max 64 chars
description: What this skill does and when to use it.
---
```

The body contains concise, actionable instructions for the AI agent.

## License

Apache License 2.0 — see [LICENSE](LICENSE).
