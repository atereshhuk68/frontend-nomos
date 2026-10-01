# frontend-nomos

> An agent skill that enforces modern frontend conventions — **React, TypeScript, CSS, browser-native APIs, accessibility, performance, and security** — so AI coding agents write and review frontend code the way a senior engineer would.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Agent skill](https://img.shields.io/badge/agent-skill-orange.svg)](#install)
[![React 19](https://img.shields.io/badge/React-19-61dafb.svg)](frontend-nomos/SKILL.md)
[![Accessibility](https://img.shields.io/badge/a11y-first-6a5acd.svg)](frontend-nomos/SKILL.md)

`frontend-nomos` is a single-file `SKILL.md` that plugs into **Claude Code, opencode, Codex, Cursor, and any `AGENTS.md` / `CLAUDE.md`-compatible agent**. It turns "make this component" into a review-ready result: correct hooks, native HTML/CSS before JavaScript, real accessibility, and no hallucinated state-management boilerplate.

## Why use it

AI agents default to `div`-based buttons, `useEffect` for everything, manual `useMemo` spam, and reinvented browser features. `frontend-nomos` encodes the opposite bias:

- **Browser primitives over JS** — `<dialog>`, Popover API, `<details>`, `base-select`, `:has()`, container queries, scroll-driven animation, scroll-snap carousels.
- **React fundamentals** — effects vs. events, derived state, render boundaries, immutable updates, React Compiler, React 19 `use()` / `useFormStatus` / `useActionState` / `useOptimistic`.
- **Accessibility as a default** — keyboard navigation, focus management, `inert`, native semantics, `:focus-visible`, contrast.
- **Security that holds** — CSP nonces, DOMPurify, HttpOnly cookies, CSRF, trust-boundary validation.

## Install

Drop `SKILL.md` into your agent's skills directory:

```bash
git clone git@github.com:atereshhuk68/frontend-nomos.git

# Claude Code
mkdir -p ~/.claude/skills/frontend-nomos
cp frontend-nomos/SKILL.md ~/.claude/skills/frontend-nomos/

# opencode / agents that read ~/.agents/skills
mkdir -p ~/.agents/skills/frontend-nomos
cp frontend-nomos/SKILL.md ~/.agents/skills/frontend-nomos/
```

Or reference it from your project's `AGENTS.md` / `CLAUDE.md` and let the agent load it on demand. No build step, no dependencies — it is plain Markdown.

## What it covers

| Area | Key patterns enforced |
|------|-----------------------|
| **React** | Composition, compound components, error boundaries, render boundaries, hooks, React Compiler setup and gotchas, React 19 `use()`, `useFormStatus`, `useActionState`, `useOptimistic` |
| **TypeScript** | Branded types, discriminated unions, `satisfies`, exhaustive switch, template-literal types, type predicates |
| **Browser primitives** | `<dialog>` + `showModal()`, Popover API, `<details>`, `appearance: base-select`, `:has()`, container queries, `light-dark()`, `contrast-color()`, `@starting-style`, scroll-driven & scroll-snap |
| **Performance** | State locality over memo, render boundaries, LCP with `fetchpriority`, `sizes="auto"`, virtualization, high-frequency/real-time rendering (workers, `SharedArrayBuffer`, `OffscreenCanvas`) |
| **Accessibility** | Semantic HTML, keyboard nav, focus return, `inert`, `aria-live`, labels, contrast |
| **Data** | TanStack Query invalidation, TanStack Router prefetching via Route Context, no server state in `useState` |
| **Security** | CSP nonces, DOMPurify for `dangerouslySetInnerHTML`, HttpOnly/Secure/SameSite cookies, CSRF tokens |
| **Testing** | Vitest, Playwright, property-based (fast-check), mutation testing |
| **Tooling** | Vite, Turborepo, React Hook Form, TanStack Form, shadcn/ui, Tailwind v4, Vercel AI SDK |

## Anti-patterns it catches

Derived state in effects · event logic in effects · state lifted too high · `useEffect` for what render or a handler should do · `key={index}` · `React.FC<Props>` · component-inside-component · barrel `index.ts` in app code · `<div onClick>` instead of `<button>` · unsanitized `dangerouslySetInnerHTML` · missing `alt` or unlabeled controls · copying query data into `useState` · mutating props/state in place · speculative `async`.

## Example prompts

Once installed, the skill triggers automatically on prompts like:

- "Build an accessible modal — no dependencies."
- "Why does this list re-render on every keystroke?"
- "Review this component for React 19 patterns and a11y issues."
- "Refactor this dark-mode toggle to use native browser features."
- "Convert this custom carousel to CSS scroll-snap."

## FAQ

**Which agents support this skill?**
Any agent that reads a `SKILL.md` or follows `AGENTS.md` / `CLAUDE.md` instructions — Claude Code, opencode, Codex, and similar. It is plain Markdown with YAML frontmatter.

**Does it require a specific framework?**
No. It is framework-agnostic within the frontend — React, TypeScript, HTML, and CSS. TanStack Query/Router and Vite appear as defaults, not requirements.

**Do I need extra dependencies?**
No. The skill is a single file. It may *recommend* libraries (DOMPurify, Vitest, Playwright), but installs nothing itself.

**Is it opinionated?**
Yes, deliberately. It encodes a "browser first, simplest correct solution" bias so agents stop over-engineering UI with JavaScript.

**How do I keep it current?**
`SKILL.md` is versioned in this repo; pull the latest and re-copy it into your skills directory.

## Topics

`react` · `typescript` · `css` · `accessibility` · `a11y` · `frontend` · `web-performance` · `agent-skill` · `claude-code` · `opencode` · `ai-coding-assistant` · `coding-standards` · `browser-apis`

## License

[MIT](LICENSE) © 2026 Oleksandr Tereshchuk
