# Day 4: Advanced Component API Patterns

Patterns covered today:

1. [Compound Components Pattern](#1-compound-components-pattern)
2. [Provider + Compound Components Pattern](#2-provider--compound-components-pattern)
3. [Dual-Mode (Controlled/Uncontrolled) API Pattern](#3-dual-mode-controlleduncontrolled-api-pattern)

Same running scenario as Day 3: an e-commerce admin dashboard, this time focused on the reusable UI components (Tabs, Select, Modal) that every dashboard screen shares.

Study method for each pattern: understand the problem, see the bad approach, see the pattern applied, real-world example, practice exercise, know when (not) to use it.

---

## 1. Compound Components Pattern

### The problem

A `Tabs` component that takes a `tabs` prop (an array of `{ label, content }`) looks simple at first, but every customization request turns into a new prop: an icon next to one tab's label, a badge on another, a disabled state on a third. The prop shape keeps growing to cover cases it was never designed for.

### Bad approach (config-object API)

```tsx
type TabConfig = {
  label: string;
  content: React.ReactNode;
  icon?: string;
  disabled?: boolean;
};

function Tabs({ tabs }: { tabs: TabConfig[] }) {
  const [active, setActive] = useState(0);
  return (
    <div>
      <div className='tab-list'>
        {tabs.map((tab, i) => (
          <button
            key={tab.label}
            disabled={tab.disabled}
            onClick={() => setActive(i)}
            className={active === i ? 'active' : ''}
          >
            {tab.icon && <img src={tab.icon} />}
            {tab.label}
          </button>
        ))}
      </div>
      <div className='tab-panel'>{tabs[active].content}</div>
    </div>
  );
}
```

Every new visual variation (a tooltip on a tab, a close button, a badge count) means editing `TabConfig` and the render logic inside `Tabs` again.

### Pattern applied

Split `Tabs` into cooperating subcomponents that share state through React Context, so the consumer writes plain JSX instead of a config object.

```tsx
type TabsContextValue = { active: number; setActive: (i: number) => void };
const TabsContext = createContext<TabsContextValue | null>(null);

function useTabsContext() {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error('Tabs subcomponents must be used inside <Tabs>');
  return ctx;
}

function Tabs({
  children,
  defaultIndex = 0,
}: {
  children: React.ReactNode;
  defaultIndex?: number;
}) {
  const [active, setActive] = useState(defaultIndex);
  return (
    <TabsContext.Provider value={{ active, setActive }}>
      {children}
    </TabsContext.Provider>
  );
}

function TabList({ children }: { children: React.ReactNode }) {
  return <div className='tab-list'>{children}</div>;
}

function Tab({
  index,
  children,
}: {
  index: number;
  children: React.ReactNode;
}) {
  const { active, setActive } = useTabsContext();
  return (
    <button
      onClick={() => setActive(index)}
      className={active === index ? 'active' : ''}
    >
      {children}
    </button>
  );
}

function TabPanel({
  index,
  children,
}: {
  index: number;
  children: React.ReactNode;
}) {
  const { active } = useTabsContext();
  return active === index ? <div className='tab-panel'>{children}</div> : null;
}

Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

// Usage, plain JSX instead of a config array
<Tabs>
  <Tabs.List>
    <Tabs.Tab index={0}>
      <img src='products.svg' /> Products
    </Tabs.Tab>
    <Tabs.Tab index={1}>Orders (12)</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel index={0}>
    <ProductTable />
  </Tabs.Panel>
  <Tabs.Panel index={1}>
    <OrderTable />
  </Tabs.Panel>
</Tabs>;
```

A tab with an icon, a badge, or a tooltip is just JSX inside `Tabs.Tab`. `Tabs` itself never changes.

### Real-world example

Radix UI's, Headless UI's, and Reach UI's Tabs, Accordion, Menu, and Select components are all built as compound components. In a dashboard, the collapsible filter sections in a product listing sidebar (an accordion of "Category," "Price," "Brand" filters) is a textbook use case.

### Practice exercise

Build an `Accordion` compound component: `Accordion`, `Accordion.Item`, `Accordion.Header`, `Accordion.Body`, where clicking a header toggles that item's open state, sharing state through context the same way `Tabs` does above.

### When to use

- Related UI pieces (tabs, accordions, menus, selects) that need to coordinate through shared internal state, while letting the consumer freely arrange the JSX and add per-item customization.

### When NOT to use

- A simple, single-purpose component with no internal coordination between parts. A compound component adds a Context and multiple subcomponents for a problem plain props already solve.

---

## 2. Provider + Compound Components Pattern

### The problem

`Tabs` above only shares one small piece of state (`active`). A `Select` (dropdown) component needs more: which value is selected, whether the menu is open, how to notify a parent form when the value changes, and keyboard navigation state. Passing all of that through props to every subcomponent by hand gets unmanageable fast, and there's no safety net if a subcomponent is used outside `<Select>`.

### Pattern applied

Combine the Provider Pattern (Day 2, a dedicated provider owning state plus a custom hook that throws if misused) with the Compound Components structure from above.

```tsx
type SelectContextValue = {
  value: string | null;
  isOpen: boolean;
  select: (value: string) => void;
  toggle: () => void;
};

const SelectContext = createContext<SelectContextValue | null>(null);

function useSelectContext() {
  const ctx = useContext(SelectContext);
  if (!ctx)
    throw new Error('Select subcomponents must be used inside <Select>');
  return ctx;
}

function Select({
  children,
  onChange,
}: {
  children: React.ReactNode;
  onChange?: (value: string) => void;
}) {
  const [value, setValue] = useState<string | null>(null);
  const [isOpen, setIsOpen] = useState(false);

  const select = (v: string) => {
    setValue(v);
    setIsOpen(false);
    onChange?.(v);
  };
  const toggle = () => setIsOpen((prev) => !prev);

  return (
    <SelectContext.Provider value={{ value, isOpen, select, toggle }}>
      <div className='select'>{children}</div>
    </SelectContext.Provider>
  );
}

function SelectTrigger() {
  const { value, toggle } = useSelectContext();
  return <button onClick={toggle}>{value ?? 'Choose an option'}</button>;
}

function SelectContent({ children }: { children: React.ReactNode }) {
  const { isOpen } = useSelectContext();
  return isOpen ? <ul className='select-content'>{children}</ul> : null;
}

function SelectItem({
  value,
  children,
}: {
  value: string;
  children: React.ReactNode;
}) {
  const { select } = useSelectContext();
  return <li onClick={() => select(value)}>{children}</li>;
}

Select.Trigger = SelectTrigger;
Select.Content = SelectContent;
Select.Item = SelectItem;

// Usage
<Select onChange={(v) => console.log('selected:', v)}>
  <Select.Trigger />
  <Select.Content>
    <Select.Item value='pending'>Pending</Select.Item>
    <Select.Item value='shipped'>Shipped</Select.Item>
    <Select.Item value='delivered'>Delivered</Select.Item>
  </Select.Content>
</Select>;
```

`SelectTrigger`, `SelectContent`, and `SelectItem` never receive props for `value` or `isOpen` directly. They all read from the same provider through `useSelectContext()`, exactly like `useAuth()` or `useCart()` in Day 2, except the consumer of this provider is a fixed family of subcomponents instead of arbitrary app code.

### Real-world example

This is precisely how shadcn/ui's `Select`, `DropdownMenu`, and `Dialog` components are structured internally (they wrap Radix UI primitives, which use this exact provider-plus-compound-components shape). An order-status filter dropdown in a dashboard, built this way, can grow to support search, multi-select, or grouped options later without changing how consumers write the JSX.

### Practice exercise

Extend the `Select` above with a `SelectItem` that supports a `disabled` prop (clicking it should not call `select`), and add keyboard support: pressing `Escape` while `isOpen` is true should close the menu (add the listener inside `Select` using `useEffect`).

### When to use

- A compound component whose internal state is complex enough (multiple fields, side effects, notifying a parent through a callback) that it needs the same rigor as Day 2's Provider Pattern: a dedicated provider, a custom hook, and a clear error if misused.

### When NOT to use

- If the shared state really is just one or two simple values (like `Tabs`'s `active` index), plain Context without the extra "custom hook that throws" ceremony is enough. Reach for the fuller Provider Pattern only once the state and its rules of use actually earn it.

---

## 3. Dual-Mode (Controlled/Uncontrolled) API Pattern

### The problem

A `Modal` component in a shared UI library has two kinds of consumers. Some pages want full control (open the modal from a "Save changes" button, close it after an API call succeeds). Other pages just want a simple "click to open" modal with no external logic. Building two separate components (`ControlledModal`, `Modal`) duplicates the rendering logic; forcing every consumer into controlled mode makes the simple case unnecessarily verbose.

### Bad approach (controlled-only, forced on every consumer)

```tsx
function Modal({
  open,
  onOpenChange,
  children,
}: {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  children: React.ReactNode;
}) {
  if (!open) return null;
  return (
    <div className='modal'>
      {children}
      <button onClick={() => onOpenChange(false)}>Close</button>
    </div>
  );
}

// Every consumer is forced to own state, even for a trivial "click to open" case
function SimpleInfoModal() {
  const [open, setOpen] = useState(false);
  return (
    <>
      <button onClick={() => setOpen(true)}>Info</button>
      <Modal open={open} onOpenChange={setOpen}>
        <p>This dashboard updates every 5 minutes.</p>
      </Modal>
    </>
  );
}
```

### Pattern applied

Let `Modal` work in both modes: if the consumer passes `open`, it behaves as controlled (Day 1's Controlled Component Pattern); if not, it manages its own state internally, seeded by `defaultOpen` (Day 1's Uncontrolled Component Pattern).

```tsx
function useControllableState<T>(
  controlledValue: T | undefined,
  defaultValue: T,
) {
  const [internalValue, setInternalValue] = useState(defaultValue);
  const isControlled = controlledValue !== undefined;
  const value = isControlled ? controlledValue : internalValue;
  return [value, setInternalValue] as const;
}

type ModalProps = {
  open?: boolean;
  defaultOpen?: boolean;
  onOpenChange?: (open: boolean) => void;
  children: React.ReactNode;
};

function Modal({
  open,
  defaultOpen = false,
  onOpenChange,
  children,
}: ModalProps) {
  const [isOpen, setIsOpen] = useControllableState(open, defaultOpen);

  const close = () => {
    setIsOpen(false);
    onOpenChange?.(false);
  };

  if (!isOpen) return null;
  return (
    <div className='modal'>
      {children}
      <button onClick={close}>Close</button>
    </div>
  );
}

// Uncontrolled, no external state needed
<Modal defaultOpen>
  <p>This dashboard updates every 5 minutes.</p>
</Modal>;

// Controlled, the app decides exactly when it opens and closes
function SaveChangesModal() {
  const [open, setOpen] = useState(false);
  async function handleSave() {
    await saveChanges();
    setOpen(false);
  }
  return (
    <Modal open={open} onOpenChange={setOpen}>
      <button onClick={handleSave}>Save</button>
    </Modal>
  );
}
```

One component, both usage styles. `useControllableState` is the reusable piece: it checks whether a value was passed in from outside, and falls back to internal state when it wasn't.

### Real-world example

Radix UI ships this exact `useControllableState` hook internally, used across `Dialog`, `Select`, `Checkbox`, and every other stateful primitive, which is why shadcn/ui components built on Radix all support both `<Dialog open={open} onOpenChange={setOpen}>` and plain `<Dialog defaultOpen>` out of the box. The native `<input>` element works the same way: pass `value` and it is controlled, pass `defaultValue` and it manages itself.

### Practice exercise

Give the `Accordion` you built in Pattern 1's practice exercise the same treatment: add optional `openItems`/`onOpenItemsChange` props so it can be used as controlled (a parent tracks which sections are open) or uncontrolled (it manages that internally, starting from a `defaultOpenItems` prop).

### When to use

- Building a shared component (library-style) where some consumers need to orchestrate its state from outside (sync it with a form, another component, or a URL parameter) and others just want it to work with zero setup.

### When NOT to use

- An app-specific, one-off component with exactly one consumer. Supporting both modes is extra code for flexibility nobody but a component library actually needs. Pick controlled or uncontrolled, whichever that one consumer requires, and move on.

---

## How the three patterns fit together

Compound Components (Pattern 1) is the baseline structure: subcomponents sharing state through Context. Provider + Compound Components (Pattern 2) is that same structure once the shared state is complex enough to need the full rigor of a Provider and a custom hook. Dual-Mode API (Pattern 3) is an orthogonal concern, about how a component's _top-level_ state is owned (by itself or by its consumer), and it applies equally well to a plain component or a compound one: a real `Select` in a component library typically combines all three, compound subcomponents, a provider-backed context, and a dual-mode `value`/`defaultValue` API on the outer `Select`.
