---
description: Scaffold a new Claude Code skill — creates commands/<name>.md with correct frontmatter and registers it in marketplace.json.
argument-hint: [skill-name]
---

A Claude Code plugin command is a markdown file in `commands/` with YAML frontmatter. The marketplace registry (`.claude-plugin/marketplace.json`) is how plugins are discovered and installed.

If `$ARGUMENTS` is provided, use it as the skill/command name.

---

## Mental Model

```
.claude-plugin/
  marketplace.json          ← root registry

plugins/<plugin-name>/
  commands/
    <command-name>.md       ← shows up in / command list
  agents/                   ← optional sub-agents
  README.md
```

Each `commands/*.md` file becomes one slash command. The filename (without `.md`) is the command name.

---

## Step 1 — Create the command file

File path: `plugins/<plugin>/commands/<name>.md`

```markdown
---
description: One-line summary shown in the / command list
argument-hint: [optional-arg]
---

Your command content here. Use $ARGUMENTS to reference what the user typed after the command name.
```

### Body conventions

- **Lead with the key mental model or gotcha** — the "why" before the "how"
- Use fenced code blocks with language tags
- Mark correct vs. wrong patterns:
  ```ts
  import { something } from 'correct-path'   // ✅
  import { something } from 'wrong-path'     // ❌
  ```
- End with a **Quick Reference** table for commands covering multiple patterns
- Reference `$ARGUMENTS` to narrow scope when a topic is passed

---

## Step 2 — Register the plugin in marketplace.json

The `source` points to the **plugin root** (not the individual command file).

**Local marketplace** (relative path string):
```json
{
  "name": "<plugin-name>",
  "description": "<one-liner>",
  "source": "./plugins/<plugin-name>"
}
```

**Remote GitHub subdirectory:**
```json
{
  "name": "<plugin-name>",
  "description": "<one-liner>",
  "source": {
    "source": "git-subdir",
    "url": "<owner>/<repo>",
    "path": "plugins/<plugin-name>",
    "ref": "main"
  }
}
```

### Required top-level fields

```json
{
  "name": "my-marketplace",
  "description": "...",
  "owner": { "name": "<github-username>", "url": "https://github.com/<username>" },
  "plugins": []
}
```

---

## Common schema errors

- `owner` must be an **object** `{ name, url }`, not a plain string  ❌ `"owner": "perymimon"`
- The key inside a source object is `"source"`, not `"type"`  ❌ `{ "type": "git-subdir" }`
- `source` points to **plugin root**, not to the command file  ❌ `"./plugins/foo/commands/bar.md"`

---

## Install & test locally

```sh
# Add your local marketplace
claude plugin marketplace add /path/to/your/skills-repo

# Install a plugin from it
claude plugin install <plugin-name>

# Refresh after making changes
claude plugin marketplace refresh <marketplace-name>
claude plugin update <plugin-name>
```

---

## Checklist

- [ ] `plugins/<plugin>/commands/<name>.md` created
- [ ] Frontmatter has `description` and optionally `argument-hint`
- [ ] Body leads with mental model / key gotcha
- [ ] `$ARGUMENTS` used to optionally narrow scope
- [ ] Plugin entry added to `.claude-plugin/marketplace.json`
- [ ] `source` points to plugin root, not command file
- [ ] `marketplace.json` has top-level `owner` as an object

---

## Quick Reference

| Field           | Value           | Notes                                      |
|-----------------|-----------------|--------------------------------------------|
| `description`   | string          | shown in `/` command list                  |
| `argument-hint` | `[topic]`       | shown as hint when typing the command      |
| `source`        | `"./plugins/x"` | string = local path to plugin root         |
| `source.source` | `"git-subdir"`  | key is `"source"`, not `"type"`            |
| `owner`         | `{ name, url }` | object, not string                         |
