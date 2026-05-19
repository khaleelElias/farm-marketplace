# Claude Instructions

Read ARCHITECTURE.md and RULES.md before starting any task.

## Mandatory rules (enforced on every task)

- Zero hardcoded strings. Always use `t('key')`. Add new keys to `packages/i18n/locales/en.json`.
- TypeScript strict mode everywhere. No `any`.
- Every new component, hook, or function gets a test (Vitest + RTL).
- Error, loading, and empty states required on every data-dependent view.
- All colors/spacing/radii from design tokens only (`constants/tokens.ts` or CSS vars). No raw hex values.
- Authorization is always re-validated server-side in the Edge Function. Client role checks are cosmetic only.
- No comments explaining what — only why (and only when non-obvious).
- No premature abstractions, no unused code, no backwards-compat shims.

## Project structure

See ARCHITECTURE.md for the full monorepo layout, DB schema, API surface, and env vars.

## Design reference

`design-system/` is read-only. Do not edit it. Reference it for component structure and visual patterns.
