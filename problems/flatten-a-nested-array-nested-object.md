# Flatten a Nested Array / Nested Object

## Problem

Write two utilities:

1. `flattenArray(arr)` — flatten a nested array of arbitrary depth into a single-level array.
2. `flattenObject(obj)` — flatten a nested object into a single-level object with dot-separated keys.

```js
flattenArray([1, [2, [3, 4], 5], 6]); // [1, 2, 3, 4, 5, 6]

flattenObject({ a: 1, b: { c: 2, d: { e: 3 } }, f: null });
// { a: 1, 'b.c': 2, 'b.d.e': 3, f: null }
```

## Why it matters for frontend

- API responses are nested; UI often needs flat shapes (e.g. turning a nested category tree into `<option>` lists, or flattening grouped filters into query params).
- Form libraries and analytics payloads normalize nested state to dot-path keys (`"address.city"`).
- Breadcrumb trails, sitemap trees, and comment threads are all "nested → flat with path" problems.

## Approach

**Array:** depth-first traversal. The recursive version is the interview default; the iterative version (explicit stack) avoids call-stack overflow on pathological depth — worth mentioning unprompted.

**Object:** recursive walk, accumulating the key path. Arrays and `null` are treated as leaf values (not recursed into) — a deliberate choice for form-data-style payloads. A `Set` of visited objects guards against circular references.

## Solution (TypeScript)

```ts
/** Flatten a nested array of arbitrary depth (iterative — no recursion limit). */
export function flattenArray<T>(arr: readonly unknown[]): T[] {
  const out: T[] = [];
  // Reversed copy so pop() yields items in original order.
  const stack: unknown[] = [...arr].reverse();
  while (stack.length > 0) {
    const item = stack.pop();
    if (Array.isArray(item)) {
      for (let i = item.length - 1; i >= 0; i--) stack.push(item[i]);
    } else {
      out.push(item as T);
    }
  }
  return out;
}

/** Recursive variant — cleanest to write in an interview; O(depth) call stack. */
export function flattenArrayRecursive<T>(arr: unknown[]): T[] {
  const out: T[] = [];
  for (const item of arr) {
    if (Array.isArray(item)) out.push(...flattenArrayRecursive<T>(item));
    else out.push(item as T);
  }
  return out;
}

type FlatObject = Record<string, unknown>;

/** Flatten a nested object to dot-path keys. Arrays/null are kept as leaf values. */
export function flattenObject(
  obj: Record<string, unknown>,
  prefix = '',
  out: FlatObject = {},
  seen = new Set<object>(),
): FlatObject {
  if (seen.has(obj)) throw new Error(`Circular reference at path "${prefix}"`);
  seen.add(obj);
  for (const [key, value] of Object.entries(obj)) {
    const path = prefix ? `${prefix}.${key}` : key;
    const isPlainObject =
      value !== null &&
      typeof value === 'object' &&
      !Array.isArray(value) &&
      Object.keys(value as object).length > 0;
    if (isPlainObject) {
      flattenObject(value as Record<string, unknown>, path, out, seen);
    } else {
      out[path] = value;
    }
  }
  seen.delete(obj);
  return out;
}

/** Inverse helper — often asked as the follow-up ("unflatten"). */
export function unflattenObject(flat: FlatObject): Record<string, unknown> {
  const out: Record<string, unknown> = {};
  for (const [path, value] of Object.entries(flat)) {
    const keys = path.split('.');
    let node = out;
    keys.forEach((k, i) => {
      if (i === keys.length - 1) node[k] = value;
      else node = (node[k] ??= {}) as Record<string, unknown>;
    });
  }
  return out;
}
```

## Complexity

| Function | Time | Space |
|---|---|---|
| `flattenArray` (either) | O(n), n = total elements incl. nested | O(n) output (+ O(depth) stack) |
| `flattenObject` | O(k), k = total keys incl. nested | O(k) output (+ O(depth) recursion/seen set) |

Both are linear — each element/key is visited exactly once.

## Edge cases

- **Empty inputs:** `flattenArray([])` → `[]`; `flattenObject({})` → `{}`.
- **Sparse arrays:** holes read as `undefined` via iteration — `for...of` yields `undefined` for holes; decide explicitly whether to keep or drop them.
- **`null` / `undefined` values in objects:** kept as leaves (`typeof null === 'object'` is the classic trap — the `value !== null` guard handles it).
- **Empty nested objects:** `{ a: {} }` — treated as a leaf (`{}` assigned to `'a'`) rather than vanishing; document the choice.
- **Circular references:** `seen` set throws a clear error instead of infinite recursion. Alternative: silently skip already-seen nodes — state which you chose and why.
- **Arrays inside objects:** kept intact (`{ tags: ['a', 'b'] }` → `'tags': ['a','b']`). If the interviewer wants bracket paths (`tags[0]`), that's a one-line change in the leaf check.
- **Very deep arrays:** prefer the iterative array version — V8's default call stack (~10k frames) will blow up on deeply nested JSON from an API.
- **Key collisions:** a literal key containing a dot (`{ 'a.b': 1, a: { b: 2 } }`) collides after flattening — call it out; real-world fix is escaping or using array paths (`['a','b']`) instead of strings.
