---
name: skill-creator-flo
description: Create a new Claude Code skill OR modify an existing one, in a personal marketplace (Type A, stable or beta), as a personal standalone (Type B), or for another tool compatible with agentskills.io (Type C). Auto-detects reference files passed as arguments and skills installed on the machine. Preserves the original slug on modification, snapshots the skill into /tmp/ before editing. Persists config in ~/.claude/skill-creator-flo/config.json (stable + beta_* keys). Invoke when the user wants to create or modify a skill, regardless of setup or intent.
---

# skill-creator-flo — Create or modify a skill

Personal skill creator aligned with Anthropic's authoring principles: explanatory prose over rigid ALL-CAPS tables, theory of mind over rote directives, lean instructions over over-specification. Covers four targets: creation in a personal marketplace (A), personal standalone (B), other tools (C), and modification of an already-installed skill (D).

**Respond in the user's language (English by default if ambiguous). Be concise. Confirm each action in one sentence.**

---

## Principles

This skill assumes nothing about the current user's identity. It works for anyone with their own GitHub config and their own paths. Three creation targets (A, B, C) plus modification (D) of skills already installed on the machine.

**The frontmatter description is the primary trigger.** It tells Claude *what* the skill does and *when* to invoke it. If Claude doesn't trigger a skill at the right moment, the first instinct should be to revisit the description before the logic. Anthropic puts it this way: *"When to trigger, what it does. This is the primary triggering mechanism — include both what the skill does AND specific contexts for when to use it."*

**Progressive disclosure**: this SKILL.md targets fewer than 500 lines. Longer workflows (reference-file detection, modification of an existing skill) live in `references/`, read on demand only when the relevant step fires.

**Theory of mind over ALL-CAPS**: Claude responds better to reasoning than rigid directives. When this skill imposes a guardrail (for example "preserve the original slug on modification"), it explains why (the slug identifies the skill on the Claude Code side; renaming it breaks other users' configs). That logic lives in the "Design rationale" section at the end.

**This skill never**: assumes a GitHub account (local mode available), assumes Claude Code (Type C covers other tools), hardcodes an absolute path, renames a skill during a modification.

---

## Step -1 — Auto-detect reference files (always first)

This step runs before all others, as soon as the skill is invoked, so the user doesn't have to write free-form instructions like "analyze this file deeply".

Scan the initial message. If you spot one or more tokens that look like file paths and the file(s) exist on disk (any extension), read `${CLAUDE_SKILL_DIR}/references/REFERENCE_DETECTION.md` and follow the detailed workflow there: read → questions about role/depth → real analysis → dynamic question on components to reproduce → brief injected into Steps 4, 6a, 6c (creation) or M.5c (modification, regenerate mode).

If no path is detected, go straight to Step 0.0.

The list of components proposed to the user is derived from the actual analysis of the files provided. A `.py` file produces different options than a `.md` or a CSV. Details are in the reference file.

---

## Step 0.0 — What do you want to do? (MAIN BRANCH)

Ask via `AskUserQuestion`:

> "What would you like to do?
>
> - **A) Create a skill in a personal marketplace** (recommended for shareable skills): full plugin marketplace structure, multi-skills, share via `/plugin marketplace add`.
>
> - **B) Create a personal standalone skill** (just for me, here, now): a single file `~/.claude/skills/<slug>/SKILL.md`, no marketplace, no Git, usable immediately.
>
> - **C) Create a skill for another tool** (OpenCode, GitHub Copilot, Cursor, Claude Desktop, etc.): SKILL.md following the agentskills.io standard, placed in the target tool's skills directory.
>
> - **D) Modify an existing skill** (mine or one installed): I list the skills detected on the machine, you pick one, I snapshot before editing, we modify together. I preserve the original slug."

