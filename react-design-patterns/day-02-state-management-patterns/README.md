# Day 2 — State Management Patterns

Patterns covered today:

1. [Provider Pattern](#1-provider-pattern)
2. [State Reducer Pattern](#2-state-reducer-pattern)

> **Why only 2 patterns, not 3?** Raw `createContext`/`useContext` on its own is rarely how Context is actually used in production — in real projects, Context is *always* wrapped in a dedicated Provider component and a custom hook. Treating "Context" and "Provider" as two separate patterns to learn is misleading; they're one pattern with two halves that only make sense together. The established name for that complete pattern is the **Provider Pattern** — Context is just the underlying React API it's built on, not a separate pattern of its own.

Study method for each pattern: understand the problem → see the bad approach → see the pattern built step by step → real-world example → practice exercise → know when (not) to use it.

---

## 1. Provider Pattern

### The problem — in two parts

**Part A — prop drilling.** Some data is needed by many components scattered across the tree — theme, logged-in user, locale — but not by the components sitting in between them. Passing it down as a prop through every intermediate layer forces every layer to know about data it never actually uses:

```tsx
function App() {
  const [theme, setTheme] = useState("light");
  return <Page theme={theme} />;
}

function Page({ theme }: { theme: string }) {
  return <Sidebar theme={theme} />; // Page doesn't use theme, just relays it
}

function Sidebar({ theme }: { theme: string }) {
  return <ThemeToggleButton theme={theme} />; // Sidebar doesn't use it either
}
```

`Page` and `Sidebar` are forced to accept and forward `theme` purely so it can reach `ThemeToggleButton`. Add one more layer and you edit three files just to thread one value through.

**Part B — raw Context is unsafe on its own.** React's `createContext`/`useContext` solves the drilling problem, but used raw, it opens a *new* problem:

```tsx
const AuthContext = createContext<{ user: User | null } | undefined>(undefined);

// Any file in the app can do this — nothing stops it, nothing warns you
function Profile() {
  const ctx = useContext(AuthContext);
  return <p>{ctx?.user?.name}</p>; // if <AuthContext.Provider> is missing above, this silently renders blank
}
```

If someone forgets to wrap the app in `<AuthContext.Provider>`, there's no error — `ctx` is just `undefined`, and `Profile` quietly renders nothing. You find out from a confused bug report, not from React telling you what went wrong. This is exactly why, in real projects, nobody stops at raw Context — everyone wraps it in a Provider component with a safety check. That combined, safe form is what this pattern actually is.

### Building the pattern — step by step

**Step 1 — create the context**, typed so consumers know exactly what shape of data they'll get:

```tsx
type AuthContextValue = {
  user: User | null;
  login: (u: User) => void;
  logout: () => void;
};

const AuthContext = createContext<AuthContextValue | null>(null);
```

Note the type is `AuthContextValue | null` — `null` is the "nobody has provided this yet" signal, which Step 3 will check for.

**Step 2 — build a Provider component that owns the actual state.** This is the piece raw Context alone doesn't give you: a single place where the state lives and the update logic is written once.

```tsx
export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  const value = { user, login: setUser, logout: () => setUser(null) };

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}
```

Notice there's no manual `useMemo` around `value` here. In React 18 and earlier, that object literal would be recreated on every render, giving every consumer a new reference and forcing them to re-render even when `user` hadn't changed — so hand-wrapping it in `useMemo` was necessary. **React 19's React Compiler removes that need**: it analyzes the component at build time and automatically memoizes values like this for you, so you no longer hand-write `useMemo`/`useCallback` for cases like this in a codebase that has the compiler enabled. This only applies if your project has the React Compiler turned on (Next.js 15+ and most React 19 starters enable it by default now) — in an older codebase without it, you'd still need to add `useMemo` yourself to avoid the extra re-renders.

**Step 3 — export a custom hook, never the raw context.** This is the safety net that fixes Part B's problem:

```tsx
export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error("useAuth must be used within an AuthProvider");
  return ctx;
}
```

Now if `<AuthProvider>` is missing anywhere above a component calling `useAuth()`, you get an immediate, readable crash pointing at the exact component — not a silent `undefined` bug discovered days later.

**Step 4 — consume it.** Notice the consuming component never imports `AuthContext` at all — only the hook:

```tsx
function Profile() {
  const { user, logout } = useAuth();
  return <p onClick={logout}>{user?.name}</p>;
}
```

That's the complete pattern: **Context (the transport) + Provider (the owner of state) + custom hook (the safe, encapsulated door in) — all three pieces, always together.**

### Real-world example

This exact three-piece shape is how virtually every major React library exposes shared state:

- React Query: `<QueryClientProvider>` + `useQueryClient()` / `useQuery()`
- Redux: `<Provider store={store}>` + `useSelector()` / `useDispatch()`
- NextAuth: `<SessionProvider>` + `useSession()`
- Radix UI / shadcn: `<ThemeProvider>` + `useTheme()`

And at the product level: dark/light mode on GitHub and Twitter/X, the logged-in user available to every page on any app with auth, feature-flag values read by components anywhere in the tree — all built with the Provider Pattern, never raw Context alone.

### Practice exercise

Build your own `CartProvider` + `useCart()` following the same 4 steps above: typed context value, a provider component owning `items` state, a hook that throws if called outside `<CartProvider>`, and a consumer component that only ever calls `useCart()`.

### When to use

- Cross-cutting data needed by many, unrelated-in-the-tree components: theme, locale, current user, feature flags, cart contents.
- Data that changes relatively rarely compared to how often the app re-renders (theme toggles occasionally; it isn't like typing into an input on every keystroke).
- Building a component library or design system where consumers shouldn't need to know your internal Context object even exists.

### When NOT to use

- For state only shared between a parent and its direct children — just pass props; Context adds indirection you don't need.
- For high-frequency-changing values (mouse position, a value updating every animation frame) — every component consuming that context re-renders on every change, which gets expensive fast.
- As a replacement for server state (data from an API) — that's the **Server State Pattern** (Day 7), which needs caching/refetching logic Context doesn't provide out of the box.

---

## 2. State Reducer Pattern

### The problem

Once a piece of state has several related fields that change together through distinct "events" (add item, remove item, apply discount, clear cart), managing each field with its own `useState` and hand-written update logic scattered across handlers gets tangled and error-prone — it's easy to update one field and forget a related one.

### Bad approach (scattered useState)

```tsx
function Cart() {
  const [items, setItems] = useState<CartItem[]>([]);
  const [total, setTotal] = useState(0);

  function addItem(item: CartItem) {
    const next = [...items, item];
    setItems(next);
    setTotal(next.reduce((sum, i) => sum + i.price, 0)); // easy to forget this line elsewhere
  }

  function removeItem(id: string) {
    const next = items.filter((i) => i.id !== id);
    setItems(next);
    setTotal(next.reduce((sum, i) => sum + i.price, 0)); // duplicated recompute logic
  }

  // every new action repeats the same "update items, then recompute total" dance
}
```

### Pattern applied

`useReducer` centralizes all valid state transitions into one function, described as explicit named actions.

```tsx
type CartState = { items: CartItem[]; total: number };
type CartAction =
  | { type: "ADD_ITEM"; item: CartItem }
  | { type: "REMOVE_ITEM"; id: string }
  | { type: "CLEAR" };

function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case "ADD_ITEM": {
      const items = [...state.items, action.item];
      return { items, total: items.reduce((sum, i) => sum + i.price, 0) };
    }
    case "REMOVE_ITEM": {
      const items = state.items.filter((i) => i.id !== action.id);
      return { items, total: items.reduce((sum, i) => sum + i.price, 0) };
    }
    case "CLEAR":
      return { items: [], total: 0 };
  }
}

function Cart() {
  const [state, dispatch] = useReducer(cartReducer, { items: [], total: 0 });
  return (
    <button onClick={() => dispatch({ type: "ADD_ITEM", item: someItem })}>
      Add
    </button>
  );
}
```

Every transition goes through one function, so `items` and `total` can never drift out of sync — the reducer is the single place that knows how to keep them consistent.

### Real-world example

Shopping cart logic on any e-commerce site (Daraz, Amazon), multi-step checkout wizards, and undo/redo in editors are natural fits. Redux itself is this exact pattern (a reducer function + dispatched actions) promoted to an app-wide, external-store version — if you understand `useReducer`, you already understand Redux's core mental model.

### Practice exercise

Extend the cart reducer above with an `UPDATE_QUANTITY` action (`{ type: "UPDATE_QUANTITY"; id: string; quantity: number }`) that updates one item's quantity and recomputes `total` correctly.

### When to use

- State with multiple related fields that must update together, or many distinct "events" that can occur (more than 3–4 different kinds of updates).
- When the *next* state depends on the *previous* state in a non-trivial way (not just "replace this one value").
- When you want the state-transition logic testable in isolation (a reducer is a pure function — easy to unit test without rendering anything).

### When NOT to use

- Simple, independent fields with no shared transition logic — plain `useState` per field is clearer and has less boilerplate.
- Don't reach for `useReducer` just because "it looks more professional" — if there's only one or two simple setters, it's overkill.

---

## How the two patterns fit together

`useReducer` and the Provider Pattern aren't competitors — they combine. A `<CartProvider>` (Provider Pattern) commonly holds its state via `useReducer` (State Reducer Pattern) internally, then shares `{ state, dispatch }` through context instead of individual `setState` calls:

```tsx
const CartContext = createContext<{ state: CartState; dispatch: React.Dispatch<CartAction> } | null>(null);

export function CartProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(cartReducer, { items: [], total: 0 });
  const value = { state, dispatch };
  return <CartContext.Provider value={value}>{children}</CartContext.Provider>;
}

export function useCart() {
  const ctx = useContext(CartContext);
  if (!ctx) throw new Error("useCart must be used within a CartProvider");
  return ctx;
}
```

This is the standard way to get app-wide, moderately-complex state (a cart, a multi-step form, a wizard) without reaching for an external library like Redux.

