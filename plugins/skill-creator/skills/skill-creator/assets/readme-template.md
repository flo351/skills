# `<repo_name>` — `<author_name>`'s skills marketplace

Claude Code marketplace bundling `<author_name>`'s public skills. One command to add them all; you enable the ones you want.

## Installation

### For Claude Code

```
/plugin marketplace add <github_user>/<repo_name>
```

Open the **Discover** tab and toggle the skills you want (Space to toggle), or install via CLI:

```
/plugin install <skill-name>@<repo_name>
```

- **Update**: `/plugin marketplace update <repo_name>`
- **Uninstall**: `/plugin uninstall <skill-name>@<repo_name>`

### For other agent tools (Claude Desktop, OpenCode, Codex, Antigravity, GitHub Copilot, Cursor, etc.)

Skills follow the open [agentskills.io](https://agentskills.io) standard. Clone the repo and copy the skill you want into your tool's skills directory:

```bash
git clone https://github.com/<github_user>/<repo_name>.git
cp -R <repo_name>/plugins/<skill-name>/skills/<skill-name> <YOUR_TOOL_SKILLS_DIR>/
```

## Available skills

| Name | Description | Use case |
|---|---|---|
| _(none yet — add some with `skill-creator`)_ | | |

## License

MIT — see [LICENSE](./LICENSE).
