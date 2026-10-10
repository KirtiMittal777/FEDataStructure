# LRU Cache — Map-Based, O(1) get/put (think: client-side caching)

Implement a Least Recently Used (LRU) cache: a key-value store with a fixed
capacity that evicts the *least recently used* entry when full. Both `get` and
`put` must run in O(1).

## Problem statement

Design a class `LRUCache` with:

- `constructor(capacity: number)` — maximum number of entries held.
- `get(key: number): number` — return the value if the key exists (and mark it
  as most recently used), otherwise return `-1`.
- `put(key: number, value: number): void` — insert or update the key and mark
  it most recently used. If the cache is full, evict the least recently used
  entry first.

Constraints: `get` and `put` must each be **O(1)** average time.

## Why it matters for frontend

- **Client-side caching:** API responses, React Query / SWR staleness
  strategies, and service-worker caches all decide "what do I keep and what do
  I throw away" — eviction policy is the core of that.
- **Memoization layers:** component prop caches, computed-selector caches
  (e.g. Reselect), and `useMemo`-style stores are bounded caches where recency
  is the eviction signal.
- **Image / route prefetch budgets:** browsers and apps cap how many images,
  fonts, or prefetched route bundles they hold; LRU is the classic replacement
  policy.
- Interviewers often follow up with: "wrap this in an `async` memoizer for
  fetch calls" — knowing the underlying structure cold is what makes that
  follow-up easy.

## Approach

The trick is combining two structures in one:

1. **A hash map** (`key → node`) for O(1) lookup of any entry.
2. **A doubly-linked list** for O(1) eviction-order bookkeeping: the head is
   the most recently used, the tail the least. On every `get`/`put`, the
   touched node is unlinked and moved to the head — pointer surgery, no
   traversal.

JavaScript gives us a shortcut: `Map` **preserves insertion order**, so we can
delete-and-re-insert a key to move it to the "most recent" end. That yields a
compact O(1) implementation without hand-rolled pointers — say this explicitly
in the interview, then (optionally) show you could also do the classic
doubly-linked-list version.

## Solution (JavaScript/TypeScript)

```ts
class LRUCache {
  private capacity: number;
  private cache = new Map<number, number>();

  constructor(capacity: number) {
    if (!Number.isInteger(capacity) || capacity <= 0) {
      throw new Error("capacity must be a positive integer");
    }
    this.capacity = capacity;
  }

  get(key: number): number {
    if (!this.cache.has(key)) return -1;
    // Refresh recency: delete + re-insert moves key to the end (most recent).
    const value = this.cache.get(key)!;
    this.cache.delete(key);
    this.cache.set(key, value);
    return value;
  }

  put(key: number, value: number): void {
    // If the key already exists, remove it first so re-insertion refreshes it.
    if (this.cache.has(key)) {
      this.cache.delete(key);
    } else if (this.cache.size >= this.capacity) {
      // Map iteration order is insertion order — the FIRST key is the LRU.
      const lruKey = this.cache.keys().next().value!;
      this.cache.delete(lruKey);
    }
    this.cache.set(key, value);
  }

  /** O(1) helpers interviewers love to see — never re-derive size from keys. */
  get size(): number {
    return this.cache.size;
  }

  has(key: number): boolean {
    return this.cache.has(key);
  }

  clear(): void {
    this.cache.clear();
  }
}

// Sanity check:
const c = new LRUCache(2);
c.put(1, 1); // cache: {1=1}
c.put(2, 2); // cache: {1=1, 2=2}
console.log(c.get(1)); // 1   → cache: {2=2, 1=1}  (1 is now MRU)
c.put(3, 3); // evicts 2 → cache: {1=1, 3=3}
console.log(c.get(2)); // -1
c.put(4, 4); // evicts 1 → cache: {3=3, 4=4}
console.log(c.get(1)); // -1
console.log(c.get(3)); // 3
console.log(c.get(4)); // 4
```

**The classic doubly-linked-list version** (mention it; write it only if asked —
interviews at FAANG often accept the `Map` version when you explain the order
guarantee):

```ts
class ListNode {
  key: number;
  value: number;
  prev: ListNode | null = null;
  next: ListNode | null = null;
  constructor(key: number, value: number) {
    this.key = key;
    this.value = value;
  }
}

// Head = MRU sentinel, tail = LRU sentinel. Touching a node = unlink + insert
// after head; evicting = remove node before tail and delete its map entry.
```

## Time / space complexity

- `get`: **O(1)** — map lookup + delete + re-insert (all O(1) amortized).
- `put`: **O(1)** — at most one delete (for eviction) + one insert.
- Space: **O(capacity)** — every key stored exactly once in the map.
- The `Map`-iteration-order trick makes eviction O(1) without extra pointers;
  the doubly-linked-list variant is identical asymptotically.

## Edge cases

- **capacity = 0 or negative** → throw on construction (fail fast, don't
  silently evict everything on every put).
- **`put` with an existing key** → updates the value *and* refreshes recency;
  forgetting the delete-first step is the classic bug (it would evict the
  wrong entry later).
- **Eviction target when full** → the first key in iteration order is the LRU
  *before* inserting the new key; ordering matters — evict, then insert.
- **`get` on a missing key** → returns `-1` per spec, and must NOT alter
  recency or insert anything.
- **Non-integer keys** (e.g. strings, objects) → `Map` handles any key type;
  note that object keys compare by reference, a real concern in frontend
  memoizers.
- **Concurrency** (frontend-flavored follow-up): two in-flight `put`s for the
  same key race — for a fetch cache, store pending promises instead of values
  so duplicate requests share one flight.
- **Eviction callbacks** (real-world caches): emit an event or call a hook on
  eviction so consumers (e.g. image placeholders) can clean up — don't just
  drop entries silently.
