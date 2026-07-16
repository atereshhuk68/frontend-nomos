---
name: frontend-nomos
description: 'TypeScript patterns (discriminated unions, branded types, satisfies, never-exhaustive, Zod), React component patterns (composition, compound, error boundary), anti-patterns (derived state, unnecessary effect, state too high), performance (render boundaries, React Compiler, LCP), CSS-over-JS (has, popover, container queries, scroll-driven, light-dark, contrast-color, starting-style, in-n-out 3-state), security (CSP, DOMPurify, HttpOnly cookies), a11y (keyboard nav, semantic HTML), testing (Vitest, Playwright, property-based, mutation), DOM manipulation (WeakMap, AbortController), naming conventions, and modern React ecosystem (TanStack Query, Zustand, shadcn/ui, Vite).'
---

## Naming

```
const marketSearchQuery = 'election';          // not q
function fetchMarketData(id: string) {}         // verb-noun, not market(id)
const isAuthenticated = true;                   // is/has prefix

// File naming: lowercase-hyphens (kebab-case), 2-5 words, descriptive
// ✅ fetch-market-data.ts   ❌ fetchMarketData.ts
```

## Semantic HTML

Native elements carry behavior + a11y — don't rebuild them:

```
// ❌ <div role="button" tabIndex={0} onKeyDown={...}>
// ✅ <button>  (tab order, Enter/Space, disabled, form participation)
// ✅ <details>/<summary> for disclosure (built-in keyboard, a11y)
// ✅ <label>, type="email", autocomplete, required — form semantics
```

Decision checklist: (1) Is there an element that already does this? (2) Does the custom version preserve keyboard behavior? (3) Does it work without a mouse? (4) Does it still make sense inside a form?

## Code Smells

```
// Long func → split
function process(data) { return saveData(transformData(validateData(data))); }

// Deep nest → early return
if (!user || !user.isAdmin) return;

// Magic number → const
const MAX_RETRIES = 3;

// Unnecessary async — don't mark sync async "just in case"
// ❌ async function getConfig() { return defaultConfig; }
// ✅ function getConfig() { return defaultConfig; }
```

## TypeScript

```ts
// Discriminated union — illegal states unrepresentable
type Req<T> = { status: 'idle' } | { status: 'loading' }
            | { status: 'success'; data: T } | { status: 'error'; error: Error };

// Branded type
type UserId = string & { readonly __brand: 'UserId' };

// Const assertion
const ROLES = ['admin', 'user'] as const;
type Role = typeof ROLES[number];

// Exhaustive switch
function process(s: Status): string {
  switch (s) { case "a": return "x"; case "b": return "y"; }
  const _: never = s; throw new Error(`unhandled ${s}`);
}

// tsconfig
{ "strict": true, "noUncheckedIndexedAccess": true, "noImplicitReturns": true }

// satisfies — validates without losing inference
const palette = { red: [255, 0, 0] } satisfies Record<string, readonly number[]>;

// unknown over any — forces validation
function parse(raw: unknown): Data { ... }

// Template literal types for routes, events
type Route = `/blog/${string}`;

// Prefer inferable generics — great APIs rarely need manual generic args
// Build types from existing types: Pick, Omit, Partial, Required
// Type predicates: function isUser(v: unknown): v is User { ... }

// Avoid enum — literal unions are simpler to refactor and serialize
// ❌ enum Role { Admin, User }
// ✅ type Role = 'admin' | 'user';
```

## Zod

```ts
const S = z.object({ id: z.string().uuid(), email: z.string().email() });
type T = z.infer<typeof S>;
const parsed = S.parse(await res.json()); // trust boundary — throws
const safe = S.safeParse(formData); // user input — expected fail
```

## Component Patterns

```tsx
// Composition
function Card({ children, variant = 'default' }: { children: ReactNode; variant?: string }) {
  return <div className={`card-${variant}`}>{children}</div>;
}
function CardHeader({ children }: { children: ReactNode }) {
  return <div>{children}</div>;
}

// Compound — Context-based tabs
const TabsCtx = createContext<{ active: string; set: (t: string) => void } | undefined>(undefined);
function Tabs({ children, defaultTab }: { children: ReactNode; defaultTab: string }) {
  const [active, setActive] = useState(defaultTab);
  return <TabsCtx.Provider value={{ active, set }}>{children}</TabsCtx.Provider>;
}

// Error boundary
class ErrorBoundary extends React.Component<{ children: ReactNode }, { hasError: boolean; error: Error | null }> {
  state = { hasError: false, error: null };
  static getDerivedStateFromError(e: Error) {
    return { hasError: true, error: e };
  }
  render() {
    return this.state.hasError ? <div>Error: {this.state.error?.message}</div> : this.props.children;
  }
}
```

## Anti-Patterns

```tsx
// Derived state in effect → compute during render
// ❌ useEffect(() => setFull(first + last), [first, last])
// ✅ const full = first + last;

// Event logic in effect → handler
// ❌ useEffect(() => { if (x) notify(); }, [x]);
// ✅ function handleClick() { notify(); }

// Mutation → immutable
// ❌ items.push(x); setItems(items);
// ✅ setItems([...items, x]);

// TanStack Query: never copy to local state
// ❌ useEffect(() => setTodos(data), [data]);
// ✅ const { data: todos } = useQuery(['todos']);

// State lifted too high — render boundaries > memo
// ❌ Page owns search + selection + modal state → every keystroke re-renders table
// ✅ SearchSection(search), DataTable(selection), Modal(visibility)

// Derived from props in state → compute during render
// ❌ const [filtered, setFiltered] = useState(() => heavy(items));
// ✅ const filtered = useMemo(() => heavy(items), [items]);

// Speculative async — see Code Smells section
```

