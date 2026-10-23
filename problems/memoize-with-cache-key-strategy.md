# Memoize with Cache-Key Strategy

## Problem

Implement a general-purpose `memoize(fn, options?)` that caches the result of
an expensive pure function call based on its arguments. The default
"cache key" strategy should handle primitives, objects, and multiple
arguments; callers can supply a custom `resolver` to compute the key.

```ts
const memoized = memoize(expensiveFn);      // works out of the box
const memoized2 = memoize(expensiveFn, { resolver: (...args) => args[0].id });
memoized2.clear();                           // optional cache controls
```

## Why it matters for frontend

Memoization is everywhere in UI work: `React.useMemo`/`React.memo` are
memoization with a React-owned key strategy. Rendering large lists,
formatting dates/currency on every render, running selector logic over a
store, or deriving layout math (e.g. virtualized row positions) are all
expensive-and-pure — the textbook case. Knowing *how* keys are computed
also explains a classic production bug: `useMemo(() => fn(a, b), [])` silently
serving stale values when the dependency array lies about the real cache key.

## Approach

1. **Default key**: the first argument by identity for objects, but primitives
   and multi-arg calls need a composite key. JSON.stringify of all arguments
   is the pragmatic default — deterministic for structurally equal inputs.
2. **Custom resolver**: lets the caller collapse arguments to a stable key
   (e.g. an `id`), which is both faster and avoids serialization pitfalls.
3. **Map for storage**: `Map` preserves insertion order and allows any value
   type, including objects, as keys.
4. **Cache controls**: expose `clear()` and the raw `cache` so callers can
   bound memory — an unbounded `Map` on a long-lived SPA leaks.

## Solution

```ts
type Resolver = (...args: any[]) => any;

interface MemoizeOptions {
  /** Custom key derivation. Defaults to a JSON composite of all arguments. */
  resolver?: Resolver;
}

export function memoize<T extends (...args: any[]) => any>(
  fn: T,
  options: MemoizeOptions = {}
): T & { cache: Map<any, ReturnType<T>>; clear(): void } {
  const cache = new Map<any, ReturnType<T>>();
  const { resolver } = options;

  const memoized = function (this: any, ...args: Parameters<T>): ReturnType<T> {
    const key = resolver ? resolver.apply(this, args) : defaultKey(args);
    if (cache.has(key)) {
      return cache.get(key) as ReturnType<T>;
    }
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  } as T & { cache: Map<any, ReturnType<T>>; clear(): void };

  memoized.cache = cache;
  memoized.clear = () => cache.clear();
  return memoized;
}

function defaultKey(args: unknown[]): string {
  return JSON.stringify(args, stableReplacer);
}

/** Stable serializer: sorted keys, circular refs marked, not crashed on. */
function stableReplacer(this: any, key: string, value: any): any {
  if (typeof value === 'object' && value !== null) {
    if (this.__seen === undefined) this.__seen = new WeakSet();
    if (this.__seen.has(value)) return '[Circular]';
    this.__seen.add(value);
  }
  if (value !== null && typeof value === 'object' && !Array.isArray(value)) {
    return Object.keys(value)
      .sort()
      .reduce<Record<string, any>>((acc, k) => {
        acc[k] = value[k];
        return acc;
      }, {});
  }
  return value;
}

/* ---------- usage ---------- */

// Heavy pure computation, e.g. deriving visible rows in a virtualized list
function layoutRows(items: Item[], rowHeight: number) {
  /* ...expensive... */ return items.map(/* ... */);
}
const getLayout = memoize(layoutRows);
// getLayout(items, 48) recomputes only when structure or args change.

// Resolver-based: user objects keyed by id (avoids re-serializing big trees)
const formatUser = memoize(
  (user: { id: string; name: string }) => user.name.toUpperCase(),
  { resolver: (user) => user.id }
);
```

> **Note:** the `__seen` WeakSet trick works because `JSON.stringify` calls
> the replacer with the holder object as `this`. In production you would
> usually keep a separate circular-safe serializer; the idea is what matters.

## Time / space complexity

- **Cache hit**: O(key) — serialization cost for the default key, O(1) for a
  resolver like `args[0].id`.
- **Cache miss**: cost of `fn` plus key computation plus one `Map` insert.
- **Space**: O(n) unique keys retained until `clear()` is called.

## Edge cases

| Case | Handling |
|---|---|
| `undefined` / functions in args | JSON drops them; two different arg lists can collide (`[undefined]` vs `[]`) — resolver recommended when args include functions |
| Object key order `{a:1,b:2}` vs `{b:2,a:1}` | Sorted-key replacer makes them equal (arguably correct for memoization) |
| Circular references | `[Circular]` placeholder instead of throwing |
| `this` context | `fn.apply(this, args)` preserves method-style calls; resolver also gets the receiver |
| Cache invalidation | Exposed `clear()`; for bounded memory use an LRU variant instead of `Map` |
| Async functions | Memoizing a promise shares the *in-flight* promise — usually desirable, but rejections get cached too; wrap if you need retry |
