# sinep skills

Agent skills from [sinep.dev](https://sinep.dev), usable by Claude Code, Codex, opencode and any agent
that reads `SKILL.md` skills.

| Skill | What it does |
|---|---|
| [`architecture-docs`](skills/architecture-docs/SKILL.md) | Writes and updates `docs/architecture/`: how the project links to the services around it (each linking to *their* architecture docs), its runtime containers, and its layered dependency tree generated from the code, with Mermaid diagrams. Same layout in every repo, so documented services form one navigable map. |
| [`sinep-messaging`](skills/sinep-messaging/SKILL.md) | Messages people through the sinep.dev MCP chat tools: finds the right conversation, reads context, writes like the user does in that chat, shows the draft before sending. Needs the sinep.dev MCP. |

## Recommend it in a project (Claude Code)

Commit this to the project's `.claude/settings.json`. Everyone who opens the repo in Claude Code
gets the marketplace and the plugin listed and enabled for that project:

```json
{
  "extraKnownMarketplaces": {
    "sinep-skills": { "source": { "source": "github", "repo": "skrcka/sinep-skills" } }
  },
  "enabledPlugins": { "sinep-skills@sinep-skills": true }
}
```

## Install it yourself

Claude Code:

```
/plugin marketplace add skrcka/sinep-skills
/plugin install sinep-skills@sinep-skills
```

Other agents: copy a folder from `skills/` into the agent's skills directory, e.g.
`~/.agents/skills/`, `~/.codex/skills/` or `~/.config/opencode/skills/`, or into the project
(`.claude/skills/`, `.agents/skills/`).

In sinep, every skill here is built in: Settings → Agent configs → *sinep skills* switches each one
on for all your agents.

## Source

Maintained in the sinep repo under `skills/` and published here; edit there.
