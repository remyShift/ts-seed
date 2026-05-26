---
name: tdd-cycle
description: TDD cycle guide for ts-seed. Invoke before writing any code.
---

# TDD in ts-seed

## The cycle

```
RED   → write a failing test for the right reason
GREEN → write the minimal code that makes the test pass
REFACTOR → improve without changing behaviour
```

## RED — what "right failure" means

A good red test fails because **the behaviour does not exist yet**, not because a file is missing or there is a syntax error.

Before implementing, verify the error message is the expected one.

```bash
npm run test:run
# Expected: FAIL with "cannot find module" or an assertion failure — not a TypeScript compile error
```

## GREEN — the minimum rule

Write the **simplest possible code** to make the test pass. Not the final code. Not the clean code. The minimal code.

If a `return hardcodedValue` makes the test pass, that is enough for this step. Refactor comes next.

## REFACTOR — the key question

At every refactor, ask:

> **"What does this code depend on that it doesn't need to know?"**

Examples of dependencies to question:

- Does this code instantiate something it could receive as a parameter?
- Does this code know about the filesystem when its only role is to compute?
- Does this code know how to persist when its role is business logic?

Every time the answer is "yes", extract an interface. That is where ports emerge naturally.

## Rules for this project

**In `brick/domain/` and `catalog/*/domain/`:**

- Zero imports of `fs`, `path`, `next`, or any Node/framework module
- If a test needs a mock, that is a signal that an interface is missing

**In tests:**

- Test observable behaviours, not implementations
- A test must never assert that a method was called, only what it produced
- Exception: asserting that `ProjectWriter.apply` was called in `BrickApplicationService` is legitimate because it is the observable side-effect

**Test naming:**

```ts
it('should <expected behaviour> when <condition>', ...)
// e.g. it('should install typescript ^5.0.0 for version 5.x', ...)
```

## When tests are green

Before committing:

```bash
npm run test:run
npm run lint
```

Commit as soon as one coherent behaviour is added — not at the very end.
