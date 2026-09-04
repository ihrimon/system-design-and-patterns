# React Design Patterns — Interview Q&A

Interview questions and answers, organized by day, matching [README.md](README.md)'s 10-day checklist. Add each day's questions here as you complete it.

---

## Day 1 — Component Fundamentals Patterns

### Q1. What is the Component Composition Pattern, and why does React favor it over inheritance?

**A.** Composition means building complex UI by combining small, focused components (via JSX nesting), instead of one component trying to handle every variation through props, or extending a base component through class inheritance. React's own docs state "React has a powerful composition model" and recommend composition over inheritance because:

- Small components are individually easy to understand, test, and reuse.
- New UI variants are just new JSX arrangements — you don't touch the base component's code or its prop list.
- Inheritance couples a child class tightly to its parent's implementation; composition only depends on the parent's public interface (its props/children), which is far more flexible.

### Q2. What's the difference between the `children` prop and passing a component through a regular named prop?

**A.** Both are composition mechanisms, but:

- `children` is a single, implicit slot — whatever is nested between a component's opening and closing JSX tags. Best when a component wraps *one* variable region of content (`<Button>Save</Button>`).
- A named prop (e.g. `<Layout header={<Header />} />`) gives you an explicit, named slot, and you can have several of them for different regions (header, sidebar, footer). This becomes the **Slot Pattern**, and combining multiple such slots with shared internal state becomes the **Compound Components Pattern** — both covered later in this checklist.

`children` is the right tool when there's one region of injected content; named props/slots are the right tool when there are several distinct regions.

### Q3. What is "prop drilling" and how does composition help avoid it?

**A.** Prop drilling is passing a prop down through several component layers that don't use it themselves, just to reach a deeply nested child. Composition avoids it by letting the top-level owner render the deep child directly as `children` (or via a slot prop), instead of relaying data through every layer:

```tsx
// Prop drilling
<Page user={user}>
  <Sidebar user={user}>
    <UserPanel user={user} />
  </Sidebar>
</Page>

// Composition — Page and Sidebar never need to know about `user`
<Page>
  <Sidebar>
    <UserPanel user={user} />
  </Sidebar>
</Page>
```

(For state that genuinely needs to reach many unrelated components, the **Provider Pattern** — Day 2 — is the complementary solution.)

### Q4. What is the difference between a Controlled and an Uncontrolled component?

**A.**
- **Controlled**: React state is the single source of truth. The input's `value` comes from state, and `onChange` updates that state — the DOM element always reflects React's state.
- **Uncontrolled**: The DOM itself owns the current value. React sets only an initial value via `defaultValue`/`defaultChecked`, and reads the current value on demand via a `ref`, rather than on every change.

### Q5. Why does React recommend controlled components for most forms?

**A.** Because React state as the single source of truth makes several things trivial that are awkward otherwise: live validation per keystroke, conditionally enabling/disabling submit buttons, formatting input as the user types, resetting or pre-filling fields programmatically, and keeping multiple inputs in sync with each other. All of these require reading or writing the value from *outside* the input at arbitrary times — which is exactly what controlled state gives you for free.

### Q6. What happens if you switch an input from uncontrolled to controlled (or vice versa) during its lifetime? Why does React warn about this?

**A.** React logs a warning: *"A component is changing an uncontrolled input to be controlled."* This happens if an input's `value` prop starts as `undefined` (uncontrolled) and later becomes a defined string (controlled), e.g.:

```tsx
const [value, setValue] = useState(); // undefined initially — uncontrolled
<input value={value} onChange={(e) => setValue(e.target.value)} />
```

React warns because an input can't reliably switch who "owns" its value mid-life without risking lost keystrokes or inconsistent DOM state. **Fix:** always initialize controlled state with a defined value, e.g. `useState("")` instead of `useState()`.

### Q7. Give a real reason you'd be forced to use an uncontrolled component.

**A.** `<input type="file">`. For security reasons, browsers do not allow JavaScript (and therefore React) to set the `value` of a file input programmatically — only the user can pick a file. So a file input can never be a `value`-driven controlled component; you must read `inputRef.current.files` via a `ref` instead.

### Q8. Can a single reusable Input/Checkbox component support both controlled and uncontrolled usage for consumers of your component library?

