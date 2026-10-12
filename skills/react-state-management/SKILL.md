---
name: react-state-management
description: Decides where each piece of state belongs in a React or React Native app (server cache, client store, form, URL, local component state, or derived during render) and implements it with the libraries the project already uses, covering TanStack Query (React Query), SWR, RTK Query, Zustand, Redux Toolkit, Jotai, Context, React Hook Form, Formik, Zod, Yup, Valibot and router search params. Use when adding data fetching, a store, a form, filters, persistence or sign-out; when writing mutations, query keys, cache invalidation or optimistic updates; when choosing between useState, Context and a state library; or when fixing stale data after an edit, duplicated or out-of-sync state, needless re-renders, useEffect sync loops, slow saves, filters lost on refresh, or one user's or workspace's data showing for another.
license: MIT
metadata:
  author: Seyi Adedokun
  version: "1.0.0"
---

# React state management

Many state bugs share one cause: **the same fact is stored in two places**, and the copies
drift apart. Give every piece of state exactly one owner, then handle each kind with the
libraries the project already uses.

Applies to React (Next.js, Vite, React Router / Remix) and React Native (Expo).

## Workflow

1. **Detect the stack.** Read the app's `package.json` (in a monorepo, the package you are
   working in) and one nearby feature, to see which libraries are installed and where keys,
   stores and schemas live.
2. **Open only the references you need**, from the table in "Use what the project already
   has". Skip the rest.
3. **Pick one owner** for every piece of state the task touches, using "Decide where the
   state belongs".
4. **Implement** with the rules below and the project's existing patterns. For every write,
   decide how the cache changes: patch it from the response, invalidate specific keys, or
   update optimistically.
5. **Fixing a bug?** Start from "Symptoms and fixes". Remove the second copy or add the
   missing cache update; don't add an effect that keeps two copies in sync.
6. **Verify.** Run the project's type-check and tests, then go through "Before you finish".

## Use what the project already has

Follow the libraries already installed. **Don't add a second library for a job that already
has one** (a second cache, store, form library or validator). If the project has two for the
same job, follow whichever the surrounding feature uses.

Apply these rules to the code you write or change for the task. Don't migrate existing
features (for example `createAsyncThunk` slices, or server data kept in a store) unless the
task is that migration; point the problem out instead.

| Job | If the project has… | Read |
|---|---|---|
| Server data | `@tanstack/react-query` | [references/server-tanstack-query.md](references/server-tanstack-query.md) |
| | `swr` | [references/server-swr.md](references/server-swr.md) |
| | `@reduxjs/toolkit` (use its RTK Query) | [references/server-rtk-query.md](references/server-rtk-query.md) |
| Client state | `zustand` | [references/client-zustand.md](references/client-zustand.md) |
| | `@reduxjs/toolkit` / `react-redux` | [references/client-redux-toolkit.md](references/client-redux-toolkit.md) |
| | `jotai` | [references/client-jotai.md](references/client-jotai.md) |
| | none of these | [references/client-context.md](references/client-context.md) |
| Forms | `react-hook-form` (with Zod, Yup or Valibot) | [references/forms-react-hook-form.md](references/forms-react-hook-form.md) |
| | `formik` | [references/forms-formik.md](references/forms-formik.md) |
| URL state | Next.js, React Router, Expo Router or `nuqs` | [references/url-state.md](references/url-state.md) |
| Persistence, sign-out, switching workspace | any | [references/persistence-and-sign-out.md](references/persistence-and-sign-out.md) |

**Nothing installed for a job the task needs?** Use these defaults, and say in your reply
which dependency you added and why:

- Server data: TanStack Query, or RTK Query if the app already uses Redux Toolkit.
- Client state: `useState` and Context first. Add Zustand only when shared state changes
  often or is read in many places.
- Forms: React Hook Form with Zod, for anything beyond a couple of fields.
- URL state: the router's own search-param API.

