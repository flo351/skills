# `skills` — flo351's Claude Code skills marketplace

Personal marketplace bundling flo351's public Claude Code skills. One command to add them all; you enable the ones you want.

## Installation

### For Claude Code

```
/plugin marketplace add flo351/skills
```

Open the **Discover** tab and toggle the skills you want (Space to toggle), or install via CLI:

```
/plugin install <skill-name>@skills
```

- **Update**: `/plugin marketplace update skills`
- **Uninstall**: `/plugin uninstall <skill-name>@skills`

### For other tools (Claude Desktop, OpenCode, GitHub Copilot, Cursor, etc.)

Skills follow the open [agentskills.io](https://agentskills.io) standard. Clone the repo and copy the skill you want into your tool's skills directory:

```bash
git clone https://github.com/flo351/skills.git
cp -R skills/plugins/<skill-name>/skills/<skill-name> <YOUR_TOOL_SKILLS_DIR>/
```

## Available skills

| Name | Description | Use case |
|---|---|---|
| `skill-creator-flo` | Create or modify a Claude Code skill | End-to-end scaffolding with GitHub marketplace integration |

## License

MIT — see [LICENSE](./LICENSE).