**A.** Yes — this is a distinct, more advanced pattern (**Dual-Mode Controlled/Uncontrolled API Pattern**, covered on Day 4). The component checks whether a `value` prop was passed: if it was, it operates in controlled mode (uses the prop + calls `onChange`); if not, it falls back to internal state seeded from `defaultValue`. Libraries like Radix UI and Headless UI components are built this way so consumers can choose either mode.

### Q9. Coding question: Convert this uncontrolled input into a controlled one that disables the submit button until the value is non-empty.

```tsx
function NameForm() {
  const ref = useRef<HTMLInputElement>(null);
  return (
    <form>
      <input ref={ref} defaultValue="" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

**A.**

```tsx
function NameForm() {
  const [name, setName] = useState("");
  return (
    <form>
      <input value={name} onChange={(e) => setName(e.target.value)} />
      <button type="submit" disabled={name.trim().length === 0}>
        Submit
      </button>
    </form>
  );
}
```

Key points an interviewer looks for: state initialized to `""` (not `undefined`, to avoid the controlled/uncontrolled warning), `value` bound to state, `onChange` updating state, and the `disabled` condition derived from that same state.

### Q10. Does using controlled components hurt performance in large forms? How would you address that?

**A.** Every keystroke in a controlled input triggers a state update and a re-render of that component (and its subtree, unless memoized). For a handful of fields this is invisible; for very large forms (50+ fields) it can add up. Mitigations: colocate each field's state as close to that field as possible (avoid one giant state object for the whole form so unrelated fields don't re-render together), memoize expensive child components with `React.memo`, or use a form library (React Hook Form) that keeps most fields uncontrolled internally via refs and only re-renders on submit/validation.

---

## Day 2 — State Management Patterns

### Q1. What problem does React Context solve, and when should you reach for it?

**A.** Context lets any descendant component read a value directly from an ancestor, without every intermediate component having to accept and forward it as a prop ("prop drilling"). Reach for it when data is genuinely cross-cutting — theme, locale, current user, feature flags — needed by components scattered around the tree, not just a parent and its immediate child (plain props are simpler there).

### Q2. What's the difference between just using `createContext`/`useContext` directly versus the "Provider Pattern"?

**A.** Raw Context only gives you a way to pass a value down the tree. The Provider Pattern wraps that in a dedicated `<XProvider>` component that owns the state/logic, plus a custom hook (`useAuth()`, `useTheme()`) that reads the context *and* validates it was actually provided — throwing a clear error like `"useAuth must be used within an AuthProvider"` instead of silently returning `undefined`. It also hides the raw context object from consumers entirely, so they never import it directly.

### Q3. Why should you always expose a custom hook (`useAuth`) instead of letting consumers call `useContext(AuthContext)` directly?

**A.** Three reasons: (1) **Safety** — the hook can check `if (!ctx) throw new Error(...)`, catching "used outside provider" bugs immediately at the call site instead of a silent `undefined` bug appearing later. (2) **Encapsulation** — consumers never need to know a Context object exists; you can refactor the internal implementation (even move away from Context entirely) without touching consumer code. (3) **Discoverability** — `useAuth()` documents intent far better than `useContext(AuthContext)`.

### Q4. What's a major performance pitfall of Context, and how do you fix it?

**A.** Every component that consumes a context re-renders whenever that context's value changes — even if the consumer only cares about part of it. Two common causes and fixes:
- Passing a new object literal as the `value` prop on every render (`<Ctx.Provider value={{ user, login }}>`) creates a new reference every time, forcing all consumers to re-render even if nothing meaningful changed. In React 18 and earlier the fix is to wrap the value in `useMemo` so the reference is stable unless a dependency actually changes; in React 19 with the React Compiler enabled, this is memoized automatically and manual `useMemo` is no longer needed for this case.
- One large context holding many unrelated fields means a change to any field re-renders every consumer of the whole context. Fix: split into multiple smaller, focused contexts (e.g. separate `UserContext` and `ThemeContext`) so consumers only re-render for the slice they actually use.

### Q5. When would you reach for `useReducer` instead of several `useState` calls?

**A.** When state has multiple related fields that must update together and staying in sync by hand across several `useState` setters is error-prone (e.g. cart `items` and `total`), when there are many distinct kinds of updates (more than 3–4 "events"), or when the next state depends on the previous state in a non-trivial way. If it's a couple of independent, simple fields, plain `useState` is simpler and has less boilerplate — don't reach for `useReducer` just because it "looks more professional."

### Q6. How does `useReducer` compare to Redux?

**A.** They share the exact same core idea: a pure reducer function `(state, action) => newState`, and you trigger transitions by dispatching action objects instead of calling setters directly. The difference is scope — `useReducer` manages state local to one component subtree (usually paired with Context to share it), while Redux manages a single global store outside the component tree, with middleware, devtools, and time-travel debugging built around that same reducer concept. If you're comfortable with `useReducer`, you already understand Redux's core mental model.

### Q7. Coding question: write a reducer for a shopping cart supporting `ADD_ITEM` and `REMOVE_ITEM`, keeping a running `total` in sync.

```tsx
type CartItem = { id: string; price: number };
type CartState = { items: CartItem[]; total: number };
type CartAction =
  | { type: "ADD_ITEM"; item: CartItem }
  | { type: "REMOVE_ITEM"; id: string };
