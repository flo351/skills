# Guidelines for writing a good SKILL.md

To read when guiding the user through authoring a new skill, OR when the user asks to "check the quality" of an existing skill.

## Official sources (2026)

- [Anthropic Claude Code docs](https://code.claude.com/docs/en/skills) — Claude Code reference
- [Anthropic Claude API docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) — cross-product view
- [agentskills.io specification](https://agentskills.io/specification) — open standard

## Concrete limits

| Item | Limit |
|---|---|
| `name` (frontmatter) | 1-64 chars, lowercase kebab-case, no double-hyphen, no leading/trailing hyphen |
| `description` (frontmatter) | 1-1024 chars; must say WHAT the skill does AND WHEN to invoke it |
| Combined `description` + `when_to_use` | truncated at 1536 chars in the skills listing |
| `SKILL.md` body | **< 500 lines** recommended (Anthropic + agentskills.io) |
| `SKILL.md` body (tokens) | **< 5,000 tokens** when the skill is loaded into context (Anthropic) |
| `compatibility` (optional) | 1-500 chars |

## "Progressive disclosure" pattern

When SKILL.md exceeds 500 lines, **split it**: keep the main workflow in `SKILL.md`, move details into reference files read on demand.

```
my-skill/
├── SKILL.md          # main workflow (< 500 lines)
├── references/       # detailed docs (read on demand)
│   └── REFERENCE.md
├── scripts/          # executables (never loaded into context)
│   └── helper.py
└── assets/           # templates, schemas, examples
    └── template.md
```

Reference from `SKILL.md`:

```markdown
For the detailed rules, see [references/REFERENCE.md](references/REFERENCE.md).
The README template is at `${CLAUDE_SKILL_DIR}/assets/readme-template.md`.
```

`${CLAUDE_SKILL_DIR}` is the Claude Code variable pointing to the skill's directory (useful for absolute paths when cwd may vary).

## Minimal compliant frontmatter

```yaml
---
name: pr-review
description: Review a pull request for bugs, security, and performance. Starts by fetching the diff via gh, then analyzes along 4 axes (business logic, security, perf, readability). Invoke when the user asks to review a PR by number or wants a second opinion on changes.
---
```

Frontmatter anti-patterns:
- `version`, `tags`, `keywords` — not in agentskills.io spec
- Vague description ("helps with X") → Claude won't know when to invoke
- Description without "Invoke when..." → missing the trigger

## Body style

- **Action verb in the first sentence** (e.g., "Coordinate...", "Generate...", "Analyze...").
- **No narration** ("This skill will...") → write instructions directly.
- **No meta comments** ("Note: this skill is cool") → keep only what helps Claude execute.
- **Numbered steps** when the workflow has an order. Markdown sections otherwise.
- **Explicit style header** (language, brevity): `**Respond in the user's language. Be concise.**`
- **Concrete examples** (GOOD vs BAD) when a precise format is critical.

## Good structural examples

### Short skill (< 100 lines) — inline reference

```markdown
---
name: api-conventions
description: REST API conventions for this codebase. Apply when the user writes or modifies an endpoint.
---

When writing an API endpoint:
- RESTful naming (`/users/:id`, not `/getUser`)
- Error format `{error: {code, message, details}}`
- Systematic input validation
```

### Workflow skill (200-500 lines) — structured steps

7 clear steps, a reference table, GOOD/BAD examples. Good model.

### Long skill (> 500 lines) — progressive disclosure

If you exceed 500 lines, split. The main SKILL.md contains the workflow and references annexed files for details.

## Adaptability (generic vs personal)

- **Generic skill** (target: public sharing):
  - No absolute paths (`/Users/<you>/...`)
  - No hardcoded GitHub login
  - No hardcoded org name
  - Runtime detection: `git rev-parse --show-toplevel`, `gh api user`, `git config`, `$HOME`
  - Mention in the description: "Invoke when any user..."

- **Personal skill** (target: private use):
  - Hardcoding OK but **mention explicitly** in the description: "Personal skill for the \<X\> workflow, not designed for sharing."
  - Better placed as Type B (standalone `~/.claude/skills/`) rather than in a public marketplace.

## Common mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| `description` < 50 chars, vague | Claude doesn't trigger the skill | Add keywords and "Invoke when X..." |
| Monolithic body > 500 lines | High token cost in context | Progressive disclosure (references/, assets/) |
| Hardcoded absolute paths | Unusable by others | Runtime variables (`$HOME`, `git rev-parse`) |
| Frontmatter with custom fields | Claude Code validators reject | Stick to the agentskills.io standard |
| Symlinks to the skill | Cross-machine bugs | Distribute via marketplace or cp -R |
| No "Respond in X" line | Skill may answer in the wrong language | Add `**Respond in the user's language. Be concise.**` |

---

## Anthropic principles

Source: [github.com/anthropics/skills](https://github.com/anthropics/skills). Four principles guide Anthropic's official skill authoring, which also apply to any personal skill.

### 1. The description is the primary trigger

> *"When to trigger, what it does. This is the primary triggering mechanism — include both what the skill does AND specific contexts for when to use it."*

If Claude doesn't trigger a skill at the right moment, the first reflex must be to revisit the description before the logic. Good description = an action verb up front + an "Invoke when..." sentence + domain-specific keywords. No vagueness ("helps with X").

### 2. Progressive disclosure

Metadata (frontmatter ~100 words) always loaded in context. SKILL.md body < 500 lines, loaded only when the skill fires. Bundled resources (`references/`, `assets/`, `scripts/`) read on demand, never preloaded.

Concretely: if a section of SKILL.md grows beyond occasional use (an edge-case workflow, a reference table, a long template), externalize it into `references/<topic>.md` and leave just a pointer in SKILL.md.

### 3. Lean instructions over ALL-CAPS

> *"Remove things that aren't pulling their weight. Explain why behind requests rather than issuing rigid ALL-CAPS directives — today's LLMs respond better to reasoning than rote rules."*

No "STRICT RULE: NEVER X". Prefer "Reason: X. If you see Y, that's probably a leak; flag it." Claude will then generalize to unforeseen cases instead of trying to match the rule literally.

### 4. Theory of mind

> *"They have good theory of mind and when given a good harness can go beyond rote instructions."*

When you describe a workflow, give the context and intent, not just the procedure. Claude infers edge cases better when it understands why a step exists. Avoid over-specifications like "do exactly X, then Y, then Z" when you can say "the goal is Q, and here are the tools you have".

### Link to the anti-patterns above

These Anthropic principles complement (not replace) the anti-patterns listed. A SKILL.md can respect the < 500 lines limit (technical anti-pattern) while being filled with ALL-CAPS directives (stylistic anti-pattern). Good authoring = both levels.
