# skill-creator

> Create a new agent skill OR modify an existing one, in a personal marketplace (Type A, stable or beta), as a personal standalone (Type B), or for another tool compatible with agentskills.io (Type C, e.g. Codex, Antigravity, OpenCode, Cursor, Copilot, Claude Desktop). Auto-detects reference files passed as arguments and skills installed on the machine. Preserves the original slug on modification, snapshots the skill into /tmp/ before editing. Persists config in ~/.claude/skill-creator/config.json (stable + beta_* keys). Invoke when the user wants to create or modify a skill, regardless of setup or intent.

```mermaid
flowchart LR
    A[user request] --> B[skill-creator]
    B --> C[skill files]
```

## Installation

```
/plugin install skill-creator
/plugin install skill-creator@<marketplace>
```

First invocation asks for GitHub CLI auth, marketplace names, repo visibility, and (if not Claude Code) the target tool. No identity, no path, no GitHub login is hardcoded — everything is detected or asked at runtime.

Slash: `/skill-creator`.

## License

MIT — see [LICENSE](../../LICENSE).
