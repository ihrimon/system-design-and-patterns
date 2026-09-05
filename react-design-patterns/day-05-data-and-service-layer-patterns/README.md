# Day 5: Data & Service Layer Patterns

Patterns covered today:

1. [Service Layer Pattern](#1-service-layer-pattern)
2. [Repository Pattern](#2-repository-pattern)
3. [Facade Pattern](#3-facade-pattern)
4. [Dependency Injection Pattern](#4-dependency-injection-pattern)

Same running scenario as Days 3–4: an e-commerce admin dashboard. Today's focus shifts away from UI components and onto how that dashboard talks to the backend — where API calls live, how business logic around that data is organized, and how components get access to both without being welded to one concrete implementation.

Study method for each pattern: understand the problem, see the bad approach, see the pattern applied, real-world example, practice exercise, know when (not) to use it.

---

## 1. Service Layer Pattern

### The problem

A component that needs data reaches for `fetch` directly, inline, in a `useEffect`. The URL, headers, error handling, and response shape all live inside the component. The next component that needs similar data copies the same `fetch` call, slightly differently. A year later, the API's base URL, auth header, or error format changes, and that change has to be found and repeated in every component that ever called `fetch`.

### Bad approach (fetch scattered across components)

```tsx
function OrderList() {
  const [orders, setOrders] = useState<Order[]>([]);

  useEffect(() => {
    fetch('https://api.example.com/v1/orders', {
      headers: { Authorization: `Bearer ${localStorage.getItem('token')}` },
    })
      .then((res) => res.json())
      .then((data) => setOrders(data));
  }, []);

  // ...
}

function OrderDetail({ id }: { id: string }) {
  const [order, setOrder] = useState<Order | null>(null);

  useEffect(() => {
    // Same base URL, same header, copy-pasted with one path segment changed
    fetch(`https://api.example.com/v1/orders/${id}`, {
      headers: { Authorization: `Bearer ${localStorage.getItem('token')}` },
    })
      .then((res) => res.json())
      .then((data) => setOrder(data));
  }, [id]);

  // ...
}
```

Every component owns a private copy of "how to talk to the orders API." Nothing enforces that the copies stay in sync.

### Pattern applied

Pull all network calls for one resource into a dedicated service module. Components call methods on the service; they never see a URL, a header, or a `fetch` call.

```tsx
// services/orderService.ts
const BASE_URL = 'https://api.example.com/v1';

function authHeaders() {
  return { Authorization: `Bearer ${localStorage.getItem('token')}` };
}

export const orderService = {
  async list(): Promise<Order[]> {
    const res = await fetch(`${BASE_URL}/orders`, { headers: authHeaders() });
    if (!res.ok) throw new Error(`Failed to load orders: ${res.status}`);
    return res.json();
  },

  async getById(id: string): Promise<Order> {
    const res = await fetch(`${BASE_URL}/orders/${id}`, {
      headers: authHeaders(),
    });
    if (!res.ok) throw new Error(`Failed to load order ${id}: ${res.status}`);
    return res.json();
  },

  async cancel(id: string): Promise<void> {
    const res = await fetch(`${BASE_URL}/orders/${id}/cancel`, {
      method: 'POST',
      headers: authHeaders(),
    });
    if (!res.ok) throw new Error(`Failed to cancel order ${id}: ${res.status}`);
  },
};

// Components only ever call the service
function OrderList() {
  const [orders, setOrders] = useState<Order[]>([]);
  useEffect(() => {
    orderService.list().then(setOrders);
  }, []);
  // ...
}

