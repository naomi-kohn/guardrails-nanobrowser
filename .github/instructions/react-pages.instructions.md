---
description: "Use when writing or editing React components inside pages/side-panel, pages/options, or pages/content. Covers component style, the `cn()` utility, `useStorage`, the long-lived port to the background service worker, Tailwind/theme conventions, and Vite path aliases."
applyTo: "pages/**/src/**/*.{ts,tsx}"
---

# Page (React) Conventions

The three page workspaces (`pages/side-panel`, `pages/options`, `pages/content`) share the same conventions. Build configuration comes from `withPageConfig` in [packages/vite-config/index.mjs](packages/vite-config/index.mjs); the `@src` alias points at the page-local `src/`.

## Components

- Functional components only, named export with `PascalCase`. File name matches component name (`ChatInput.tsx`).
- No `import React from 'react'` — JSX runtime is `react-jsx` (see [packages/tsconfig/base.json](packages/tsconfig/base.json)).
- Type-only imports use `import type { ... }` — `@typescript-eslint/consistent-type-imports` is enforced.
- Reuse primitives from `@extension/ui` (e.g. `Button`, `cn`) rather than reimplementing buttons or class-merge helpers.

```tsx
import { cn } from '@extension/ui';
import { t } from '@extension/i18n';

export function PrimaryAction({ disabled, label }: { disabled: boolean; label: string }) {
  return (
    <button
      className={cn('rounded px-3 py-1 text-sm', disabled && 'opacity-50')}
      disabled={disabled}
    >
      {t('chat_primaryAction_label', [label])}
    </button>
  );
}
```

## State & data

- Read persisted state via `useStorage(...)` from `@extension/shared` paired with a store from `@extension/storage`. Don't subscribe manually or wrap stores in custom hooks unless the existing API is insufficient.
- Local UI state: `useState` / `useReducer`. Keep state colocated with the component that owns it; lift only when two siblings need it.
- Don't introduce a global state library (Redux, Zustand, Jotai, …) without prior agreement — the existing storage + hooks pattern is the project norm.

## Messaging the background service worker

The side panel uses a **long-lived port** named `'side-panel-connection'`:

```ts
const port = chrome.runtime.connect({ name: 'side-panel-connection' });
port.postMessage({ type: 'new_task', task, tabId, taskId });
port.onMessage.addListener(msg => { /* ... */ });
```

Rules:
- Reuse the existing port name `'side-panel-connection'` ([pages/side-panel/src/SidePanel.tsx](pages/side-panel/src/SidePanel.tsx)). Do not invent a new name unless adding a brand-new channel.
- Keep payloads as discriminated unions on `type`; mirror the type definition in any shared types file (e.g. [pages/side-panel/src/types/event.ts](pages/side-panel/src/types/event.ts)).
- The content script (`pages/content`) does **not** talk to the side panel directly. Route everything through the background worker.
- Use `chrome.runtime.sendMessage` only for one-shot fire-and-forget requests; prefer the port for any stream of events.

## Styling

- Tailwind utility classes via [pages/<page>/tailwind.config.ts](pages/options/tailwind.config.ts) (each page extends `withUI` from `@extension/ui`).
- Use `cn(...)` for conditional class lists.
- Respect dark mode via Tailwind's `dark:` variants — don't read `prefers-color-scheme` manually.
- Don't add new design tokens to a single page; promote them to [packages/tailwind-config/tailwind.config.ts](packages/tailwind-config/tailwind.config.ts).

## i18n

All visible strings go through `t(...)` from `@extension/i18n`. See [i18n.instructions.md](.github/instructions/i18n.instructions.md) for keys and placeholders.

## Don'ts

- Don't import from another page's `src/` — the `@src` alias is page-local on purpose.
- Don't reach into `chrome.storage.*` directly; go through `@extension/storage`.
- Don't add a runtime dependency to a page workspace's `package.json` without checking it's already available via `@extension/shared` or `@extension/ui`.
