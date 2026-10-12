# Server state with SWR (v2)

The SWR cache owns server data. Components read it through `useSWR`; writes keep it correct
with `mutate` and `useSWRMutation`. Examples are checked against SWR 2.5.

## Contents

- [Name keys in one place](#name-keys-in-one-place)
- [Reading](#reading)
- [Writes own the cache](#writes-own-the-cache): instant update, optimistic update
- [Infinite lists (feeds)](#infinite-lists-feeds)
- [Scoped keys and clearing the cache](#scoped-keys-and-clearing-the-cache)
- [Next.js (App Router)](#nextjs-app-router)

## Name keys in one place

SWR keys are just values, so give each feature one helper instead of scattering strings.

```ts
export const orderKeys = {
  list: (filters: OrderFilters) => ['orders', 'list', filters] as const,
  feedPage: (filters: OrderFilters, cursor: string | null) => ['orders', 'feed', filters, cursor] as const,
  detail: (id: string) => ['orders', 'detail', id] as const,
  counts: () => ['orders', 'counts'] as const,
};

// A key filter for `mutate`: matches every cached paged list, whatever its filters.
export const isOrderList = (key: unknown) =>
  Array.isArray(key) && key[0] === 'orders' && key[1] === 'list';
```

- Array keys are compared by value, so a new `filters` object on each render still hits
  the same cache entry.
- The fetcher receives the key, so the key carries every input the request needs.

## Reading

```ts
const fetchOrder = ([, , id]: ReturnType<typeof orderKeys.detail>) => getOrder(id);

export function useOrder(id: string | null) {
  return useSWR(id ? orderKeys.detail(id) : null, fetchOrder);
}

export function useOrderList(filters: OrderFilters) {
  return useSWR(orderKeys.list(filters), ([, , f]) => listOrders(f), {
    keepPreviousData: true, // keep the current page on screen while the next one loads
  });
}
```

- A `null` key means "don't fetch yet": no id, not signed in, or waiting on another query
  (`useSWR(order ? customerKeys.detail(order.customerId) : null, getCustomer)`).
- By default SWR revalidates on mount, window focus and reconnect, and shares one request
  between hooks for 2 s (`dedupingInterval`). Tune per kind of data:
  `revalidateOnFocus: false` and `revalidateIfStale: false` for reference data,
  `refreshInterval` for live data.
- Prefetch when you can predict the next screen: `preload(orderKeys.detail(id), fetchOrder)`.

## Writes own the cache

`useSWRMutation` defaults worth knowing:

- `trigger` throws when the request fails (`throwOnError: true`). Catch it in the screen
  and tell the user, or pass `throwOnError: false` and read `error`.
- `populateCache` is `false` (unlike `mutate`). Set it to `true` only when the response is
  the full record for the key.
- `revalidate` is `true`: the key refetches after the write.

### Instant update, then a background refresh

```ts
export function useUpdateOrderStatus(id: string) {
  const { mutate } = useSWRConfig();
  return useSWRMutation(
    orderKeys.detail(id),
    (_key, { arg }: { arg: OrderStatus }) => updateOrderStatus(id, arg),
    {
      populateCache: true, // the response is the full order: it becomes the cached detail
      revalidate: false, // so the detail needs no refetch
      onSuccess: (updated) => {
        // Patch every cached paged list now. Revalidation (on by default) then refetches the
        // lists on screen, because a status change can move the order between filtered lists.
        // Pass { revalidate: false } for changes that can't.
        void mutate<OrderPage>(isOrderList, (page) =>
          page && { ...page, items: page.items.map((o) => (o.id === id ? { ...o, ...updated } : o)) },
        );
        void mutate(orderKeys.counts()); // refresh what the response can't tell us
      },
    },
  );
}
```

- The screen wraps `await trigger(status)` in `try`/`catch`, closes the dialog on success,
  and shows a toast on failure. Don't `await` a refetch of everything first.
- `mutate(key)` with no data only refetches hooks that are mounted. Unmounted keys keep
  their data and revalidate when they mount again.
- If the API returns no record (`{ ok: true }`, 204), leave `populateCache` off and let
  `revalidate` refetch the detail. To show the change before that refetch lands, use
  `optimisticData` as below.

### Optimistic update with rollback

```ts
export function useToggleFavorite(id: string) {
  const { data: order } = useOrder(id);
  const { trigger, isMutating } = useSWRMutation(
    orderKeys.detail(id),
    (_key, { arg }: { arg: boolean }) => setFavorite(id, arg),
  );
  async function toggle() {
    if (!order) return;
    const favorite = !order.favorite;
    await trigger(favorite, {
      optimisticData: { ...order, favorite },
      rollbackOnError: true, // put the old value back if the request fails
      populateCache: true, // the API returns the full order
      revalidate: false,
    });
  }
  return { toggle, isMutating };
}
```

- `toggle()` rejects when the request fails. The screen catches it and tells the user.
- `populateCache: true` assumes the API returns the full order. If it returns `{ ok: true }`,
  pass `populateCache: false` and `revalidate: true`: the optimistic value stays in the
  cache, and a refetch confirms it.
- Once a newer write to the same key starts, SWR drops the older write's result, so rapid
  taps don't flash old values.
- Lists that show the star: patch them in `onSuccess` with `mutate(isOrderList, …)`, as in
  the status example.
- Don't use optimistic updates for payments or irreversible actions.

## Infinite lists (feeds)

```ts
// One getKey per filter set; each page is cached under orderKeys.feedPage(...).
export const orderFeedKey = (filters: OrderFilters) => (_index: number, previous: OrderPage | null) =>
  previous && !previous.nextCursor ? null : orderKeys.feedPage(filters, previous?.nextCursor ?? null);

export function useOrderFeed(filters: OrderFilters) {
  return useSWRInfinite(orderFeedKey(filters), ([, , f, cursor]) => listOrders({ ...f, cursor }));
}
```

- Load the next page with `setSize((size) => size + 1)` when the user reaches the end and
  the last loaded page has a `nextCursor`.
- **Key filters can't see feeds.** `mutate(filterFn, …)` skips `useSWRInfinite` caches, so
  `isOrderList` never reaches them. Address a feed by its getKey instead, with
  `unstable_serialize` from `swr/infinite`:

```ts
// Patch the feed for these filters; revalidation (on by default) then refetches its pages.
void mutate<OrderPage[]>(unstable_serialize(orderFeedKey(filters)), (pages) =>
  pages?.map((p) => ({ ...p, items: p.items.map((o) => (o.id === id ? { ...o, ...updated } : o)) })),
);
```

- This needs the feed's filters (read them from the URL). Inside the feed screen, the
  `mutate` returned by `useOrderFeed` does the same without serializing.

## Scoped keys and clearing the cache

- If users can switch workspace or account, put the scope in the key after the feature name
  (`['orders', workspaceId, 'list', filters]`) and let the fetcher read it from the key, not
  from the store at request time. Update key filters like `isOrderList` to match.
- Or clear on switch and remount: wrap the scoped screens in
  `<SWRConfig key={workspaceId} value={{ provider: () => new Map() }}>`. Each workspace gets
  a new, empty cache and the screens remount with it, so no fetcher from the previous render
  runs against the new scope. This works in any SWR 2 version.
- To clear the cache in place (sign-out), SWR 2.5+ has `unload({ revalidate: false })` from
  `useSWRConfig()`. It clears every entry, feeds included, and drops in-flight requests.
  `mutate(() => true, undefined, { revalidate: false })` is not enough: key filters skip
  feeds.

## Next.js (App Router)

- `useSWR` and `useSWRMutation` only run in client components (`'use client'`). `SWRConfig`
  and `unstable_serialize` can also be used in server components.
- Hand server-fetched data to SWR through `fallback`, with array keys serialized:
  `<SWRConfig value={{ fallback: { [unstable_serialize(orderKeys.detail(id))]: order } }}>`.
- Server Actions and `revalidatePath` refresh server-rendered data, not the SWR cache.
  Return the updated record from the action and write it on the client with `mutate`.
