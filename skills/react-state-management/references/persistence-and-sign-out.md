# Persistence and sign-out

These rules hold whatever store or cache library the project uses. Library-specific calls
are listed at the end of each section.

## Contents

- [What to persist](#what-to-persist)
- [Picking the storage](#picking-the-storage)
- [Version everything you persist](#version-everything-you-persist)
- [Wait for hydration](#wait-for-hydration)
- [Sign-out](#sign-out)
- [Switching workspace or account](#switching-workspace-or-account)

## What to persist

Persist only what must survive a restart: the session, a few preferences, the selected
workspace, and (optionally) a cached copy of server data for a fast cold start. Don't
persist transient UI state, form drafts you don't intend to restore, or anything derived.

## Picking the storage

| Platform | Use | Notes |
|---|---|---|
| Web | `localStorage` (or `sessionStorage` for per-tab state) | Any script on the page can read it, so one XSS bug exposes all of it. For tokens, prefer httpOnly, `Secure`, `SameSite` cookies set by the server when the backend supports them: scripts can't read them. Protect state-changing requests against CSRF. |
| React Native: settings, query cache | `react-native-mmkv` (v4: `createMMKV`) | Fast and synchronous. Plain files unless you pass an `encryptionKey`, and that key must itself live in secure storage. |
| React Native: tokens | `expo-secure-store` or `react-native-keychain` | iOS Keychain / Android Keystore. Store small values (tokens and ids, not objects): some iOS versions rejected values over about 2 KB. SecureStore keys allow only letters, digits, `.`, `-` and `_`. |
| React Native: never for tokens | `AsyncStorage` | Unencrypted. |

- iOS keeps Keychain items after the app is uninstalled. On the first launch after an
  install (keep a flag in MMKV or AsyncStorage, which uninstalling does wipe), delete any
  stored tokens, or a reinstall signs the previous user straight back in.
- Never put tokens or personal data in logs, URLs or analytics events.

## Version everything you persist

A new app version can change the shape of stored data. Old data in a new shape crashes the
app or shows junk.

- Give persisted stores a **version** and a **migration** (or drop the data when the
  version is old).
- Give a persisted server cache a **buster** that changes with the data shape: the app
  version plus the OTA update id on mobile, a build id on web. Add a **max age** so very old
  data is dropped.

| Library | How |
|---|---|
| Zustand `persist` | `version` and `migrate`. Without `migrate`, data with another version is dropped. |
| redux-persist | `version` and `migrate: createMigrate(migrations)`. Without `migrate`, `version` is ignored and old data loads as-is. |
| TanStack Query persister | `buster` and `maxAge` in the persist options |
| Jotai `atomWithStorage` | A version in the key (`'recent-searches:v2'`); remove the old key |

## Wait for hydration

State persisted in **async** storage (SecureStore, AsyncStorage, IndexedDB) arrives after
the first render. If routing or auth checks run before that, a signed-in user is bounced to
sign-in on a cold start. Render a splash until the store reports it has hydrated, then
route.

| Library | Gate |
|---|---|
| Zustand | Only needed with async storage: with `localStorage` or MMKV the store is hydrated before the first render. Use `hasHydrated()` and `onFinishHydration()` through `useSyncExternalStore` (see `client-zustand.md`). |
| redux-persist | Always (its storage API is async): `<PersistGate loading={<LoadingScreen />} persistor={persistor}>` |
| TanStack Query persister | `PersistQueryClientProvider` holds queries until the cache is restored; `useIsRestoring()` says when |
| Jotai `atomWithStorage` | Reads storage on mount by default, so the first render shows the initial value. Without SSR and with sync storage, `{ getOnInit: true }` reads it at once. |

With SSR (Next.js), the server can't read browser storage, so the server render uses the
initial state. Don't route or check auth on persisted client state during that render; if
the server must know who is signed in, use a cookie. Reading storage during the first
browser render (`useState(() => localStorage.getItem(...))`, Jotai's `getOnInit`) causes a
hydration mismatch. Zustand v5 avoids it: it renders the hydration pass with the initial
state, then switches to the stored state.

## Sign-out

One function owns sign-out and removes **everything** a user owns, so the next person on
the device never sees it. Order matters:

1. **Run once at a time.** Several requests can fail with 401 together, and each calls
   sign-out.
2. **Revoke on the server without waiting.** Send the revoke request if the API has one,
   but never block local cleanup on it.
3. **Drop the session first**, and its stored tokens, so the API client stops sending a
   token and auth-gated queries stop.
4. **Leave the signed-in screens**, so nothing mounted refetches or shows old data and a
   back gesture can't return to them. Expo Router: wrap them in
   `<Stack.Protected guard={!!session}>`, which redirects and removes their history when the
   session becomes null. Otherwise call `router.dismissAll()` (when `router.canDismiss()`),
   then `router.replace('/sign-in')`. On web, navigate to sign-in, and key per-user
   providers by user id.
5. **Clear the server-state cache and its persisted copy.**
6. **Reset every other client store and its storage**, and clear other per-user data
   (recent searches, drafts).

Call it from the sign-out button **and** from the API client when a token refresh fails.

### Library calls

| Library | Clear it |
|---|---|
| TanStack Query | `queryClient.clear()` (it also cancels in-flight queries), then `await persister.removeClient()` if you persist |
| SWR | SWR 2.5+: `unload({ revalidate: false })` from `useSWRConfig()`, or `import { unload } from 'swr'` for the default cache. Any version: give `<SWRConfig value={{ provider: () => new Map() }}>` a `key` of the user id, so each user gets a new, empty cache. `mutate(() => true, undefined, { revalidate: false })` is not enough: key filters skip `useSWRInfinite` and `useSWRSubscription` entries. Clear any custom provider's storage too. |
| RTK Query | `dispatch(api.util.resetApiState())`, after the `signedOut` action |
| Zustand | `useStore.getState().reset(); useStore.persist.clearStorage();` (in that order) |
| Redux Toolkit | dispatch one `signedOut` action that every slice resets on; `await persistor.purge()` with redux-persist |
| Jotai | `set(atom, RESET)` for `atomWithReset` and `atomWithStorage` atoms (the latter also removes the stored key); a fresh `Provider` store resets memory only |

### Example (TanStack Query + Zustand, Expo Router)

```ts
let running: Promise<void> | null = null;

// The sign-out button and the API client (after a failed refresh) both call this.
export function signOut(queryClient: QueryClient) {
  running ??= clearUserData(queryClient).finally(() => {
    running = null;
  });
  return running;
}

async function clearUserData(queryClient: QueryClient) {
  const session = useAuthStore.getState().session;
  if (session) void revokeSession(session.refreshToken).catch(() => {}); // not awaited

  // Drop the session and its stored tokens: no token is sent from here on.
  useAuthStore.getState().reset();
  useAuthStore.persist.clearStorage();

  // Leave the signed-in screens. Skip these two lines with <Stack.Protected>.
  if (router.canDismiss()) router.dismissAll();
  router.replace('/sign-in');

  queryClient.clear(); // cancels in-flight queries and empties the cache
  useWorkspaceStore.getState().reset();
  useWorkspaceStore.persist.clearStorage();
  clearRecentSearches();

  await queryPersister.removeClient(); // last, after all the synchronous work
}
```

## Switching workspace or account

Data that depends on the selected workspace, tenant or account must not leak across a
switch. Either put the scope in every cache key, or clear the server cache when it
changes and remount the scoped screens (see `client-zustand.md`). A persisted workspace id
may point at a workspace the user has lost access to: check it against the user's current
workspaces and fall back.
