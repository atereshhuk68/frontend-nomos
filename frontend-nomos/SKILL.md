---
name: frontend-nomos
description: 'MUST USE for any React/TypeScript/CSS frontend work — writing, reviewing, refactoring, or debugging. Covers React (hooks, effect-vs-event, derived state, render boundaries, React Compiler, React 19 use()/useFormStatus/useActionState/useOptimistic, real-time rendering), browser primitives over JS (dialog, popover, details, invoker commands, base-select, :has, container/style queries, anchor positioning, view transitions, scroll-driven, scroll-snap), real-time transport (WebSockets, SSE), Temporal/Intl, accessibility (keyboard nav, focus management, inert, native semantics, contrast), TypeScript, performance (LCP, content-visibility, ship-less-JS), security (XSS/CSP/CSRF, CSS exfil), testing (Vitest, Playwright), DOM APIs. Trigger whenever the task mentions React, a component, JSX, hooks, useEffect, re-render, state/props, TanStack Query/Router, or frontend HTML/CSS — and when debugging browser runtime errors (Uncaught Error, minified stack, hydration mismatch) or cross-browser bugs.'
---
Deep dives in `references/` — read when task touches area:

- `references/realtime.md` — high-frequency rendering ladder, WebSocket reconnect/heartbeat/backpressure, SSE-vs-WS call, stale-response abort races
- `references/css-2026.md` — anchor positioning / view transitions / style queries failure modes, layout & root scroller, base-select fallback, CSS exfiltration, micro-UX
- `references/typescript-react.md` — assertion functions, `as const satisfies`, Context cost, Baseline ship-less-JS judgment, Temporal, ES2026, Intl, TS6

## Naming

```
const marketSearchQuery = 'election';          // not q
function fetchMarketData(id: string) {}         // verb-noun, not market(id)
const isAuthenticated = true;                   // is/has prefix

// File naming: lowercase-hyphens (kebab-case), 2-5 words, descriptive
// ✅ fetch-market-data.ts   ❌ fetchMarketData.ts
```

## Semantic HTML

Pick element that already does job — native elements ship behavior, keyboard, a11y that custom code must re-implement (and usually breaks: disabled state, form participation, Enter/Space activation).

```tsx
// Accordion with radio behavior — shared name, zero JS
<details name="faq">...</details>
<details name="faq">...</details>   // opening one closes the siblings

// Declarative modal control — no onClick handler needed
<button command="show-modal" commandfor="checkout-dialog">Open</button>
<dialog id="checkout-dialog">...</dialog>

// Collapsed content that find-in-page reveals automatically
<section hidden="until-found">...</section>

// Declarative trigger for the newer close sequence
<button command="request-close" commandfor="dlg">✕</button>
```

Decision checklist: (1) Element already does this? (2) Custom version keeps keyboard behavior? (3) Works without mouse? (4) Still makes sense inside form?

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

// useState(null) infers the type as null — useState<User | null>(null) or delete the dead state

// More: assertion functions, as const satisfies, Context cost, ship-less-JS judgment,
// TimeoutError gotcha, ES2026, Intl, Temporal — references/typescript-react.md
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

## Data Fetching (TanStack Query + Router)

Server state stay in cache, never local copies. Let keying solve correctness (races, staleness) that hand-rolled effects re-introduce:

```ts
// queryKey is hierarchical — scope invalidations, never copy server data into local state
queryClient.invalidateQueries({ queryKey: ['users'] });      // all users queries
queryClient.invalidateQueries({ queryKey: ['users', 42] });  // one user

// Prefetch as early as the route loader, but keep loader + component in sync — one queryOptions
// ❌ loader ensureQueryData(...) + component useSuspenseQuery(...) duplicated → drift, waterfalls
// ✅ share ONE queryOptions via Route Context; loader/component differ only in the call
context: ({ params, deps }) => ({ dashboardQueryOptions: dashboardQueryOptions(params.id, deps) }),
loader: async ({ context }) => context.queryClient.ensureQueryData(context.dashboardQueryOptions),
// loaderDeps names the params that uniquely identify the loaded data (don't re-run on every param)

// Cancellation is not failure: when you do hand-roll a signal, swallow AbortError silently
// (never log/metric it as an error) — and remember only the newest response may paint.
// Reconnect/heartbeat/backpressure for streams → references/realtime.md
```

