# Modifying an existing skill — M.1 to M.7 workflow

To read when the user has chosen "Modify an existing skill" at Step 0.0 of the main SKILL.md. Describes the complete sub-workflow to modify a skill while preserving its original slug and keeping a rollback snapshot.

Core principle: always modify the **source** of a skill (a file the user can edit), never the **installed cache copy** (which will be overwritten on the next `/plugin update`). When the source isn't accessible locally, the skill offers to clone first.

---

## M.1 — Detect all modifiable skills on the machine

Scan four families of locations in parallel, read-only:

### M.1.1 — Personal standalone skills

```bash
ls -d ~/.claude/skills/*/  2>/dev/null
```

Each folder that contains a `SKILL.md` is a standalone. Directly editable.

### M.1.2 — Skills in cloned local marketplaces

Read the config (`~/.claude/skill-creator-flo/config.json`) to fetch known `repo_dir` paths:
- `repo_dir` (stable, e.g. `~/code/skills`)
- `beta_repo_dir` (beta, e.g. `~/code/skills-beta`)

For each existing `<repo_dir>`:
```bash
ls -d <repo_dir>/plugins/*/skills/*/  2>/dev/null
```

Each folder containing a `SKILL.md` is a local marketplace skill. Editable. Source of truth for the matching `/plugin install`.

### M.1.3 — Skills installed via /plugin install (read-only copies)

```bash
ls -d ~/.claude/plugins/cache/*/plugins/*/skills/*/  2>/dev/null
ls -d ~/.claude/plugins/marketplaces/*/plugins/*/skills/*/  2>/dev/null
```

These locations contain installed copies of plugins. Read-only: do not edit directly (overwritten on next update). Shown in M.2 with note "(installed, source in repo X)" if the source can be resolved locally.

### M.1.4 — Metadata extraction

For each SKILL.md found, extract via Read:
- `name:` from frontmatter
- First line of `description:` (truncated to ~100 chars for display)
- Absolute path of the skill folder (parent of SKILL.md)
- Family among: `standalone`, `marketplace-local-stable`, `marketplace-local-beta`, `installed-cache`, `bundled-marketplace`

For skills in cache/bundled, try to resolve the source: if the marketplace name in the path matches a known `repo_name` from config, propose the matching source (`<repo_dir>/plugins/<slug>/`).

---

## M.2 — Present the list and ask which to modify

`AskUserQuestion` with the consolidated list, grouped by family. If more than ~10 entries, first ask to filter by family or keyword to shrink the list.

Display format:

```
<slug> (<family>) — <short description>
Path: <path>
```

### If the user picks an installed-cache or bundled skill

Check whether the source exists locally:
- **Source present locally** → propose: "Skill `<slug>` is installed read-only. Its source is in `<repo_dir>/plugins/<slug>/`. I'll edit the source; you'll then `/plugin update <slug>@<repo_name>` to refresh. OK?"
- **Source absent** → propose two options:
  1. Clone the source repo first and edit the source (recommended).
  2. Make a standalone copy in `~/.claude/skills/<slug>-fork/` and edit it (discouraged: creates a divergent fork).

Never propose editing the cache copy directly.

---

## M.3 — Snapshot before editing

Create a timestamped backup of the full skill folder (not just SKILL.md):

```bash
SNAPSHOT_DIR="/tmp/<slug>-snapshot-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$SNAPSHOT_DIR"
cp -R "<skill-dir>" "$SNAPSHOT_DIR/"
```

Store `$SNAPSHOT_DIR` in a local variable for the final recap (M.7).

Confirm to the user: "Snapshot: `<SNAPSHOT_DIR>` (will be purged on reboot — copy elsewhere if you want to keep it). Rollback: `cp -R <SNAPSHOT_DIR>/<slug>/* <original-path>/`".

---

## M.4 — Read the full skill

Read all the chosen skill's content:
1. The current SKILL.md (capture frontmatter + full body).
2. List and read files in `references/` and `assets/` of the skill folder (if present).
3. If Type A (marketplace): also read `<dir>/plugins/<slug>/.claude-plugin/plugin.json` and find the skill's row in `<dir>/.claude-plugin/marketplace.json`.

