# Day 3 — Logic Reuse Patterns

Patterns covered today:

1. [Custom Hooks Pattern](#1-custom-hooks-pattern)
2. [Hook Composition Pattern](#2-hook-composition-pattern)
3. [Hook Factory Pattern](#3-hook-factory-pattern)

All three examples today use one running scenario — an **e-commerce admin dashboard** (products, orders, users) — the kind of codebase most frontend engineers actually work in, so the bad/good code is something you can map directly onto your own project.

Study method for each pattern: understand the problem → see the bad approach → see the pattern applied → real-world example → practice exercise → know when (not) to use it.

---

## 1. Custom Hooks Pattern

### The problem

Data-fetching logic (loading state, error state, the fetch itself) is close to identical across every list page in a dashboard — products, orders, users — but without a shared abstraction, that logic gets copy-pasted into every component that needs it, and a bug fix in one copy doesn't reach the others.

### Bad approach (duplicated fetch logic)

```tsx
// ProductList.tsx
function ProductList() {
  const [products, setProducts] = useState<Product[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    setLoading(true);
    fetch("/api/products")
      .then((res) => res.json())
      .then(setProducts)
      .catch((err) => setError(err.message))
      .finally(() => setLoading(false));
  }, []);

  if (loading) return <Spinner />;
  if (error) return <ErrorBanner message={error} />;
  return <ul>{products.map((p) => <li key={p.id}>{p.name}</li>)}</ul>;
}

// OrderList.tsx — nearly identical block, copy-pasted
function OrderList() {
  const [orders, setOrders] = useState<Order[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    setLoading(true);
    fetch("/api/orders")
      .then((res) => res.json())
      .then(setOrders)
      .catch((err) => setError(err.message))
      .finally(() => setLoading(false));
  }, []);

  if (loading) return <Spinner />;
  if (error) return <ErrorBanner message={error} />;
  return <ul>{orders.map((o) => <li key={o.id}>{o.total}</li>)}</ul>;
}
```

Six months later someone fixes a race condition in `ProductList`'s fetch — `OrderList` and every other copy still has the bug, because nobody remembered they all needed the same fix.

### Pattern applied

Extract the shared logic into a hook — any component that needs "fetch this URL, give me data/loading/error" reuses the same, single implementation.

```tsx
function useFetch<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    let isActive = true;
    setLoading(true);
    fetch(url)
      .then((res) => res.json())
      .then((json) => isActive && setData(json))
      .catch((err) => isActive && setError(err.message))
      .finally(() => isActive && setLoading(false));
    return () => {
      isActive = false; // avoids setting state after the component unmounts
    };
  }, [url]);

  return { data, loading, error };
}

// ProductList.tsx
function ProductList() {
  const { data: products, loading, error } = useFetch<Product[]>("/api/products");
  if (loading) return <Spinner />;
  if (error) return <ErrorBanner message={error} />;
  return <ul>{products?.map((p) => <li key={p.id}>{p.name}</li>)}</ul>;
}

// OrderList.tsx — same hook, zero duplicated logic
function OrderList() {
  const { data: orders, loading, error } = useFetch<Order[]>("/api/orders");
  if (loading) return <Spinner />;
  if (error) return <ErrorBanner message={error} />;
  return <ul>{orders?.map((o) => <li key={o.id}>{o.total}</li>)}</ul>;
}
```

Fix a bug in `useFetch` once, every consumer gets the fix automatically.

### Real-world example

The `useDebounce` hook behind Amazon's or Daraz's search-as-you-type box (wait for the user to stop typing before firing an API call), a `useLocalStorage` hook behind "remember my login" checkboxes, and `useClickOutside` behind every dropdown/menu that closes when you click elsewhere — these are the exact kind of small, reusable behaviors every production codebase accumulates as custom hooks.

### Practice exercise

Build a `useDebounce(value, delay)` hook yourself and wire it to a product search input that only calls the API 300ms after the user stops typing.

### When to use

- The same stateful logic (fetch, subscription, event listener, timer) is needed in 2+ components.
- The logic has enough moving parts (loading/error/cleanup) that inlining it everywhere is error-prone.

### When NOT to use

- One-off logic used in exactly one component — extracting it into a hook adds an indirection layer with no reuse benefit yet. Wait until you actually need it in a second place.

---

## 2. Hook Composition Pattern

### The problem

A Product Detail page in the dashboard needs several independent pieces of data — the logged-in user, whether this product is in their wishlist, and how many are in their cart. Calling all three lower-level hooks directly inside the page component mixes data-orchestration logic with rendering logic, and the same combination is needed again on the Product Card component in the listing page.

### Bad approach (page component juggles every hook itself)

```tsx
function ProductDetailPage({ productId }: { productId: string }) {
  const { user } = useAuth();
  const { items: wishlistItems } = useWishlist();
  const { items: cartItems } = useCart();

  const isWishlisted = wishlistItems.some((i) => i.productId === productId);
  const cartQuantity = cartItems.find((i) => i.productId === productId)?.quantity ?? 0;
  const canPurchase = !!user; // must be logged in to buy

  // ... same 5 lines of derivation logic will be re-written on ProductCard too
  return <ProductView isWishlisted={isWishlisted} cartQuantity={cartQuantity} canPurchase={canPurchase} />;
}
```

Every screen that needs "is this product wishlisted + how many in cart + can this user buy" re-derives the same three values from three separate hooks.

### Pattern applied

Compose the lower-level hooks into one higher-level hook that returns exactly what a "product state" consumer needs.

```tsx
function useProductState(productId: string) {
  const { user } = useAuth();
  const { items: wishlistItems } = useWishlist();
  const { items: cartItems } = useCart();

  return {
    isWishlisted: wishlistItems.some((i) => i.productId === productId),
    cartQuantity: cartItems.find((i) => i.productId === productId)?.quantity ?? 0,
    canPurchase: !!user,
  };
}

// ProductDetailPage.tsx
function ProductDetailPage({ productId }: { productId: string }) {
  const { isWishlisted, cartQuantity, canPurchase } = useProductState(productId);
  return <ProductView isWishlisted={isWishlisted} cartQuantity={cartQuantity} canPurchase={canPurchase} />;
}

// ProductCard.tsx — same derived state, zero repeated logic
function ProductCard({ productId }: { productId: string }) {
  const { isWishlisted, cartQuantity } = useProductState(productId);
  return <WishlistIcon active={isWishlisted} /* ... */ />;
}
```

### Real-world example

Admin dashboards (Stripe's, Shopify's) commonly have a `useDashboardAccess()`-style hook that composes `useUser()` + `usePermissions()` + `useFeatureFlags()` — every gated page or button calls this one hook instead of re-deriving "is this user allowed to see this" from three sources every time.

### Practice exercise

Build `useCheckoutSummary()` that composes `useCart()` (items) with a `useShipping()` hook (shipping cost) and a `useDiscount()` hook (applied coupon), returning a single `{ subtotal, shipping, discount, total }` object ready for a checkout page to render directly.

### When to use

- Multiple components need the same *combination* of data derived from 2+ hooks.
- You want a page/screen component to stay focused on rendering, not on orchestrating several data sources itself.

### When NOT to use

- If only one component ever needs the combination, composing into a named hook is premature — inline the two hook calls there directly until a second consumer shows up.

---

## 3. Hook Factory Pattern

### The problem

The same dashboard needs near-identical CRUD hooks for `users`, `products`, and `orders` — fetch a list, fetch one by id, create, update, delete. Hand-writing `useUsers()`, `useProducts()`, `useOrders()` from scratch each duplicates the same boilerplate for a different URL.

### Bad approach (hand-written, duplicated per resource)

```tsx
function useUsers() {
  const [data, setData] = useState<User[]>([]);
  useEffect(() => { fetch("/api/users").then((r) => r.json()).then(setData); }, []);
  return data;
}

function useProducts() {
  const [data, setData] = useState<Product[]>([]);
  useEffect(() => { fetch("/api/products").then((r) => r.json()).then(setData); }, []);
  return data;
}

function useOrders() {
  const [data, setData] = useState<Order[]>([]);
  useEffect(() => { fetch("/api/orders").then((r) => r.json()).then(setData); }, []);
  return data;
}
```

Three files, one line different (`"/api/users"` vs `"/api/products"` vs `"/api/orders"`) — and the next resource (`invoices`, `coupons`, ...) means writing a fourth near-copy.

### Pattern applied

A **hook factory** is a function that *generates* a hook, parameterized by whatever changes between resources (here, the endpoint).

```tsx
function createResourceHook<T>(endpoint: string) {
  return function useResource() {
    const [data, setData] = useState<T[]>([]);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
      fetch(endpoint)
        .then((r) => r.json())
        .then(setData)
        .finally(() => setLoading(false));
    }, []);

    return { data, loading };
  };
}

// One factory call per resource — no copy-pasted boilerplate
const useUsers = createResourceHook<User>("/api/users");
const useProducts = createResourceHook<Product>("/api/products");
const useOrders = createResourceHook<Order>("/api/orders");

// Usage is identical to hand-written hooks
function UserList() {
  const { data: users, loading } = useUsers();
  return loading ? <Spinner /> : <ul>{users.map((u) => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

Adding a fourth resource (`invoices`) is a single line: `const useInvoices = createResourceHook<Invoice>("/api/invoices");` — no new logic to write or maintain.

### Real-world example

At larger scale, RTK Query's `injectEndpoints` and OpenAPI-generated hook libraries are productionized versions of this same idea — a factory that turns an API schema into a full set of typed hooks automatically.

### More factory examples in this repo

The factory idea isn't limited to fetching — anything with a repeated "same shape, different parameter" need fits. Two more factory examples with the same idea:

**`createPersistedState.ts`** — generates a `useState`-like hook that's automatically backed by `localStorage`, so the value survives a page refresh:

```ts
export const createPersistedState = <T>(key: string, initialValue: T) => {
  return () => {
    const [state, setState] = useState<T>(() => {
      const storedValue = localStorage.getItem(key);
      return storedValue ? JSON.parse(storedValue) : initialValue;
    });

    useEffect(() => {
      localStorage.setItem(key, JSON.stringify(state));
    }, [state]);

    return [state, setState] as const;
  };
};

// Usage — behaves exactly like useState, but persists automatically
type CartItem = { id: number; name: string; price: number; quantity: number };
export const useCartState = createPersistedState<CartItem[]>("cart", []);

function CartWidget() {
  const [cart, setCart] = useCartState(); // survives a page refresh, no extra code
  return <span>{cart.length} items</span>;
}
```

Every dashboard has a handful of these — remembering a "dark mode" toggle, a collapsed sidebar, a saved filter — and each one is a one-line call to this same factory instead of a hand-written `useEffect` + `localStorage` pair.

**`createWebSocketHook.ts`** — generates a hook that subscribes to one real-time event over a shared WebSocket connection:

```ts
type CreateWebSocketOptions = { url?: string };

export const createWebSocketHook = <T>(
  eventName: string,
  options: CreateWebSocketOptions = {}
) => {
  const url = options.url || "wss://api.example.com/ws";
  return () => {
    const [data, setData] = useState<T | null>(null);

    useEffect(() => {
      const ws = new WebSocket(url);
      ws.onopen = () => ws.send(JSON.stringify({ subscribe: eventName }));
      ws.onmessage = (event) => setData(JSON.parse(event.data));
      return () => ws.close();
    }, [eventName]);

    return data;
  };
};

// Usage — one factory call per real-time event type
type OrderUpdate = { orderId: string; status: string };
export const useOrderStatusUpdates = createWebSocketHook<OrderUpdate>("order.status");

function OrderTracker({ orderId }: { orderId: string }) {
  const update = useOrderStatusUpdates(); // live order-status pushes, e.g. "Out for delivery"
  return <p>{update?.status ?? "Waiting for updates..."}</p>;
}
```

This is the same live-order-tracking experience you see on Daraz/food-delivery apps — a live "Order Confirmed → Preparing → Out for Delivery" status feed. Instead of hand-writing WebSocket connect/subscribe/cleanup logic separately for order updates, chat messages, and notifications, one factory call per event type produces a ready-to-use hook for each.

### Practice exercise

Extend `createResourceHook` from above with a `create(item: T)` function that `POST`s a new item to the endpoint and appends it to `data` on success — so `const { data, loading, create } = useProducts(); create(newProduct)` adds a product and updates the list without a manual refetch.

### When to use

- You need the same *shape* of hook for 3+ resources/endpoints/keys, differing only in a parameter (URL, storage key, event name).
- You're building a small internal library of hooks that should stay consistent in behavior across every resource.

### When NOT to use

- Only 1–2 resources, or each one's logic genuinely diverges (different error handling, different caching rules) — forcing them through one factory then adds special-case branches inside it, which is worse than just writing two separate hooks.

---

## How the three patterns fit together

`useFetch` (Custom Hook) is the low-level building block. `createResourceHook` (Hook Factory) is a machine that stamps out many `useFetch`-like hooks consistently. `useProductState` (Hook Composition) sits one level up, combining multiple already-existing hooks (which may themselves come from a factory) into the exact shape one screen needs. In a real dashboard codebase you'll typically have all three layered: factory-generated resource hooks at the bottom, composed into page-specific hooks above them.


