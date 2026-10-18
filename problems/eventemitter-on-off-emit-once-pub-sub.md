# EventEmitter — on/off/emit/once (pub-sub)

## Problem

Implement a minimal, robust publish–subscribe class:

```ts
emitter.on(event, listener)   // subscribe; returns an unsubscribe function
emitter.once(event, listener) // subscribe for exactly one emission; returns an unsubscribe function
emitter.off(event, listener?) // unsubscribe one listener, or all listeners for the event if listener omitted
emitter.emit(event, ...args)  // invoke all listeners for the event with the given args
```

Requirements:

- Listeners fire in registration order.
- A `once` listener fires at most one time, even across multiple emissions.
- `emit` must be safe if a listener adds or removes listeners mid-emission.
- `off`/`emit` for unknown events are no-ops (never throw).

## Why it matters for frontend

Pub-sub is the backbone of decoupled frontend code: a single WebSocket connection
dispatching typed messages to interested modules, a global event bus for
analytics/telemetry, cross-widget communication without prop drilling, and
store subscribers in state-management libraries. Interviewers also use it to
test your grasp of closures, iteration safety, and API design.

## Approach

- Keep `Map<event, Set<listener>>` — a `Set` makes `off` O(1) and naturally
  dedupes the same listener registered twice.
- `once` wraps the listener in a closure that removes itself before calling the
  original — self-removal first guarantees "at most once" even if the listener
  throws or re-emits.
- `emit` snapshots listeners into an array before invoking, so add/remove during
  emission doesn't skip or double-call listeners.
- Return an unsubscribe closure from `on`/`once` — the modern ergonomic API
  (`useEffect(() => emitter.on('tick', handler), [])` works naturally).

## Solution (TypeScript)

```ts
type Listener<T extends unknown[] = unknown[]> = (...args: T) => void;

export class EventEmitter {
  private events = new Map<string, Set<Listener>>();

  on(event: string, listener: Listener): () => void {
    let listeners = this.events.get(event);
    if (!listeners) {
      listeners = new Set();
      this.events.set(event, listeners);
    }
    listeners.add(listener);
    return () => this.off(event, listener);
  }

  once(event: string, listener: Listener): () => void {
    const wrapper: Listener = (...args) => {
      this.off(event, wrapper); // remove BEFORE calling: guarantees at-most-once
      listener(...args);
    };
    return this.on(event, wrapper);
  }

  off(event: string, listener?: Listener): void {
    if (!this.events.has(event)) return;
    if (listener === undefined) {
      this.events.delete(event);
    } else {
      const listeners = this.events.get(event)!;
      listeners.delete(listener);
      if (listeners.size === 0) this.events.delete(event);
    }
  }

  emit(event: string, ...args: unknown[]): void {
    const listeners = this.events.get(event);
    if (!listeners || listeners.size === 0) return;
    // Snapshot: safe against listeners that subscribe/unsubscribe mid-emit
    for (const listener of [...listeners]) {
      listener(...args);
    }
  }

  listenerCount(event: string): number {
    return this.events.get(event)?.size ?? 0;
  }
}
```

### Quick sanity check

```ts
const bus = new EventEmitter();
const log: number[] = [];
const unsub = bus.on('data', (n: number) => log.push(n));
bus.once('data', (n: number) => log.push(n * 10));
bus.emit('data', 1); // log: [1, 10]
bus.emit('data', 2); // log: [1, 10, 2]
unsub();
bus.emit('data', 3); // log unchanged: [1, 10, 2]

// Mutation during emit: removing a listener inside emit must not skip others
const e2 = new EventEmitter();
const order: string[] = [];
const b = () => order.push('b');
e2.on('x', () => {
  order.push('a');
  e2.off('x', b);
});
e2.on('x', b);
e2.emit('x'); // order: ['a', 'b'] — b still fired this round
```

## Complexity

| Operation | Time |
|---|---|
| `on` / `once` | O(1) amortized |
| `off` | O(1) |
| `emit` | O(n) — n = listeners for the event (plus the snapshot copy) |
| Space | O(total registered listeners) |

## Edge cases

- **Emitting an event with no listeners** — no-op, no throw.
- **`off` with no `listener` argument** — clears the whole event (avoids one-by-one removal boilerplate).
- **Duplicate registration** — `Set` dedupes; registering the same function twice fires it once. (Document this; some implementations use arrays to allow duplicates — a deliberate design tradeoff to mention in interviews.)
- **Listener throws mid-emit** — propagates and aborts remaining listeners. If guaranteed delivery matters, wrap each call in try/catch and aggregate errors (mention as follow-up).
- **`once` + manual `off` before emit** — the returned unsubscribe removes the internal wrapper, so the original never fires. Correct because `on`'s unsubscribe closure closes over the wrapper, not the original.
- **Memory leaks** — the classic EventEmitter footgun. Mitigate with the returned unsubscribe functions and `off(event)` bulk-clear on teardown (e.g., in `useEffect` cleanup).
