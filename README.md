# Agent Skills

A collection of reusable [Agent Skills](https://agentskills.io/specification) — markdown-based instructions that teach AI coding agents specialized workflows, domain knowledge, and best practices.

## Available Skills

| Skill | Description |
|-------|-------------|
| [starrocks-table-design](skills/starrocks-table-design/) | Design StarRocks tables — table types, partitioning with `date_trunc()`, bucketing/distribution, and indexes (prefix, bloom filter, bitmap, ngram). |
| [starrocks-data-loading](skills/starrocks-data-loading/) | Load data into StarRocks via Stream Load, Broker Load, Routine Load, and INSERT INTO, including column mapping, status checks, and load-failure troubleshooting. |
| [starrocks-query-optimization](skills/starrocks-query-optimization/) | Optimize StarRocks queries — EXPLAIN, partition pruning, JOIN strategy, predicate pushdown, parallelism, index usage, and slow-query diagnosis. |
| [starrocks-materialized-views](skills/starrocks-materialized-views/) | Create and manage StarRocks materialized views — synchronous rollups, asynchronous MVs, transparent query rewrite, and refresh/management. |
| [starrocks-etl-and-tasks](skills/starrocks-etl-and-tasks/) | Build StarRocks ETL pipelines and async/scheduled tasks with `SUBMIT TASK` — batch/incremental/real-time pipelines, SCD Type 2, and task monitoring. |
| [starrocks-cluster-setup](skills/starrocks-cluster-setup/) | Set up and manage StarRocks cluster topology — FE/BE/Broker roles, `fe.conf`/`be.conf`, node lifecycle, and status checks. |
| [starrocks-resource-management](skills/starrocks-resource-management/) | Manage StarRocks resources — session limits, resource groups for workload isolation, and storage volumes for remote/tiered storage. |
| [starrocks-monitoring](skills/starrocks-monitoring/) | Monitor and maintain a StarRocks cluster — health metrics, compaction, tablet repair/rebalancing, statistics for CBO, and routine maintenance. |
| [starrocks-backup-recovery](skills/starrocks-backup-recovery/) | Back up and restore StarRocks data — backup repository setup, full snapshots, status checks, and restore from snapshot. |

## Installation

### Via skills CLI (recommended)

Uses the [`skills` CLI](https://github.com/vercel-labs/skills) (`npx skills`), the open agent-skills tool that installs into 70+ agents (Claude Code, Codex, Cursor, OpenCode, and more):

```bash
# List available skills without installing
npx skills add TiansuYu/agent-skills --list

# Install all skills
npx skills add TiansuYu/agent-skills

# Install a specific skill
npx skills add TiansuYu/agent-skills --skill starrocks-table-design

# Install globally (available across all projects)
npx skills add -g TiansuYu/agent-skills
```

Manage installed skills with `npx skills list`, `npx skills update`, and `npx skills remove`.

### Via git clone

Clone the repo and copy (or symlink) the skills into your agent's skills location — e.g. `~/.claude/skills/`, `~/.cursor/skills/`, or `~/.agents/skills/` (shared across agents):

```bash
git clone https://github.com/TiansuYu/agent-skills.git
cd agent-skills

# Copy all skills into a global location
cp -r skills/* ~/.claude/skills/

# Or symlink a single skill so `git pull` keeps it up to date
ln -s "$PWD/skills/starrocks-table-design" ~/.claude/skills/starrocks-table-design
```

### Manual

Copy individual skill folders (`skills/<name>/`) into the respective agent skill locations, e.g. `~/.cursor/skills/`.

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
