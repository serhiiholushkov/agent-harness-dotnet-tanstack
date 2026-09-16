# Skills

Skills are on-demand procedures for repeatable tasks in this stack. A skill owns the _how_ — decision points, steps, a worked example, and a done-when checklist. It never restates standards: those live in [../rules/](../rules/README.md) and [../docs/architecture.md](../docs/architecture.md), which skills cite.

## Format and discovery

Each skill is a folder with a `SKILL.md`:

```yaml
---
name: add-vertical-slice
description: 'Purpose, "use when" guidance, and trigger terms — written for the harness to match against the task.'
---
```

The body stays under ~150 lines: decision points, the procedure, a stack-correct example, and "done when". Depth beyond that lives in `references/*.md` inside the same folder (progressive disclosure — read them only when the body points there).

Harnesses discover skills by their frontmatter `description`. When a task matches a skill, follow the skill instead of improvising; when two skills apply (an API slice plus its web feature), run them in pipeline order — API first, regenerate, then web.

## Index

| Skill                                                              | Use when                                                           |
| ------------------------------------------------------------------ | ------------------------------------------------------------------ |
| [plan-feature/](plan-feature/SKILL.md)                             | Turning a feature request into a reviewable plan before any code   |
| [add-vertical-slice/](add-vertical-slice/SKILL.md)                 | Adding one use case (endpoint) to an existing module               |
| [add-module/](add-module/SKILL.md)                                 | Creating a new business module (ADR-gated)                         |
| [add-web-feature/](add-web-feature/SKILL.md)                       | Adding the web side: feature folder, adapters, queries, UI, route  |
| [version-endpoint/](version-endpoint/SKILL.md)                     | Making a breaking change to a released endpoint                    |
| [add-ef-migration/](add-ef-migration/SKILL.md)                     | Changing a module's schema                                         |
| [regenerate-api-client/](regenerate-api-client/SKILL.md)           | Refreshing OpenAPI docs + TypeScript client; fixing drift failures |
| [cross-module-communication/](cross-module-communication/SKILL.md) | One module needs another module's data or must react to it         |
| [scaffold-workspace/](scaffold-workspace/SKILL.md)                 | Bootstrapping the whole repository from an empty folder            |
| [fix-verify-failures/](fix-verify-failures/SKILL.md)               | `pnpm verify` or CI is red and needs triage                        |

## Installation

The catalog here is canonical; installed copies are regenerated, never edited.

- **GitHub Copilot** — copy each folder to `.github/skills/<name>/`.
- **Claude Code** — copy to `.claude/skills/<name>/`.
- **Codex** — copy to `.agents/skills/<name>/` (repo-shared; user-level `~/.agents/skills/`).
- **Cursor** — copy to `.cursor/skills/<name>/`, or rely on its compatibility loading of `.claude/skills/` and `.agents/skills/`.

Exact destinations and translation steps live in [../docs/install.md](../docs/install.md); verify against each harness's current documentation when installing.
