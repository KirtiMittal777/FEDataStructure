# Valid Parentheses — Stack

**Difficulty:** Easy · **Classic asked at:** Meta, Amazon, Google — often as a warm-up before harder parsing questions

## Problem

Given a string `s` containing bracket characters, determine whether the
brackets are balanced: every opening bracket is closed by the same type of
bracket, in the correct (LIFO) order.

```ts
isValidParentheses("()[]{}"); // true
isValidParentheses("([)]");   // false — interleaved
isValidParentheses("(((");     // false — unclosed
isValidParentheses("");        // true — vacuously valid
```

## Why it matters for frontend

This is the same algorithm a JSX-aware editor runs when it underlines a
mismatched tag, and what template compilers do when they check nesting. If you
can explain it in terms of "push opens, match closes," interviewers trust you
with real parsing tasks — validating HTML fragments in a CMS, linting markup,
or writing a bracket-matching feature for a code playground.

## Approach

Scan left to right with a stack. Push every opening bracket. On a closing
bracket, pop the top of the stack — it must be the matching opener. If the
stack is empty when a closer arrives, or anything is left on the stack at the
end, the string is invalid.

Non-bracket characters are ignored here (so the function can scan raw JSX);
the strict variant is noted under edge cases.

## Solution

```ts
function isValidParentheses(s: string): boolean {
  const pairs: Record<string, string> = { ")": "(", "]": "[", "}": "{" };
  const openers = new Set(["(", "[", "{"]);
  const stack: string[] = [];

  for (const ch of s) {
    if (openers.has(ch)) {
      stack.push(ch);
    } else if (ch in pairs) {
      if (stack.pop() !== pairs[ch]) return false; // mismatch or empty stack
    }
    // other characters: ignored (scanning mode for JSX/template text)
  }

  return stack.length === 0;
}

// Sanity checks
console.log(isValidParentheses("()[]{}"));      // true
console.log(isValidParentheses("{[()]}"));      // true — nested
console.log(isValidParentheses("([)]"));       // false — wrong order
console.log(isValidParentheses(")("));         // false — closer with empty stack
console.log(isValidParentheses("((("));        // false — leftovers
console.log(isValidParentheses("<div>{x}</div>")); // true — ignores non-brackets
```

## Complexity

- **Time:** O(n) — one pass, O(1) work per character.
- **Space:** O(n) — worst case the stack holds all n characters (e.g. `"((("`).

## Edge cases

- **Empty string** → `true` (nothing to mismatch; state the choice — some specs want `false`).
- **Closer on empty stack** (`")("`): `stack.pop()` returns `undefined`, which never equals an opener — safe without a separate length check.
- **Interleaved types** (`"([)]"`): the pop must match *this* closer, not just any opener — this is the case a counter-based solution gets wrong.
- **Non-bracket characters**: ignored above; a strict validator would `return false` on any unexpected char — say which you're implementing.
- **Unicode**: `for...of` iterates by code point, so astral-plane characters (emoji in JSX text) never split a surrogate pair and corrupt indexing.

## Follow-ups interviewers ask

- "Extend it to HTML tags" → tokenize with `/<\/?([a-zA-Z][\w-]*)[^>]*>/g`, push tag *names*, ignore self-closing `<img/>`; closers must match the popped name.
- "Return the position of the first mismatch" → push `{ ch, index }` pairs instead of bare characters.
- "What about escaped brackets, `\(`?" → when you see a backslash, skip the next character.
- "Validate with only O(1) space, one bracket type" → a simple counter works when there's exactly one kind of pair — and explaining *why* it breaks for multiple types shows you understand the stack's role.
