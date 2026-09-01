# Day 1 — Component Fundamentals Patterns

Patterns covered today:

1. [Component Composition Pattern](#1-component-composition-pattern)
2. [Children Pattern](#2-children-pattern)
3. [Controlled Component Pattern](#3-controlled-component-pattern)
4. [Uncontrolled Component Pattern](#4-uncontrolled-component-pattern)

Study method for each pattern: understand the problem → see the bad approach → see the pattern applied → do the practice exercise → know when (not) to use it.

---

## 1. Component Composition Pattern

### The problem

Beginners often build one component that tries to handle every visual variation through boolean/string props. It grows into an unreadable prop list and every new UI variant needs a code change inside the component itself.

### Bad approach (no pattern)

```tsx
type CardProps = {
  title: string;
  showHeader?: boolean;
  showFooter?: boolean;
  footerText?: string;
  showAvatar?: boolean;
  avatarUrl?: string;
  showBadge?: boolean;
  badgeText?: string;
};

function Card({
  title,
  showHeader = true,
  showFooter = false,
  footerText,
  showAvatar = false,
  avatarUrl,
  showBadge = false,
  badgeText,
}: CardProps) {
  return (
    <div className="card">
      {showHeader && (
        <div className="card-header">
          {showAvatar && <img src={avatarUrl} />}
          <h3>{title}</h3>
          {showBadge && <span className="badge">{badgeText}</span>}
        </div>
      )}
      <div className="card-body">{/* body content has nowhere to go */}</div>
      {showFooter && <div className="card-footer">{footerText}</div>}
    </div>
  );
}
```

Every new requirement (two badges? a button in the footer? custom body content?) means editing `Card` again and adding more props.

### Pattern applied

Build small components and let the consumer combine them with JSX — this is React's core philosophy: **composition over configuration**.

```tsx
function Card({ children }: { children: React.ReactNode }) {
  return <div className="card">{children}</div>;
}

function CardHeader({ children }: { children: React.ReactNode }) {
  return <div className="card-header">{children}</div>;
}

function CardBody({ children }: { children: React.ReactNode }) {
  return <div className="card-body">{children}</div>;
}

function CardFooter({ children }: { children: React.ReactNode }) {
  return <div className="card-footer">{children}</div>;
}

// Usage — the consumer decides the shape, Card never needs new props
<Card>
  <CardHeader>
    <img src="avatar.jpg" />
    <h3>John Doe</h3>
    <span className="badge">Pro</span>
  </CardHeader>
  <CardBody>
    <p>Bio text goes here.</p>
  </CardBody>
  <CardFooter>
    <button>Follow</button>
  </CardFooter>
</Card>;
```

No boolean explosion. New layouts are just new JSX arrangements, not new props.

### Practice exercise

Build a `Page` layout out of composed pieces: `Page`, `PageHeader`, `PageSidebar`, `PageContent`. Then render two different pages (a dashboard and a settings page) that reuse the same pieces in different arrangements.

### When to use

- Anytime you're tempted to add a new boolean prop to control rendering — stop and ask if composition solves it instead.
- Cards, layouts, dashboards, any "shell + variable content" UI.

### When NOT to use

- When every consumer truly needs the exact same fixed structure with only text/data changing — a simple props-based component is simpler than composition there.

---

## 2. Children Pattern

### The problem

`Component Composition` is the general principle. The **Children Pattern** is the specific mechanic: use React's built-in `children` prop so a component can accept *arbitrary* nested content without declaring a specific prop for it.

### Bad approach (no pattern)

```tsx
type ButtonProps = { text: string; iconUrl?: string };

function Button({ text, iconUrl }: ButtonProps) {
  return (
    <button>
      {iconUrl && <img src={iconUrl} />}
      {text}
    </button>
  );
}
```

This only supports "icon + text". Want a spinner instead of text while loading? Want two icons? You're back to adding more props.

### Pattern applied

```tsx
function Button({ children }: { children: React.ReactNode }) {
  return <button className="btn">{children}</button>;
}

// Usage — Button doesn't care what's inside it
<Button>
  <SpinnerIcon /> Saving...
</Button>

<Button>
  <span>Delete</span> <TrashIcon />
</Button>
```

### Practice exercise

Refactor a `Tooltip` or `Badge` component you've written before (or write one fresh) so it takes `children` instead of a `text` prop, then render it with plain text, with an icon + text, and with a small inline list — without touching the `Tooltip`/`Badge` component itself.

### When to use

- Wrapper/container components: buttons, cards, modals, tooltips, list items.
- Whenever the "content" is genuinely variable and the wrapper only provides layout/behavior.

### When NOT to use

- When a component needs to render content in **multiple distinct places** (e.g. a header area AND a footer area) — a single `children` slot can't do that; you need multiple named props (that's the **Slot Pattern**, coming later) or **Compound Components** (Day 4).

---

## 3. Controlled Component Pattern

### The problem

Forms need React to know the current value of an input — for validation, conditional logic (disable submit until valid), or resetting fields programmatically.

### Bad approach (fighting the DOM)

```tsx
function SearchBox() {
  function handleSubmit() {
    const input = document.querySelector("input"); // fragile, breaks with multiple instances
    console.log(input?.value);
  }
  return (
    <>
      <input />
      <button onClick={handleSubmit}>Search</button>
    </>
  );
}
```

React has no idea what's in the input. You can't validate as the user types, can't disable the button conditionally, can't reset the field from a "Clear" button.

### Pattern applied

React state is the **single source of truth**; the DOM input just reflects it.

```tsx
function SearchBox() {
  const [query, setQuery] = useState("");
  const isValid = query.trim().length > 0;

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <button disabled={!isValid} onClick={() => console.log(query)}>
        Search
      </button>
      <button onClick={() => setQuery("")}>Clear</button>
    </>
  );
}
```

Every keystroke updates state, which re-renders the input with the new `value` — React drives the DOM, not the other way around.

### Practice exercise

Build a signup form with `email` and `password` controlled fields. Show a live validation message ("Enter a valid email") as the user types, and disable the submit button until both fields are valid.

### When to use

- Any form where you need: live validation, conditional UI based on input, formatting-as-you-type, resetting fields from outside the input, or syncing multiple inputs together.

### When NOT to use

- Simple "fire and forget" forms with no validation, or performance-sensitive inputs with extremely high update frequency where re-rendering on every keystroke is measurably too slow — an uncontrolled input (or a debounced/library-managed form) can be lighter.

---

## 4. Uncontrolled Component Pattern

### The problem

Sometimes you don't need React to track a value on every keystroke — you only need it *once*, e.g. on submit. Making everything controlled adds state and re-renders you don't need.

### Bad approach (over-engineered)

```tsx
function SimpleForm() {
  const [name, setName] = useState("");

  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    console.log(name);
  }

  return (
    <form onSubmit={handleSubmit}>
      <input value={name} onChange={(e) => setName(e.target.value)} />
      <button type="submit">Submit</button>
    </form>
  );
}
```

If `name` is never used for validation or conditional rendering, this state + re-render on every keystroke is unnecessary overhead.

### Pattern applied

Let the DOM own the value; read it only when you need it via a `ref`.

```tsx
function SimpleForm() {
  const nameRef = useRef<HTMLInputElement>(null);

  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    console.log(nameRef.current?.value);
  }

  return (
    <form onSubmit={handleSubmit}>
      <input ref={nameRef} defaultValue="" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

Note: `defaultValue`, not `value` — that's what tells React "don't manage this, DOM owns it."

### Practice exercise

Build a file upload form (`<input type="file" />` is *always* uncontrolled — the browser doesn't allow React/JS to set its value for security reasons) with a text `note` field, also uncontrolled via `ref`. Read both values only on submit.

### When to use

- `<input type="file">` (must be uncontrolled — browser restriction).
- Simple, one-shot forms with no live validation.
- Integrating with non-React/third-party DOM libraries that expect to manage the input themselves.
- Performance-critical forms with many fields where you truly don't need per-keystroke reactivity.

### When NOT to use

- Any time you need live validation, conditional rendering, or to programmatically clear/update the field from elsewhere in the UI — that requires the Controlled pattern.

---

## Controlled vs Uncontrolled — quick comparison

| | Controlled | Uncontrolled |
|---|---|---|
| Source of truth | React state | The DOM |
| Prop used | `value` + `onChange` | `defaultValue` + `ref` |
| Live validation | Easy | Requires manual event listener |
| Re-renders per keystroke | Yes | No |
| Reset/clear programmatically | Easy (`setState('')`) | Needs `ref.current.value = ''` |
| `<input type="file">` | Not possible | Required |
