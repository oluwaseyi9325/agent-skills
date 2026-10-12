# Shared client state with Redux Toolkit (v2)

Slices hold **client** state only: the session, the selected workspace, preferences, UI
state shared across distant screens. Fetched data belongs in RTK Query (see
`server-rtk-query.md`), never in a slice.

## Contents

- [Slice shape](#slice-shape)
- [Typed hooks and narrow selectors](#typed-hooks-and-narrow-selectors)
- [Persistence](#persistence)
- [Sign-out](#sign-out)

## Slice shape

```ts
import { createAction, createSlice, type PayloadAction } from '@reduxjs/toolkit';

export const signedOut = createAction('auth/signedOut');

type WorkspaceState = { workspaceId: string | null };
const initialState: WorkspaceState = { workspaceId: null };

export const workspaceSlice = createSlice({
  name: 'workspace',
  initialState,
  reducers: {
    workspaceSelected: (state, action: PayloadAction<string | null>) => {
      state.workspaceId = action.payload;
    },
  },
  extraReducers: (builder) => {
    builder.addCase(signedOut, () => initialState);
  },
});
```

- One slice per concern. Every slice resets on the shared `signedOut` action, so sign-out
  is one dispatch.
- Name actions as events (`workspaceSelected`), not setters (`setWorkspace`).
- No fetched lists in slices. If you're about to write `state.orders = action.payload`,
  use an RTK Query endpoint.

## Typed hooks and narrow selectors

```ts
import { useDispatch, useSelector } from 'react-redux';

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
export const useAppSelector = useSelector.withTypes<RootState>();
export const useAppDispatch = useDispatch.withTypes<AppDispatch>();

// Good: re-renders only when this value changes
const workspaceId = useAppSelector((s) => s.workspace.workspaceId);

// Bad: returns a new object every time, so it re-renders on every action
const { workspaceId, theme } = useAppSelector((s) => ({
  workspaceId: s.workspace.workspaceId,
  theme: s.preferences.theme,
}));
// Fix: one useAppSelector per field, or pass shallowEqual (from react-redux) as the
// second argument.
```

A derived value that builds a new array or object needs a memoised selector, or it has
the same problem:

```ts
import { createSelector } from '@reduxjs/toolkit';

export const selectPinnedOrders = createSelector(
  [(s: RootState) => s.preferences.pinnedOrderIds, (_s: RootState, orders: Order[]) => orders],
  (pinnedIds, orders) => orders.filter((o) => pinnedIds.includes(o.id)),
);

// orders come from an RTK Query hook; the pins are client state
const pinned = useAppSelector((s) => selectPinnedOrders(s, orders));
```

A plain lookup (`s.auth.session !== null`) doesn't need `createSelector`.

## Persistence

With redux-persist, persist only the slices that must survive a restart, version them
with a migration, and tell RTK's serializability check to ignore redux-persist's actions:

```ts
import { combineReducers, configureStore } from '@reduxjs/toolkit';
import {
  FLUSH, PAUSE, PERSIST, PURGE, REGISTER, REHYDRATE,
  createMigrate, persistReducer, persistStore, type MigrationManifest,
} from 'redux-persist';
import storage from 'redux-persist/lib/storage'; // localStorage; see below for React Native

const migrations: MigrationManifest = {
  // v2 renamed workspace.selected to workspace.workspaceId
  2: (state) => {
    const old = state as typeof state & { workspace?: { selected?: string | null } };
    return old && { ...old, workspace: { workspaceId: old.workspace?.selected ?? null } };
  },
};

const rootReducer = combineReducers({
  auth: authSlice.reducer,
  workspace: workspaceSlice.reducer,
  preferences: preferencesSlice.reducer,
  [api.reducerPath]: api.reducer,
});

const persistedReducer = persistReducer(
  {
    key: 'root',
    storage,
    version: 2,
    migrate: createMigrate(migrations), // without it, `version` does nothing
    whitelist: ['workspace', 'preferences'], // never the RTK Query cache; tokens: see below
  },
  rootReducer,
);

export const store = configureStore({
  reducer: persistedReducer,
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: { ignoredActions: [FLUSH, REHYDRATE, PAUSE, PERSIST, PURGE, REGISTER] },
    }).concat(api.middleware),
});

export const persistor = persistStore(store);
```

Render nothing that reads persisted state or sends requests until it's restored:

```tsx
import { Provider } from 'react-redux';
import { PersistGate } from 'redux-persist/integration/react';

<Provider store={store}>
  <PersistGate loading={<LoadingScreen />} persistor={persistor}>
    <App />
  </PersistGate>
</Provider>
```

- redux-persist's storage methods must return Promises. On React Native, use
  `AsyncStorage` for non-secret slices, or wrap MMKV:
  `getItem: (key) => Promise.resolve(mmkv.getString(key) ?? null)`, and the same for
  `setItem` and `removeItem`.
- Tokens never go in `localStorage` or `AsyncStorage`. On web, prefer httpOnly cookies and
  keep tokens out of Redux. On React Native, give the auth slice its own `persistReducer`
  backed by SecureStore. redux-persist names its keys `persist:<key>`, and SecureStore
  rejects `:`, so map the key:

```ts
import * as SecureStore from 'expo-secure-store';
import type { Storage } from 'redux-persist';

const safeKey = (key: string) => key.replace(/[^\w.-]/g, '_');

// Wrapped, not passed directly: redux-persist passes extra arguments that SecureStore
// would read as options.
const secureStorage: Storage = {
  getItem: (key: string) => SecureStore.getItemAsync(safeKey(key)),
  setItem: (key: string, value: string) => SecureStore.setItemAsync(safeKey(key), value),
  removeItem: (key: string) => SecureStore.deleteItemAsync(safeKey(key)),
};

// In combineReducers, instead of the plain auth reducer:
auth: persistReducer({ key: 'auth', storage: secureStorage }, authSlice.reducer),
```

## Sign-out

```ts
export const signOut = () => async (dispatch: AppDispatch) => {
  dispatch(signedOut());              // every slice back to its initial state
  dispatch(api.util.resetApiState()); // drop the RTK Query cache
  await persistor.purge();            // remove every persisted copy
};
```

The session goes first, then the caches. For the full order (one run at a time, leaving
the signed-in screens), see `persistence-and-sign-out.md`.
