# TypeScript / React Typing & JS-Language 2026

Read when: designing type-level APIs, choosing native-vs-library replacements, touching dates/formatting.

## Type patterns people don't know exist

**Assertion functions**: narrow *caller's* scope, not just return boolean. Prefer over scattered `if (!x) throw`:

```ts
function assertIsDefined<T>(v: T): asserts v is NonNullable<T> {
  if (v == null) throw new Error('expected defined');
}
assertIsDefined(config); // config: NonNullable<...> from here on
```

**`as const satisfies`**: freeze literals AND validate. Either alone loses something:

```ts
export const ROUTES = {
  home: '/', markets: '/markets',
} as const satisfies Record<string, `/${string}`>;
// literal types preserved + typo/shape validated
```

## React Context cost — the shipped reality

Provider value change re-renders EVERY consumer. `useContextSelector` never shipped, `calculateChangedBits` removed. Stop looking. React Compiler auto-memoization matches hand-rolled split-wrapper trick. Productive move: make renders cheap + split providers by concern, not prevent renders.

## Ship less JS — Baseline audit

Typical mid-size app still ships 60–90 KB gzipped of replaceable libs. Check quarterly vs Baseline status (webstatus.dev): `toSorted/toReversed/toSpliced/with` (lodash), `Object.groupBy`, `structuredClone` (cloneDeep), Set methods (`union/intersection/difference/isSubsetOf`), `crypto.subtle`, `AbortSignal.any()`.

Judgment matters, not list. Three gates before dropping lib: (1) audience in Baseline range? (2) polyfill heavier than lib? (3) native covers *actual* usage?

Common gotcha: `AbortSignal.timeout(ms)` rejects with **`TimeoutError`, not `AbortError`**. `if (e.name === 'AbortError') return;` silently misses timeouts:

```ts
try { await fetch(url, { signal: AbortSignal.timeout(5000) }); }
catch (e) { if (e.name === 'TimeoutError') retry(); else if (e.name === 'AbortError') return; }
```

## Dates & formatting

**Temporal** shipped Chrome 144 (Stage 4): immutable `PlainDate`/`ZonedDateTime`, DST-aware math, 1-based months. Don't swap yet: polyfill (19–44 KB) outweighs dayjs until Safari lands. `Intl.DateTimeFormat` covers most today.

**Intl more capable than teams assume** (zero libs): `Intl.RelativeTimeFormat` (`numeric: 'auto'` → "yesterday"), `NumberFormat` (currency/`notation: 'compact'`/units), `ListFormat` ("a, b, and c"), `PluralRules`, `DisplayNames`, `DurationFormat`. Hoist formatter construction to module scope. Building formatter per render expensive; formatting cheap.

## ES2026 (know these exist)

`Map.getOrInsert/getOrInsertComputed` (memoization without dance), `Iterator.concat`, `JSON.parse(text, reviver, context)`: `context.source` gives raw text slice (parse snowflake IDs > 2^53 into BigInt losslessly), `JSON.rawJSON`, `Error.isError`, `Math.sumPrecise`, `Uint8Array.toBase64()/fromHex()`. Caveat: `Array.fromAsync` sequential. Use `Promise.all` for parallelism.

## TS 6 migration traps

`strict: true` now default on. `types` defaults to `[]` (declare ambient globals explicitly). `baseUrl` deprecated → `paths` alone (resolves relative to tsconfig). Codemod: `npx ts5to6`.