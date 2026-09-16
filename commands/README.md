# Commands

Commands are thin user entry points: a slash command routes to a skill or agent, states required inputs, and names the expected report. Procedure never lives here — a command that needs steps points at the skill that owns them. Keep every file ≤ ~30 lines.

## Format

```yaml
---
name: implement
description: 'One line shown in the command picker.'
argument-hint: <what the user supplies>
---
```

The body has three fixed parts: what it activates (skill/agent links), **Inputs** (with `$ARGUMENTS` as the placeholder harnesses substitute), and **Expected report**.

## Routing table

| Command                                  | Routes to                                                                      | Use for                                 |
| ---------------------------------------- | ------------------------------------------------------------------------------ | --------------------------------------- |
| [/plan](plan.md)                         | `planner` agent → `plan-feature` skill                                         | Plan before code                        |
| [/implement](implement.md)               | `add-vertical-slice` → `regenerate-api-client` → `add-web-feature`             | Build a full vertical in pipeline order |
| [/add-module](add-module.md)             | `add-module` skill                                                             | New business module (ADR-gated)         |
| [/version-endpoint](version-endpoint.md) | `version-endpoint` skill                                                       | Breaking change to a released endpoint  |
| [/migrate](migrate.md)                   | `add-ef-migration` skill                                                       | Schema change in one module             |
| [/generate-client](generate-client.md)   | `regenerate-api-client` skill                                                  | Refresh contract + client; drift check  |
| [/audit](audit.md)                       | `boundary-auditor` (+ `contract-reviewer`, `security-reviewer` when triggered) | Read-only review of a change set        |
| [/verify](verify.md)                     | `pnpm verify` → `fix-verify-failures` skill on red                             | Run and triage the PR gate              |

## Installation

The catalog here is canonical; installed copies are regenerated, never edited.

- **GitHub Copilot** — copy each file to `.github/prompts/<name>.prompt.md` (`$ARGUMENTS` maps to the prompt input); prompt files are deprecated for Agent Host sessions — prefer installing commands as skills.
- **Claude Code** — copy to `.claude/commands/<name>.md`; `$ARGUMENTS` is native.
- **Codex** — custom prompts are deprecated; install each command as a skill in `.agents/skills/<name>/`.
- **Cursor** — slash commands merged into skills; install as skills for `/<name>` invocation.

Exact destinations and steps live in [../docs/install.md](../docs/install.md); verify against each harness's current documentation when installing.
