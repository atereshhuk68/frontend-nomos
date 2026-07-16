# frontend-nomos

Agent skill enforcing frontend conventions for TypeScript, React, CSS, security, a11y, and testing.

## Install

Copy `SKILL.md` to your agent's skills directory or reference it in your project's AGENTS.md / CLAUDE.md.

## Covered Areas

| Area | Key Patterns |
|------|-------------|
| **TypeScript** | Branded types, discriminated unions, `satisfies`, exhaustive switch, Zod validation |
| **React** | Composition, compound components, ErrorBoundary, render boundaries, React 19 `use()`, `useFormStatus` |
| **CSS** | `:has()`, popover, container queries, `light-dark()`, `contrast-color()`, `@starting-style`, in-n-out 3-state |
| **Performance** | State locality over memo, fetchpriority, sizes="auto" |
| **Security** | CSP nonce, DOMPurify, HttpOnly cookies, Zod server validation |
| **A11y** | Semantic HTML, keyboard nav, focus management |
| **Testing** | Vitest, Playwright, property-based, mutation |
| **Ecosystem** | TanStack Query, Zustand, shadcn/ui, Vite, Better Auth, Vercel AI SDK |

## Anti-Patterns Caught

Derived state in effects, state lifted too high, `React.FC`, barrel exports, index-as-key, component-in-component, speculative async.

## Usage

Load the skill in your agent config. The agent will follow these conventions when writing or reviewing code.
