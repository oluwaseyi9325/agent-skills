# Shared client state with Zustand (v5)

Use a store only for state that is **not** server data and is needed by components far
apart in the tree: the session, the selected workspace, an onboarding step, a few user
preferences. Everything else has a better owner (see the table in `SKILL.md`).

## Contents

- [Store shape](#store-shape)
- [Selecting state](#selecting-state)
- [Persistence](#persistence)
- [Wait for hydration](#wait-for-hydration)
- [Next.js and SSR](#nextjs-and-ssr)
- [Resetting](#resetting)
- [Switching workspace](#switching-workspace)

## Store shape

Small stores, one concern each. State and the actions that change it live together, and
an `initialState` makes reset trivial.

```ts
import { create } from 'zustand';

type WorkspaceState = { workspaceId: string | null };
type WorkspaceActions = {
  selectWorkspace: (id: string | null) => void;
  reset: () => void;
};

const initialState: WorkspaceState = { workspaceId: null };

export const useWorkspaceStore = create<WorkspaceState & WorkspaceActions>()((set) => ({
  ...initialState,
  selectWorkspace: (workspaceId) => set({ workspaceId }),
  reset: () => set(initialState),
}));
```

- No "god store" holding unrelated things.
- No server data in a store. If you're about to `set({ orders })`, use the server-state
  cache instead.
- Business rules belong in pure functions or on the server, not inside store actions.
- The `devtools` middleware is optional. If you add it, make it the outermost one:
  `create<T>()(devtools(persist(...), { name: 'workspace' }))`.

## Selecting state

```ts
import { useShallow } from 'zustand/react/shallow';

// Good: re-renders only when this value changes
const workspaceId = useWorkspaceStore((s) => s.workspaceId);
const selectWorkspace = useWorkspaceStore((s) => s.selectWorkspace); // actions never change

// Good: several fields at once, compared shallowly
const { workspaceId, selectWorkspace } = useWorkspaceStore(
  useShallow((s) => ({ workspaceId: s.workspaceId, selectWorkspace: s.selectWorkspace })),
);

// Bad: re-renders on every change to anything in the store
const store = useWorkspaceStore();

// Bad in v5: a selector that returns a new object or array without useShallow
// loops forever ("Maximum update depth exceeded")
const { workspaceId } = useWorkspaceStore((s) => ({ workspaceId: s.workspaceId }));
```

Outside React (API client interceptors, sign-out), use `useAuthStore.getState()`,
`setState()` and `subscribe()`.

## Persistence

Persist only what must survive a restart. Partialize, and version it so a later shape
change migrates instead of crashing. Non-secret state (selected workspace, preferences)
goes in fast, synchronous storage:

```ts
import { createJSONStorage, persist } from 'zustand/middleware';

export const useWorkspaceStore = create<WorkspaceState & WorkspaceActions>()(
  persist(
    (set) => ({
      ...initialState,
      selectWorkspace: (workspaceId) => set({ workspaceId }),
      reset: () => set(initialState),
    }),
    {
      name: 'workspace',
      storage: createJSONStorage(() => appStorage),
      partialize: (s) => ({ workspaceId: s.workspaceId }), // state only, no actions
      version: 2,
      // Runs only when the stored version differs. Upgrade shapes you know; drop the rest.
      migrate: (persisted, version) => {
        if (version === 1) {
          const v1 = persisted as { selectedId?: string | null }; // v1 called it selectedId
          return { workspaceId: v1.selectedId ?? null };
        }
        return { workspaceId: null };
      },
    },
  ),
);
```

Without `migrate`, stored data with another version is dropped (with a console error) and
the store starts from its initial state.

`appStorage` is `localStorage` on web (also the default when you leave `storage` out). On
React Native, wrap MMKV (v4):

```ts
import { createMMKV } from 'react-native-mmkv';
import type { StateStorage } from 'zustand/middleware';

const mmkv = createMMKV({ id: 'app-storage' });

export const appStorage: StateStorage = {
  getItem: (key) => mmkv.getString(key) ?? null,
  setItem: (key, value) => mmkv.set(key, value),
  removeItem: (key) => mmkv.remove(key),
};
```

Tokens don't go there. On React Native, persist the session to the Keychain / Keystore
through `expo-secure-store`:

```ts
import * as SecureStore from 'expo-secure-store';

const secureStorage: StateStorage = {
  getItem: (key) => SecureStore.getItemAsync(key),
  setItem: (key, value) => SecureStore.setItemAsync(key, value),
  removeItem: (key) => SecureStore.deleteItemAsync(key),
};

type Session = { userId: string; accessToken: string; refreshToken: string };
type AuthState = { session: Session | null };
type AuthActions = { setSession: (session: Session | null) => void; reset: () => void };
const initialAuth: AuthState = { session: null };

export const useAuthStore = create<AuthState & AuthActions>()(
  persist(
    (set) => ({
      ...initialAuth,
      setSession: (session) => set({ session }),
      reset: () => set(initialAuth),
    }),
    {
      name: 'auth', // SecureStore keys allow only letters, digits, ".", "-" and "_"
      storage: createJSONStorage(() => secureStorage),
      partialize: (s) => ({ session: s.session }), // keep it small: tokens and ids, no profile
      version: 1,
    },
  ),
);
```

On web, prefer httpOnly cookies set by the server; then the auth store holds no token and
needn't be persisted. See `persistence-and-sign-out.md` for the storage choices.

## Wait for hydration

`localStorage` and MMKV are synchronous: the store has its stored state before the first
render. Async storage (SecureStore, AsyncStorage) loads after the first render, so don't
route or check auth until it has finished:

```tsx
import { useSyncExternalStore } from 'react';

// Subscribes, then reads, so a hydration that finishes in between can't be missed.
export function useAuthHydrated() {
  return useSyncExternalStore(
    useAuthStore.persist.onFinishHydration,
    useAuthStore.persist.hasHydrated,
    () => false, // server render
  );
}

export function RootLayout() {
  const hydrated = useAuthHydrated();
  if (!hydrated) return <LoadingScreen />;
  return <AppRoutes />;
}
```

Don't use `useState(hasHydrated())` with `onFinishHydration` in a `useEffect`: if
hydration finishes between the render and the effect, the flag stays `false` forever.

## Next.js and SSR

A store made with `create` at module level is one instance per server process, shared by
every request.

- Only write to it in the browser (event handlers, effects). Never fill it with a user's
  data during a server render: the next request, from another user, would see it.
- If the store must start with per-user data from the server (say, the workspace from a
  cookie), create one store per request and pass it down through Context:

```tsx
'use client';
import { createContext, useContext, useState, type ReactNode } from 'react';
import { createStore, useStore } from 'zustand';

type WorkspaceStore = WorkspaceState & WorkspaceActions;

const createWorkspaceStore = (workspaceId: string | null) =>
  createStore<WorkspaceStore>()((set) => ({
    workspaceId,
    selectWorkspace: (id) => set({ workspaceId: id }),
    reset: () => set(initialState),
  }));

const WorkspaceStoreContext = createContext<ReturnType<typeof createWorkspaceStore> | null>(null);

export function WorkspaceStoreProvider(props: { workspaceId: string | null; children: ReactNode }) {
  // One store per request on the server, one per page load in the browser.
  const [store] = useState(() => createWorkspaceStore(props.workspaceId));
  return (
    <WorkspaceStoreContext.Provider value={store}>{props.children}</WorkspaceStoreContext.Provider>
  );
}

export function useWorkspaceStore<T>(selector: (s: WorkspaceStore) => T): T {
  const store = useContext(WorkspaceStoreContext);
  if (!store) throw new Error('useWorkspaceStore must be used inside WorkspaceStoreProvider');
  return useStore(store, selector);
}
```

  The trade-off: there's no module-level `getState()`, so code outside React needs the
  store passed in.
- Server Components can't use a store. Pass server data to client components as props.
- Persisted stores read storage only in the browser. Zustand v5 renders the hydration pass
  with the initial state (so no hydration mismatch), then switches to the stored state.
  Gate UI that depends on stored values with a hydration hook like the one above.

## Resetting

Every store exposes `reset()`. Sign-out calls `reset()` and then, on persisted stores,
`persist.clearStorage()`. Keep that order: `reset()` writes the initial state to storage,
then `clearStorage()` removes the key. See `persistence-and-sign-out.md`.

## Switching workspace

Data from one workspace must never show in another. Best: the workspace id is part of
every server-cache key, so switching just changes the keys and nothing needs clearing.

If it isn't, clear the server cache where the switch happens, not inside the store action
(the store shouldn't depend on the cache library, and on Next.js there's no module-level
client to import). Then remount the workspace's screens so every query starts again with
the new id:

```tsx
export function useSwitchWorkspace() {
  const queryClient = useQueryClient();
  const selectWorkspace = useWorkspaceStore((s) => s.selectWorkspace);
  return (workspaceId: string) => {
    queryClient.removeQueries(); // drops the old workspace's data and cancels its fetches
    selectWorkspace(workspaceId);
  };
}

// In the signed-in layout
const workspaceId = useWorkspaceStore((s) => s.workspaceId);
return <WorkspaceScreens key={workspaceId ?? 'none'} />;
```

With SWR, clear with `unload({ revalidate: false })` from `useSWRConfig()` (SWR 2.5+);
with RTK Query, `dispatch(api.util.resetApiState())`.

A persisted `workspaceId` can point at a workspace the user can no longer open. Check it
against the user's workspaces from the server cache during render, fall back, and have
screens read the workspace through this hook:

```ts
export function useCurrentWorkspaceId() {
  const storedId = useWorkspaceStore((s) => s.workspaceId);
  const { data: workspaces } = useWorkspaces();
  if (!workspaces) return null;
  return workspaces.some((w) => w.id === storedId) ? storedId : (workspaces[0]?.id ?? null);
}
```