## Decide where the state belongs

Ask in table order and stop at the first match.

| The state is… | Example | Owner |
|---|---|---|
| Computable from props or other state | totals of data you already hold, a filtered view of a loaded list, "is valid" | **Nothing.** Compute it during render. |
| Data that lives on a server | orders, profile, balance, permissions | **The server-state cache** (or the framework, for data loaded in Server Components or route loaders) |
| Something refresh, back or a shared link should restore | filters, search, sort, page, active tab | **The URL** |
| A draft the user is editing before submitting | create-order or edit-profile form | **The form library** and its schema |
| Needed by components far apart, and not from a server | signed-in session, selected workspace, theme | **A client store** (Zustand store, Redux slice, Jotai atom or Context) |
| Needed by one component, or a parent and its children | sheet open, hovered row, section expanded | **`useState`** in the closest common parent |

Server-side aggregates (a badge count across all pages, a total from the API) are server
data, not derived state: you can't compute them from one loaded page.

Persistence is not an owner. It is something you add to a client store or the server cache
when that state must survive a restart.

## Rules

### Server data
- Read it through the server-state library, not `useEffect` + `useState`. Data loaded in a
  React Server Component or a route loader is owned by the framework: pass it down, or
  hydrate it into the client cache, but don't copy it into a store or `useState`.
- **Keep one copy.** Don't put server data in a store, Context or `useState`. Read it from
  the cache wherever it's needed; the cache dedupes the requests. The one deliberate fork is
  an edit form's draft (see Forms).
- Name cache entries in **one place per feature** (a key factory, a key helper, or RTK Query
  tags), never as ad-hoc strings in components.
- **The write owns the cache update.** The mutation updates or invalidates what it changed.
  Screens call it and show feedback; they don't touch the cache.
- After a write, **show the result straight away**:
  - The response contains the updated record: write it into the cached detail and every
    cached list row that shows it.
  - The change can move the record into or out of a list (a status change and a
    status-filtered list, a new sort position, a different page): also refresh those lists
    in the background. A patch alone is enough only when list membership and order can't
    change.
  - Some effects can't be read from the response (counts, totals, other records):
    invalidate just those keys and let them refetch in the background.
  - Don't `await` a refetch of everything before closing a dialog, and don't invalidate the
    whole cache.
- **Optimistic updates** only when the write is very likely to succeed and waiting would feel
  slow (toggles, reordering). Cancel in-flight fetches for those keys first, roll back on
  error, and tell the user it failed. Not for payments or irreversible actions.