Show a short recap to the user:

> "Skill `<slug>` (family <X>). Current frontmatter — name: `<slug>`, description: `<start of description...>`. Body: <N> lines. References/: <list or none>. Assets/: <list or none>. Plugin.json: version `<version>`."

---

## M.5 — Modification mode

`AskUserQuestion` with 3 options:

- **M.5a — Interactive (recommended)**: you tell me what to change in free text or by sections, I propose a section-by-section diff you validate.
- **M.5b — Paste**: you paste the new body (or frontmatter) in full, I replace.
- **M.5c — Regenerate from reference files**: you've given me reference files (Step -1 produced a brief), I regenerate a new body that replaces the old one — useful for a major rewrite.

### M.5a — Interactive mode

Ask: "What do you want to change?" (free text).

Typical answers:
- "Add a step between Step 2 and Step 3 to validate X."
- "Rewrite the frontmatter description to mention Y."
- "Replace the table in section Z with explanatory prose stating the why."

For each request:
1. Cite the current section (lines or title).
2. Propose the modified version (readable before/after diff).
3. Wait for explicit confirmation (`OK` / `adjust` / `cancel`).
4. Apply only after OK.

Repeat until the user says "that's all".

### M.5b — Paste mode

Ask: "Paste the new content here. Specify whether it's the full body (without frontmatter) or the full SKILL.md (with frontmatter)."

Verify:
- If frontmatter included: `name:` still matches the original slug (otherwise warn — no silent rename).
- If body only: keep the existing frontmatter unchanged.

### M.5c — Regenerate mode (with reference brief)

If Step -1 produced a brief (see `REFERENCE_DETECTION.md` -1.7), use it to regenerate a new body:
1. Fetch the components checked by the user at -1.6.
2. Generate a new structured body: steps matching the components, style instructions, format consistent with the reference files.
3. The frontmatter `name:` stays unchanged. The `description:` may be proposed for update if the brief suggests it (ask for confirmation).

---

## M.6 — Structural validation

Before writing changes, check:

1. **Frontmatter complete**: `name:` and `description:` present, valid YAML. If missing or broken, refuse and explain.
2. **Slug invariant**: the modified frontmatter `name:` is identical to the original. If different, warn explicitly ("You changed name from `<old>` to `<new>`. Confirm? This may break configs of users who installed this skill.").
3. **SKILL.md size**: count lines (`wc -l`). If > 500, propose progressive disclosure (create a file in `references/`, move a long section). Not blocking, just a warning.
4. **Single YAML frontmatter**: a single `---open---close---` block at the start of the file. No double frontmatter.
5. **Description length**: between 50 and 1024 characters. If too short (< 50), Claude risks not triggering the skill (remind of the "primary trigger" principle).

If a non-blocking validation fails, ask the user whether to continue or fix.

---

## M.7 — Commit, push, recap

### M.7.0 — Patch the "Last updated" date in the plugin README (Type A only)

Before committing, update the `**Last updated**: ` line of `<repo_dir>/plugins/<slug>/README.md` with today's date (`date +%Y-%m-%d`). The `**Created**: ` line stays unchanged — it's immutable, the original plugin creation date.

If the README is in the old format (before the introduction of dates and Mermaid diagram), offer the user via `AskUserQuestion` to migrate it to the new short format (title + description + dates + mermaid + install + license — see `${CLAUDE_SKILL_DIR}/assets/readme-plugin-template.md`). If yes, apply the conversion using the current date for **Created** and **Last updated** (best effort: the real creation date is lost); flag it to the user so they can fix it manually if needed by consulting `git log --follow --diff-filter=A` on the plugin's `plugin.json` file.

### M.7.0bis — Bump the version in plugin.json (Type A only) — CRITICAL

