# enonic-marketplace

Enonic plugins for AI coding agents, following the [Agent Skills specification](https://agentskills.io/specification).

## Layout

```
.claude-plugin/marketplace.json       # Claude Code marketplace registry
plugins/
  agent-skills/
    .claude-plugin/plugin.json        # Plugin manifest
    skills/
      xp-app-upgrader/SKILL.md        # compatibility: Claude Code, Codex
      xp-app-debugger/SKILL.md        # compatibility: Claude Code
```

Each skill is a self-contained directory with a `SKILL.md` (YAML frontmatter + Markdown instructions) and optional `references/`,
`scripts/`, and `assets/` subdirectories. The `compatibility` frontmatter field declares which agents the skill targets.

## Use with Claude Code

```
/plugin marketplace add enonic/ai-enonic-marketplace
/plugin install agent-skills@enonic-marketplace
```

## Use with Codex

Install a skill directly from this repo into `~/.codex/skills`:

```bash
python ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --repo enonic/ai-enonic-marketplace \
  --path plugins/agent-skills/skills/xp-app-upgrader
```

Only skills whose frontmatter includes `Codex` in the `compatibility` field are supported on Codex.

## Use with other agents

Any agent that follows the Agent Skills specification can load `SKILL.md` files directly from `plugins/agent-skills/skills/<skill-name>/`.
The skill body uses Claude Code tool names; agents on other platforms should map them to their equivalent tools.

## Writing skills for this marketplace

When a skill targets more than one agent (`compatibility: Claude Code, Codex`, etc.), write instructions in **generic action verbs** and
mention the agent-specific tool in parentheses — never as the primary phrasing. This keeps the body readable on every supported agent while
still giving Claude Code (or whichever agent) the exact tool name to bind to.

- Good: `Edit the file in-place (Edit tool in Claude Code) to change build.gradle.`
- Good: `Search the project for usages (Grep tool in Claude Code).`
- Bad: `Use Edit for in-place changes.` — names a Claude tool as the imperative; other agents have no such tool to bind to.
- Bad: `Use the Grep tool.` — same problem.

Single-agent skills (`compatibility: Claude Code`) can use Claude tool names directly, but the generic-verb-plus-parenthetical form is still
preferred — it costs nothing and survives a later widening of `compatibility`.
