# Installation guide — mapping the catalog to your harness

This catalog is harness-agnostic; harnesses read from harness-specific locations. Installing means copying (or generating) artifacts from here into those locations and translating frontmatter where the target uses a different key.

**The single-source rule.** The catalog is canonical. Installed copies are disposable build outputs: regenerate them from the catalog, never edit them in place, and never let them drift into a second source of truth. Treat a hand-edited installed copy the same way this architecture treats a hand-edited generated client — revert and regenerate.

Harness formats change quickly. This table was verified against vendor documentation in 2026-09; re-verify before installing, especially anything marked Preview or deprecated.

## Mapping table

| Artifact | Canonical            | GitHub Copilot / VS Code                                        | Claude Code                     | Codex                                       | Cursor                                                  |
| -------- | -------------------- | --------------------------------------------------------------- | ------------------------------- | ------------------------------------------- | ------------------------------------------------------- |
| Contract | `AGENTS.md`          | read natively (repo root)                                       | via `CLAUDE.md` import          | read natively (root + nested + `~/.codex/`) | read natively (root + nested)                           |
| Rules    | `rules/<pack>/*.md`  | `.github/instructions/<pack>/*.instructions.md` (`applyTo`)     | `.claude/rules/*.md` (`paths`)  | linked from `AGENTS.md`                     | `.cursor/rules/*.mdc` (`globs`, `alwaysApply`)          |
| Skills   | `skills/<name>/`     | `.github/skills/<name>/`                                        | `.claude/skills/<name>/`        | `.agents/skills/<name>/`                    | `.cursor/skills/<name>/` (also reads `.claude/skills/`) |
| Agents   | `agents/<name>.md`   | `.github/agents/<name>.agent.md`                                | `.claude/agents/<name>.md`      | role prompts / subagent config — see below  | custom modes — see below                                |
| Commands | `commands/<name>.md` | `.github/prompts/<name>.prompt.md` (deprecated → prefer skills) | `.claude/commands/<name>.md`    | skills (custom prompts are deprecated)      | skills (slash commands merged into skills)              |
| Hooks    | `hooks/scripts/`     | `.github/hooks/*.json` (Preview) / git hooks / CI               | `.claude/settings.json` `hooks` | git hooks / CI (verify native hook docs)    | `.cursor/hooks.json`                                    |

Hook wiring details and the honest support matrix live in [../hooks/README.md](../hooks/README.md); this guide covers the other five artifact kinds.

## GitHub Copilot / VS Code

1. **Contract** — `AGENTS.md` at the repository root is read natively (`chat.useAgentsMdFile`). `CLAUDE.md` is also detected, so the shim causes no conflict — both point at the same content.
2. **Rules** — copy each rule to `.github/instructions/<pack>-<name>.instructions.md` (subfolders are allowed; discovery is recursive). Keep `title`/`description`; translate `appliesTo` to the `applyTo` frontmatter key (glob relative to the workspace root, comma-separated patterns supported). Rules without a matching `applyTo` are not auto-applied.
3. **Skills** — copy each skill folder to `.github/skills/<name>/` unchanged, including `references/`.
4. **Agents** — copy each agent to `.github/agents/<name>.agent.md`. Map the canonical `tools` shorthand to Copilot tool names (reviewers: read/search tools plus terminal for the listed read-only commands; no edit tools).
5. **Commands** — prompt files (`.github/prompts/<name>.prompt.md`, `$ARGUMENTS` → prompt input) still work for the local agent but are **deprecated for Agent Host sessions**. Prefer installing each command as a skill folder (`.github/skills/<name>/SKILL.md`); the body already reads as an instruction to route to the target skill/agent.
6. **Monorepo note** — if the repo is opened from a subfolder, enable `chat.useCustomizationsInParentRepositories`.

## Claude Code

