# Two Sum — Hash Map Approach

**Difficulty:** Easy · **Classic asked at:** Meta, Amazon, Google, every frontend interview

## Problem

Given an array of integers `nums` and an integer `target`, return the indices of
the two numbers that add up to `target`. Exactly one solution exists, and the
same element may not be used twice.

```ts
twoSum([2, 7, 11, 15], 9); // [0, 1]
twoSum([3, 2, 4], 6);      // [1, 2]
```

## Why it matters for frontend

This is the canonical "complement lookup" pattern — the same mental model behind
deduplicating event listeners by ID, resolving cached API responses by key, or
pairing related DOM updates in a batch. Interviewers use it to test whether you
reach for a hash map instead of nested loops.

## Approach

Single pass with a `Map` storing `{ value → index }` seen so far. For each
element `x`, check whether `target − x` is already in the map. If yes, return
`[map.get(target − x), i]`; otherwise record `x → i` and continue.

## Solution

```ts
function twoSum(nums: number[], target: number): [number, number] {
  const seen = new Map<number, number>(); // value → index

  for (let i = 0; i < nums.length; i++) {
    const complement = target - nums[i];
    if (seen.has(complement)) {
      return [seen.get(complement)!, i];
    }
    seen.set(nums[i], i);
  }

  throw new Error("No two-sum solution exists");
}

// Sanity checks
console.log(twoSum([2, 7, 11, 15], 9)); // [0, 1]
console.log(twoSum([3, 2, 4], 6));      // [1, 2]
console.log(twoSum([3, 3], 6));         // [0, 1]
```

## Complexity

- **Time:** O(n) — one pass, O(1) average map operations.
- **Space:** O(n) — the map holds at most n entries.

Contrast with the brute-force O(n²) nested loop — the map trades memory for
time, which is almost always the right tradeoff in client code.

## Edge cases

- **Duplicate values** (`[3, 3]`, target 6): works, because we check the map
  *before* inserting the current element — an element is never paired with itself.
- **Negative numbers / zero**: the complement logic is value-agnostic.
- **No solution**: the problem guarantees one; the `throw` is defensive (returning
  `null` is also acceptable — state the choice).
- **Large inputs**: `Map` handles millions of entries; fine for realistic UI data.

## Follow-ups interviewers ask

- "What if the array is sorted?" → two-pointer from both ends, O(1) space.
- "Return all pairs?" → track a list per value; careful not to double-count.
- "What if the input is a stream?" → same single pass, stateful — the map *is*
  your streaming state.
