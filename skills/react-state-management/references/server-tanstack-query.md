# Server state with TanStack Query (v5)

Server data has one owner: the query cache. Screens read it through query hooks, and
mutations keep it correct after writes. Examples are checked against v5.104; notes mark
APIs that changed within v5.

## Contents

- [Layers](#layers)
- [Key factories](#key-factories)
- [Queries](#queries)
- [Mutations own the cache](#mutations-own-the-cache): update strategy, patch helper,
  instant update, optimistic update, background jobs
- [Infinite lists (feeds)](#infinite-lists-feeds)
- [Scoped keys](#scoped-keys)
- [Persisting the query cache](#persisting-the-query-cache)
- [Next.js (App Router)](#nextjs-app-router)
- [Errors](#errors)

## Layers

Follow the project's existing layout. If it has none, this split works:

| Layer | Holds | Must not |
|---|---|---|
| API functions | One function per endpoint. Parses the response (e.g. with Zod) and returns it. | Use hooks, touch the cache, show toasts |
| Query hooks | The key factory, `queryOptions`, `useQuery` / `useInfiniteQuery` hooks | Render JSX |
| Mutation hooks | `useMutation` hooks, including every cache update the write causes | Show toasts (the screen decides) |
| Screens | Call hooks, render states, show toasts | Call `useQueryClient`, `invalidateQueries` or the API directly |

## Key factories

One factory per feature. Keys go from broad to narrow so you can invalidate a whole
branch at once.

```ts
export const orderKeys = {
  all: ['orders'] as const,
  lists: () => [...orderKeys.all, 'list'] as const,
  // One branch per data shape: a paged list holds an OrderPage, a feed holds InfiniteData<OrderPage>.
  pagedLists: () => [...orderKeys.lists(), 'paged'] as const,
  list: (filters: OrderFilters) => [...orderKeys.pagedLists(), filters] as const,
  feeds: () => [...orderKeys.lists(), 'feed'] as const,
  feed: (filters: OrderFilters) => [...orderKeys.feeds(), filters] as const,
  details: () => [...orderKeys.all, 'detail'] as const,
  detail: (id: string) => [...orderKeys.details(), id] as const,
  counts: () => [...orderKeys.all, 'counts'] as const,
};
```

- **Invalidate broad, patch narrow.** `invalidateQueries({ queryKey: orderKeys.lists() })`
  hits every list, paged or feed, whatever its filters. Invalidation doesn't care about
  the data shape, but patching does. `setQueriesData` runs your updater on every query under
  the key. If one prefix holds two shapes, the updater throws on the wrong one, and the
  throw turns a save that succeeded into a failed mutation.
- Filters match partially: `{ queryKey: orderKeys.list({ status: 'open' }) }` hits every
  paged list whose filters include `status: 'open'`.
- Never write `['orders', id]` by hand in a component.

## Queries

```ts
export const orderQuery = (id: string) =>
  queryOptions({ queryKey: orderKeys.detail(id), queryFn: () => getOrder(id) });

export function useOrder(id: string) {
  return useQuery({ ...orderQuery(id), enabled: Boolean(id) });
}

export function useOrderList(filters: OrderFilters) {
  return useQuery({
    queryKey: orderKeys.list(filters),
    queryFn: () => listOrders(filters),
    placeholderData: keepPreviousData, // keep the current page on screen while the next one loads
  });
}
```

- `queryOptions` keeps a key and its fetcher together. `useQuery`, prefetching and server
  rendering share it, and `queryClient.getQueryData(orderQuery(id).queryKey)` is typed.
- `enabled` holds a query back: `Boolean(token)` for signed-in data,
  `Boolean(order?.customerId)` for a query that needs another query's result first.
- A disabled query stays `isPending`, because it has no data. If a query can be disabled,
  show the skeleton on `isLoading` (pending **and** fetching), or it never goes away.
- `staleTime` defaults to 0: data is stale at once and refetches on every mount, window
  focus and reconnect. Set it per kind of data: minutes or hours for reference data
  (countries, plans), seconds for live operational data.
- Render all four states: loading (a skeleton shaped like the content), error (with
  retry), empty (with a next step), and data.
- Prefetch when you can predict the next screen (press-in, hover, route loader), from the
  query layer: `void queryClient.query(orderQuery(id)).catch(() => {})`. `query()` is new
  in 5.102; on older versions use `queryClient.prefetchQuery(orderQuery(id))` (now
  deprecated, still works).

## Mutations own the cache

The screen calls `mutate` and shows feedback. The mutation hook keeps the cache right.

- Put cache updates in the `useMutation` callbacks: they always run. Callbacks passed to
  `mutate(variables, { onSuccess })` are skipped if the component unmounted first, so use
  those only for UI (a toast, closing a dialog, navigating).
- The third argument of `onSuccess`, `onError` and `onSettled` is whatever `onMutate`
  returned. Current types name it `onMutateResult`; older v5 docs call it `context`.

### Pick the update strategy

| The write… | Do this |
|---|---|
| Returns the updated record | **Patch** the detail and every cached list row from the response |
| Returns nothing, or only an id | Patch from the variables you sent, then invalidate the detail in the background for fields the server sets (`updatedAt`) |
| Can move the record in or out of a list (filter, sort order, page) | Patch, then invalidate the lists in the background |
| Changes things you can't compute (counts, totals, other records) | Invalidate those keys in the background |
| Is slow, and very likely to succeed | **Optimistic update** with rollback |

Patching alone is enough only when the change can't affect which lists hold the record
or where it sits in them.

### One helper patches every copy

```ts
export function patchOrder(queryClient: QueryClient, id: string, patch: Partial<Order>) {
  const apply = (o: Order) => (o.id === id ? { ...o, ...patch } : o);
  queryClient.setQueryData<Order>(orderKeys.detail(id), (o) => o && { ...o, ...patch });
  queryClient.setQueriesData<OrderPage>({ queryKey: orderKeys.pagedLists() }, (page) =>
    page && { ...page, items: page.items.map(apply) },
  );
  queryClient.setQueriesData<InfiniteData<OrderPage>>({ queryKey: orderKeys.feeds() }, (data) =>
    data && { ...data, pages: data.pages.map((p) => ({ ...p, items: p.items.map(apply) })) },
  );
}
```

`setQueriesData` only touches queries already in the cache, and an updater that returns
`undefined` leaves a query as it is.

### Instant update, then a background refresh

This is the fix for "did it save?". Show the new value as soon as the API replies, then
refresh only what could be out of date, without blocking the UI.

```ts
export function useUpdateOrderStatus(id: string) {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (status: OrderStatus) => updateOrderStatus(id, status),
    onSuccess: (updated) => {
      // 1. Show it now: write the response into the detail and every cached list row.
      patchOrder(queryClient, id, updated);
      // 2. In the background: the order may move between status-filtered lists, and the
      //    counts change. Only queries on screen refetch now; the rest are marked stale.
      void queryClient.invalidateQueries({ queryKey: orderKeys.lists() });
      void queryClient.invalidateQueries({ queryKey: orderKeys.counts() });
      // 3. Another feature's heavy views: mark them stale, refetch when next viewed.
      void queryClient.invalidateQueries({ queryKey: reportKeys.all, refetchType: 'none' });
    },
  });
}
```

- Don't `await` the refresh before closing the dialog. Close on success and let the
  refresh run behind it. (In one app this took a status dialog from several seconds to
  about 0.2 s.)
- Don't invalidate everything (`invalidateQueries()` with no key). One status change
  refetching a dozen screens is how apps feel slow.
- Narrow further when many lists are on screen. On a board with one column per status,
  invalidate only the old and new columns, e.g. `orderKeys.list({ status: 'open' })`.
- `refetchType: 'none'` marks data stale without a network burst now. Good for reports
  and heavy dashboards.

If the API returns no record, patch from what you sent:

```ts
onSuccess: (_data, status) => {
  patchOrder(queryClient, id, { status }); // from the variables
  void queryClient.invalidateQueries({ queryKey: orderKeys.detail(id) }); // server-set fields
  void queryClient.invalidateQueries({ queryKey: orderKeys.lists() });
},
```

### Optimistic update with rollback

Use when the change is very likely to succeed and waiting would feel slow (toggles,
reordering, moving a card between columns).

```ts
export function useToggleFavorite(id: string) {
  const queryClient = useQueryClient();
  const mutationKey = ['orders', 'favorite', id] as const;
  return useMutation({
    mutationKey,
    mutationFn: (favorite: boolean) => setFavorite(id, favorite),
    onMutate: async (favorite) => {
      // Stop in-flight fetches so they can't overwrite the optimistic value.
      await queryClient.cancelQueries({ queryKey: orderKeys.detail(id) });
      await queryClient.cancelQueries({ queryKey: orderKeys.lists() });
      // Snapshot what you are about to change, then change it.
      const snapshot = [
        ...queryClient.getQueriesData({ queryKey: orderKeys.detail(id) }),
        ...queryClient.getQueriesData({ queryKey: orderKeys.lists() }),
      ];
      patchOrder(queryClient, id, { favorite });
      return { snapshot };
    },
    onError: (_error, _favorite, onMutateResult) => {
      onMutateResult?.snapshot.forEach(([key, data]) => queryClient.setQueryData(key, data));
    },
    onSettled: () => {
      // After rapid taps, refetch once, when the last toggle settles. A refetch that lands
      // between two writes can flash the old value.
      if (queryClient.isMutating({ mutationKey }) === 1) {
        void queryClient.invalidateQueries({ queryKey: orderKeys.detail(id) });
      }
    },
  });
}
```

- Cancel first, then snapshot, then patch. Restore exactly what you patched in `onError`.
- Tell the user it failed, from the screen:
  `mutate(next, { onError: () => toast('Could not save. Try again.') })`.
- Don't use optimistic updates for payments, irreversible actions, or anything where
  showing success before the server confirms would mislead.

### When the server finishes the work later

Some writes start background work (a rollup, a recalculation). The response is right, but
related data keeps changing for a while. Patch what you know. If the API reports the job's
state, poll that one query until it's done:

```ts
refetchInterval: (query) => (query.state.data?.status === 'processing' ? 2000 : false),
```

If it doesn't, schedule one delayed re-check of the affected keys instead of polling.

## Infinite lists (feeds)

```ts
export function useOrderFeed(filters: OrderFilters) {
  return useInfiniteQuery({
    queryKey: orderKeys.feed(filters),
    queryFn: ({ pageParam }) => listOrders({ ...filters, cursor: pageParam }),
    initialPageParam: null as string | null,
    getNextPageParam: (lastPage) => lastPage.nextCursor, // null means no more pages
  });
}
```

- Load more from `onEndReached` (mobile) or an intersection observer (web), guarded by
  `hasNextPage && !isFetchingNextPage`.
- Paginate on the server. Never fetch 1,000 rows and slice them in the client.
- A refetch of a feed refetches every loaded page, one after another. Patch feeds after a
  write (`patchOrder` does) and invalidate them only when list membership can change.

## Scoped keys

If users can switch organisation, workspace or account, the scope goes into every key
that depends on it, and the fetcher takes the scope from the same place as the key.

```ts
export const orderKeys = {
  all: (workspaceId: string) => ['orders', workspaceId] as const,
  details: (workspaceId: string) => [...orderKeys.all(workspaceId), 'detail'] as const,
  detail: (workspaceId: string, id: string) => [...orderKeys.details(workspaceId), id] as const,
  // ...every other key takes workspaceId too
};

export function useOrder(id: string) {
  const workspaceId = useWorkspaceId(); // from the client store that owns it
  return useQuery({
    queryKey: orderKeys.detail(workspaceId, id),
    queryFn: () => getOrder(workspaceId, id), // the scope travels with the key
  });
}
```

- Don't let the API client read the current workspace from the store when a request goes
  out. A background refetch of workspace A's key would then store workspace B's data under
  it.
- Prefer scope in keys when the cache is persisted, or when switching back should be
  instant (A's data is still cached).
- The simpler fallback is to clear on switch and remount. In the handler that changes the
  scope, call `queryClient.removeQueries()` (it also cancels in-flight fetches), and render
  the scoped screens under `key={workspaceId}` so they remount with fresh queries. Without
  the remount, mounted screens keep showing the old scope's data. Avoid `resetQueries()`
  here: it refetches before the screens re-render, so a `queryFn` that closed over the old
  scope fetches the old data again.

## Persisting the query cache

Persisting makes a cold start show data immediately. Do it carefully:

```tsx
const DAY = 24 * 60 * 60 * 1000;

export const queryClient = new QueryClient({
  // At least maxAge. With the 5-minute default, unused queries are garbage-collected
  // and drop out of the persisted copy long before maxAge.
  defaultOptions: { queries: { gcTime: DAY } },
});

// localStorage on web (undefined during SSR); AsyncStorage or an MMKV adapter on React Native.
export const queryPersister = createAsyncStoragePersister({
  storage: typeof window === 'undefined' ? undefined : window.localStorage,
});

export function AppQueryProvider({ children }: { children: ReactNode }) {
  return (
    <PersistQueryClientProvider
      client={queryClient}
      persistOptions={{
        persister: queryPersister,
        maxAge: DAY,
        buster: BUILD_ID, // change it whenever the cached data's shape can change
        dehydrateOptions: {
          // Opt in per query with `meta: { persist: true }`, and keep the default
          // check, which only persists successful queries.
          shouldDehydrateQuery: (query) =>
            defaultShouldDehydrateQuery(query) && query.meta?.persist === true,
        },
      }}
    >
      {children}
    </PersistQueryClientProvider>
  );
}
```

- Packages: `@tanstack/react-query-persist-client` and
  `@tanstack/query-async-storage-persister`. The async persister also takes synchronous
  storage such as `localStorage`; `createSyncStoragePersister` is deprecated.
- `buster`: the app version plus the OTA update id on mobile, a build id on web. A cache
  saved with a different buster is thrown away on restore.
- Persist only safe, non-sensitive queries. Overriding `shouldDehydrateQuery` replaces the
  default, so keep `defaultShouldDehydrateQuery(query)` in it.
- Queries wait while the cache restores. If a cold start must not flash empty states, gate
  the first screen on `useIsRestoring()`.
- On sign-out, remove the persisted copy too: `queryClient.clear()` (it also cancels
  in-flight queries), then `await queryPersister.removeClient()`. See
  `persistence-and-sign-out.md`.

## Next.js (App Router)

```ts
// get-query-client.ts: a plain module (no 'use client'), so both sides can import it.
function makeQueryClient() {
  return new QueryClient({
    // Above 0, so the browser doesn't refetch what the server just fetched.
    defaultOptions: { queries: { staleTime: 60 * 1000 } },
  });
}

let browserClient: QueryClient | undefined;

export function getQueryClient() {
  if (typeof window === 'undefined') return makeQueryClient(); // server: a new client per request
  return (browserClient ??= makeQueryClient()); // browser: one client for the whole session
}
```

```tsx
// app/orders/[id]/page.tsx (a server component)
export default async function OrderPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const queryClient = getQueryClient();
  await queryClient.query(orderQuery(id)).catch(() => {}); // prefetchQuery() before 5.102
  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <OrderDetail id={id} />
    </HydrationBoundary>
  );
}
```

- Never share a module-level `QueryClient` across server requests: one user's data would
  end up in another user's page.
- The client provider calls `getQueryClient()`. Don't create the client in `useState`
  unless a Suspense boundary sits between it and anything that suspends; React throws the
  client away if the first render suspends.
- Keep `queryOptions` factories in files without `'use client'`, so server components can
  import them. `OrderDetail` is a client component that calls `useOrder(id)`.
- A failed prefetch isn't sent to the browser; the client fetches that query itself.

## Errors

- Normalise API errors into a display string in one place (the API client), so screens
  show `error.message` without parsing.
- A 401 is handled once, in the API client (refresh the token, or sign out), not in each
  query.
- Background refetch failures: report them once, globally, and only when data is already
  on screen (otherwise the screen's error state shows it):
  `new QueryCache({ onError: (error, query) => { if (query.state.data !== undefined) toast(error.message); } })`.