**Problem to avoid**: Claude Code caches installed plugins in `~/.claude/plugins/cache/<repo>/<slug>/<version>/`. When the user runs `/plugin marketplace update <repo>`, Claude Code refreshes the marketplace snapshot but **only re-downloads the plugin into its cache if the `version` in `plugin.json` has changed**. If you modify content (SKILL.md, references/, assets/) without bumping the version, existing sessions keep reading the old cache and `/plugin marketplace update` appears to do nothing — a very confusing bug for the user.

**Rule**: for any content change to a Type A plugin (SKILL.md, files in references/ or assets/, README, plugin.json itself), **bump the version** in `<repo_dir>/plugins/<slug>/.claude-plugin/plugin.json` before committing.

**How to bump**:

1. Read the current version in `plugin.json` (`"version"` field).
2. Ask the user via `AskUserQuestion` which bump type to apply, pre-computing the three next versions:
   - **Patch** (recommended for most edits: rule tweak, fix, clarification): `X.Y.(Z+1)`
   - **Minor** (new step, new variant, structural refactor): `X.(Y+1).0`
   - **Major** (slug rename, rewrite breaking existing usage): `(X+1).0.0`
3. Write the new version into `plugin.json`.

**Exception — when not to bump**: if the only thing modified is a file not distributed in the plugin (e.g., an internal note outside the plugin folder, or a typo fix in a comment outside the skill). But when in doubt, bump: a needless patch costs nothing; a missed bump costs a debugging session.

**For Type B (standalone) and Type C (other tool)**: no `plugin.json`, so no bump. The file is read directly from source on every invocation. Skip this step.

### Action depending on the original Type of the modified skill

| Type | Action |
|---|---|
| **A — local stable/beta marketplace** | `git -C <repo_dir> add -A && git -C <repo_dir> commit -m "Update <slug>: <1-line summary from the user>"`. Ask whether to push: if yes, `git -C <repo_dir> push`. |
| **B — standalone** | No commit. The file is updated in place at `~/.claude/skills/<slug>/SKILL.md`. |
| **C — other tool** | Typically no Git commit. The file is updated in place at `<target_dir>/<slug>/`. |

### Final recap

Show:

```
✓ Skill <slug> modified.

Snapshot         : <SNAPSHOT_DIR>
Rollback         : cp -R <SNAPSHOT_DIR>/<slug>/* <original-path>/
                   (snapshot will be purged on next reboot)

Diff summary     : <N> lines added, <M> removed in SKILL.md
                   <K> other files modified (references, plugin.json, marketplace.json…)

[Type A pushed]
Version          : <old> → <new>
GitHub           : https://github.com/<github_user>/<context>_name/tree/main/plugins/<slug>
To refresh       : /plugin marketplace update <context>_name  →  /plugin update <slug>@<context>_name
If no effect     : the local cache may be stale. Delete it:
                   rm -rf ~/.claude/plugins/cache/<context>_name/<slug>
                   then /reload-plugins.

[Type A not pushed]
To do            : cd <repo_dir> && git push

[Type B]
Slash command    : /<slug> (already active, reload if needed)

[Type C]
Activation       : per target tool's docs (often auto at next startup)
```

---

## Edge cases

### User wants to rename the slug despite the warning
Acceptable if the user explicitly confirms after the M.6 warning. The workflow becomes hybrid:
1. The snapshot is kept under the old name.
2. The skill folder is renamed (`mv <old-path> <new-path>`).
3. The frontmatter `name:` is updated.
4. For Type A: `plugin.json` (`name`), `marketplace.json` (entry `name`), repo `README.md` are updated.
5. Warn that users with the old slug installed will need to `/plugin uninstall <old>@<repo>` then `/plugin install <new>@<repo>`.

### Target skill has no `references/` nor `assets/`
M.4 skips reading those folders, M.5a only proposes SKILL.md edits, M.6 validates against SKILL.md only.

### Target skill exceeds 500 lines
M.6 emits the warning, proposes progressive disclosure. If the user wants to add even more content, strongly recommend splitting into `references/<topic>.md` before continuing.

### Multiple installed skills share the same slug (multi-marketplace collision)
Possible if the user installed `<slug>@stable` and `<slug>@beta` at the same time. M.1 must list both. M.2 must distinguish them by full path so the choice is unambiguous.
