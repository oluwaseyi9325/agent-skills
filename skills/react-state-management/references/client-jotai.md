# Shared client state with Jotai (v2)

Atoms hold small pieces of client state. Components subscribe to exactly the atoms they
read, so re-renders stay narrow by default.

## Atoms

```ts
import { atom } from 'jotai';
import { atomWithReset } from 'jotai/utils';

export const workspaceIdAtom = atomWithReset<string | null>(null);

// Derived: computed, never stored
export const hasWorkspaceAtom = atom((get) => get(workspaceIdAtom) !== null);
```

```ts
const workspaceId = useAtomValue(workspaceIdAtom);   // read only
const setWorkspaceId = useSetAtom(workspaceIdAtom);  // write only: no re-render on change
```

- Use `useAtomValue` / `useSetAtom` instead of `useAtom` when a component only reads or only
  writes.
- Derived values are derived atoms, not atoms kept in sync with effects.
- No server data in atoms. Use the project's server-state cache. If you want atoms over it,
  use `jotai-tanstack-query` rather than copying responses into atoms.

## Persistence

```ts
import { atomWithStorage } from 'jotai/utils';

export const themeAtom = atomWithStorage<'light' | 'dark'>('theme', 'light'); // device preference
export const recentSearchesAtom = atomWithStorage<string[]>('recent-searches:v1', []); // per user
```

- `atomWithStorage` uses `localStorage` by default. On React Native, pass a storage made
  with `createJSONStorage(() => appStorage)` from `jotai/utils`, where `appStorage` is a
  synchronous MMKV wrapper (see `client-zustand.md`). With async storage such as
  `AsyncStorage` the value can be a Promise, so read it under `<Suspense>`.
- By default the first render shows the initial value and storage is read on mount, which
  keeps SSR safe. Without SSR and with synchronous storage, pass `{ getOnInit: true }` as
  the fourth argument to read storage straight away and avoid a flash.
- Put a version in the key (`'recent-searches:v2'`) when the stored shape changes, and
  remove the old key.
- No tokens in `localStorage` or `AsyncStorage` (see `persistence-and-sign-out.md`).

## Reset and sign-out

```ts
import { RESET } from 'jotai/utils';

// Per-user atoms only: the theme is a device preference and stays.
export const resetUserAtomsAtom = atom(null, (_get, set) => {
  set(workspaceIdAtom, RESET);
  set(recentSearchesAtom, RESET); // atomWithStorage: also removes the key from storage
});
```

- In a component: `const resetUserAtoms = useSetAtom(resetUserAtomsAtom)`.
- Outside React (sign-out from the API client), set it on the store the app renders with:
  `getDefaultStore().set(resetUserAtomsAtom)` when there's no `Provider`, or
  `appStore.set(resetUserAtomsAtom)` when the app renders `<Provider store={appStore}>`
  with `const appStore = createStore()` (both from `jotai`).
- A fresh store resets every atom in memory: `<Provider key={userId}>` remounts with a new
  store when the user changes. `atomWithStorage` atoms read storage again when they mount,
  so still `RESET` them (or remove their keys).