```

**A.**

```tsx
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
    default:
      return state;
  }
}
```

Key points an interviewer looks for: the reducer is a pure function (no side effects, returns a new state object rather than mutating), every action recomputes `total` from `items` so the two fields can never drift out of sync, and a `default` case returning unchanged state for safety.

### Q8. Can you combine Context and `useReducer`? Why would you?

**A.** Yes — this is one of the most common patterns for app-wide state without an external library. A Provider internally holds `const [state, dispatch] = useReducer(reducer, initialState)`, then shares `{ state, dispatch }` through Context. Any descendant can then call a custom hook (e.g. `useCart()`) to read state or dispatch actions, without prop drilling and without needing Redux for moderate-complexity global state.

### Q9. What's a common bug when creating a context's `value` inline in JSX, and how do you fix it?

**A.** Writing `<AuthContext.Provider value={{ user, login, logout }}>` creates a brand-new object every single render, even if `user`, `login`, and `logout` haven't actually changed — because object literals are never `===` to a previous render's object. Every consumer re-renders as a result, regardless of whether anything meaningful changed. **Fix (pre-React 19 / no compiler):** wrap it in `useMemo(() => ({ user, login, logout }), [user])` so the reference only changes when a real dependency changes. **In React 19 with the React Compiler enabled**, the compiler detects this at build time and memoizes it for you automatically — you can write the plain object literal and get the same stable-reference behavior without touching `useMemo` yourself. It's still worth knowing the manual fix, since plenty of production codebases haven't adopted the compiler yet.

### Q10. Name two popular libraries whose public API is essentially the Provider Pattern.

**A.** React Query (`<QueryClientProvider>` + `useQueryClient()`/`useQuery()`), Redux (`<Provider store={store}>` + `useSelector()`/`useDispatch()`), NextAuth (`<SessionProvider>` + `useSession()`), and Radix/shadcn UI's `<ThemeProvider>` + `useTheme()` all follow the exact same shape: a provider component owning state, paired with a custom hook for safe, encapsulated access.

---

## Day 3 — Logic Reuse Patterns

### Q1. What is a custom hook, and what problem does it solve?

**A.** A custom hook is just a function whose name starts with `use` that calls other hooks (`useState`, `useEffect`, etc.) internally, letting you extract stateful logic out of a component so it can be reused. It solves duplication: without it, logic like "fetch this URL, track loading/error state, clean up on unmount" gets copy-pasted into every component that needs it, and a bug fix in one copy never reaches the others. With a shared `useFetch(url)` hook, every consumer gets the fix automatically.

### Q2. Does a custom hook share state between the components that use it?

**A.** No — this is a common misconception. Each call to a custom hook gets its own independent state. If `ProductList` and `OrderList` both call `useFetch(url)`, they each get their own `data`/`loading`/`error` state; nothing is shared between them. What's shared is the *logic* (the implementation), not the state itself.

### Q3. Coding question: extract the duplicated fetch logic below into a reusable `useFetch` hook.

```tsx
function ProductList() {
  const [products, setProducts] = useState<Product[]>([]);
  const [loading, setLoading] = useState(true);
  useEffect(() => {
    fetch("/api/products").then((r) => r.json()).then((data) => {
      setProducts(data);
      setLoading(false);
    });
  }, []);
  // ...
}
```

**A.**

```tsx
function useFetch<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    let isActive = true;
    setLoading(true);
    fetch(url)
      .then((r) => r.json())
      .then((json) => isActive && setData(json))
      .finally(() => isActive && setLoading(false));
    return () => { isActive = false; };
  }, [url]);

  return { data, loading };
}

