---
description: "Use when writing, editing, or debugging Vitest unit tests for the chrome-extension workspace. Covers test layout under `__tests__`, naming, Chrome API mocking, and targeted run commands."
applyTo: "chrome-extension/src/**/__tests__/**/*.ts, chrome-extension/src/**/*.test.ts"
---

# Vitest Conventions (chrome-extension)

Unit tests live alongside the code they cover, inside a `__tests__` folder, and run via the workspace-scoped script.

## Layout

```
chrome-extension/src/background/services/guardrails/
├── sanitizer.ts
└── __tests__/
    └── guardrails.test.ts
```

- One `*.test.ts` file per module under test; name it after the module (`sanitizer.test.ts`) or after the integrated surface (`guardrails.test.ts`) when several modules collaborate.
- Use `describe` blocks named after the subject (`describe('Sanitizer', () => …)`). The `-t` flag matches on these names.
- Group `it(...)` cases by behavior, one assertion target per case. Prefer descriptive names ("removes zero-width characters from input"), not "test 1".

## Running

```bash
# Whole suite
pnpm -F chrome-extension test

# Targeted by describe / it name
pnpm -F chrome-extension test -- -t "Sanitizer"
```

Do not invoke `vitest` from the repo root or with a different package manager.

## Style

- Import the symbol under test directly; do not re-export private helpers just for tests.
- Use plain functions and `vi.fn()` spies. Reset mocks in `beforeEach` when state leaks across cases.
- Mock Chrome APIs explicitly (`globalThis.chrome = { storage: { local: { … } } } as any`) at the top of the file or in a `vi.stubGlobal` call — don't depend on the browser environment.
- Tests must be deterministic: no network, no timers without `vi.useFakeTimers()`, no reliance on real `Date.now()`.
- Prefer table-driven cases with `it.each([...])` when you're checking many input/output pairs (e.g. sanitizer threat patterns).

## What to cover

- All `act_*` action handlers: at minimum one success path and one validation/failure path.
- Sanitizer / guardrails: every new pattern needs a positive (matches) and negative (untouched) case.
- Storage helpers with non-trivial serialization or merge logic.
- Pure utilities in `@src/background/**/utils.ts`.

## What not to test here

- React components — there is no React test runner configured for the pages workspaces. Cover UI logic by extracting it into pure functions and testing those.
- End-to-end browser behavior — that belongs to `pnpm e2e`, not Vitest.