1. **Contract** — place `CLAUDE.md` (containing `@AGENTS.md`) and `AGENTS.md` at the repository root.
2. **Rules** — copy selected packs to `.claude/rules/<pack>-<name>.md`; translate `appliesTo` to the `paths` frontmatter key (array of globs; omitted = always loaded). Alternatively import pack files from `CLAUDE.md` for always-on packs.
3. **Skills** — copy each skill folder to `.claude/skills/<name>/` unchanged. The `name`/`description` frontmatter is already in Claude's expected shape.
4. **Agents** — copy each agent to `.claude/agents/<name>.md`. Frontmatter `name`, `description`, `tools` are native; map `tools` to Claude tool names (e.g. reviewers: `Read, Grep, Glob, Bash` where Bash is limited to the read-only commands the agent lists).
5. **Commands** — copy each command to `.claude/commands/<name>.md`; `$ARGUMENTS` and `argument-hint` are native. (Skills have absorbed commands in Claude Code; either location works, commands keep explicit-invocation semantics.)
6. **Hooks** — wire per [../hooks/README.md](../hooks/README.md) in `.claude/settings.json`.

## Codex

1. **Contract** — `AGENTS.md` is Codex's native format: repository root (merged with nested `AGENTS.md` files walking down to the working directory, ~32 KiB combined budget). Personal defaults go to `~/.codex/AGENTS.md`.
2. **Rules** — no glob-scoped rule mechanism is pinned here; keep the packs in the repo and link them from `AGENTS.md` (the catalog's `AGENTS.md` already points at `rules/`). Mind the instructions byte budget — link, do not inline packs.
3. **Skills** — copy each skill folder to `.agents/skills/<name>/` at the repository root (Codex scans `.agents/skills` from the working directory up to the repo root; user-level: `~/.agents/skills/`). The SKILL.md `name` + `description` frontmatter is the same open agent-skills format. Invoke with `$<name>` or `/skills`.
4. **Agents** — Codex has subagent configuration but no drop-in agent-file directory pinned here; expose each agent as a skill or role prompt referenced from `AGENTS.md`, and check the current subagents documentation when installing.
5. **Commands** — custom prompts (`~/.codex/prompts/*.md`) are **deprecated**; install commands as skills (step 3) so they are repo-shared and invocable.
6. **Hooks** — Codex documents a hooks feature; its schema is not pinned here. Use git hooks + CI as the portable path and verify current docs.

## Cursor

1. **Contract** — `AGENTS.md` at the repository root is read natively; nested `AGENTS.md` files are supported.
2. **Rules** — convert each rule to `.cursor/rules/<pack>-<name>.mdc` (the `.mdc` extension is required; plain `.md` is ignored). Translate: `description` stays; `appliesTo` → `globs` (comma-separated); set `alwaysApply: true` only for the `common` pack digest — glob-scoped rules use `alwaysApply: false` + `globs`.
3. **Skills** — copy each skill folder to `.cursor/skills/<name>/`, or rely on Cursor's compatibility loading of `.claude/skills/` and `.agents/skills/` if another harness is already installed — one installed copy is enough. Skills are invoked with `/<name>` or `@<name>`.
4. **Agents** — no drop-in agent-file directory; recreate each agent as a custom mode with equivalent tool restrictions, or install them as skills whose body instructs the read-only scope. Check current docs.
5. **Commands** — slash commands merged into skills; installing the command files as skills gives `/<name>` invocation directly.
6. **Hooks** — wire per [../hooks/README.md](../hooks/README.md) in `.cursor/hooks.json`.

## Generic AGENTS.md-reading harness

Minimum viable install: copy the whole catalog into the repo (e.g. as `harness/`), symlink or copy `AGENTS.md` to the root, and add git hooks from [../hooks/scripts/](../hooks/scripts/). The contract links to rules, skills, and the architecture by relative path, so any harness that reads `AGENTS.md` and can follow links gets the full framework without translation.

## Multi-harness installs

Installing into several harnesses simultaneously is supported — that is why the catalog is the single source. Keep one generation script or checklist per harness, re-run it after catalog changes, and never fix an installed copy directly. Where two harnesses can read the same location (Cursor reading `.claude/skills/`, every harness reading `AGENTS.md`), prefer one shared copy over duplicates.
