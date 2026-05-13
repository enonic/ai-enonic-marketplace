# Enonic Marketplace

Enonic's marketplace of [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and Codex plugins, following
the [Agent Skills specification](https://agentskills.io/specification).

## Installation

### Claude Code

Add the marketplace and install the `enonic-skills` plugin:

```
/plugin marketplace add enonic/ai-enonic-marketplace
/plugin install enonic-skills@enonic-marketplace
```

This makes every skill in the plugin available in your Claude Code sessions.

### Scopes

| Scope          | Command                                                            | Use case                |
|----------------|--------------------------------------------------------------------|-------------------------|
| User (default) | `/plugin install enonic-skills@enonic-marketplace`                 | Personal — all projects |
| Project        | `/plugin install enonic-skills@enonic-marketplace --scope project` | Team — shared via Git   |
| Local          | `/plugin install enonic-skills@enonic-marketplace --scope local`   | Project — gitignored    |

### Codex

Install a skill directly from this GitHub repo into `~/.codex/skills`:

```bash
python ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --repo enonic/ai-enonic-marketplace \
  --path plugins/enonic-skills/skills/<skill-name>
```

Only skills whose frontmatter includes `Codex` in the `compatibility` field are supported on Codex.

## Repository Layout

```
.claude-plugin/marketplace.json       # Marketplace registry
plugins/
  enonic-skills/
    .claude-plugin/plugin.json        # Plugin manifest
    skills/                           # Skill directories live here
      <skill-name>/
        SKILL.md                      # Required — frontmatter + instructions
        references/                   # Optional — additional documentation
        scripts/                      # Optional — executable code
        assets/                       # Optional — templates, images, data files
```

The marketplace is nested so additional plugins can be added under `plugins/` without restructuring.

Each `SKILL.md` is a YAML frontmatter block followed by Markdown instructions:

```markdown
---
name: example-skill
description: Does X when the user asks for Y.
compatibility: Claude Code, Codex
---

## Steps

1. First, do this.
2. Then, do that.
```

See the [Agent Skills specification](https://agentskills.io/specification) for the full frontmatter reference, and `AGENTS.md` in this repo
for the cross-agent writing convention used here.

## Available Skills

| Skill                                                            | Description                                                                                                                                                                                   | Agent              | Category    |
|------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------|-------------|
| [xp-app-upgrader](plugins/enonic-skills/skills/xp-app-upgrader/) | Upgrade an Enonic XP application from XP 7 to XP 8 — descriptor conversion via `xp8migrator`, build-system reorganization (settings plugin, `xplibs.*` catalog), code-level breaking changes. | Claude Code, Codex | Development |
| [xp-app-debugger](plugins/enonic-skills/skills/xp-app-debugger/) | Debug XP application errors — build failures (Gradle, TypeScript) and server runtime errors (Nashorn/JS stack traces in `server.log`).                                                        | Claude Code        | Development |

## Creating a Skill

1. Create a directory under `plugins/enonic-skills/skills/` matching the skill name.
2. Add a `SKILL.md` with required `name` and `description` frontmatter.
3. Write Markdown instructions in the body (keep under 500 lines).
4. Optionally add `scripts/`, `references/`, or `assets/` subdirectories.
5. Update the "Available Skills" table above.

For multi-agent skills, follow the writing convention in `AGENTS.md` — generic action verbs with the agent-specific tool name in
parentheses (e.g. "edit the file (`Edit` tool in Claude Code)").

## Releasing

1. Ensure you're on `master` with a clean working tree.
2. Bump the version in both `.claude-plugin/marketplace.json` and `plugins/enonic-skills/.claude-plugin/plugin.json`.
3. Commit: `git commit -m "Release vX.Y.Z"`.
4. Tag: `git tag vX.Y.Z`.
5. Push: `git push && git push --tags`.

## License

[MIT](LICENSE)
