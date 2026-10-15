# Deep Clone an Object (handle circular refs)

## Problem

Implement a function `deepClone<T>(value: T): T` that produces a **deep copy**
of any JavaScript value: every nested object, array, `Map`, `Set`, `Date`,
`RegExp`, and typed array is duplicated so mutating the clone never affects
the original. The clone must **not recurse infinitely on circular references**
(e.g. `obj.self = obj`) and must **preserve shared references** — if two
properties point to the same object, the clone must point them at the same
cloned object.

## Why it matters for frontend

- **Immutable state updates** (Redux / Zustand / React `useState`): snapshotting
  a subtree before an optimistic mutation lets you roll back cleanly.
- **Form editing**: clone draft form state so "Cancel" restores the pristine
  model without keeping a live reference to it.
- **Undo/redo**: cloning editor state at each step is the simplest history model
  (and why `structuredClone` / Immer exist).

## Approach

1. Primitives, `null`, and functions are returned as-is (functions are not
   meaningfully "cloned"; interviewers expect reference-copy or a thrown error —
   I keep them by reference and note the tradeoff).
2. Walk the graph recursively (iterative BFS also works). Keep a `WeakMap`
   from **original → clone**. Before recursing into an object, check the map:
   - If present, return the existing clone (this handles circular refs *and*
     shared references).
   - If absent, create the empty clone first, store it in the map, *then*
     populate it. Registering before populating is what breaks the cycle.
3. Handle special built-ins by value: `Date` → `new Date(t)`, `RegExp` →
   `new RegExp(source, flags)` (plus `lastIndex`), `Map`/`Set` → new
   containers with cloned entries, `ArrayBuffer`/typed arrays → `.slice()`.
4. Preserve the prototype and copy own property descriptors (covers getters,
   non-enumerable keys, and symbol keys — the reason a naive `for...in` /
   `Object.assign` loop is wrong).

## Solution (TypeScript)

```ts
export function deepClone<T>(value: T, seen = new WeakMap<object, unknown>()): T {
  // 1. Primitives and functions need no cloning.
  if (value === null || typeof value !== "object") {
    return value;
  }
  // 2. Circular / shared references: return the already-created clone.
  if (seen.has(value)) {
    return seen.get(value) as T;
  }

  let clone: unknown;

  if (value instanceof Date) {
    clone = new Date(value.getTime());
  } else if (value instanceof RegExp) {
    clone = new RegExp(value.source, value.flags);
    (clone as RegExp).lastIndex = value.lastIndex;
  } else if (value instanceof Map) {
    clone = new Map();
    seen.set(value, clone);
    for (const [k, v] of value) {
      (clone as Map<unknown, unknown>).set(deepClone(k, seen), deepClone(v, seen));
    }
    return clone as T;
  } else if (value instanceof Set) {
    clone = new Set();
    seen.set(value, clone);
    for (const v of value) {
      (clone as Set<unknown>).add(deepClone(v, seen));
    }
    return clone as T;
  } else if (ArrayBuffer.isView(value)) {
    // TypedArray / DataView — copy the underlying bytes.
    if (value instanceof DataView) {
      clone = new DataView(value.buffer.slice(0));
    } else {
      clone = (value as unknown as { slice(): unknown }).slice();
    }
  } else if (Array.isArray(value)) {
    clone = [];
  } else {
    // Plain objects AND class instances: keep the prototype so
    // instanceof checks still pass on the clone.
    clone = Object.create(Object.getPrototypeOf(value));
  }

  // 3. Register BEFORE populating — this is what breaks cycles.
  seen.set(value, clone);

  // 4. Copy every own key, including symbols and non-enumerables.
  for (const key of Reflect.ownKeys(value)) {
    const descriptor = Object.getOwnPropertyDescriptor(value, key)!;
    if ("value" in descriptor) {
      descriptor.value = deepClone(descriptor.value, seen);
    }
    Object.defineProperty(clone, key, descriptor);
  }

  return clone as T;
}
```

## Complexity

- **Time:** O(n), where n is the total number of reachable values — each is
  visited exactly once (the `WeakMap` lookup is O(1) amortized).
- **Space:** O(n) for the clone itself + O(n) for the `WeakMap` + O(d) call
  stack, where d is max depth (an iterative version with an explicit stack
  removes the recursion limit).

## Edge cases

- **Circular:** `const a = { x: 1 }; a.self = a;` → `deepClone(a).self === clone` ✓
- **Shared refs:** `const o = {}; const src = { p: o, q: o };` → `clone.p === clone.q` ✓
- **Diamond + cycles through Maps/Sets** — handled because Map/Set entries go
  through the same `seen` map.
- **`Date` inside an object** — cloned by value, not left as a shared reference.
- **Class instances** — prototype preserved, so methods/`instanceof` survive.
- **Getters** — descriptor copied, not eagerly evaluated; the getter runs on the
  clone with the clone as `this`.
- **Functions** — returned by reference (documented choice; `structuredClone`
  throws on them instead — name-drop this as the platform alternative).
- **Very deep graphs** — recursion can overflow the stack; mention the iterative
  variant as the follow-up.

**Interviewer follow-ups worth naming:** how `structuredClone` differs (handles
  more built-ins, lives in the platform, but throws on functions and DOM nodes),
  and why a `WeakMap` (not `Map`) is used — so cloned graphs don't pin the
  originals in memory.
