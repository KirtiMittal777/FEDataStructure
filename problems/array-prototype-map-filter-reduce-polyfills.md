# Array.prototype.map / filter / reduce Polyfills

## Problem

Implement `Array.prototype.myMap`, `myFilter`, and `myReduce` from scratch,
matching the behavior of the native methods closely enough to pass a
spec-driven interview check:

- `myMap(callback, thisArg?)` — returns a new array with `callback` applied
  to every element.
- `myFilter(callback, thisArg?)` — returns a new array of only the elements
  for which `callback` returns truthy.
- `myReduce(callback, initialValue?)` — folds the array into a single value.

## Why it matters for frontend

Every React render pipeline is a map/filter/reduce in disguise: transforming
API payloads into view models, filtering lists by search state, and deriving
totals, grouped sections, or form validity from arrays. Interviewers ask for
these polyfills because they expose whether you understand callbacks, `this`
binding, sparse arrays, and mutation discipline — the exact things that cause
real bugs in list rendering and derived state.

## Approach

For each method, follow the spec's core rules rather than the happy path:

1. **Validate the receiver.** Throw a `TypeError` if called on `null` or
   `undefined`; coerce the receiver to an object and its `length` to a
   non-negative integer (`ToLength`).
2. **Validate the callback.** It must be callable, otherwise `TypeError`.
3. **Respect `thisArg`.** Invoke the callback with `callback.call(thisArg,
   value, index, array)` so the caller's `this` is honored.
4. **Handle holes.** Native methods skip holes in sparse arrays for
   `map`/`filter` (they never invoke the callback for a missing index) and
   for `reduce` (holes are skipped, not treated as `undefined`). `map`
   preserves holes in the result; `filter` compacts them away.
5. **Snapshot the length once.** Elements appended during iteration are not
   visited; elements deleted before their turn are skipped.
6. **`reduce` without an initial value** uses the first *present* element as
   the accumulator and starts from the next index. Reducing an empty array
   (or an all-hole array) without an initial value throws a `TypeError`.

## Solution

```js
// Shared helpers: spec-faithful length coercion and "is this index present?"
function toLength(value) {
  const len = Number(value);
  if (Number.isNaN(len) || len <= 0) return 0;
  return Math.min(Math.floor(len), 2 ** 53 - 1);
}

function assertCallable(fn) {
  if (typeof fn !== 'function') {
    throw new TypeError(`${String(fn)} is not a function`);
  }
}

Array.prototype.myMap = function (callback, thisArg) {
  if (this == null) {
    throw new TypeError('Array.prototype.myMap called on null or undefined');
  }
  assertCallable(callback);
  const O = Object(this);
  const len = toLength(O.length);
  const result = new Array(len); // holes preserved: we only fill visited indices
  for (let k = 0; k < len; k++) {
    if (k in O) {
      result[k] = callback.call(thisArg, O[k], k, O);
    }
  }
  return result;
};

Array.prototype.myFilter = function (callback, thisArg) {
  if (this == null) {
    throw new TypeError('Array.prototype.myFilter called on null or undefined');
  }
  assertCallable(callback);
  const O = Object(this);
  const len = toLength(O.length);
  const result = [];
  for (let k = 0; k < len; k++) {
    if (k in O && callback.call(thisArg, O[k], k, O)) {
      result.push(O[k]); // compacted: no holes survive a filter
    }
  }
  return result;
};

Array.prototype.myReduce = function (callback, initialValue) {
  if (this == null) {
    throw new TypeError('Array.prototype.myReduce called on null or undefined');
  }
  assertCallable(callback);
  const O = Object(this);
  const len = toLength(O.length);
  let k = 0;
  let accumulator;
  if (arguments.length >= 2) {
    accumulator = initialValue;
  } else {
    // Find the first present element to seed the accumulator.
    while (k < len && !(k in O)) k++;
    if (k >= len) {
      throw new TypeError('Reduce of empty array with no initial value');
    }
    accumulator = O[k++];
  }
  for (; k < len; k++) {
    if (k in O) {
      accumulator = callback(accumulator, O[k], k, O);
    }
  }
  return accumulator;
};
```

### Worked examples

```js
[1, 2, 3].myMap((x) => x * 2);                    // [2, 4, 6]
[1, 2, 3, 4].myFilter((x) => x % 2 === 0);        // [2, 4]
[1, 2, 3, 4].myReduce((sum, x) => sum + x, 0);     // 10
[1, 2, 3].myReduce((sum, x) => sum + x);           // 6 (no initial value)
const sparse = [1, , 3];                            // hole at index 1
sparse.myMap((x) => x * 10);                       // [10, <hole>, 30]
sparse.myFilter(() => true);                        // [1, 3]

// Frontend-shaped usage: derive a rendered list + a total from one payload.
const products = [
  { id: 1, name: 'Mug', price: 12, inStock: true },
  { id: 2, name: 'Lamp', price: 40, inStock: false },
  { id: 3, name: 'Book', price: 18, inStock: true },
];
const cards = products
  .myFilter((p) => p.inStock)
  .myMap((p) => ({ ...p, label: `${p.name} — $${p.price}` }));
const total = cards.myReduce((sum, p) => sum + p.price, 0); // 30
```

## Complexity

- **Time:** O(n) per method, where n is the array length — one pass, one
  callback invocation per present element. (Chaining `filter` then `map`
  costs two passes; for hot lists, a single `reduce` that filters and maps
  in one pass is the standard optimization.)
- **Space:** O(n) for `map` (a new array of length n) and O(m) for `filter`
  (m = number of kept elements); `reduce` is O(1) beyond the accumulator.

## Edge cases

- Called on `null`/`undefined` → `TypeError`; on array-likes (e.g.
  `{ 0: 'a', 1: 'b', length: 2 }`) it works, because we coerce via `Object`
  and `toLength` — the same reason `Array.prototype.map.call` works on
  `arguments` and `NodeList`.
- Non-function callback → `TypeError`, matching native behavior.
- Sparse arrays: holes are never passed to the callback; `map` keeps them,
  `filter` drops them, `reduce` skips them (including when seeding the
  accumulator).
- Mutation during iteration: the length is snapshotted up front, so pushes
  during the loop are not visited; a deleted index is skipped via the
  `k in O` check.
- `reduce` on an empty array with an initial value returns that value; with
  no initial value it throws — a classic interview trap.
- `thisArg` is honored via `callback.call(...)`; arrow-function callbacks
  ignore it, exactly as with the native methods.
- Never mutate the input array: `map`/`filter` return new arrays, and
  `reduce` only reads. Mutating props/state arrays inside a callback is
  the bug these polyfills are meant to teach you to avoid.
