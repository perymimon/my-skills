---
name: create-skill
description: Scaffold a new Claude Code skill — creates SKILL.md with correct frontmatter and registers it in marketplace.json.
user-invokable: true
args:
  - name: name
    description: Kebab-case skill name (e.g. "my-skill")
    required: true
  - name: plugin
    description: Plugin folder name to nest the skill under. Defaults to the skill name.
    required: false
---

A Claude Code skill is a markdown file with YAML frontmatter that Claude loads
on demand. The marketplace registry (`marketplace.json` or
`.claude-plugin/marketplace.json`) is how skills are discovered.

---

## Mental Model

```
marketplace.json              ← root registry
plugins/<plugin>/
  skills/<skill-name>/
    SKILL.md                  ← the actual skill content
```

Each skill is independently addressable. The `source` block in`marketplace.json`
tells Claude where to pull the skill from at install time.

---

## Step 1 — Create the SKILL.md

File path: `plugins/<plugin>/skills/<name>/SKILL.md`

```yaml
---
name: <name>
description: <one-line summary used for skill discovery — be specific>
user-invokable: true          # false if only invoked by other skills
args:
  - name: topic               # optional: narrow the skill's focus
    description: Specific area to cover (e.g. "auth", "uploads")
    required: false
---
```

### Body conventions

- **Lead with the key mental model or gotcha** — the "why" before the "how"
- Use fenced code blocks with language tags
- Mark correct vs. wrong patterns:
  ```ts
  import { something } from 'correct-path'   // ✅
  import { something } from 'wrong-path'     // ❌
  ```
- End with a **Quick Reference** table for skills with multiple patterns

---

## Step 2 — Register in marketplace.json

Add an entry to the `plugins[]` array.

**Local marketplace** (relative path string):
```json
{
  "name": "<name>",
  "description": "<same one-liner as frontmatter>",
  "source": "./plugins/<plugin>/skills/<name>"
}
```

**Remote GitHub subdirectory** (note: key is `"source"`, not `"type"`):
```json
{
  "name": "<name>",
  "description": "<same one-liner as frontmatter>",
  "source": {
    "source": "git-subdir",
    "url": "<owner>/<repo>",
    "path": "plugins/<plugin>/skills/<name>",
    "ref": "main"
  }
}
```

### Required top-level fields (validate these exist)

```json
{
  "name": "my-claude-skills",
  "description": "...",
  "owner": {
    "name": "<github-username>",
    "url": "https://github.com/<github-username>"
  },
  "plugins": []
}
```

Common schema errors:

- `owner` must be an **object** `{ name, url }`, not a plain string  ❌ `"owner": "perymimon"`
- The key inside a source object is `"source"`, not `"type"`  ❌ `{ "type": "github" }`
- Old `"path"` field is invalid — use `"source"` string or object  ❌ `"path": "plugins/foo"`

---

## Checklist

- [ ] `plugins/<plugin>/skills/<name>/SKILL.md` created
- [ ] Frontmatter has `name`, `description`, `user-invokable`
- [ ] Body leads with mental model / key gotcha
- [ ] Code blocks use ✅ / ❌ to distinguish patterns
- [ ] Entry added to `marketplace.json` with correct `source` object
- [ ] `marketplace.json` has top-level `owner` as an object

---

## Quick Reference

| Field            | Type            | Notes                                        |
|------------------|-----------------|----------------------------------------------|
| `owner`          | `{ name, url }` | object, not string                           |
| `source`         | string          | relative path — local installs               |
| `source.source`  | `"git-subdir"`  | key is `"source"`, not `"type"`              |
| `source.url`     | `"owner/repo"`  | short form or full https URL                 |
| `source.path`    | `"plugins/..."` | subdirectory within repo                     |
| `source.ref`     | `"main"`        | branch (optional)                            |
| `user-invokable` | `true\|false`   | whether user can `/skill-name`               |
