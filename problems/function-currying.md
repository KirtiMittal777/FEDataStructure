# Function Currying

## Problem Statement

Implement a `curry` function that converts a multi-argument function into a
sequence of functions, each taking a single argument (or a partial group of
arguments), until all arguments are supplied and the original function is
invoked.

```js
function add(a, b, c) { return a + b + c; }
const curriedAdd = curry(add);

curriedAdd(1)(2)(3);   // 6
curriedAdd(1, 2)(3);   // 6  (supports partial application with multiple args)
curriedAdd(1)(2, 3);   // 6
```

## Why It Matters for Frontend

Currying shows up constantly in real frontend code, even when nobody calls it
that:

- **Event handlers:** `onChange={handleChange('email')}` — a curried handler
  factory that closes over the field name and returns the actual listener.
- **Higher-order components / hooks composition:** middleware-style pipelines
  (`compose`, `pipe`) rely on functions that return functions.
- **Config-first APIs:** libraries like styled-components (`styled.div\`...\``),
  Yup (`string().min(3)`), and i18n formatters (`t('key')(params)`) are built
  on partial application — fix the config now, supply the data later.
- **Interview signal:** it tests closures, `Function.prototype.length`
  (arity), recursion, rest/spread, and `this`-binding in one question.

## Approach

1. Read the target function's arity via `fn.length`.
2. Return a wrapper that collects arguments across calls. On each call, merge
   previously collected args with the new ones.
3. If the merged argument count is **≥ arity**, invoke the original function
   with all collected arguments and return its result.
4. Otherwise, return a new wrapper that remembers the accumulated arguments
   (recursion via closure).
5. Edge decisions:
   - Default parameters are not counted in `fn.length` — acceptable and worth
     mentioning in an interview.
   - Forward `this` so the curried function works as a method.
   - Extra arguments beyond arity are passed through (matches how `add(1,2,3,4)`
     ignores the 4th).
   - Zero-arity functions (`() => 42`) return the result immediately on the
     first call.

## Solution (TypeScript)

```ts
type Curried<F extends (...args: any[]) => any> =
  F extends (first: infer A, ...rest: infer R) => infer Ret
    ? R extends []
      ? (first: A) => Ret
      : (first: A, ...rest: any[]) => Ret | Curried<(...args: R) => Ret>
    : never;

/**
 * Curries a function of known arity. Accumulated arguments are gathered
 * across calls; when arity is reached the original function is invoked.
 */
export function curry<F extends (...args: any[]) => any>(fn: F): any {
  const arity = fn.length;

  function curried(this: unknown, ...args: any[]): any {
    if (args.length >= arity) {
      return fn.apply(this, args);
    }
    return function (this: unknown, ...nextArgs: any[]): any {
      return curried.apply(this, [...args, ...nextArgs]);
    };
  }

  return curried;
}

// ---- Usage ----

function add(a: number, b: number, c: number): number {
  return a + b + c;
}
const curriedAdd = curry(add);

console.log(curriedAdd(1)(2)(3));  // 6
console.log(curriedAdd(1, 2)(3));  // 6
console.log(curriedAdd(1)(2, 3));  // 6
console.log(curriedAdd(1, 2, 3));  // 6

// Real frontend use: handler factory
const handleFieldChange = curry(
  (field: string, e: React.ChangeEvent<HTMLInputElement>) => {
    console.log(field, e.target.value);
  }
);
// <input onChange={handleFieldChange('email')} />
```

A stricter variant that rejects "too many" accumulated arguments instead of
passing them through (some interviewers ask for this):

```ts
export function strictCurry<F extends (...args: any[]) => any>(fn: F): any {
  const arity = fn.length;

  function curried(this: unknown, ...args: any[]): any {
    if (args.length === arity) {
      return fn.apply(this, args);
    }
    if (args.length > arity) {
      throw new Error(`Too many arguments: expected ${arity}, got ${args.length}`);
    }
    return (...nextArgs: any[]) => curried.apply(this, [...args, ...nextArgs]);
  }

  return curried;
}
```

## Complexity

- **Time:** O(1) per curried call (argument concatenation is O(k) in the number
  of accumulated args, bounded by the arity, so effectively constant). The final
  call costs whatever the wrapped function costs.
- **Space:** O(n) for the closure chain, where n is the arity — each partial
  call holds one wrapper and one accumulated-args array until invocation.

## Edge Cases

| Case | Behavior |
|---|---|
| `fn.length === 0` (e.g. `() => 42`) | `curried()` invokes immediately on first call — correct, since arity 0 is already satisfied |
| Default params: `(a, b = 2, c) => ...` | `fn.length` is 1; curry treats it as 1-arity. Mention this — it's the standard gotcha |
| Rest params: `(...args) => ...` | `fn.length` is 0; falls into the zero-arity case above |
| Extra args beyond arity | Passed through to `fn` (matches normal JS call semantics) |
| `this` binding | `fn.apply(this, args)` forwards `this`, so curried functions work as object methods |
| Reusing a partial | `const add1 = curriedAdd(1); add1(2)(3); add1(5)(6);` — each branch of the closure tree accumulates independently; no shared mutable state |

## Interview Follow-ups

1. *"What if the function has a variable number of arguments?"* — You can't
   know when to stop collecting, so provide an explicit terminator (e.g. call
   with no arguments: `curriedAdd(1)(2)(3)()`), or require the caller to state
   the arity: `curry(fn, arity)`.
2. *"How is currying different from partial application?"* — Partial
   application fixes *some* args and returns a function for the rest in one
   step (`bind` does this). Currying decomposes to unary steps and can be
   partially applied repeatedly.
3. *"Where have you used this in production?"* — Event-handler factories
   (`handleChange(field)`), Redux `mapDispatchToProps` wrappers, and
   config-first API clients (`apiClient(baseURL)(endpoint)`).
