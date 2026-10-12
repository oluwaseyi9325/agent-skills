# Shared client state with React Context

Context is often enough when there's very little shared client state left once server data
lives in a cache and filters live in the URL. Use it for values that **change rarely**:
the signed-in session, theme, locale, feature flags.

## When Context is the wrong tool

Every component that reads a Context re-renders whenever its value changes. If the value
changes often (typing, timers, frequently toggled UI) or many components read it, use a
store with selectors (Zustand, Jotai, Redux) instead of fighting re-renders.

## Do it well

Split state from actions, and memoise the value, so readers of one don't re-render when
the other changes:

```tsx
import { createContext, useContext, useMemo, useState, type ReactNode } from 'react';

type Session = { userId: string; token: string } | null;

const SessionContext = createContext<Session>(null);
const SessionActionsContext = createContext<{
  signIn: (s: NonNullable<Session>) => void;
  signOut: () => void;
} | null>(null);

export function SessionProvider({ children }: { children: ReactNode }) {
  const [session, setSession] = useState<Session>(null);
  const actions = useMemo(
    () => ({ signIn: setSession, signOut: () => setSession(null) }),
    [],
  );
  return (
    <SessionActionsContext.Provider value={actions}>
      <SessionContext.Provider value={session}>{children}</SessionContext.Provider>
    </SessionActionsContext.Provider>
  );
}

export const useSession = () => useContext(SessionContext);

export function useSessionActions() {
  const actions = useContext(SessionActionsContext);
  if (!actions) throw new Error('useSessionActions must be used inside SessionProvider');
  return actions;
}
```

- One Context per concern. A single "AppContext" holding everything re-renders everything.
- Throw a clear error when a hook is used outside its provider.
- No server data in Context. Read it from the server-state cache.
- State in `useState` is lost on restart. If the session must survive one, load it from
  storage before routing (see `persistence-and-sign-out.md`).

## Resetting per-user state

Put the per-user providers **under** `SessionProvider` and key them by the user id. When
the user changes, they remount with their initial state:

```tsx
function UserProviders({ children }: { children: ReactNode }) {
  const session = useSession();
  return (
    <WorkspaceProvider key={session?.userId ?? 'signed-out'}>{children}</WorkspaceProvider>
  );
}
```

The key can't go on `SessionProvider` itself, which holds the user id. Remounting resets
memory only (and the state of every component below). Storage and the server cache are
still cleared by the sign-out function.