function OrderDetail({ id }: { id: string }) {
  const [order, setOrder] = useState<Order | null>(null);
  useEffect(() => {
    orderService.getById(id).then(setOrder);
  }, [id]);
  // ...
}
```

If the auth scheme changes, or the base URL moves, or every request needs a new retry header, there is exactly one file to edit: `orderService.ts`.

### Real-world example

Most production frontends that don't use a generated API client (like an OpenAPI-generated SDK) hand-roll a service layer exactly like this, one module per resource (`userService`, `productService`, `orderService`). Tools like `axios` instances with a shared config, or a generated tRPC/OpenAPI client, are essentially this same pattern with the boilerplate automated away.

### Practice exercise

Extract a `productService` with `list()`, `getById(id)`, and `updateStock(id, quantity)` methods for the dashboard's product catalog page, following the same shape as `orderService` above.

### When to use

- Any app that calls a backend from more than one component. Centralizing the calls means one place to change base URLs, auth, retries, and response parsing.

### When NOT to use

- A tiny prototype with one API call in one component. Introducing a `services/` folder for a single `fetch` is premature structure; inline it and extract once a second consumer shows up.

---

## 2. Repository Pattern

### The problem

`orderService` above is solid for "talk to this one HTTP API," but it hard-codes the assumption that data always comes from that specific REST endpoint, shaped exactly as that endpoint returns it. Swapping the backend (REST to GraphQL), adding an offline cache, or writing a test that doesn't hit the network all require touching the service's internals or duplicating it.

### Pattern applied

Define an interface that describes *what data operations exist*, independent of *how* they're fulfilled. The service layer becomes one implementation of that interface; a component only ever depends on the interface.

```tsx
// repositories/orderRepository.ts
export interface OrderRepository {
  list(): Promise<Order[]>;
  getById(id: string): Promise<Order>;
  cancel(id: string): Promise<void>;
}

// One implementation: talks to the real REST API
export const restOrderRepository: OrderRepository = {
  list: () => orderService.list(),
  getById: (id) => orderService.getById(id),
  cancel: (id) => orderService.cancel(id),
};

// A second implementation: in-memory, for tests and Storybook
export function createInMemoryOrderRepository(
  seed: Order[] = [],
): OrderRepository {
  let orders = [...seed];
  return {
    async list() {
      return orders;
    },
    async getById(id) {
      const order = orders.find((o) => o.id === id);
      if (!order) throw new Error(`Order ${id} not found`);
      return order;
    },
    async cancel(id) {
      orders = orders.map((o) =>
        o.id === id ? { ...o, status: 'cancelled' } : o,
      );
    },
  };
}

// Components depend on the interface, not on which implementation is active
function OrderList({ repository }: { repository: OrderRepository }) {
  const [orders, setOrders] = useState<Order[]>([]);
  useEffect(() => {
    repository.list().then(setOrders);
  }, [repository]);
  // ...
}
```

A unit test for `OrderList` passes `createInMemoryOrderRepository([...])` and asserts on rendered output, with zero network mocking. Production code passes `restOrderRepository`. The component's logic never changes.

### Real-world example

This is precisely how Angular's `HttpClient`-backed data services are conventionally structured, and it's the mental model behind ORMs like Prisma or TypeORM: the application code calls `userRepository.findById(id)`, unaware (and uninterested in) whether that's backed by Postgres, SQLite, or an in-memory test double.

### Practice exercise

Define a `ProductRepository` interface with `list()` and `updateStock(id, quantity)`, then write both a `restProductRepository` (delegating to the `productService` from the previous exercise) and an `createInMemoryProductRepository(seed)` for tests.

### When to use

- The data source might change (REST today, GraphQL later), or you want components/hooks that are unit-testable without mocking `fetch`/network calls.

### When NOT to use

- A small app with one data source that will realistically never change, and no test suite that would benefit from swapping it. The interface plus two implementations is real overhead — pay it only when you actually need to swap what's behind it.

---

## 3. Facade Pattern

### The problem

A "place an order" action in the dashboard isn't one API call — it means checking stock via `productService`, reserving inventory, calling `orderService.create(...)`, then calling `notificationService.send(...)` to alert the warehouse. If every component that can trigger a checkout re-implements that whole sequence (in what order, with what error handling if step 2 fails after step 1 succeeded), the business rule "how a checkout actually happens" lives in N different components instead of one place.

### Pattern applied

Wrap the multi-step, multi-service sequence behind one simple function (a facade) that hides which underlying services it coordinates and in what order.

```tsx
// facades/checkoutFacade.ts
export const checkoutFacade = {
  async placeOrder(cart: CartItem[]): Promise<Order> {
    for (const item of cart) {
      const product = await productService.getById(item.productId);
      if (product.stock < item.quantity) {
        throw new Error(`${product.name} is out of stock`);
      }
    }

    const order = await orderService.create({ items: cart });

    await Promise.all(
      cart.map((item) =>
        productService.updateStock(
          item.productId,
          -item.quantity, // decrement
        ),
      ),
    );

    await notificationService.send({
      type: 'new-order',
      orderId: order.id,
    });

    return order;
  },
};

