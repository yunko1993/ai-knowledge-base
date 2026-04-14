---
name: code-comment-style
description: Use when writing, refactoring, or reviewing code where comments should be improved. Guides Codex to add useful intent-focused comments for complex logic, business rules, edge cases, SQL, regex, state transitions, and integration boundaries without creating noisy line-by-line comments.
---

# Code Comment Style

Help code stay readable for humans and future AI agents by adding comments where they reduce re-reading cost.

## Core Principle

Write comments that explain why the code exists, why a branch matters, or what hidden constraint must be preserved.

Do not write comments that merely repeat the syntax.

## When To Add Comments

Add succinct comments for:

1. Business rules that are not obvious from names alone.
2. Edge cases, compatibility workarounds, and defensive checks.
3. Non-trivial algorithms, data transformations, batching, caching, retries, locks, and concurrency.
4. State machines, workflow transitions, permission differences, and lifecycle hooks.
5. Complex SQL, dynamic SQL conditions, joins with non-obvious semantics, and migration scripts.
6. Regex, date/time handling, encoding, locale, timezone, path, proxy, or OS-specific behavior.
7. Public APIs, integration boundaries, external service assumptions, and side effects.
8. Code that future maintainers or AI agents are likely to misread or "simplify" incorrectly.

## When Not To Add Comments

Avoid comments for:

1. Straightforward assignments, getters, setters, constructors, and obvious loops.
2. Restating the method or variable name in prose.
3. Explaining language syntax or common library calls.
4. Large comment blocks that duplicate nearby code.
5. Temporary speculation, TODOs without owner/context, or comments that will quickly become stale.

## Comment Style

1. Prefer short comments placed immediately above the relevant block.
2. Explain intent, invariant, risk, or tradeoff.
3. Keep comments stable when implementation details change.
4. Use the project's existing comment language and style.
5. If the file already uses Chinese comments, continue in Chinese.
6. If the file uses English comments or has no established style, prefer concise English unless the user asks otherwise.

## During Code Changes

When editing code:

1. Add comments for newly introduced non-obvious logic.
2. Update stale comments touched by the change.
3. Remove misleading comments instead of leaving them around.
4. Do not add comment-only noise just to increase comment count.

## During Code Review

When reviewing code:

1. Flag missing comments only when the lack of explanation creates maintenance risk.
2. Prefer actionable suggestions such as "explain why this fallback is needed" over generic "add comments".
3. Treat unclear business rules, hidden side effects, and fragile ordering as high-value comment targets.

## Examples

Good:

```java
// Keep parent departments in the result so tree construction does not drop visible children.
```

Good:

```typescript
// Retry only idempotent requests; POST may already have changed remote state.
```

Bad:

```java
// Increment i by one.
i++;
```

