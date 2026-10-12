# URL state: filters, search, page and tab

If refreshing the page, pressing back, or sharing a link should bring the view back, the
state belongs in the URL. Component state loses it.

Put in the URL: filters, search text, sort, page, selected tab, an open detail id.
Keep out of the URL: tokens, personal data, anything long or sensitive.

## Contents

- [Rules](#rules)
- [Parse params with defaults](#parse-params-with-defaults)
- [Next.js (App Router)](#nextjs-app-router)
- [nuqs](#nuqs)
- [React Router](#react-router)
- [Expo Router (React Native)](#expo-router-react-native)
- [Search box](#search-box)

## Rules

- **Read from the URL, write to the URL.** Don't mirror params into `useState`. The one
  exception is the text in a search box while the user types (see [Search box](#search-box)).
- **Changing a filter resets the page** to 1 (or drops the cursor).
- Use `replace` for filter changes, so back leaves the list instead of stepping through
  every filter. Use `push` for real navigation.
- **Parse params with a schema that falls back to defaults.** URLs are user input; a bad or
  hand-edited param must not throw.
- Pass the parsed filters straight into the query key (`orderKeys.list(filters)`), so each
  combination is cached separately.

## Parse params with defaults

```ts
import { z } from 'zod';

export const ORDER_STATUSES = ['open', 'shipped', 'cancelled'] as const;
export type OrderStatus = (typeof ORDER_STATUSES)[number];

// .catch() swaps a missing or bad value for the default, so parse never throws.
export const orderFiltersSchema = z.object({
  status: z.enum(ORDER_STATUSES).optional().catch(undefined),
  q: z.string().catch(''),
  page: z.coerce.number().int().min(1).catch(1),
});
export type OrderFilters = z.infer<typeof orderFiltersSchema>;

export function parseOrderFilters(params: URLSearchParams): OrderFilters {
  return orderFiltersSchema.parse({
    status: params.get('status') ?? undefined,
    q: params.get('q') ?? undefined,
    page: params.get('page') ?? undefined,
  });
}
```

With another schema library, give each field its own fallback the same way (Valibot:
`v.fallback`), so one bad param doesn't reset the others.

## Next.js (App Router)

```ts
'use client';
import { usePathname, useSearchParams } from 'next/navigation';

export function useOrderFilters() {
  const params = useSearchParams();
  const pathname = usePathname();
  const filters = parseOrderFilters(params);

  function setFilter(key: 'status' | 'q', value: string | null) {
    const next = new URLSearchParams(params.toString());
    if (value) next.set(key, value);
    else next.delete(key);
    next.delete('page'); // a new filter starts at page 1
    const query = next.toString();
    const url = query ? `${pathname}?${query}` : pathname;
    // The list is fetched on the client (TanStack Query, SWR): change the URL without a
    // server round trip. If a Server Component renders the list from `searchParams`, use
    // router.replace(url, { scroll: false }) instead.
    window.history.replaceState(null, '', url);
  }

  return { filters, setFilter };
}
```

Wrap the component that calls `useSearchParams` in `<Suspense>`. On a prerendered (static)
route, `next build` fails without it.

## nuqs

If the project uses `nuqs` (Next.js, React Router, Remix or plain React), it handles
parsing, defaults and history.

```ts
import { parseAsInteger, parseAsString, parseAsStringLiteral, useQueryStates } from 'nuqs';

const [filters, setFilters] = useQueryStates({
  status: parseAsStringLiteral(ORDER_STATUSES), // any other value reads as null
  q: parseAsString.withDefault(''),
  page: parseAsInteger.withDefault(1),
});

// Changing a filter resets the page. `null` removes a param from the URL, and a param
// with a default then reads as that default.
void setFilters({ status: 'open', page: null });
```

- Wrap the app once in `NuqsAdapter` from the path for its router:
  `nuqs/adapters/next/app` (in the root layout), `nuqs/adapters/next/pages`,
  `nuqs/adapters/react-router/v7`, `nuqs/adapters/remix` or `nuqs/adapters/react`. The
  unversioned `nuqs/adapters/react-router` is deprecated and only for React Router v6.
- Defaults: `history: 'replace'`, `shallow: true` (the URL changes on the client, with no
  server request) and `clearOnDefault: true` (a value equal to its default leaves the URL).
- Set `shallow: false` only when a Server Component renders the list from `searchParams`.
- In Next.js, nuqs reads `useSearchParams`, so the Suspense rule above applies.

## React Router

```ts
import { useSearchParams } from 'react-router'; // v7

const [params, setParams] = useSearchParams();
const filters = parseOrderFilters(params);

function setStatus(status: OrderStatus | null) {
  setParams(
    (prev) => {
      // `prev` is a fresh copy of the current params, so changing it is safe.
      if (status) prev.set('status', status);
      else prev.delete('status');
      prev.delete('page');
      return prev;
    },
    { replace: true, preventScrollReset: true },
  );
}
```

Unlike React's `setState`, the function form isn't queued: two calls in the same tick don't
see each other's changes, so change several params in one call.

## Expo Router (React Native)

Route params play the URL's role, and deep links restore them.

```ts
import { router, useLocalSearchParams } from 'expo-router';

const params = useLocalSearchParams(); // each value is a string, or string[] if repeated
const filters = orderFiltersSchema.parse(params);

function setStatus(status: OrderStatus | undefined) {
  router.setParams({ status, page: undefined });
}
```

- `router.setParams` merges into the current screen's params and adds no history entry,
  like `replace`. Set a param to `undefined` to clear it. Use `router.push` for real
  navigation.
- Prefer `useLocalSearchParams` to `useGlobalSearchParams`, which re-renders screens in the
  background on every param change.
- On mobile, a list's filters often only need to survive navigation, not an app restart.
  Route params cover that. Persist them in a store only if users expect them to survive a
  restart.

## Search box

Each search value is a new query key, so a new request. If every keystroke went to the URL,
every keystroke would fetch. So the search box is the one place a URL value gets a local
copy: it holds what is being typed and writes to the URL once typing pauses for about
300 ms. The query key reads the URL, so it changes once.

```tsx
'use client';
import { useEffect, useRef, useState } from 'react';

export function SearchBox({ value, onSearch }: { value: string; onSearch: (q: string) => void }) {
  const [text, setText] = useState(value); // what is being typed
  const [urlValue, setUrlValue] = useState(value); // the URL value this box last saw
  const [sent, setSent] = useState(value); // the last value this box wrote

  // The URL changed and this box didn't do it ("Clear filters", a link): show the new value.
  if (value !== urlValue) {
    setUrlValue(value);
    if (value !== sent) {
      setText(value);
      setSent(value);
    }
  }

  const timer = useRef<ReturnType<typeof setTimeout>>(undefined);
  useEffect(() => () => clearTimeout(timer.current), []); // don't write after leaving the screen

  function handleChange(next: string) {
    setText(next);
    clearTimeout(timer.current);
    timer.current = setTimeout(() => {
      setSent(next.trim());
      onSearch(next.trim());
    }, 300);
  }

  return <input type="search" value={text} onChange={(e) => handleChange(e.target.value)} />;
}
```

```tsx
// Next.js hook above
<SearchBox value={filters.q} onSearch={(q) => setFilter('q', q || null)} />
// nuqs
<SearchBox value={filters.q} onSearch={(q) => void setFilters({ q: q || null, page: null })} />
// Expo Router (same component with a TextInput and onChangeText)
<SearchBox value={filters.q} onSearch={(q) => router.setParams({ q: q || undefined, page: undefined })} />
```

Use this box with nuqs too. nuqs updates its hook state on every keystroke;
`limitUrlUpdates: debounce(300)` only delays the URL write, so a query key built from that
state would still change on every keystroke.

In Expo Router, `router.setParams` writes to the focused screen, and a stack keeps the list
mounted under the next screen. Also clear the timer when the list loses focus
(`useFocusEffect`), so a late write can't land on the screen the user just opened.
