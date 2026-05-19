# Engineering Rules

These rules apply to every file, every PR, every task — no exceptions.

---

## 1. No Hardcoded Strings

Never write user-facing text directly in code. Every label, button, error message,
placeholder, toast, or heading must use the translation function.

```ts
// ✗ Wrong
<Text>Save changes</Text>
throw new Error('User not found')

// ✓ Correct
<Text>{t('common.save_changes')}</Text>
throw new ApiError(t('errors.user_not_found'))
```

Add new keys to `packages/i18n/locales/en.json` as part of the same PR.
Key naming: `snake_case`, grouped by feature (`auth.`, `farm.`, `farmer.`, `admin.`, `common.`, `errors.`, `validation.`).

---

## 2. TypeScript Strict Mode

- `strict: true` in all `tsconfig.json` files
- No `any` — use `unknown` and narrow it, or define a proper type
- All shared types live in `packages/types/` — never duplicate them
- API responses must be typed end-to-end (request body and response shape)

---

## 3. Error Handling

Every API call must handle three states: loading, success, error.
Every Edge Function must return structured errors:

```ts
// Edge Function error shape
{ error: { code: 'farm_not_found', message: string } }

// HTTP status must match:
// 400 bad input, 401 unauthenticated, 403 forbidden, 404 not found, 500 server error
```

On the client, never silently swallow errors. Surface them to the user via i18n error keys.

---

## 4. Authorization Must Be Server-Side

The client must never hide or show features based solely on a local role check.
Role checks in the UI are purely cosmetic (hiding buttons the user cannot use).
The Edge Function always re-validates the JWT and role — no trust from the client.

---

## 5. Comments

Write no comments by default. Add one only when the **why** is non-obvious:
a hidden constraint, a workaround for a library bug, a subtle invariant.
Never comment what the code does — only why it does it that way.

---

## 6. Tests

- Every new function, hook, or non-trivial component gets a unit test
- Every important user flow (login, save farm, approve application) gets an integration test
- Use **Vitest** + **React Testing Library**
- Tests must be deterministic — mock network, time, and location
- Test files live next to the source file: `FarmCard.tsx` → `FarmCard.test.tsx`

---

## 7. Accessibility

- All interactive elements have an `accessibilityLabel` (mobile) or `aria-label` (web)
- Minimum tap target: 48×48px on mobile, 40px on web (matches design system `--tap-min`)
- Color is never the only indicator of state — always pair with text or icon
- Focus states must be visible (design system `--shadow-focus` / `:focus-visible`)

---

## 8. Component Rules

- Functional components only — no class components
- No prop drilling past two levels — lift state or use Zustand / context
- Components receive only what they need — no passing the entire store
- Reusable atoms live in `components/ui/` and must have no business logic
- Loading and empty states are required for every data-dependent view

---

## 9. Performance

- Memoize with `useMemo` / `useCallback` only when a profiler shows it helps
- Farm list queries are paginated — never fetch all farms in one request
- Images are lazy-loaded
- TanStack Query handles caching — do not duplicate caching logic

---

## 10. Design System Fidelity

The reference is `design-system/` — that folder is READ-ONLY.

All colors, spacing, radii, shadows, and typography must come from:
- `constants/tokens.ts` (mobile)
- `colors_and_type.css` (web)

Never hardcode a hex value or pixel size outside the token file.
If a new token is needed, add it to the token file and the CSS file together.

---

## 11. No Premature Abstraction

Do not build for hypothetical requirements.
If something is used once, write it inline.
If it is used twice, consider extracting.
If it is used three or more times, extract it.

Do not add feature flags, backwards-compat shims, or "reserved for later" code.

---

## Summary checklist before any PR

- [ ] Zero hardcoded strings — all keys added to `en.json`
- [ ] Types are strict — no `any`
- [ ] Loading, error, and empty states handled
- [ ] Tests written and passing
- [ ] Auth/authorization checked server-side
- [ ] Design tokens used — no raw hex/px values
- [ ] Accessibility labels present on interactive elements
