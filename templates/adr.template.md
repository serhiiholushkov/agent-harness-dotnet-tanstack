# ADR {{NUMBER}}: {{TITLE}}

<!-- One decision per record. File name: NNNN-short-slug.md. Never edit an accepted ADR; supersede it. -->

- **Status:** {{Proposed | Accepted | Superseded by ADR-NNNN}}
- **Date:** {{YYYY-MM-DD}}
- **Deciders:** {{NAMES}}
- **Trigger:** {{Which ADR trigger from the architecture this is — e.g. new module, new dependency, cross-module write strategy, boundary exception, new major version, shared-code promotion, gate change}}

## Context

{{What forces this decision now. The business or technical situation, the constraint that makes the default insufficient, and what happens if nothing is decided. 2–3 paragraphs maximum.}}

## Options considered

<!-- Include the do-nothing option. For cross-module writes, the architecture's consistency table is the option list. -->

1. **{{OPTION 1}}** — {{one-line description}}. Pros: {{...}}. Cons: {{...}}.
2. **{{OPTION 2}}** — {{one-line description}}. Pros: {{...}}. Cons: {{...}}.
3. **{{OPTION 3 / do nothing}}** — {{...}}

## Decision

{{The chosen option, stated as a rule the codebase can be held to. Name the modules/packages affected and the boundary the decision touches.}}

## Consequences

- {{What becomes easier}}
- {{What becomes harder or is now forbidden}}
- {{What must be built/changed to comply — link issues}}
- {{What test or check enforces this decision (architecture test, drift check, review scope) — if nothing enforces it, say so explicitly}}

## Links

- {{Related ADRs, the feature spec, the architecture section this interprets}}