## React Performance: Render Boundaries First

Most perf problems from **state too high**, not missing memo. Three failure modes: too wide (half page re-renders), too frequent (keystroke propagates through heavy trees), too expensive (heavy work on hot path).

Before useMemo, ask: (1) What changed? (2) Why did it reach this component? (3) Can state move lower? (4) Oversized objects in props?

React Compiler kills manual memo in components, but can't fix poor separation — boundary design still on you. Context especially: any provider-value change re-renders EVERY consumer; `useContextSelector` never shipped. Reframe: don't prevent renders, make renders cheap → `references/typescript-react.md`.

```tsx
// LCP optimization: fetchpriority="high" on above-fold images
<img src="hero.webp" fetchpriority="high" alt="" />

// Never lazy-load above-fold images — adds ~360ms delay
// sizes="auto" with srcset for fully automatic responsive images
<img srcset="..." sizes="auto" loading="lazy" />

// Hand-rolled fetch races and WebSocket transport judgments → references/realtime.md
```

## React Compiler

Build-time Babel plugin — auto-memoizes; drop manual `useMemo`/`useCallback`/`React.memo`.

```bash
npx react-compiler-healthcheck       # audit readiness before adopting
npm i -D babel-plugin-react-compiler eslint-plugin-react-compiler
```

```js
// Vite:      react({ babel: { plugins: [['babel-plugin-react-compiler', {}]] } })
// Next 15:   experimental: { reactCompiler: true }
// Gradual:   { compilationMode: 'annotation' } → opt in per component with "use memo"
```

Manual control still needed: `useSyncExternalStore` (external stores), Web Workers / Canvas / imperative DOM via refs, `requestAnimationFrame` loops, class components (not processed).

```
// Gotcha: mutating during render breaks compiler caching — copy first
// ❌ items.sort(...)          ✅ [...items].sort(...)
```

## High-Frequency / Real-Time Data

Under ~30 updates/sec per stream: skip all this — RAF-coalesce `setState`. 1kHz per-sample setState ≈ 19k renders/s; batch ~16 samples per frame = 60 renders/s. Whole trick.

Above threshold: external store (Float32Array ring buffer, not React state) → split shape from values → Web Worker with transferable buffers → offscreen canvas → decimate before drawing. Full ladder, SharedArrayBuffer COOP/COEP headers, WebSocket production discipline (full-jitter backoff, app-level heartbeats, dedupe by sequence, bufferedAmount backpressure; prefer SSE for one-way streams), stale-response abort handling: → `references/realtime.md`.

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

// useActionState — form action + error + pending in one
const [error, submitAction, pending] = useActionState(serverAction, initialState);

// useOptimistic — for overlapping writes, never build from stale rendered state
// ❌ each optimistic write derived from current rendered value → lost updates
// ✅ useActionState as an ordered async reducer (prev confirmed + change), layer useOptimistic on top;
//    on failure show the error and fall back to the last confirmed state
const [optimistic, addOptimistic] = useOptimistic(items, (cur, next) => [...cur, next]);
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
// Validate external/user input at the trust boundary — TS only covers compile time
// Add CSRF tokens (X-CSRF-Token header) for state-changing requests
// Test CSP with Content-Security-Policy-Report-Only before enforcing
// CSS is an attack surface too: remote-font oracles + :has() leak user data/state from
// untrusted CSS — sandbox iframes, never render user CSS in your origin → references/css-2026.md
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

