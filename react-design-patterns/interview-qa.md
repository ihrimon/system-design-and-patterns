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

<!-- Add Day 2 questions below as you complete Day 2 -->
