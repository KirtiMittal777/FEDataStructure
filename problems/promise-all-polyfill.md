# Promise.all Polyfill

**Category:** DSA (JavaScript) · **Difficulty:** Medium

## Problem statement

Implement `Promise.myAll(promises)` — a polyfill for the native `Promise.all`.

Behavioral contract (match native semantics exactly):

1. Accepts an **iterable** of values (promises, thenables, or plain values).
2. Returns a promise that **resolves with an array of results in the same order as the input**, regardless of which promise settles first.
3. **Rejects as soon as any input rejects** — with that first rejection reason ("fail fast").
4. Resolves to `[]` for an empty iterable.
5. Rejects with a `TypeError` if the argument is not iterable.

## Why it matters for frontend

Dashboards, product pages, and onboarding flows routinely fan out parallel
fetches (`Promise.all([fetchUser(), fetchCart(), fetchRecs()])`). The ordering
guarantee is what makes destructuring (`const [user, cart] = ...`) safe, and
fail-fast is why one slow/broken endpoint kills the whole render unless you
switch to `allSettled`. Interviewers use this to probe whether you understand
promise scheduling, closure capture, and the difference between eager and lazy
evaluation.

## Approach

- Eagerly iterate the input with a counter — inputs start executing as soon as
  the iterable is consumed (native `Promise.all` does not defer).
- Normalize each entry with `Promise.resolve(entry)` so thenables and plain
  values are handled uniformly (a thenable's `then` is adopted).
- Keep a results array indexed by position and a `remaining` counter.
- On each fulfillment, store at its index; when `remaining` hits 0, resolve.
- On the **first** rejection, reject immediately and ignore everything after
  (guard with a `settled` flag so a late fulfillment can't resolve after a
  rejection, and a second rejection can't overwrite the first).
- Note the subtle correctness detail: because a promise can settle
  **synchronously** (already-resolved), we must increment `remaining` in the
  same synchronous loop before attaching handlers — otherwise an
  already-resolved entry could drop `remaining` to 0 prematurely. We handle
  this by first collecting entries into an array.

## Complete solution (TypeScript)

```ts
type Awaitable<T> = T | PromiseLike<T>;

function myAll<T>(iterable: Iterable<Awaitable<T>>): Promise<T[]> {
  return new Promise<T[]>((resolve, reject) => {
    if (iterable == null || typeof (iterable as any)[Symbol.iterator] !== "function") {
      reject(new TypeError("myAll: argument must be an iterable"));
      return;
    }

    const entries = Array.from(iterable);
    const results = new Array<T>(entries.length);
    let remaining = entries.length;
    let settled = false; // ensures fail-fast + exactly one outcome

    if (remaining === 0) {
      resolve([]);
      return;
    }

    entries.forEach((entry, index) => {
      // Adopt thenables / plain values uniformly
      Promise.resolve(entry).then(
        (value) => {
          if (settled) return;
          results[index] = value;
          remaining -= 1;
          if (remaining === 0) {
            settled = true;
            resolve(results);
          }
        },
        (reason) => {
          if (settled) return;
          settled = true;
          reject(reason); // fail fast: first rejection wins
        }
      );
    });
  });
}

// Usage mirrors the native API
const [user, cart, recs] = await myAll([
  fetch("/api/user").then((r) => r.json()),
  fetch("/api/cart").then((r) => r.json()),
  fetch("/api/recs").then((r) => r.json()),
]);
```

## Complexity

- **Time:** O(n) bookkeeping work, where n = number of inputs. Wall-clock time is
  dominated by the slowest input (that's the point of parallelism).
- **Space:** O(n) for the results array plus one closure per input.

## Edge cases

| Case | Expected behavior |
|---|---|
| Empty iterable `myAll([])` | Resolves to `[]` |
| Plain values `myAll([1, "a", null])` | Resolves to `[1, "a", null]` via `Promise.resolve` adoption |
| Foreign thenable `{ then(r) { r(42); } }` | Adopted — resolves to `42` |
| One input rejects | Whole thing rejects with that reason, even if others later resolve |
| Two inputs reject | First rejection reason wins; second is ignored (unhandled-rejection risk if not otherwise handled — same as native) |
| Non-iterable argument | `TypeError` |
| Sparse input array `[p1, , p2]` | Hole becomes `undefined` entry — adopted as a plain value |
| Inputs already settled | Handled — we count `entries.length` before attaching handlers |

## Follow-up talking points (what a FAANG interviewer asks next)

- **Why can't you implement this without closures?** — each `.then` callback
  must remember its own `index`; a `for` loop with `var` would break this.
- **How would you change it for `allSettled`?** — remove fail-fast: count both
  fulfillments and rejections, resolve with `{ status, value | reason }` objects.
- **`race`?** — same shape, but the first *settlement of any kind* wins.
- **Why does native `Promise.all` evaluate eagerly?** — the iterable is consumed
  synchronously, so side-effectful promise constructors already ran before you
  can cancel; there is no cancellation primitive.