- If users can switch workspace, tenant or account, prefer putting that scope in **every
  cache key** that depends on it, and have the request take the scope from the key, not
  from global state at fetch time (otherwise a background refetch can store one workspace's
  data under another's key). Clearing the cache on switch is the simpler fallback when the
  cache isn't persisted and switching back needn't be instant.

### Shared client state
- Small stores, split by concern (auth, workspace, preferences). No store that holds
  everything.
- Components **subscribe to the slice they use** (a selector, a single atom), not the whole
  store. With Context, split by concern so readers of one value don't re-render when another
  changes.
- Updates live next to the state they change (actions, reducers, atom setters).
- Persist only what must survive a restart, and version it. Secrets such as tokens go in
  secure storage, not the same store as preferences. If the storage is async (SecureStore,
  AsyncStorage), wait for it to load before routing on it.

### Forms
- **Validation rules live in one place**: the schema, when the project uses a schema
  library. Infer the form's types from it. Don't repeat the rules in the component (for
  example in a `disabled` condition); read the form's validity state instead.
- When a rule depends on runtime data, build the schema from that data.
- **Edit forms:** the server cache owns the saved record; the form owns the unsaved draft.
  Seed the form's initial values from the cached record, not from a `useState` copy. Don't
  write the draft into the cache while the user types, and don't let a background refetch
  overwrite fields they have changed. Save through a mutation that updates the cache, then
  reset or close the form.
- A form with one or two fields and no shared rules can use plain `useState` (or the
  framework's form actions). Don't add a form library just for it.

### URL state
- Filters, search, sort, page and tab go in the URL when refresh, back or a shared link
  should restore them. Read them from the URL; don't mirror them into `useState`.
- Params are user input: parse them and fall back to defaults instead of throwing.
- Changing a filter resets the page. Use replace, not push, for filter changes.
- Debounce search, because every new value is a new cache key and a new request. The search
  box is the one exception to "don't mirror params": it keeps what is being typed in local
  state, and the URL gets the value once typing pauses.

### Derived state
- Compute it during render. Add `useMemo` only when the calculation is measurably expensive.
- **Don't `setState` in a `useEffect` to mirror other state.** It costs an extra render,
  shows stale values for a frame and can loop. To reset a component's state when an id
  changes, render it with `key={id}`.

### Sign-out and account switching
One function clears everything a user owns, in an order that stops old data leaking back:

1. Return early if a sign-out is already running (a 401 during sign-out must not start a
   second one).
2. Mark the session signed out first, so gated requests stop and no token is sent.
3. Leave the signed-in screens so they unmount (on a native stack, don't leave them mounted
   underneath the sign-in screen).
4. Cancel in-flight requests, then clear the server cache **and its persisted copy**.
5. Reset every store **and its storage**, and other per-user local data.

Call it from the sign-out button and when a token refresh fails. The calls for each library
are in [references/persistence-and-sign-out.md](references/persistence-and-sign-out.md).

## Symptoms and fixes

| Symptom | Usual cause | Fix |
|---|---|---|
| List shows the old value after an edit | The write didn't update the list's cache | Patch or invalidate the list in the mutation |
| Edited item stays in the wrong filtered list or page | Rows were patched but lists weren't refreshed | Also refresh lists whose membership or order can change |
| "Did it save?": the UI waits seconds after a write | Awaiting a refetch of many queries | Write the response into the cache; refresh the rest in the background |
| Two parts of the screen disagree | Server data copied into a store, Context or `useState` | Delete the copy; read from the cache |
| Typing in one field re-renders the whole screen | Component reads a whole store or a big Context, or form state lives in a store | Select one value; split the store or Context; keep form state in the form library |
| Flicker or render loop | `useEffect` copying one state into another | Derive the value, or reset with `key` |
| Edit form loses what the user typed | Form re-seeded when the query refetched | Seed once, or keep dirty fields when re-seeding |
| Filters lost on refresh or back | Filters in component state | Move them to the URL |
| Signed-in user sent to sign-in on a cold start | Routing ran before persisted state loaded | Wait for hydration, then route |
| Next user sees the previous user's data | Cache or store not cleared on sign-out | One sign-out function that clears everything |
| Wrong workspace's data after switching | Scope missing from cache keys | Add scope to the keys, or clear the cache on switch |
| App crashes or shows junk after a release | Old persisted data in a new shape | Version persisted data; migrate or discard old versions |
| Form and component disagree about validity | Validation in two places | Validate only in the schema |

## Before you finish

- [ ] No library added for a job the project already covers; any new dependency is named in
      the reply.
- [ ] Every new piece of state has one owner from the table above.
- [ ] No server data is kept in a second place (store, Context, `useState`); edit forms only
      seed their initial values from it.
- [ ] Cache keys come from the feature's one naming place, and include the scope if users
      can switch.
- [ ] Each write updates or invalidates exactly what it changed (including lists the record
      can move into or out of), and the UI shows the result without waiting for a full
      refetch.
- [ ] No `setState` inside `useEffect` mirrors other state.
- [ ] Components select slices, not whole stores or catch-all Contexts.
- [ ] Persisted state is partialized, versioned, and cleared on sign-out.
- [ ] Filters, search and page that should survive a refresh are in the URL.
- [ ] The project's type-check and tests pass.
