# Promise.race / Promise.allSettled — Polyfills

## Problem Statement

Implement `Promise.race` and `Promise.allSettled` from scratch in JavaScript, matching the semantics of the native implementations.

- `race(iterable)`: returns a promise that settles as soon as the **first** promise in the iterable settles — resolving with its value or rejecting with its reason.
- `allSettled(iterable)`: returns a promise that resolves with an array of outcome objects **after every** promise has settled. Each object is `{ status: "fulfilled", value }` or `{ status: "rejected", reason }`, in the same order as the input.

## Why It Matters for Frontend

- `Promise.race` is the standard pattern for **request timeouts**: `race([fetch(url), timeout(5000)])`. It's how you keep a page from hanging on a slow API.
- `Promise.allSettled` is for **partial failure tolerance**: prefetching 6 product images where one 404s shouldn't nuke the other five; loading multiple widget data sources and rendering each that succeeds with an error state for the ones that fail.
- Interviewers use this to check you understand **settlement semantics, ordering guarantees, and how `then` adoption works** — all things that show up in real async UI code.

## Approach

Both are thin wrappers over the native `Promise` constructor:

1. Normalize every input with `Promise.resolve(item)` — this adopts non-promise values **and thenables** automatically, exactly like the native implementations.
2. `race`: attach the outer promise's `resolve`/`reject` directly as the `then` handlers. The first handler to fire wins; subsequent calls are ignored (a settled promise ignores later `resolve`/`reject`).
3. `allSettled`: pre-allocate a results array (preserves input order regardless of completion order), count settlements, and resolve the outer promise only when the counter reaches the input length. An empty iterable resolves immediately with `[]`.

## Complete Solution

```ts
/**
 * Polyfill for Promise.race.
 * Resolves/rejects with the outcome of the first settled input.
 */
function race<T>(iterable: Iterable<T | PromiseLike<T>>): Promise<T> {
  return new Promise<T>((resolve, reject) => {
    for (const item of iterable) {
      // Promise.resolve adopts values AND thenables; if item is a thenable
      // that misbehaves, its handlers still drive resolve/reject correctly.
      Promise.resolve(item).then(resolve, reject);
    }
    // Note: an empty iterable never settles, matching native behavior.
  });
}

/**
 * Polyfill for Promise.allSettled.
 * Always fulfills with an array of per-promise outcome descriptors.
 */
type SettledResult<T> =
  | { status: "fulfilled"; value: T }
  | { status: "rejected"; reason: unknown };

function allSettled<T>(
  iterable: Iterable<T | PromiseLike<T>>
): Promise<SettledResult<T>[]> {
  const items = Array.from(iterable);

  return new Promise<SettledResult<T>[]>((resolve) => {
    if (items.length === 0) {
      resolve([]);
      return;
    }

    const results = new Array<SettledResult<T>>(items.length);
    let settledCount = 0;

    items.forEach((item, index) => {
      Promise.resolve(item).then(
        (value) => {
          results[index] = { status: "fulfilled", value };
          settledCount += 1;
          if (settledCount === items.length) resolve(results);
        },
        (reason) => {
          results[index] = { status: "rejected", reason };
          settledCount += 1;
          if (settledCount === items.length) resolve(results);
        }
      );
    });
  });
}
```

### Usage in frontend code

```js
// 1. Timeout wrapper around fetch
function fetchWithTimeout(url, ms = 8000) {
  const timeout = new Promise((_, reject) =>
    setTimeout(() => reject(new Error("Request timed out")), ms)
  );
  return race([fetch(url), timeout]);
}

// 2. Bulk load with per-item error states
async function loadDashboardWidgets(widgets) {
  const outcomes = await allSettled(
    widgets.map((w) => fetch(`/api/widgets/${w.id}`).then((r) => r.json()))
  );
  return widgets.map((w, i) =>
    outcomes[i].status === "fulfilled"
      ? { ...w, data: outcomes[i].value }
      : { ...w, error: outcomes[i].reason }
  );
}
```

## Complexity

- **Time:** O(n) — one `.then` attachment per input promise. The underlying work is driven by the promises themselves, not by the combinator.
- **Space:** O(n) — the results array in `allSettled` (one entry per input). `race` uses O(1) extra space.

## Edge Cases

| Case | `race` | `allSettled` |
|---|---|---|
| Empty iterable | Never settles (matches native) | Resolves `[]` |
| Non-promise values | Adopted as resolved values via `Promise.resolve` | Same |
| Thenables (custom `.then`) | Adopted; a hostile thenable that calls back twice still settles once (first wins) | Same |
| Already-settled promises | First attached handler in iteration order wins | All recorded in input order |
| Rejection | First rejection wins, even if an earlier-started promise would resolve later | Recorded as `{ status: "rejected", reason }`; the returned promise never rejects |
| Sparse array input (`new Array(3)`) | `for..of` yields `undefined` holes as `undefined` → resolves `undefined` on first tick | `Array.from` converts holes to `undefined` → three fulfilled entries |
| `iterable` throws on iteration | The `new Promise` executor catches it → returned promise rejects | Same |
