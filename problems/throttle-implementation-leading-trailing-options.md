# Throttle Implementation (Leading + Trailing Options)

## Problem Statement

Implement a `throttle(fn, wait, options)` utility that guarantees `fn` is
invoked **at most once per `wait` milliseconds**, no matter how often the
throttled function is called. Support `leading` and `trailing` edge options
and expose `cancel()` / `flush()`, mirroring lodash's `throttle`.

```ts
interface ThrottleOptions { leading?: boolean; trailing?: boolean }

declare function throttle<F extends (...args: any[]) => any>(
  fn: F,
  wait: number,
  options?: ThrottleOptions
): F & { cancel(): void; flush(): ReturnType<F> | undefined };
```

Requirements:
1. Default (`leading: true, trailing: true`): the first call in a burst fires
   immediately; further calls within `wait` ms are coalesced into one trailing
   invocation with the latest arguments.
2. `leading: false`: no immediate invocation; the first call is deferred to the
   trailing edge.
3. `trailing: false`: calls within the window are dropped entirely.
4. `cancel()` drops any pending trailing invocation; `flush()` invokes it now.
5. Preserve `this` and return the last computed result.

## Why It Matters for Frontend

Throttle is the twin of debounce and the right tool whenever you need a
**steady rate** rather than quiet-time detection: scroll handlers feeding
sticky-header logic, `resize` listeners re-laying-out a canvas, mousemove
tracking, and API polling guards. Interviewers ask throttle right after
debounce to check you understand the distinction.

## Approach

Track `lastInvokeTime` (last time `fn` actually ran), a pending timer, and the
latest args/this.

- If `leading` and enough time has passed since the last invocation → invoke
  immediately.
- Otherwise, if `trailing`, schedule (or keep) a timer for the remainder of the
  window; when it fires, invoke with the latest captured args.
- `cancel()` clears the timer and resets state; `flush()` runs the pending
  trailing call immediately.

## Complete Solution (TypeScript)

```ts
type AnyFn = (...args: any[]) => any;

interface ThrottleOptions {
  leading?: boolean;
  trailing?: boolean;
}

export interface Throttled<F extends AnyFn> {
  (...args: Parameters<F>): ReturnType<F> | undefined;
  cancel(): void;
  flush(): ReturnType<F> | undefined;
}

export function throttle<F extends AnyFn>(
  fn: F,
  wait: number,
  options: ThrottleOptions = {}
): Throttled<F> {
  const { leading = true, trailing = true } = options;

  let timerId: ReturnType<typeof setTimeout> | undefined;
  let lastArgs: any[] | undefined;
  let lastThis: unknown;
  let lastInvokeTime = 0; // 0 = "never invoked" (leading fires on first call)
  let lastResult: ReturnType<F> | undefined;

  function invoke(time: number): ReturnType<F> | undefined {
    lastInvokeTime = time;
    const args = lastArgs;
    const context = lastThis;
    lastArgs = undefined;
    lastThis = undefined;
    if (args !== undefined) {
      lastResult = fn.apply(context, args);
    }
    return lastResult;
  }

  function remainingWait(time: number): number {
    return wait - (time - lastInvokeTime);
  }

  function timerExpired(): ReturnType<F> | undefined {
    const time = Date.now();
    if (trailing && lastArgs !== undefined) {
      // Time's up and there's a pending call → trailing-edge invocation.
      timerId = undefined;
      return invoke(time);
    }
    timerId = undefined;
    lastArgs = undefined;
    lastThis = undefined;
    return lastResult;
  }

  function cancel(): void {
    if (timerId !== undefined) clearTimeout(timerId);
    timerId = undefined;
    lastArgs = undefined;
    lastThis = undefined;
    lastInvokeTime = 0;
  }

  function flush(): ReturnType<F> | undefined {
    if (timerId === undefined) return lastResult;
    return timerExpired();
  }

  function throttled(this: unknown, ...args: Parameters<F>): ReturnType<F> | undefined {
    const time = Date.now();
    const timeSinceLastInvoke = time - lastInvokeTime;
    const isFirstCall = lastInvokeTime === 0;

    lastArgs = args;
    lastThis = this;

    // Leading edge: fire now if the window has fully elapsed.
    if (leading && (isFirstCall || timeSinceLastInvoke >= wait)) {
      if (timerId !== undefined) {
        clearTimeout(timerId);
        timerId = undefined;
      }
      return invoke(time);
    }

    // Otherwise coalesce into a trailing call at the end of the window.
    if (trailing && timerId === undefined) {
      const delay = isFirstCall ? wait : Math.max(0, remainingWait(time));
      timerId = setTimeout(timerExpired, delay);
    }

    return lastResult;
  }

  throttled.cancel = cancel;
  throttled.flush = flush;
  return throttled as Throttled<F>;
}
```

Usage:

```ts
const onScroll = throttle(() => updateStickyHeader(), 100);
window.addEventListener("scroll", onScroll, { passive: true });
// Cleanup:
window.removeEventListener("scroll", onScroll);
onScroll.cancel();
```

## Complexity

- Time: O(1) per call — timer bookkeeping and arg capture only.
- Space: O(1) extra (one timer + one args array per throttled function).

## Edge Cases & Interview Follow-ups

1. **Debounce vs throttle**: debounce waits for a *quiet period* then fires
   once; throttle enforces a *maximum rate* and fires steadily during a burst.
   Search-as-you-type → debounce; scroll/resize handlers → throttle.
2. **`leading: false` first call**: with `lastInvokeTime = 0`, the first call
   would look "idle" — but because `leading` is false we skip the immediate
   invoke and just schedule the trailing timer for a full `wait`.
3. **`trailing: false`**: burst calls inside the window only refresh
   `lastArgs`; with no trailing timer nothing fires until the next call after
   the window — a pure rate limiter.
4. **Timer fires early**: `timerExpired` checks `lastArgs` rather than trusting
   the clock; if there's no pending call it just clears state.
5. **`wait = 0`**: degrades to "invoke at most once per task" — the leading
   call fires, and a trailing call is scheduled via `setTimeout(0)`.
6. **React usage**: create the throttled function once (`useMemo`/`useRef`) and
   `cancel()` on unmount; never recreate it per render or the rate limiting
   resets.
7. **Clock jumps**: like debounce, a backward system-clock jump can stretch the
   window; `Math.max(0, ...)` prevents negative timeouts.
8. **`requestAnimationFrame` alternative**: for scroll-driven visual updates,
   rAF-throttling (fire at most once per frame) is often better than
   time-based throttling — mention it as the follow-up.
