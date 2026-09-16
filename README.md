# Agent Harness: TanStack Start + .NET Modular Monolith (Vertical Slices)

Rules, skills, agents, commands, and hooks that equip any coding agent (Copilot, Claude Code, Codex, Cursor) to build a TanStack Start + .NET modular-monolith monorepo.

## Catalog

| Component                                                          | What it is                                                                                 | Loaded                      | Status |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------ | --------------------------- | ------ |
| [AGENTS.md](AGENTS.md)                                             | The slim always-on contract: precedence, stack, commands, boundary digest, completion gate | Always                      | Ready  |
| [CLAUDE.md](CLAUDE.md)                                             | Claude Code shim — imports `AGENTS.md`                                                     | Always (Claude)             | Ready  |
| [docs/architecture.md](docs/architecture.md)                       | The complete normative architecture every artifact cites                                   | On demand                   | Ready  |
| [rules/](rules/README.md)                                          | Durable standards in seven opt-in packs, installed per agent role                          | Always (per installed pack) | Ready  |
| [skills/](skills/README.md)                                        | Procedures for repeatable stack tasks (slice, module, migration, codegen, web feature, …)  | When the task matches       | Ready  |
| [agents/](agents/README.md)                                        | Planner + read-only reviewers (boundaries, contracts, security)                            | When delegated              | Ready  |
| [commands/](commands/README.md)                                    | Thin slash entry points routing to skills/agents                                           | When invoked                | Ready  |
| [hooks/](hooks/README.md)                                          | Deterministic guards (generated-file edits, contract drift, format, verify)                | On harness/git events       | Ready  |
| [docs/usage.md](docs/usage.md), [docs/install.md](docs/install.md) | How to drive the framework; per-harness installation mapping                               | Human reference             | Ready  |
| [templates/](templates/README.md)                                  | ADR, glossary, and feature-spec seeds for a consuming repository                           | Copied out                  | Ready  |

## Quickstart

1. Copy this folder into the target repository (or keep it as a subtree, e.g. `harness/`).
2. Wire the always-on contract:
   - **Copilot / Codex / Cursor and other AGENTS.md readers** — place `AGENTS.md` at the repository root (or symlink it).
   - **Claude Code** — place `CLAUDE.md` beside it; it imports `AGENTS.md`.
3. Install the rule packs matching each agent's role — see [rules/README.md](rules/README.md) for the pack combinations and per-harness mechanics.
4. Make skills, agents, and commands discoverable — [docs/install.md](docs/install.md) maps every artifact kind to its exact destination per harness.
5. Optionally wire the deterministic guards — [hooks/README.md](hooks/README.md) has the support matrix and snippets; CI + git hooks work everywhere.
6. Point agents at [docs/architecture.md](docs/architecture.md) as the authority, and read [docs/usage.md](docs/usage.md) for how to drive them.

## What this framework deliberately excludes

Runtime software is out of scope for a template: memory vaults, continuous-learning loops, eval pipelines, security scanners, installers, and plugin manifests. ECC-style add-ons of that kind can coexist with this catalog — they consume the same `AGENTS.md` contract and rules — but they are products, not template content.

## Layout

```text
harness/
├── README.md            # this file
├── AGENTS.md            # always-on contract
├── CLAUDE.md            # @AGENTS.md import
├── docs/
│   ├── architecture.md  # the authority
│   ├── usage.md         # driving the framework
│   └── install.md       # per-harness mapping
├── rules/               # seven packs + README
├── skills/              # ten procedures + README
├── agents/              # planner + three reviewers + README
├── commands/            # eight entry points + README
├── hooks/               # four guard scripts + README
└── templates/           # ADR, glossary, feature-spec seeds
```
