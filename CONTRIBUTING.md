# Contributing

Thanks for considering a contribution!

## Adding a new skill

1. Create a new directory under `./skills/` with a lowercase, hyphenated name.
2. Add a `SKILL.md` file with YAML frontmatter and Markdown instructions.

Required frontmatter fields:

```yaml
---
name: my-new-skill          # Must match folder name. Lowercase, hyphens, max 64 chars.
description: When and why to use this skill (max 1024 chars)
license: MIT                 # Optional but recommended
---
```

The body of `SKILL.md` should describe:

- When the skill should trigger
- Prerequisites (CLI commands, IDs, prior skills)
- The exact CLI command(s) with parameters
- Expected output shape
- How to present results to the user
- Common errors and how to handle them

## Spec

See the [Agent Skills specification](https://agentskills.io/specification) for the full schema.

Validate your skill before committing:

```bash
npx -y @agentskills/skills-ref validate ./skills/<your-skill>
```

## Versioning the underlying CLI

All skills pin a major version of the `travel-cli` via `npx @tvl-justin/travel-cli@latest ...`. When you publish a breaking CLI change, bump the major version and update each skill's command.
