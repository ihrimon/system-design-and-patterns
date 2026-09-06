# Day 6: Application Architecture Patterns

Patterns covered today:

1. [Feature-Based Architecture Pattern](#1-feature-based-architecture-pattern)
2. [Atomic Design Pattern](#2-atomic-design-pattern)
3. [Container/Presentational Pattern](#3-containerpresentational-pattern)

Same running scenario as Days 3–5: an e-commerce admin dashboard. Days 1–5 looked at individual components, hooks, and the data layer in isolation. Today's focus is one level up: how files and folders across the *whole app* are organized so it stays navigable as it grows past a handful of screens.

Study method for each pattern: understand the problem, see the bad approach, see the pattern applied, real-world example, practice exercise, know when (not) to use it.

---

## 1. Feature-Based Architecture Pattern

### The problem

A new React project usually starts with folders organized by *file type*: one `components/` folder, one `hooks/` folder, one `services/` folder. It looks tidy at first. But once the dashboard has Orders, Products, and Users screens, each of those folders fills up with files from every feature mixed together, and touching "how orders work" means jumping between `components/OrderList.tsx`, `components/OrderDetail.tsx`, `hooks/useOrders.ts`, `services/orderService.ts`, and `types/order.ts` — five different folders for one feature.

### Bad approach (organized by file type)

```
src/
  components/
    OrderList.tsx
    OrderDetail.tsx
    ProductList.tsx
    ProductCard.tsx
    UserList.tsx
  hooks/
    useOrders.ts
    useProducts.ts
    useUsers.ts
  services/
    orderService.ts
    productService.ts
    userService.ts
  types/
    order.ts
    product.ts
    user.ts
```

Deleting the entire Orders feature (a real thing that happens during a rewrite or a deprecation) means hunting through all four folders for every file whose name happens to contain "Order," hoping nothing is missed.

### Pattern applied

Group files by *feature/domain* first, and only by file type inside each feature folder.

```
src/
  features/
    orders/
      OrderList.tsx
      OrderDetail.tsx
      useOrders.ts
      orderService.ts
      order.types.ts
    products/
      ProductList.tsx
      ProductCard.tsx
      useProducts.ts
      productService.ts
      product.types.ts
    users/
      UserList.tsx
      useUsers.ts
      userService.ts
  components/        # only truly shared, cross-feature UI (Button, Modal, Tabs)
  services/
    apiClient.ts     # the one shared piece from Day 5's Service Layer
```

Everything needed to understand or delete the Orders feature now lives in one folder. `components/` and the shared `services/apiClient.ts` are deliberately kept thin — they hold only what's genuinely reused across multiple features, not a dumping ground for everything.

### Real-world example

This is the structure recommended by most large-scale React/Next.js style guides today (including Bulletproof React, a widely referenced open-source architecture guide), and it's how most mature codebases using Nx or Turborepo-style monorepos organize their `apps/` and `libs/` — one folder per business domain, not per technical layer.

### Practice exercise

Reorganize a hypothetical `src/components/CartIcon.tsx`, `src/components/CartDrawer.tsx`, `src/hooks/useCart.ts`, and `src/services/cartService.ts` into a single `src/features/cart/` folder, and decide which (if any) of those files would stay in a shared top-level folder instead.

### When to use

- Any app expected to grow past 3–4 screens/domains. The cost of "which folder is this file in" should scale with the number of features, not the number of technical categories.

### When NOT to use

- A very small app (a handful of components total) where file-type folders are still easy to scan. Introducing a `features/` split for three files total is structure without a problem to solve yet.

---

## 2. Atomic Design Pattern

### The problem

Inside a single feature's UI, or in the shared `components/` folder, it's easy to end up with components of wildly different sizes sitting at the same level: a raw `Button` next to a full `OrderTable` next to an entire `OrderDetailPage`. Nothing in the folder signals which components are safe, generic building blocks versus which ones already encode business-specific layout and logic — so a change meant for "make all buttons rounder" risks touching a component that isn't really a generic button anymore.

### Pattern applied

Organize UI components into explicit layers of increasing complexity, borrowed from Brad Frost's Atomic Design methodology: atoms → molecules → organisms → templates/pages.

```
src/components/
  atoms/
    Button.tsx        # one HTML element, no business meaning
    Input.tsx
    Badge.tsx
  molecules/
    SearchBar.tsx      # Input + Button composed together
    PriceTag.tsx        # Badge + formatted currency logic
  organisms/
    OrderTableRow.tsx    # several molecules/atoms + real Order data
    OrderTable.tsx
  templates/
    DashboardLayout.tsx   # page skeleton: header + sidebar + content slot
```

```tsx
// atoms/Button.tsx — no knowledge of "orders" exists here
function Button({ children, ...props }: ButtonProps) {
  return <button className="btn" {...props}>{children}</button>;
}

// molecules/PriceTag.tsx — composes an atom, still no business logic
function PriceTag({ amount }: { amount: number }) {
  return <Badge>{formatCurrency(amount)}</Badge>;
}

// organisms/OrderTableRow.tsx — this is where real domain data enters
function OrderTableRow({ order }: { order: Order }) {
  return (
    <tr>
      <td>{order.id}</td>
      <td><PriceTag amount={order.total} /></td>
      <td><Button onClick={() => cancelOrder(order.id)}>Cancel</Button></td>
    </tr>
  );
}
```

A change to `atoms/Button.tsx` is guaranteed to be safe everywhere (it never assumes anything about orders, products, or users). A change to `organisms/OrderTableRow.tsx` is expected to be order-specific.

### Real-world example

Design systems like Google's Material UI, Shopify's Polaris, and IBM's Carbon Design System are documented and structured using almost exactly this atoms/molecules/organisms vocabulary, which is also why designers and engineers can use the same shared language when discussing a component library.

### Practice exercise

Classify these five components from a dashboard into atoms/molecules/organisms: `Avatar`, `UserMenu` (Avatar + dropdown), `Checkbox`, `BulkActionBar` (several Checkboxes' selection state + action buttons), `Tooltip`.

### When to use

- A shared component library or design system, especially one used by more than one team, where a consistent vocabulary for "how generic is this component" prevents business logic from leaking into supposedly reusable pieces.

### When NOT to use

- A small app with a handful of one-off components and no shared design system ambitions. Forcing every component into an atom/molecule/organism bucket when there are only 8 components total is bureaucracy without payoff — a flat `components/` folder is fine until the count (or the team) grows.

---

## 3. Container/Presentational Pattern

### The problem

An `OrderTable` component that both fetches data (`useEffect` + `fetch`) *and* renders the table is hard to reuse and hard to test. Reusing the same table layout with different data (search results, a filtered subset) means either duplicating the whole component or bolting on more props to control its fetching behavior. Testing the rendering logic alone means mocking the network every time, even though the test has nothing to do with the fetch.

### Bad approach (data-fetching and rendering mixed together)

```tsx
function OrderTable() {
  const [orders, setOrders] = useState<Order[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    orderService.list().then((data) => {
      setOrders(data);
      setLoading(false);
    });
  }, []);

  if (loading) return <Spinner />;

  return (
    <table>
      {orders.map((order) => (
        <OrderTableRow key={order.id} order={order} />
      ))}
    </table>
  );
}
```

There is no way to render this table with, say, a `<SearchResultsPage>`'s already-fetched orders without editing `OrderTable` itself.

### Pattern applied

Split into a **container** (owns data/state, no markup of its own) and a **presentational** component (pure rendering, no data fetching, driven entirely by props).

```tsx
// Presentational — pure function of its props, trivially testable
function OrderTableView({
  orders,
  loading,
}: {
  orders: Order[];
  loading: boolean;
}) {
  if (loading) return <Spinner />;
  return (
    <table>
      {orders.map((order) => (
        <OrderTableRow key={order.id} order={order} />
      ))}
    </table>
  );
}

// Container — owns the data, renders the presentational component
function OrderTableContainer() {
  const [orders, setOrders] = useState<Order[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    orderService.list().then((data) => {
      setOrders(data);
      setLoading(false);
    });
  }, []);

  return <OrderTableView orders={orders} loading={loading} />;
}

// Now the same view works with any data source
function SearchResultsPage({ searchResults }: { searchResults: Order[] }) {
  return <OrderTableView orders={searchResults} loading={false} />;
}
```

`OrderTableView` can be unit-tested (or shown in Storybook) with a hardcoded `orders` array and zero network mocking. `OrderTableContainer` can be tested separately for its data-fetching behavior, without asserting on rendered markup.

### Real-world example

This is precisely the split that custom hooks made mostly unnecessary in modern React (a `useOrders()` hook from Day 3 often replaces the container entirely), which is why the pattern is now described as slightly dated by some — but it's still exactly the mental model behind Storybook-driven development, where every story renders a presentational component directly with hardcoded props, never through a real data-fetching container.

### Practice exercise

Take the `Users` component from Day 5's Dependency Injection example (the one calling `useService()` and rendering a list) and split it into a `UserListView` (props: `users`, `loading`) and a `UserListContainer` (owns `useService()` and the fetch).

### When to use

- A component's rendering logic needs to be tested or previewed (Storybook) independent of its real data source, or the same view needs to be driven by more than one data source (a live fetch, search results, a cached value).

### When NOT to use

- In most modern React code, prefer extracting a custom hook (Day 3) over a full Container component — `const { orders, loading } = useOrders()` gives the same separation with less boilerplate (no extra component, no extra layer of props-drilling for the container's own children). Reach for an actual Container component mainly when the "container" needs to render server-side, wrap multiple presentational children, or when working in a codebase that already follows this pattern.

---

## How the three patterns fit together

Feature-Based Architecture answers "where do files live" at the scale of the whole app — one folder per business domain. Atomic Design answers a narrower question inside the shared, cross-feature parts of that structure — how UI building blocks are layered by complexity so business logic doesn't creep into generic components. Container/Presentational answers a different question again, at the level of a single component — how data-fetching is separated from rendering. A mature dashboard typically uses all three at once: `features/orders/` (feature-based) contains an `OrderTable` organism (atomic design) that's split into a view and a container, or more commonly today, a view plus a `useOrders()` hook (container/presentational, hook-flavored).