function ProductList() {
  const { data: products, loading } = useFetch<Product[]>("/api/products");
  // ...
}
```

Key points an interviewer looks for: generic `<T>` so it works for any resource type, an `isActive` flag (or `AbortController`) in the cleanup function to avoid setting state after unmount, and `url` in the dependency array so the hook refetches if the URL changes.

### Q4. What is Hook Composition, and how is it different from just calling multiple hooks inside a component?

**A.** Hook Composition means building a higher-level hook by calling several lower-level hooks *inside another custom hook*, then returning the combined/derived result — e.g. `useProductState(id)` internally calling `useAuth()`, `useWishlist()`, and `useCart()` to return `{ isWishlisted, cartQuantity, canPurchase }`. The difference from calling all three hooks directly inside a component is reuse and separation of concerns: if two different components (a product detail page and a product card) both need that same combination, composing it into one hook means the derivation logic exists in exactly one place instead of being copy-pasted into every consuming component.

### Q5. What is the Hook Factory Pattern, and when would you reach for it instead of writing hooks by hand?

**A.** A hook factory is a function that *generates* a hook, parameterized by whatever varies between use cases (a URL, a storage key, an event name) — e.g. `createResourceHook<T>(endpoint)` returns a ready-to-use `useResource()` hook. Reach for it when you need the same *shape* of hook for 3+ resources that differ only in a parameter (`useUsers`, `useProducts`, `useOrders` all fetching different endpoints with identical logic). For only one or two resources, or when each one's logic genuinely diverges, a factory just adds indirection — write them by hand instead.

### Q6. In a `createApiHook<T>(url)`-style factory, what does the factory actually return, and why is that useful?

**A.** It returns a *function* (the hook itself), not the data directly — calling `createApiHook<T>(url)` produces a new `useX()` hook that, when called inside a component, internally manages its own `data`/`loading`/`error` state via `useState`/`useEffect`. This is useful because you can generate an unlimited number of independent, fully-functional hooks (one per endpoint) from a single implementation, and each generated hook still follows React's rules of hooks correctly since it's a real hook, not a plain utility function.

### Q7. What's a real risk of overusing custom hooks or hook composition?

**A.** Over-abstracting a single, one-off piece of logic into a named hook before it has a second consumer — this adds an indirection layer (you now have to jump to another file to understand what a component does) with no actual reuse benefit yet. The rule of thumb from this checklist's Day 1–3 patterns: extract only once you have (or clearly anticipate) a second consumer, not preemptively.

### Q8. How do Custom Hooks, Hook Composition, and Hook Factory relate to each other in a real codebase?

**A.** They typically layer: a Hook Factory generates many low-level, `useFetch`-style Custom Hooks consistently (e.g. one per API resource); Hook Composition then sits a level above, combining several of those (possibly factory-generated) hooks into the exact shape a specific screen or component needs. You rarely pick just one — a mature dashboard codebase usually has all three working together.

### Q9. What's the relationship between the Custom Hooks Pattern here and libraries like React Query or SWR?

**A.** React Query and SWR are productionized, battle-tested versions of the same `useFetch`-style custom hook idea, extended with caching, request deduplication, background refetching, and stale-data handling that a hand-rolled `useFetch` doesn't have. Understanding why you'd extract `useFetch` in the first place is exactly what makes adopting React Query later make sense — it's solving the same duplication problem, just with far more production-grade behavior built in. (This connects directly to the **Server State Pattern**, Day 7.)

---

## Day 4: Advanced Component API Patterns

### Q1. What is the Compound Components Pattern, and what problem does it solve?

**A.** Compound Components is a set of components that work together as a family (like `Tabs`, `Tabs.List`, `Tabs.Tab`, `Tabs.Panel`), sharing internal state through React Context instead of the parent passing that state down as props. It solves the rigidity of a config-object API: a `Tabs` component that takes a `tabs: { label, content }[]` prop can only support what that shape anticipated. Compound components let the consumer write plain JSX, so adding an icon, a badge, or a tooltip to one tab is just JSX, not a new prop on the config object.

### Q2. How does a subcomponent like `Tabs.Tab` get access to shared state without the parent passing it a prop?

**A.** The top-level component (`Tabs`) wraps its children in a Context Provider holding the shared state (e.g. `{ active, setActive }`). Each subcomponent calls a custom hook (`useTabsContext()`) that reads from that context. This is the same mechanism as the Provider Pattern from Day 2, just applied so the "consumers" are a fixed family of subcomponents instead of arbitrary parts of the app.

### Q3. Why attach subcomponents as properties on the parent (`Tabs.List = TabList`) instead of exporting them separately?

**A.** It keeps the API self-documenting and scoped: importing `Tabs` gives you `Tabs.List`, `Tabs.Tab`, and `Tabs.Panel` together, signaling they're meant to be used only inside `<Tabs>`, rather than as standalone components that happen to share a name convention. It also avoids polluting the module's export list with several tightly coupled pieces.

### Q4. What's the difference between the Compound Components Pattern and the Provider + Compound Components Pattern?

**A.** They're the same underlying mechanism (subcomponents sharing state via Context) at different levels of rigor. Plain Compound Components (like `Tabs` sharing just an `active` index) can get away with a simple Context and a small hook. Provider + Compound Components is for cases where the shared state is complex enough (multiple fields, side effects, a callback notifying a parent form) that it needs the fuller treatment from Day 2's Provider Pattern: a dedicated provider component, and a custom hook that throws a clear error if a subcomponent is used outside its provider.

### Q5. Coding question: a `SelectItem` component reads `select` from `useSelectContext()`. What happens if someone renders `<Select.Item>` outside of a `<Select>`, and how do you make the failure clear instead of confusing?

**A.**

```tsx
function useSelectContext() {
  const ctx = useContext(SelectContext);
  if (!ctx) throw new Error("Select subcomponents must be used inside <Select>");
  return ctx;
}
```

Without the `if (!ctx) throw`, `useContext` would silently return the context's default value (often `null` or `undefined`), and `SelectItem` would fail later with a cryptic error like "Cannot read properties of null" when it tries to call `select(...)`. The explicit throw fails immediately, at the exact call site, with a message that tells the developer exactly what's wrong.

### Q6. What is the Dual-Mode (Controlled/Uncontrolled) API Pattern, and how is it different from the Controlled/Uncontrolled Component Pattern from Day 1?

**A.** Day 1's Controlled and Uncontrolled patterns describe two different ways *you*, as the app developer, can use a component: pass `value` and own the state yourself, or pass `defaultValue` and let the DOM/component own it. The Dual-Mode API Pattern is about *building* a component (typically for a shared library) that supports both of those usage styles at once, so its consumers can choose either one without the library author writing two separate components.

### Q7. Coding question: explain what `useControllableState` does and why `controlledValue !== undefined` is the check used to decide the mode.

```tsx
function useControllableState<T>(controlledValue: T | undefined, defaultValue: T) {
  const [internalValue, setInternalValue] = useState(defaultValue);
  const isControlled = controlledValue !== undefined;
  const value = isControlled ? controlledValue : internalValue;
  return [value, setInternalValue] as const;
}
```

**A.** It always keeps an internal `useState` around (for the uncontrolled case), but the actual `value` it returns is either the caller-supplied `controlledValue` (if one was passed) or that internal state. The `!== undefined` check is the signal for "was a value passed in at all": if the consumer never passes the `open` prop, it's `undefined`, so the hook falls back to managing its own state; the moment a consumer passes any defined value (even `false`), the component treats it as controlled and defers to that value entirely.

### Q8. If a component using `useControllableState` is controlled, does calling the returned setter function actually change what's rendered?

**A.** No, and this is a common source of bugs when implementing this pattern: in controlled mode, `setInternalValue` still updates the internal state, but the hook returns `controlledValue` (the prop), not `internalValue`, so the render is unaffected. The component must call the consumer's `onChange`/`onOpenChange` callback so the consumer updates their own state, which flows back down as a new `controlledValue` on the next render. This is exactly how the native `<input value={...} onChange={...}>` works too.

### Q9. When would you deliberately *not* build a dual-mode component, even in a shared UI library?

**A.** When every realistic consumer needs the same mode. Supporting both adds real complexity (the `useControllableState` indirection, and the mental overhead of "which mode am I in" for every future contributor touching the component). If a component is genuinely only ever used one way in practice, hardcode that mode and add the other later if a real need for it shows up, rather than building flexibility speculatively.

---

<!-- Add Day 5 questions below as you complete Day 5 -->