// Any component that needs "place an order" calls one method
function CheckoutButton({ cart }: { cart: CartItem[] }) {
  const [isPlacing, setIsPlacing] = useState(false);

  async function handleCheckout() {
    setIsPlacing(true);
    try {
      await checkoutFacade.placeOrder(cart);
    } finally {
      setIsPlacing(false);
    }
  }

  return (
    <button onClick={handleCheckout} disabled={isPlacing}>
      {isPlacing ? 'Placing order...' : 'Place order'}
    </button>
  );
}
```

`CheckoutButton` (and any other place that can trigger a checkout — a "reorder" button on a past order, a bulk-order admin tool) never needs to know that placing an order touches three separate services in a specific sequence.

### Real-world example

Payment SDKs like Stripe's `confirmPayment` are facades: internally they may coordinate 3D Secure redirects, tokenization, and multiple API round-trips, but the calling code sees one function call. Firebase's `signInWithPopup` similarly hides OAuth-flow orchestration behind one method.

### Practice exercise

Build a `returnFacade.processReturn(orderId, itemIds)` that: calls `orderService.getById`, validates the items belong to that order, calls a `refundService.issueRefund(...)`, then calls `productService.updateStock(...)` to restock the returned items. Callers should only ever call `returnFacade.processReturn(...)`.

### When to use

- A business operation spans multiple services/repositories with rules about ordering, error recovery, or rollback. A facade gives that operation exactly one home instead of letting it be re-derived (and drift) in every component that triggers it.

### When NOT to use

- A single service call with no coordination logic. Wrapping `orderService.getById(id)` in a facade that does nothing but call it adds a layer with no behavior — call the service directly.

---

## 4. Dependency Injection Pattern

### The problem

`checkoutFacade` above imports `productService`, `orderService`, and `notificationService` directly at the top of the file. That hard-wires it to one concrete set of implementations: a component can't render `CheckoutButton` in a test or a Storybook story with fake services, and there's no single place to swap, say, `restOrderRepository` for `createInMemoryOrderRepository` without editing every file that imports the real one.

### Bad approach (hard-coded imports)

```tsx
import { restOrderRepository } from '../repositories/orderRepository';
import { restProductRepository } from '../repositories/productRepository';

// This facade can never be tested or reused with different implementations
export const checkoutFacade = {
  async placeOrder(cart: CartItem[]) {
    const product = await restProductRepository.getById(cart[0].productId);
    // ...
  },
};
```

### Pattern applied

Provide the dependencies from outside (via React Context, building on Day 2's Provider Pattern) instead of importing concrete implementations inside the modules that use them. Components and facades receive their dependencies; they never construct or import them directly.

```tsx
// services/ServiceProvider.tsx
type Services = {
  orderRepository: OrderRepository;
  productRepository: ProductRepository;
  notificationService: NotificationService;
};

const ServiceContext = createContext<Services | null>(null);

