# Installation

## Claude Code

```bash
/plugin marketplace add anyways-open/agents-skills
/plugin install anyways@agents-skills
```

Then restart Claude Code or run `/reload-plugins`.

To make the plugin available to your whole team, add to your project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "agents-skills": {
      "source": {"source": "github", "repo": "anyways-open/agents-skills"}
    }
  }
}
```

## Gemini CLI

```bash
gemini extensions install https://github.com/anyways-open/agents-skills
```

## Codex

Add to your project's `codex.md` or agent instructions:

```
Fetch and follow instructions from
https://raw.githubusercontent.com/anyways-open/agents-skills/refs/heads/main/skills/anyways-mcp/SKILL.md
https://raw.githubusercontent.com/anyways-open/agents-skills/refs/heads/main/skills/anyways-projects/SKILL.md
https://raw.githubusercontent.com/anyways-open/agents-skills/refs/heads/main/skills/anyways-datasets/SKILL.md
```

## Other agents

The skills are standard markdown files. Point your agent at the relevant file:

- **MCP setup:** `skills/anyways-mcp/SKILL.md`
- **Projects, scenarios, networks:** `skills/anyways-projects/SKILL.md`
- **Datasets (manual / counters / locations):** `skills/anyways-datasets/SKILL.md`

Each `SKILL.md` has a YAML frontmatter `description` field explaining when to use it.

Clone the repo and point your agent at the files, or fetch them directly:

```
https://raw.githubusercontent.com/anyways-open/agents-skills/refs/heads/main/skills/anyways-mcp/SKILL.md
https://raw.githubusercontent.com/anyways-open/agents-skills/refs/heads/main/skills/anyways-projects/SKILL.md
https://raw.githubusercontent.com/anyways-open/agents-skills/refs/heads/main/skills/anyways-datasets/SKILL.md
```
