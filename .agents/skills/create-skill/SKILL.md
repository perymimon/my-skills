---
name: create-skill
description: Scaffold a new Claude Code skill in this repo — creates .claude/skills/<name>/SKILL.md with correct frontmatter and updates .claude-plugin files.
user-invocable: true
argument-hint: "[skill-name]"
---

A Claude Code skill is a markdown file with YAML frontmatter in `.claude/skills/<name>/SKILL.md`. The `.claude-plugin/` directory tells Claude Code where to find them.

If an argument is provided, use it as the skill name.

---

## Repo structure

```
.claude-plugin/
  marketplace.json    ← registry for npx skills add / claude plugin marketplace add
  plugin.json         ← declares where skills live: "skills": "./.claude/skills"
.claude/
  skills/
    <skill-name>/
      SKILL.md        ← shows up in /skills list
```

---

## Step 1 — Create .claude/skills/<name>/SKILL.md

```yaml
---
name: <skill-name>
description: One-line summary shown in /skills list — be specific about when to load this skill.
user-invocable: true
argument-hint: "[optional-arg]"
---
```

### Body conventions

- **Lead with the key mental model or gotcha** — the "why" before the "how"
- Use fenced code blocks with language tags
- Mark correct vs. wrong patterns:
  ```ts
  import { x } from 'correct'   // ✅
  import { x } from 'wrong'     // ❌
  ```
- Reference `$ARGUMENTS` (or the argument-hint) to narrow scope when a topic is passed
- End with a **Quick Reference** table for skills covering multiple patterns

---

## Step 2 — Update .claude-plugin/marketplace.json

No changes needed — `"source": "./"` already covers the whole repo.
Just bump the `version` in both `marketplace.json` and `plugin.json` when adding skills.

---

## Common frontmatter mistakes

| Wrong | Right |
|-------|-------|
| `user-invokable: true` | `user-invocable: true` |
| `args: [{name: topic}]` | `argument-hint: "[topic]"` |
| `name` mismatches directory | `name` must match the folder name |

---

## Install this skills repo

```sh
# Via npx
npx skills add perymimon/my-skills

# Via Claude Code CLI
claude plugin marketplace add https://github.com/perymimon/my-skills
claude plugin install my-claude-skills
```
