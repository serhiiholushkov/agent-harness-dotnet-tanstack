---
title: Components and styling
description: shadcn primitives in packages/ui, Tailwind 4 conventions, and the accessibility floor.
appliesTo: 'apps/web/src/**/*.tsx, packages/ui/**/*.tsx'
---

# Components and styling

Placement:

- Generic, product-agnostic shadcn primitives and design tokens live in `packages/ui`. Anything that knows a product concept — a `ProductCard`, a checkout summary — stays in its feature's `components/`.
- Cross-feature composition (layout shells, navigation) lives in `apps/web/src/components/`.
- shadcn components are added via its CLI into the configured paths (`components.json`); customize the copied source rather than wrapping it in pass-through layers.

Styling:

- Tailwind CSS 4 utilities are the styling mechanism; design decisions (colors, spacing, radii) come from the shared tokens/theme in `packages/ui`, not hard-coded values scattered in features.
- No CSS-in-JS libraries, no ad hoc global stylesheets beyond `styles/`; component-scoped classes via `class-variance-authority`/`cn` patterns as shadcn sets up.
- Class-name sprawl indicating a reusable variant belongs as a variant on the `packages/ui` primitive.

Components:

- Files kebab-case; component per file; colocated `.test.tsx` ([../testing/web-testing.md](../testing/web-testing.md)).
- Components receive typed data via props/hooks; they do not fetch, read config, or import server modules.

Accessibility floor:

- Semantic elements first (`button`, `nav`, `label`, headings in order); ARIA only where semantics fall short.
- Every interactive element is keyboard-reachable and operable with a visible focus state; every form control has an associated label; images carry meaningful `alt`.
- Color is never the only signal; dialogs/popovers use the shadcn primitives, which manage focus correctly — do not hand-roll them.

Caught by: Testing Library queries by role/label (fail on inaccessible markup), ESLint a11y rules, review.
