# Server state with RTK Query (Redux Toolkit 2)

If the app already uses Redux Toolkit, RTK Query is its server-state cache. Don't add
TanStack Query or SWR next to it, and don't store fetched data in your own slices.
Examples are checked against Redux Toolkit 2.13.

## Contents

- [One API, tags name the cache](#one-api-tags-name-the-cache)
- [Reading](#reading)
- [Mutations own the cache](#mutations-own-the-cache): instant update, optimistic update
- [Infinite lists (feeds)](#infinite-lists-feeds)
- [Scoped data and sign-out](#scoped-data-and-sign-out)

## One API, tags name the cache

Tags play the role of query keys: queries **provide** tags, mutations **invalidate** them
or **patch** the cached data.

```ts
export const api = createApi({
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  tagTypes: ['Order', 'OrderCounts'],
  endpoints: (build) => ({
    getOrder: build.query<Order, string>({
      query: (id) => `orders/${id}`,
      providesTags: (_result, _error, id) => [{ type: 'Order', id }],
    }),
    listOrders: build.query<OrderPage, OrderFilters>({
      query: (filters) => ({ url: 'orders', params: filters }),
      providesTags: (result) => [
        ...(result?.items.map((o) => ({ type: 'Order' as const, id: o.id })) ?? []),
        { type: 'Order', id: 'LIST' },
      ],
    }),
    getOrderCounts: build.query<OrderCounts, void>({
      query: () => 'orders/counts',
      providesTags: ['OrderCounts'],
    }),
    // orderFeed, updateOrderStatus and toggleFavorite are shown below.
  }),
});
```

- Each list provides one tag per order plus `LIST`. Invalidating `{ type: 'Order', id }`
  refetches that order's detail **and** every list that contains it. Invalidating
  `{ type: 'Order', id: 'LIST' }` refetches every list.
- Invalidation refetches only queries that are in use; unused entries are dropped and load
  fresh next time.
- Screens use the generated hooks (`useGetOrderQuery`, `useUpdateOrderStatusMutation`) and
  never dispatch cache updates themselves.

## Reading

- RTK Query serves cached data and refetches only when a tag is invalidated or you ask. For
  live data, pass `refetchOnMountOrArgChange: 30` (seconds) or `pollingInterval`.
  `refetchOnFocus` and `refetchOnReconnect` work only after `setupListeners(store.dispatch)`.
- Data nothing uses is dropped after `keepUnusedDataFor` (60 s by default).
- When the args change (next page, new filter), `data` keeps the previous result while the
  new one loads, and `isFetching` is true. Use `currentData` when you must not show it.
- Wait for an input with `skipToken`: `useGetOrderQuery(orderId ?? skipToken)`.
- Prefetch when you can predict the next screen: `usePrefetch('getOrder', { ifOlderThan: 60 })`
  returns a function to call on press-in or hover.

## Mutations own the cache

### Instant update, then a background refresh

One helper patches every cached copy of an order and returns the patches, so an
optimistic update can undo them:

```ts
type ApiDispatch = ThunkDispatch<any, any, UnknownAction>; // both types from '@reduxjs/toolkit'
type ApiState = Parameters<typeof api.util.selectCachedArgsForQuery>[0];

function patchOrder(dispatch: ApiDispatch, state: ApiState, id: string, patch: Partial<Order>) {
  const apply = (items: Order[]) => {
    const row = items.find((o) => o.id === id);
    if (row) Object.assign(row, patch);
  };
  const patches = [
    dispatch(api.util.updateQueryData('getOrder', id, (draft) => { Object.assign(draft, patch); })),
  ];
  for (const args of api.util.selectCachedArgsForQuery(state, 'listOrders')) {
    patches.push(dispatch(api.util.updateQueryData('listOrders', args, (draft) => apply(draft.items))));
  }
  for (const args of api.util.selectCachedArgsForQuery(state, 'orderFeed')) {
    patches.push(
      dispatch(api.util.updateQueryData('orderFeed', args, (draft) => draft.pages.forEach((p) => apply(p.items)))),
    );
  }
  return patches;
}
```

```ts
updateOrderStatus: build.mutation<Order, { id: string; status: OrderStatus }>({
  query: ({ id, status }) => ({ url: `orders/${id}/status`, method: 'PATCH', body: { status } }),
  async onQueryStarted({ id }, { dispatch, getState, queryFulfilled }) {
    try {
      const { data: updated } = await queryFulfilled;
      // Show it now: write the response into every cached copy of the order.
      patchOrder(dispatch, getState(), id, updated);
    } catch {
      // The write failed, so there is nothing to patch. The screen shows the error.
    }
  },
  // In the background: lists may gain or lose the order, and the counts change.
  // The function form skips invalidation when the request failed.
  invalidatesTags: (_result, error) => (error ? [] : [{ type: 'Order', id: 'LIST' }, 'OrderCounts']),
}),
```

- Always wrap `await queryFulfilled` in `try`/`catch`. It rejects when the request fails,
  and nothing handles the promise `onQueryStarted` returns, so a missing `catch` becomes an
  unhandled rejection.
- An array `invalidatesTags` also fires when the server returns an error. Use the function
  form to skip that.
- Invalidate `LIST` only when the change can move the order in or out of a list (filter,
  sort order, page). Otherwise the patch is enough and nothing refetches.
- Invalidate only data you can't compute. If you can (a status change moves one count from
  `open` to `closed`), patch it too:
  `api.util.updateQueryData('getOrderCounts', undefined, (c) => { c[from] -= 1; c[to] += 1; })`.
- The screen calls `await updateStatus({ id, status }).unwrap()` in `try`/`catch`: close the
  dialog on success, show a toast on failure.

### Optimistic update with rollback

```ts
toggleFavorite: build.mutation<Order, { id: string; favorite: boolean }>({
  query: ({ id, favorite }) => ({ url: `orders/${id}/favorite`, method: 'PUT', body: { favorite } }),
  async onQueryStarted({ id, favorite }, { dispatch, getState, queryFulfilled }) {
    const patches = patchOrder(dispatch, getState(), id, { favorite });
    try {
      await queryFulfilled;
    } catch {
      patches.forEach((patch) => patch.undo()); // roll back every copy
    }
  },
}),
```

- RTK Query can't cancel a query that is already in flight. A refetch that started before
  the toggle can land after it and overwrite the optimistic value. If that matters, add
  `invalidatesTags: (_result, _error, { id }) => [{ type: 'Order', id }]` to refetch once
  the write settles (this also refetches every list that contains the order).
- Don't use optimistic updates for payments or irreversible actions.

## Infinite lists (feeds)

Redux Toolkit 2.6+ has infinite query endpoints:

```ts
orderFeed: build.infiniteQuery<OrderPage, OrderFilters, string | null>({
  infiniteQueryOptions: {
    initialPageParam: null,
    getNextPageParam: (lastPage) => lastPage.nextCursor, // null means no more pages
  },
  query: ({ queryArg, pageParam }) => ({ url: 'orders', params: { ...queryArg, cursor: pageParam ?? undefined } }),
  providesTags: (result) => [
    ...(result?.pages.flatMap((p) => p.items.map((o) => ({ type: 'Order' as const, id: o.id }))) ?? []),
    { type: 'Order', id: 'LIST' },
  ],
}),
```

- The hook is `useOrderFeedInfiniteQuery(filters)`. Render `data.pages`; load more with
  `fetchNextPage()`, guarded by `hasNextPage && !isFetchingNextPage`.
- A refetch reloads every loaded page in order, so patch feeds after a write (`patchOrder`
  does) and invalidate them only when list membership can change.
- On versions before 2.6, keep one cache entry per page (`listOrders` with a cursor in the
  args) and render the pages you have loaded.

## Scoped data and sign-out

- If users can switch workspace or account, include the scope in the query args, so it is
  part of the cache key, and build the request from the args. If `prepareHeaders` reads the
  workspace from the store instead, a background refetch of workspace A's entry goes out
  with B's header and stores B's data under A.
- Or reset on switch: `dispatch(api.util.resetApiState())`, and remount the scoped screens
  (`key={workspaceId}`). Hooks keep some state in components, so the reset alone may not
  clear what is on screen.
- Sign-out dispatches `api.util.resetApiState()` along with the slices' reset (see
  `client-redux-toolkit.md` and `persistence-and-sign-out.md`).
