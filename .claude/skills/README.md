# claude-skills

A collection of [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) platform ops skills related to this repository.

## Skills

| Skill | Description |
|---|---|
| [cluster-analysis](.claude/skills/cluster-analysis/SKILL.md) | Read-only inspection of Kubernetes workloads across all namespaces to surface errors, crashes, failed pods, and unhealthy resources. |

## Installation

Skills must live in a directory Claude Code scans. Pick one:

- **Personal (all sessions):** `~/.claude/skills/<skill-name>/`
- **Project-scoped:** `<project>/.claude/skills/<skill-name>/`

Symlink the skill you want so you can keep editing it in this repo:

```sh
mkdir -p ~/.claude/skills
ln -s "$(pwd)/.claude/skills/cluster-analysis" ~/.claude/skills/cluster-analysis
```

Restart Claude Code (or start a new session) so it re-scans.

## Triggering a skill

- **Explicit:** type `/<skill-name>` in Claude Code (e.g. `/cluster-analysis`).
- **Automatic:** ask something matching the skill's `description` — Claude decides to invoke it.

## Adding a new skill

1. Create `.claude/skills/<skill-name>/SKILL.md`.
2. Add YAML frontmatter with `name` and a specific `description` (the description is what triggers automatic invocation).
3. Write the skill body: when to use, when NOT to use, step-by-step instructions, inputs, examples.
4. Symlink into `~/.claude/skills/` and restart.

Folder name MUST match the `name` field in frontmatter.

## Folder layout

```
.claude/
  settings.json            # team-shared settings (commit)
  settings.local.json      # personal overrides (gitignored)
  skills/
    <skill-name>/SKILL.md
```