export function ServiceProvider({
  services,
  children,
}: {
  services: Services;
  children: React.ReactNode;
}) {
  return (
    <ServiceContext.Provider value={services}>
      {children}
    </ServiceContext.Provider>
  );
}

export function useServices() {
  const ctx = useContext(ServiceContext);
  if (!ctx) throw new Error('useServices must be used within a ServiceProvider');
  return ctx;
}

// The facade takes its dependencies as arguments instead of importing them
function createCheckoutFacade(services: Services) {
  return {
    async placeOrder(cart: CartItem[]): Promise<Order> {
      for (const item of cart) {
        const product = await services.productRepository.getById(
          item.productId,
        );
        if (product.stock < item.quantity) {
          throw new Error(`${product.name} is out of stock`);
        }
      }
      const order = await services.orderRepository.create({ items: cart });
      await services.notificationService.send({
        type: 'new-order',
        orderId: order.id,
      });
      return order;
    },
  };
}

// Components ask for whatever they need through the provider
function CheckoutButton({ cart }: { cart: CartItem[] }) {
  const services = useServices();
  const checkoutFacade = useMemo(() => createCheckoutFacade(services), [services]);

  async function handleCheckout() {
    await checkoutFacade.placeOrder(cart);
  }
  return <button onClick={handleCheckout}>Place order</button>;
}

// Production: real REST-backed services
<ServiceProvider
  services={{
    orderRepository: restOrderRepository,
    productRepository: restProductRepository,
    notificationService: restNotificationService,
  }}
>
  <App />
</ServiceProvider>;

// Tests / Storybook: fakes, no network involved
<ServiceProvider
  services={{
    orderRepository: createInMemoryOrderRepository(),
    productRepository: createInMemoryProductRepository([{ id: '1', stock: 0, name: 'Mug' }]),
    notificationService: { send: async () => {} },
  }}
>
  <CheckoutButton cart={[{ productId: '1', quantity: 1 }]} />
</ServiceProvider>;
```

`CheckoutButton` and `createCheckoutFacade` never know or care whether they're wired to real REST repositories or in-memory fakes. Only the one `<ServiceProvider>` at the composition root decides that.

### Real-world example

A `ServiceProvider` supplying an `ApiClient` and a `Logger` through context, read via a `useService()` hook (mirroring `useServices()` above), is a common way to structure a real app: swapping a fetch-based `ApiClient` for an axios-based one — or a test double — never touches component code. React Query's `QueryClientProvider` and Redux's `<Provider store={store}>` are the same idea at framework scale: inject the one shared instance from the root, read it everywhere via a hook.

### Practice exercise

Add a `refundService` to the `Services` type and `ServiceProvider`, then rewrite the `returnFacade` from the Facade Pattern exercise so it takes `services: Services` as a parameter (via `createReturnFacade(services)`) instead of importing `refundService`/`orderRepository` directly.

### When to use

- Code (facades, complex components) that coordinates multiple services/repositories and needs to be swappable or testable without touching its internals — the same motivation as the Provider Pattern (Day 2), applied specifically to service dependencies rather than UI state.

### When NOT to use

- A leaf component or a one-off utility with a single, stable dependency that will never need to be swapped or faked in a test. Threading it through a provider "for consistency" when nothing ever needs a different implementation is unnecessary indirection.

---

## How the four patterns fit together

Service Layer (Pattern 1) is the foundation: pull network calls out of components into dedicated modules. Repository (Pattern 2) sits on top of that, defining an interface so the *shape* of data access is decoupled from *how* it's fulfilled (REST today, something else tomorrow, an in-memory fake in tests). Facade (Pattern 3) is orthogonal — it composes multiple services/repositories into one coherent business operation, hiding the coordination sequence from callers. Dependency Injection (Pattern 4) is what makes the other three swappable in practice: instead of every facade and component importing concrete repositories/services directly, a `ServiceProvider` at the root supplies them, so tests, Storybook, and production can each wire up a different set without touching any consuming code.
