---
name: typescript-strict-mode-adoption
description: Turn on `"strict": true` in `tsconfig.json` for an existing project without freezing the team. Stage the rollout flag-by-flag, fix `noImplicitAny` first, then `strictNullChecks`, and put `// @ts-expect-error` on the holdouts so the next PR sees them.
roles: [BulkRefactor, Worker]
tags: [typescript, tsconfig, strict-mode, refactor, types]
required_tools: [read_file, edit_file, tsc_check]
---

# typescript-strict-mode-adoption

Enabling `"strict": true` on a multi-thousand-file TS project all at once is a multi-week migration with a frozen main branch. Don't do that — stage the flags individually, land them as separate PRs, and use `// @ts-expect-error` as a tracked holdout.

## When to use

- A new repo: turn on `"strict": true` from day one. Stop reading.
- An existing repo where `"strict": false`: this skill is for you.
- An existing repo with `"strict": true` but lots of `as any` / `// @ts-ignore`: skip ahead to the "Cleanup" section.

## The flags `"strict": true` actually enables

`"strict": true` is a meta-flag for these eight (TS 5.5+):

| Flag                         | Stage |
| ---------------------------- | ----- |
| `noImplicitAny`              | 1     |
| `strictNullChecks`           | 2     |
| `strictFunctionTypes`        | 3     |
| `strictBindCallApply`        | 3     |
| `strictPropertyInitialization` | 3   |
| `noImplicitThis`             | 1     |
| `useUnknownInCatchVariables` | 4     |
| `alwaysStrict`               | 1 (free) |

Each numbered stage = one PR. Don't bundle them.

## Stage 1 — `noImplicitAny`

This catches every untyped function param. The fix is mechanical: add types.

```jsonc
{
  "compilerOptions": {
    "noImplicitAny": true,
    "alwaysStrict": true,
    "noImplicitThis": true
  }
}
```

Run `tsc --noEmit`, get the error list, and burn through it. Most fixes are `(x: string) =>` instead of `(x) =>`.

For genuinely untyped third-party code, prefer `unknown` over `any`. If you must use `any`, leave a comment why.

## Stage 2 — `strictNullChecks`

This is the big one — every API that returns `T | undefined` needs an explicit check at every call site.

```jsonc
{
  "compilerOptions": {
    "strictNullChecks": true
  }
}
```

Tactics:

1. Run `tsc --noEmit > errors.txt` and bucket errors by file.
2. Fix the leaf modules first (utilities with no dependents) — those propagate type fixes upward for free.
3. For each "object is possibly undefined" error, prefer optional chaining (`x?.y`) and nullish coalescing (`x ?? default`). Use `!` only when you have a *runtime invariant* that the value is non-null AND a comment explaining it.

```ts
// BAD
const id = user!.id;

// GOOD
if (!user) throw new Error("user must be set after auth");
const id = user.id;
```

## Stage 3 — `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`

These three together usually surface fewer errors than stage 2. Land them in one PR.

`strictPropertyInitialization` is the trickiest — it requires class fields to be definitely-assigned in the constructor or marked with `!`. For framework-driven classes (Angular `@Input`, NestJS `@Injectable`), use the `!:` pattern with a comment.

## Stage 4 — `useUnknownInCatchVariables`

The only annoying flag. Every `catch (err)` becomes `catch (err: unknown)` and you must narrow before reading properties.

```ts
try {
  await doThing();
} catch (err) {
  if (err instanceof Error) {
    console.error(err.message);
  } else {
    console.error("unknown error", err);
  }
}
```

Easy to fix mechanically — write a `toError(unknown): Error` helper and route every catch through it.

## Holdouts: `// @ts-expect-error`

For files you cannot fix in this PR, use `// @ts-expect-error` instead of `// @ts-ignore`. The `expect-error` form *fails* if the next refactor accidentally fixes the type — so the comment can't go stale.

```ts
// @ts-expect-error -- legacy module, planned for refactor in PDX-9999
const result = legacyApi(input);
```

Then `git grep '@ts-expect-error'` is your migration backlog.

## Cleanup

Once `"strict": true` is on:

```bash
git grep -n 'as any' src
git grep -n '@ts-ignore' src
git grep -n ': any' src
```

Each match is technical debt. Set a CI rule to fail on new `as any` — existing matches grandfathered into a `lint-allowlist.txt`.

## Anti-patterns

- One big-bang PR turning on `"strict": true`. The PR will sit unmerged for weeks. Block.
- Using `// @ts-ignore` instead of `// @ts-expect-error`. Block in review — `expect-error` is the right primitive.
- Adding `as any` to silence stage 2 errors. The whole point of strict null checks is that you have to think; `as any` opts out of thinking.
- Adding `"skipLibCheck": false` to "be more strict." That just adds noise from your `node_modules`. Leave it `true`.
- Setting `"noUncheckedIndexedAccess": true` in the same PR as `"strict": true`. It's even bigger than `strictNullChecks`. Land it as stage 5 if at all.
