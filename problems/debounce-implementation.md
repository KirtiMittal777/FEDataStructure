# Debounce Implementation

## Problem statement

Implement `debounce(fn, wait, options?)`, which returns a new function that
delays invoking `fn` until `wait` milliseconds have elapsed since the last
time the debounced function was called. Support optional
`leading`/`trailing` invocation and expose `cancel()` and `flush()` methods.

```ts
const search = debounce(fetchResults, 300);
input.addEventListener("input", (e) => search(e.target.value));
```

## Why it matters for frontend

Debounce is the standard rate-limiting tool for user-driven events:
search-as-you-type, window resize handlers, autosave, and form validation.
FAANG frontend rounds ask for it both as a standalone coding problem and as
the correctness layer under typeahead/autocomplete UI.

## Approach

- Keep a single timer (`setTimeout`) per debounced wrapper. Every call
  resets it — that's the whole debounce idea: the clock restarts on each
  keystroke.
- Track `lastArgs`/`lastThis` so the eventual invocation uses the most
  recent arguments.
- `leading: true` invokes immediately on the first call of a burst, then
  suppresses until the wait window passes.
- `trailing: true` (default) invokes once the burst settles.
- `maxWait` optionally caps how long invocations can be deferred, degrading
  debounce into throttle for sustained input.
- `cancel()` clears the timer and drops pending state. `flush()` invokes
  immediately with the pending args if any.

## Complete solution

```ts
interface DebounceOptions {
  leading?: boolean;
  trailing?: boolean;
  maxWait?: number;
}

interface Debounced<T extends (...args: any[]) => any> {
  (...args: Parameters<T>): void;
  cancel(): void;
  flush(): void;
  pending(): boolean;
}

function debounce<T extends (...args: any[]) => any>(
  fn: T,
  wait: number,
  options: DebounceOptions = {}
): Debounced<T> {
  const { leading = false, trailing = true, maxWait } = options;

  let timerId: ReturnType<typeof setTimeout> | undefined;
  let maxTimerId: ReturnType<typeof setTimeout> | undefined;
  let lastArgs: Parameters<T> | undefined;
  let lastThis: unknown;
  let lastInvokeTime = 0;
  let lastCallTime = 0;

  function invoke(time: number) {
    const args = lastArgs as Parameters<T>;
    lastArgs = undefined;
    lastThis = undefined;
    lastInvokeTime = time;
    fn.apply(lastThis, args);
  }

  function clearTimers() {
    if (timerId !== undefined) clearTimeout(timerId);
    if (maxTimerId !== undefined) clearTimeout(maxTimerId);
    timerId = maxTimerId = undefined;
  }

  function timerExpired() {
    const time = Date.now();
    // If maxWait elapsed since the last actual invoke, force a trailing call.
    if (maxWait !== undefined && time - lastInvokeTime >= maxWait) {
      trailingEdge(time);
      return;
    }
    // Otherwise wait out the remaining quiet period.
    const timeSinceLastCall = time - lastCallTime;
    const remaining = wait - timeSinceLastCall;
    if (remaining <= 0 || remaining > wait) {
      trailingEdge(time);
    } else {
      timerId = setTimeout(timerExpired, remaining);
    }
  }

  function trailingEdge(time: number) {
    clearTimers();
    if (trailing && lastArgs) {
      invoke(time);
    } else {
      lastArgs = lastThis = undefined;
    }
  }

  function debounced(this: unknown, ...args: Parameters<T>): void {
    const time = Date.now();
    const isFirstCall = lastCallTime === 0 && leading;
    lastCallTime = time;
    lastArgs = args;
    lastThis = this;

    if (timerId === undefined) {
      if (isFirstCall) {
        lastInvokeTime = time;
        fn.apply(this, args);
        lastArgs = lastThis = undefined;
      }
      if (maxWait !== undefined && maxTimerId === undefined) {
        maxTimerId = setTimeout(timerExpired, maxWait);
      }
      timerId = setTimeout(timerExpired, wait);
    } else {
      clearTimeout(timerId);
      timerId = setTimeout(timerExpired, wait);
    }
  }

  debounced.cancel = () => {
    clearTimers();
    lastArgs = lastThis = undefined;
    lastCallTime = 0;
    lastInvokeTime = 0;
  };

  debounced.flush = () => {
    if (timerId !== undefined) {
      trailingEdge(Date.now());
    }
  };

  debounced.pending = () => timerId !== undefined;

  return debounced;
}
```

**Simpler default-trailing version** (what most interviews accept):

```ts
function debounceSimple<T extends (...args: any[]) => void>(
  fn: T,
  wait: number
) {
  let timer: ReturnType<typeof setTimeout> | undefined;
  const debounced = (...args: Parameters<T>) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), wait);
  };
  debounced.cancel = () => clearTimeout(timer);
  return debounced;
}
```

## Time / space complexity

- Time: O(1) per call (timer reset + a few assignments); the wrapped
  function itself runs at most once per quiet window.
- Space: O(1) beyond the retained latest arguments.

## Edge cases

- **Arguments freshness:** always invoke with the *last* call's args, not
  the first — stale queries are the classic bug.
- **`this` binding:** use `.apply(lastThis, ...)` so methods keep their
  receiver; arrow-function callers are unaffected.
- **wait = 0:** behaves like deferring to the next macrotask; still
  coalesces synchronous bursts.
- **Component unmount / navigation:** call `cancel()` in the effect
  cleanup, or a debounced fetch may `setState` on an unmounted component.
- **Stale closures in React:** recreate the debounced function when its
  dependencies change (or hold `fn` in a ref) — a stale `fn` silently uses
  old state.
- **leading + trailing both true:** a single isolated call invokes twice
  unless guarded — decide which edge the product needs (typeahead usually
  wants trailing only).
- **Timers in tests:** use fake timers (`vi.useFakeTimers()` / jest) and
  advance time explicitly; real timers make debounce tests flaky.

## Interviewer follow-ups

- "How does this differ from throttle?" → Throttle guarantees a max
  execution rate (one call per window); debounce guarantees a quiet period
  before executing. `maxWait` turns debounce into throttle for continuous
  input.
- "What if the API call itself needs cancelling, not just delaying?" →
  combine with `AbortController`: the debounced function aborts the previous
  in-flight request before firing the new one — this is the race-safety
  layer under typeahead.