// Native <dialog>.showModal() gives focus trap + top-layer + Esc + focus return for free
// inert the background while a custom modal is open (showModal does this itself)
// UI state via attributes ([aria-busy], [aria-invalid], [aria-current]), not classes — a `.loading` div is invisible to AT
// :focus-visible for keyboard rings — no mouse-click flicker
// aria-live="polite" to announce async results; label every control (aria-label / <label>)
// contrast >= 4.5:1 — contrast-color(var(--bg)) auto-picks black/white per WCAG
```

## Browser Primitives (HTML/CSS over JS)

```css
// CSS/HTML does more — browser does the work; drop the React state + handlers
// ✅ :has() — parent state toggling, no JS classList
// ✅ popover — modals/dropdowns, zero JS, light-dismiss, top-layer
// ✅ container queries — replaces ResizeObserver; @container (width > 40em) {}
//   + container-name for scoping (name-only containers, no size cost)
// ✅ style queries — @container style(--variant: outline); ⚠ style(--n: 3) is a STRING match
//   (calc never evaluates) — use style(--n = 3) for numeric
// ✅ anchor positioning — tooltip placement with zero JS; ⚠ ancestor position/transform breaks
//   the link (use display:contents wrapper); anchor must precede the tip in DOM order
// ✅ view transitions — ⚠ 'close' fires AFTER leaving top-layer (too late to animate):
//   intercept 'request-close'/'cancel' → preventDefault → startViewTransition(() => dialog.close())
// ✅ scroll-driven animations — replaces JS scroll listeners
// ✅ scrollbar-gutter: stable — no JS scrollbar width calc
// ✅ light-dark(lightVal, darkVal) — no @media color-scheme duplication
// ✅ text-wrap: balance / pretty — no JS text balancing
// ✅ contrast-color(var(--bg)) — auto black/white per WCAG
// ✅ @starting-style — animate elements entering display:block (dialogs, popovers)
// ✅ sizes="auto" — browser determines image sizes for lazy-loaded images
// ✅ align-content: center — single-child centering without flexbox/grid
// ✅ <dialog> + showModal() — true modal: top-layer, ::backdrop, inert bg, Esc, focus return
// ✅ popover + popovertarget — menus/light popups, light-dismiss, top-layer, zero JS
// ✅ <details>/<summary> — accordion; ⚠ open + onToggle fires on mount in React → guard with a mounted ref
// ✅ appearance: base-select + ::picker(select) + <selectedcontent> — customizable native select
//   ⚠ unknown ::picker() discards the ENTIRE rule — isolate base-select in its own block, fallback separately
// ✅ scroll-snap + scroll-marker-group + ::scroll-button() + ::scroll-marker — carousel, no JS
// ✅ animation-timeline: view() + animation-range — reveal on scroll (wrap in @supports)
// ✅ :has(#toggle:checked) + color-scheme — dark mode without Context/re-render (persist with tiny JS)

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

// Deeper recipes (anchor math, VT cross-doc rules, micro-UX, bidi) → references/css-2026.md
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
// Data: TanStack Query (server state) + TanStack Router (type-safe routing / prefetch)
// Client UI state: React-native first — useState/useReducer, URL, Context; add a store only when proven
// Forms: React Hook Form or TanStack Form
// Styling: Tailwind v4, shadcn/ui, Panda CSS (type-safe atomic CSS)
// Build: Vite (default), Turborepo (monorepo)
// Auth: Better Auth, Auth.js
// AI: Vercel AI SDK (useChat hook, streaming UIs)
```

## Red Flags

| Pattern                                      | Fix              |
| -------------------------------------------- | ---------------- |
| `eslint-disable react-hooks/exhaustive-deps` | Refactor         |
| Component inside component                   | Move out         |
| `key={index}`                                | Stable id        |
| `any` without justification                  | Proper type      |
| Barrel `index.ts` in app code                | Direct import    |
| `React.FC<Props>`                            | Explicit props   |
| `useEffect` for derived state / event logic  | Render / handler |
| `<div onClick>` / `role="button"`            | `<button>`       |
| Unsanitized `dangerouslySetInnerHTML`        | DOMPurify        |
| Missing `alt` / unlabeled control            | Semantic a11y    |
| Copying query data into `useState`           | Read `useQuery`  |
| `height:100vh` + `overflow:auto` wrapper     | Root scroller: `min-height:100svh`, `overflow-x: clip` |
| `AbortError` logged as fetch failure         | Expected cancellation |