### Based on the answer
- **A** → continue to Step 0.1 (stable vs beta marketplace).
- **B** → SKIP Steps 0.1, 1, 2, 2bis, 8, 9, 10. Go directly to Step 3 (slug). At Step 7, only write `~/.claude/skills/<slug>/SKILL.md`.
- **C** → SKIP the same steps. Go to Step 0.5 (determine target directory). At Step 7, only write the SKILL.md in the target directory.
- **D** → read `${CLAUDE_SKILL_DIR}/references/MODIFICATION_WORKFLOW.md` and follow steps M.1 to M.7. Return here only for the final recap (adapted Step 11).

---

## Step 0.5 — Target directory (Type C only)

If Type C, ask via `AskUserQuestion`:

> "Which target tool are you using? (the skill will be placed in its skills directory)"

Default options:
- **OpenCode** → `~/.opencode/skills/`
- **GitHub Copilot** → varies (see [Copilot docs](https://docs.github.com/en/copilot/concepts/agents/about-agent-skills))
- **Cursor** → varies (see [Cursor docs](https://cursor.com/docs/context/skills))
- **Claude Desktop (macOS)** → `~/Library/Application Support/Claude/skills/`
- **Other / custom path** → free-text via "Other"

Verify the directory exists (otherwise offer to create it). Store as `<target_dir>` for this session, not in the persisted config (each skill may target a different tool).

---

## Step 0 — Load or initialize configuration (Type A only)

Config lives in `~/.claude/skill-creator-flo/config.json` and supports two contexts (stable and beta):

```json
{
  "mode": "github" | "local",
  "github_user": "<login>",
  "author_name": "<full name>",
  "author_url": "<profile URL>",

  "repo_name": "skills",
  "repo_dir": "/Users/.../code/skills",
  "repo_url": "https://github.com/<login>/skills.git",

  "beta_repo_name": "skills-beta",
  "beta_repo_dir": "/Users/.../code/skills-beta",
  "beta_repo_url": "https://github.com/<login>/skills-beta.git"
}
```

### 0.1 — Context choice (stable vs beta)

The skill `skill-creator-flo` detects its own context via its `name:` (suffix `-beta` → beta context by default). But the user may want to create a stable skill from a beta invocation (e.g., add directly to the stable marketplace without going through beta). Ask via `AskUserQuestion`:

> "Which marketplace should this skill go to?
>
> - **Beta** (`<beta_repo_name>`, recommended for a new creation — you can promote to stable later): `<beta_repo_dir>`
> - **Stable** (`<repo_name>`, direct public sharing): `<repo_dir>`"

Store the choice in a local variable `<context>` (values `"beta"` or `"stable"`). The rest of Step 0 uses the matching variables (`repo_name`/`beta_repo_name`, etc.) based on this choice.

### 0.2 — If config exists and is valid

Load `~/.claude/skill-creator-flo/config.json`, verify required keys for the chosen context are present. If so, show a summary ("Mode: <github|local>, repo <context>: <repo_name>, dir: <repo_dir>") and move to Step 1.

If config exists but `beta_*` keys are missing while `<context> = "beta"`, go to 0.3 to fill them in.

### 0.3 — Otherwise, detect mode and auto-fill

In parallel: `command -v gh` and (if present) `gh auth status`.

**Case A — `gh` absent or not authenticated**: propose `local` mode (recommended) or configuring `gh` first.

**Case B — `gh` present and authenticated**: propose `github` mode (recommended) or `local`.

Auto-fill fields via `gh api user` and `git config`. For context-specific fields:

| Field | github mode | local mode |
|---|---|---|
| `<context>_name` | suggest `"skills"` or `"skills-beta"` depending on context | same |
| `<context>_dir` | suggest `$HOME/code/<context>_name` | same |
| `<context>_url` | `https://github.com/<login>/<context>_name.git` | (omitted) |

Ask each choice via `AskUserQuestion` (detected value = first "Recommended" option).

### 0.4 — Save

Write config to `~/.claude/skill-creator-flo/config.json` (mkdir first). Confirm: "Config saved. You can edit it any time in that file."

All following steps use these variables with the chosen context prefix. No hardcoded paths or names.

---

## Step 1 — Pre-flight checks

### github mode
In parallel: `gh auth status`, verify `gh api user --jq .login` returns the `<github_user>` from config (otherwise mismatch — ask what to do), and `git config --global user.email` non-empty.

If anything is missing, say exactly what's missing and propose solutions (fix or switch to local mode), then stop or continue depending on the answer.

### local mode
Check `command -v git`. Without git you can't init a repo, but you can still create files — ask the user whether to continue without git. No need for `gh` or an internet connection.

---

## Step 2 — Repo state (depends on mode and context)

Let `<dir>` = `<context>_dir` (the repo matching the context chosen at Step 0.1).

### github mode
- If `<dir>` doesn't exist:
  - `gh repo view <github_user>/<context>_name --json name 2>/dev/null` to check the remote repo exists
  - If yes → `git clone <context>_url <dir>`
  - If no → bootstrap (Step 2bis)
- If `<dir>` exists → `git -C <dir> pull --ff-only`

If pull/clone fails (conflicts, divergence, permissions), stop and show the error. Reason: silently repairing hides structural problems.

### local mode
- If `<dir>` doesn't exist → bootstrap (Step 2bis local).
- If `<dir>` exists with `.claude-plugin/marketplace.json` → use as-is.
- If `<dir>` exists without `marketplace.json` → ask the user (continue or pick another folder).

---

## Step 2bis — Bootstrap a new repo (first use of the context)

### github mode

1. Confirm via `AskUserQuestion`: "Create repo `<github_user>/<context>_name` (public) now?"
2. If yes:
   - `mkdir -p <dir>/.claude-plugin <dir>/plugins`
   - Create `<dir>/.claude-plugin/marketplace.json`, `README.md`, `LICENSE` (templates below, adapted for beta if applicable — BETA banner in README).
   - `cd <dir> && git init -b main && git add -A && git commit -m "Initial commit: empty <context>_name marketplace"`
   - `gh repo create <github_user>/<context>_name --public --description "<context-appropriate description>" --source=. --push`

### local mode

1. Confirm: "Create folder `<dir>` (local marketplace) now?"
2. If yes: `mkdir -p` + files as above. Optional: `git init` for local versioning.

### Templates

**`marketplace.json` (beta context)**: include the "BETA — unstable versions" note in the top-level description.

**`marketplace.json` (stable context)**: neutral description as in the existing stable.

**`LICENSE`**: MIT with `Copyright (c) <current year> <author_name>`.

**`README.md`**: read `${CLAUDE_SKILL_DIR}/assets/readme-template.md`, substitute placeholders. For the beta context, prepend the warning banner "Personal use, unstable versions" (see the skills-beta README as a model).

---

## Step 3 — Slug of the new skill

`AskUserQuestion` with free text:

> "What's the slug for the new skill? kebab-case, e.g. `pr-review`, `weekly-recap`. If you're creating an experimental variant of an existing skill, suffix `-beta` or `-v2` (lives alongside the original)."

Validation: pattern `^[a-z][a-z0-9-]*$`, and `<dir>/plugins/<slug>/` must not exist. Otherwise explain and ask again.

---

## Step 4 — Frontmatter description (primary trigger)

`AskUserQuestion` with free text:

> "Describe in 1-3 sentences WHEN this skill should be invoked. Start with an action verb (Coordinate..., Generate..., Analyze...). This description is read by Claude to decide whether to activate the skill — it's the primary trigger, so make it specific."

If Step -1 produced a reference brief, propose a **pre-drafted** description based on the brief (e.g., "Transform ANY input into a document structured like `<reference filename>`: sections X/Y/Z, sequence diagram, narrative scenario. Invoke when..."). The user accepts, adjusts, or rewrites.

Keep the final text as-is (no truncation, no auto-rephrasing).

---

## Step 5 — Body authoring mode

`AskUserQuestion` with 3 options:
- **Interactive**: ask questions about the steps, the skill generates everything.
- **Paste a draft**: the user supplies complete markdown.
- **Minimal skeleton**: generate a template to fill in later.

---

## Step 6a — Interactive mode

Before starting, read `${CLAUDE_SKILL_DIR}/references/AUTHORING_GUIDELINES.md` to keep in mind the official limits (frontmatter, < 500 lines, < 5k tokens), progressive disclosure, and Anthropic principles (lean instructions, theory of mind).

If Step -1 produced a reference brief, use it as the starting skeleton: propose steps matching the detected sections / diagrams / rules rather than starting from scratch.

Remind the user about adaptability:

> "This skill may be installed by others. Avoid hardcoding your name, GitHub login, absolute paths, or URLs specific to your account. Prefer runtime detection (`gh api user`, `git config`, `$HOME`). If the skill is deliberately personal, mention it explicitly in the description."

Then ask: language, style (concise / detailed), scope (generic or personal), number of steps (typically 3-7), and for each step: title + free description + relevant shell commands (optional).

While drafting, if you detect hardcoded paths/names, flag them. If the SKILL.md being built exceeds 500 lines, propose progressive disclosure (create `references/` and `assets/`, move details).

---

## Step 6b — "Paste a draft" mode

`AskUserQuestion` with free text:

> "Paste the complete markdown body here (without the `---` frontmatter)."

Use as-is.

---

## Step 6c — "Minimal skeleton" mode

Default:

```markdown
# <Skill name>

<description provided at Step 4>

**Respond in English. Be concise.**

## Step 1 — TODO

To fill in.

## Step 2 — TODO

To fill in.

## Step 3 — TODO

To fill in.
```

If Step -1 produced a brief, replace the `Step N — TODO` entries with pre-titled steps matching the detected components.

---

## Step 7 — Build the new skill's files

### Type A — Personal marketplace

Structure inside `<dir>`:

```
<dir>/plugins/<slug>/
├── .claude-plugin/plugin.json
├── README.md
└── skills/<slug>/SKILL.md
```

**`plugin.json`**: `name`, `description` (full frontmatter), `version: "1.0.0"`, `author` (config), `license: "MIT"`, `homepage` (repo URL in github mode, omitted in local mode).

**`README.md`**: read `${CLAUDE_SKILL_DIR}/assets/readme-plugin-template.md`, substitute placeholders. The template targets a short README (≤ 25 lines) with only: title, full description as a blockquote, created/last-updated dates, a simple 3-node `flowchart LR` Mermaid diagram (`main input → <slug> → main output`), install block, slash command, license. No "Use case", "Contents", or "For other tools" sections — the description doubles as the use case and the root README covers multi-tool install.

Substitutions:
- `<YYYY-MM-DD>` (both occurrences) → `date +%Y-%m-%d` on creation day.
- `<main input>` and `<main output>` → 1-3 words each, derived from the frontmatter description (Step 4). If you can't infer them confidently, ask the user via `AskUserQuestion` ("What's the main input of the skill in 1-3 words?" + "What's the main output?").
- `<slug>`, `<description>`, `<github_user>`, `<context>_name` → as before.

In `local` mode, replace `/plugin marketplace add <github_user>/<context>_name` with `/plugin marketplace add <dir>` (absolute path). The HTML comment at the end of the template is informational only and must be stripped from the final written file (no pollution of the published README).

**`SKILL.md`**: frontmatter `name: <slug>` + `description: <description>` + body built at Step 6.

### Type B — Personal standalone skill

Create only `~/.claude/skills/<slug>/SKILL.md` (mkdir first). No plugin.json, no marketplace.json. Slash `/<slug>` available immediately.

### Type C — Skill for another tool

Create only `<target_dir>/<slug>/SKILL.md` (mkdir first). agentskills.io standard. No plugin.json (other tools don't use it).

**For Types B and C: SKIP Steps 8, 9, 10. Go straight to Step 11.**

---

## Step 8 — Update `marketplace.json` (Type A only)

Read `<dir>/.claude-plugin/marketplace.json`. Append an entry to the `plugins` array:

```json
{
  "name": "<slug>",
  "source": "./plugins/<slug>",
  "description": "<short 1-sentence description>"
}
```

`source` format matters: always use `"./plugins/<slug>"` (explicit relative path). The short form `"<slug>"` with `metadata.pluginRoot` is rejected by the current validator.

Preserve formatting (2 spaces, correct commas). Rewrite the full file. After writing, run `claude plugin validate <dir>` and resolve any errors before continuing.

---

## Step 9 — Update `README.md`

Read `<dir>/README.md`. Find the `## Available skills` section (or equivalent) and its table. Add a row:

```
| `<slug>` | <short 1-sentence description> | <1-sentence use case> |
```

If the section or table doesn't exist, tell the user and ask how to proceed.

---

## Step 10 — Commit and push?

### github mode
`AskUserQuestion`:
- **Yes, commit + push now**: `cd <dir> && git add -A && git commit -m "Add <slug> skill" && git push`
- **No, later**: show the manual commands.

### local mode
- **Yes, local commit** (if `.git` present): `cd <dir> && git add -A && git commit -m "Add <slug> skill"`
- **No, just leave the files**.

No push possible in local mode — by design.

---

## Step 11 — Final recap

Show `✓ Skill <slug> created.` then the matching block:

**Type A, github mode pushed**:
```
Files    : <dir>/plugins/<slug>/
GitHub   : https://github.com/<github_user>/<context>_name/tree/main/plugins/<slug>
Install  : /plugin marketplace add <github_user>/<context>_name  →  /plugin marketplace update <context>_name  →  /plugin install <slug>@<context>_name
Slash    : /<slug>:<slug>
```

**Type A, not pushed**: add `To do: cd <dir> && git add -A && git commit -m "..." && git push`.

**Type A, local mode**: replace the GitHub URL with `Install: /plugin marketplace add <dir>` and mention sharing via zip + `marketplace add <path>` on the recipient's side.

**Type B**: `File: ~/.claude/skills/<slug>/SKILL.md`; slash `/<slug>` available; uninstall `rm -rf ~/.claude/skills/<slug>`.

**Type C**: `File: <target_dir>/<slug>/SKILL.md`; activation per target tool's docs (often auto at next startup).

**Type D (modification)**: The recap is handled by `MODIFICATION_WORKFLOW.md` (M.7) — it includes the snapshot path + rollback command + summary diff + (Type A) `/plugin update` command.

---

## MODIFY sub-workflow (Step 0.0 = D)

When the user picks "Modify an existing skill", read `${CLAUDE_SKILL_DIR}/references/MODIFICATION_WORKFLOW.md` and follow steps M.1 to M.7:

- M.1: detect modifiable skills (standalones, local marketplaces, installed cache copies)
- M.2: present the list, ask which one to modify
- M.3: snapshot to `/tmp/<slug>-snapshot-<timestamp>/`
- M.4: read the full skill (SKILL.md + references/ + assets/ + plugin.json if Type A)
- M.5: modification mode (interactive section-by-section diff / paste new body / regenerate from a reference brief)
- M.6: structural validation (frontmatter, invariant slug, size < 500 lines)
- M.7: commit + push (Type A) or save file (Type B/C) + recap with snapshot + diff

The original slug is preserved unless the user explicitly confirms otherwise (Anthropic rule).

---

## Reference files (progressive disclosure)

| File | When to read |
|---|---|
| `${CLAUDE_SKILL_DIR}/references/REFERENCE_DETECTION.md` | Step -1 — invocation containing existing file paths. Workflow scan → read → analyze → dynamic components → brief. |
| `${CLAUDE_SKILL_DIR}/references/AUTHORING_GUIDELINES.md` | Step 6a/6c — when guiding body authoring, or when the user wants to check a skill's quality. Contains official limits, Anthropic principles, good/bad examples. |
| `${CLAUDE_SKILL_DIR}/references/MODIFICATION_WORKFLOW.md` | Step 0.0 = D — sub-workflow M.1 to M.7 to modify an existing skill. |
| `${CLAUDE_SKILL_DIR}/assets/readme-template.md` | Step 2bis — bootstrap of a new marketplace repo. |
| `${CLAUDE_SKILL_DIR}/assets/readme-plugin-template.md` | Step 7 Type A — README generated at the plugin root. |

`${CLAUDE_SKILL_DIR}` is the Claude Code variable pointing to this skill's folder.

---

## Design rationale

### Adaptability first
No path or name is hardcoded: everything goes through the config (Step 0). Reason: this skill is installable by other users with their own GitHub login, their own folder layout, their own name. If you see an absolute path in generated code, it's probably a leak — flag it.

### No symlinks
Distribution via `/plugin install`, never via `ln -s`. Symlinks introduce subtle cross-machine bugs (paths that work in dev but break on install). If you need to share code between skills, duplicate it or extract it into a shared reference file.

### Strict frontmatter (`name` + `description` only)
That's the agentskills.io standard, and the Anthropic validator rejects custom fields (`version`, `tags`, `keywords`). If you want to version, put it in `plugin.json` at the plugin root, not in SKILL.md.

### Preserve the slug on modify
When the user asks to modify `research-helper`, the updated file stays `research-helper` (not `research-helper-v2`). Reason: the slug identifies the skill on Claude Code's side, and renaming breaks the links, configs, and scripts of other users who have it installed. If you really want a new identity, that's a creation (Step 0.0 = A), not a modification.

### Snapshot before any edit
M.3 copies the full skill to `/tmp/<slug>-snapshot-<timestamp>/`. Reason: if the modification breaks something, rollback in one command (`cp -R /tmp/<slug>-snapshot-<ts>/* <original-path>/`). `/tmp/` is purged on reboot — if you want to persist, copy elsewhere first.

### Bump the version on every modification (Type A)
M.7.0bis bumps the `version` in `plugin.json` on every content change of a Type A plugin. Reason: Claude Code caches installed plugins in `~/.claude/plugins/cache/<repo>/<slug>/<version>/` and only re-downloads the cache if the version changes. Modifying content without bumping the version means `/plugin marketplace update` refreshes the snapshot but not the cache Claude reads at invocation — a very confusing bug, where the GitHub push is correct but the user still sees the old behavior. Bumping forces propagation.

### Multi-source detection of installed skills
M.1 scans four families of locations (standalones, local stable marketplaces, local beta marketplaces, installed cache copies via `/plugin install`). Reason: editable sources (the first two) must be prioritized over read-only cache copies, which will be overwritten at the next `/plugin update`. The skill redirects to the source when it exists locally, and proposes a clone otherwise.

### < 500 lines limit for SKILL.md
Official Anthropic + agentskills.io recommendation. Beyond that, the token cost at each invocation climbs and maintainability drops. When a workflow grows (case of modification in this skill), externalize it into `references/` read on demand — exactly the progressive disclosure pattern.

### Fail fast, don't repair silently
If `git pull` fails, if JSON is invalid, if the slug is taken: stop and say exactly what happened. Silent repairs hide structural problems (wrong GitHub account, pending conflict, remote divergence) that resurface later, more expensive to diagnose.

### GitHub account mismatch detected
If `gh api user --jq .login` doesn't match the `github_user` in config, the skill warns before continuing. Reason: pushing from the wrong account creates a commit with an incorrect author and may expose an unintended identity.