## React Performance: Render Boundaries First

Most perf problems come from **state too high**, not missing memo. Three failure modes: too wide (half page re-renders), too frequent (keystroke propagates through heavy trees), too expensive (heavy work on hot path).

Before useMemo, ask: (1) What changed? (2) Why did that reach this component? (3) Can state move lower? (4) Are oversized objects in props?

React Compiler eliminates manual memo in components, but can't fix poor separation — boundary design is still on you.

```
// LCP optimization: fetchpriority="high" on above-fold images
<img src="hero.webp" fetchpriority="high" alt="" />

// Never lazy-load above-fold images — adds ~360ms delay
// sizes="auto" with srcset for fully automatic responsive images
<img srcset="..." sizes="auto" loading="lazy" />
```

## React 19

```tsx
// useFormStatus — must be in child of <form>, not the form component itself
function Submit() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>Send</button>;
}

// use() — pass promise from parent, never create in render
function C({ p }: { p: Promise<Data> }) {
  const d = use(p);
}
```

## Hooks

```ts
function useToggle(i = false): [boolean, () => void] {
  const [v, set] = useState(i);
  return [v, useCallback(() => set((p) => !p), [])];
}
function useDebounce<T>(v: T, d: number): T {
  const [db, set] = useState(v);
  useEffect(() => {
    const h = setTimeout(() => set(v), d);
    return () => clearTimeout(h);
  }, [v, d]);
  return db;
}
```

## DOM Manipulation (vanilla contexts)

```
// Prefer modify over create: hide/show instead of destroy/create
// textContent > innerText (avoids reflow)
// insertAdjacentHTML > innerHTML (preserves existing DOM)
// Batch inserts: DocumentFragment, template + cloneNode
// WeakMap for DOM-associated data (auto-cleans on removal)
// AbortController to unbind groups of listeners
```

## Security

```tsx
// XSS: JSX auto-escapes; for dangerouslySetInnerHTML always sanitize with DOMPurify
// Auth: HttpOnly cookies, never localStorage — any XSS reads localStorage
// Cookie: HttpOnly; Secure; SameSite=Strict
// CSP: script-src 'nonce-...' over 'unsafe-inline'; strict-dynamic for code-split chunks
// Validate server inputs with Zod safeParse — TS only covers compile time
// Add CSRF tokens (X-CSRF-Token header) for state-changing requests
// Test CSP with Content-Security-Policy-Report-Only before enforcing
```

## A11y

```tsx
// Keyboard nav
switch (e.key) {
  case 'ArrowDown':
    setActive((i) => Math.min(i + 1, max));
    break;
  case 'ArrowUp':
    setActive((i) => Math.max(i - 1, 0));
    break;
  case 'Enter':
    select();
    break;
  case 'Escape':
    close();
    break;
}

// Focus management
useEffect(() => {
  if (open) {
    prev = document.activeElement as HTMLElement;
    ref.current?.focus();
  } else prev?.focus();
}, [open]);
```

## CSS

```css
// CSS does more — browser does the work
// ✅ :has() — parent state toggling, no JS classList
// ✅ popover — modals/dropdowns, zero JS, light-dismiss, top-layer
// ✅ container queries — replaces ResizeObserver
// ✅ @container (<size>)
{
}
+ container-name for scoping (name-only containers)
// ✅ scroll-driven animations — replaces JS scroll listeners
// ✅ scrollbar-gutter: stable — no JS scrollbar width calc
// ✅ light-dark(lightVal, darkVal) — no @media color-scheme duplication
// ✅ text-wrap: balance / pretty — no JS text balancing
// ✅ contrast-color(var(--bg)) — auto black/white per WCAG
// ✅ @starting-style — animate elements entering display:block (dialogs, popovers)
// ✅ sizes="auto" — browser determines image sizes for lazy-loaded images
// ✅ align-content: center — single-child centering without flexbox/grid

// In-n-out animation: 3-state system (3→2→1 source order)
dialog {
  transition:
    1s opacity,
    1s display allow-discrete,
    1s overlay allow-discrete;
  &:not(:open) {
    opacity: 0;
  }
  &:open {
    opacity: 1;
  }
  @starting-style {
    &:open {
      opacity: 0;
    }
  } /* must be last */
}

// Name-only container scoping — no build, no specificity boost
ds-card {
  container-name: ds-card;
}
@container ds-card {
  .title {
    /* scoped */
  }
}
```

## Testing

```
// Vitest > Jest (faster, Vite-native), Playwright for E2E
// Property-based testing (fast-check) asserts over arbitrary inputs
// Mutation testing (Stryker) — catches weak test coverage
// Human-owned acceptance tests: deny agent write access via settings
```

## Ecosystem (2026)

```
// Data: TanStack Query (default async state manager)
// State: Zustand (lightweight) > Jotai (atomic) > Redux Toolkit (enterprise)
// Forms: React Hook Form + Zod, or TanStack Form
// Styling: Tailwind v4, shadcn/ui, Panda CSS (type-safe atomic CSS)
// Build: Vite (default), Turborepo (monorepo)
// Auth: Better Auth, Auth.js
// AI: Vercel AI SDK (useChat hook, streaming UIs)
```

## Red Flags

| Pattern                                      | Fix            |
| -------------------------------------------- | -------------- |
| `eslint-disable react-hooks/exhaustive-deps` | Refactor       |
| Component inside component                   | Move out       |
| `key={index}`                                | Stable id      |
| `any` without justification                  | Proper type    |
| Barrel `index.ts` in app code                | Direct import  |
| `React.FC<Props>`                            | Explicit props |
