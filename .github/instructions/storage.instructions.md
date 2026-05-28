---
description: "Use when adding a new persisted setting, chat record, or any data that survives reloads — i.e. creating or consuming a `@extension/storage` module. Covers the `createStorage` factory, StorageEnum choice, live-update subscriptions, and the `useStorage` React hook."
applyTo: "packages/storage/lib/**/*.ts"
---

# Storage Conventions

All extension state that needs to survive a service-worker restart goes through [`@extension/storage`](packages/storage/index.ts). Do not call `chrome.storage.*` directly from feature code.

## Defining a new store

Create a file under [packages/storage/lib/](packages/storage/lib/) (e.g. `settings/myFeature.ts`) and export both the typed store and any helper functions:

```ts
import { createStorage } from '../base/base';
import { StorageEnum } from '../base/enums';
import type { BaseStorage } from '../base/types';

export interface MyFeatureConfig {
  enabled: boolean;
  threshold: number;
}

const DEFAULT_CONFIG: MyFeatureConfig = { enabled: false, threshold: 10 };

const storage = createStorage<MyFeatureConfig>('my-feature-settings', DEFAULT_CONFIG, {
  storageEnum: StorageEnum.Local, // .Session only for tab-scoped, ephemeral data
  liveUpdate: true,               // required for cross-page reactivity
});

export const myFeatureStore: BaseStorage<MyFeatureConfig> & {
  setThreshold: (value: number) => Promise<void>;
} = {
  ...storage,
  setThreshold: async value => {
    const current = await storage.get();
    await storage.set({ ...current, threshold: value });
  },
};
```

Then re-export from [packages/storage/index.ts](packages/storage/index.ts) so consumers import via `@extension/storage`.

## Rules

- Always provide a complete `fallback`; the type system treats `get()` as non-nullable.
- Choose `StorageEnum.Local` for persistent user data. Reserve `StorageEnum.Session` for short-lived per-session state, and only set `sessionAccessForContentScripts: true` if a content script truly needs it.
- Set `liveUpdate: true` whenever the data is consumed by React UIs (side-panel, options) — otherwise the `useStorage` hook will not re-render on cross-page changes.
- Provide a `serialization` config only when the value contains non-JSON types (Maps, Dates, class instances). Keep it pure and synchronous.
- Don't mutate the result of `get()`; always spread to a new object before calling `set()`.

## Consuming storage

**Background / agent code** — direct async calls:
```ts
import { generalSettingsStore } from '@extension/storage';
const settings = await generalSettingsStore.get();
```

**React pages (side-panel, options)** — use the hook from `@extension/shared`:
```ts
import { useStorage } from '@extension/shared';
import { generalSettingsStore } from '@extension/storage';

const settings = useStorage(generalSettingsStore); // re-renders on changes
```

- Don't introduce a custom React `useEffect` + `subscribe` pattern; use `useStorage`.
- Don't add network or business logic inside a storage module — keep it data-only. Cross-store orchestration belongs in the consuming background service or page component.

## Generated artifacts

Do not commit changes under `packages/storage/dist/` — those are produced by `build.mjs`. Source lives under `packages/storage/lib/`.
