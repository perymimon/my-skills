# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with
code in this repository.

## What This Repo Is

A Claude Code skill marketplace (`my-claude-skills`) containing reusable skills
distributed via `marketplace.json`. Skills are markdown files with YAML
frontmatter that Claude Code can invoke during sessions.

## Structure

```
marketplace.json          — root manifest listing all plugins and their paths
plugins/<name>/
  skills/<name>/
    SKILL.md              — the skill content (YAML frontmatter + markdown body)
```

Each `SKILL.md` uses this frontmatter schema:

```yaml
---
name: <skill-name>
description: <one-line summary used for skill discovery>
user-invokable: true|false
args:
  - name: <arg>
    description: <what it does>
    required: false
---
```

## Adding a New Skill

1. Create `plugins/<name>/skills/<name>/SKILL.md`
2. Add an entry to `marketplace.json` under `plugins[]` with `name`, `version`,
   `description`, and `path`

## Skill Writing Conventions

- Lead with the core mental model or the key gotcha — the "why" before the "how"
- Use fenced code blocks with language tags
- Mark correct vs. wrong patterns with `✅` / `❌` inline comments
- A Quick Reference table at the end is useful for skills with multiple patterns
- Keep skill content focused: one domain per skill, invoke with an optional
  `topic` arg to narrow scope