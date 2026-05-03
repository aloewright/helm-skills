---
name: tailwind-css-migration
description: Migrate a stylesheet (or component tree) from hand-written CSS / Sass / styled-components to Tailwind. Use `tw-migrate` for bulk class extraction, then squeeze out arbitrary values by introducing design tokens — never let `[#abc123]` literals survive into final code.
roles: [BulkRefactor, Worker]
tags: [tailwind, css, refactor, design-tokens, frontend]
required_tools: [read_file, edit_file, run_codemod]
---

# tailwind-css-migration

Migrating to Tailwind without ending up with a sea of `class="p-[7px] text-[#abc]"` arbitrary values. The two-pass approach: bulk convert, then collapse arbitraries into design tokens.

## When to use

- A repo has mixed CSS / Sass / styled-components and a new design system requires Tailwind.
- A Tailwind codebase has drifted into too many arbitrary values and needs cleanup.
- You're starting a new component and want to set the rules before the first PR lands.

## Pass 1 — Bulk convert

For component-scoped CSS (CSS Modules, `<style>` blocks):

```bash
npx tw-migrate src/components --jsx --apply
```

This rewrites CSS rules into class lists. It will produce arbitrary values for everything not in your `tailwind.config.ts` — that's fine for now, you'll squeeze them out in pass 2.

For styled-components specifically:

```bash
npx tw-migrate src --styled-components --apply
```

Commit pass 1 as a single commit. Reviewing the diff line by line is hopeless — review the rendered output instead.

## Pass 2 — Squeeze out arbitraries

```bash
rg -n '\b(p|m|gap|text|bg)-\[' src
```

For each unique arbitrary value, decide:

1. **Repeated 3+ times** → add a token to `tailwind.config.ts` and replace.
2. **One-off, semantically unique** → leave it.
3. **One-off, semantically generic** (`text-[#888]` for "muted") → snap to the closest existing token (`text-gray-500`).

Adding a token:

```ts
// tailwind.config.ts
export default {
  theme: {
    extend: {
      colors: {
        brand: {
          50: "#f0f9ff",
          500: "#0284c7",
          900: "#0c4a6e",
        },
      },
      spacing: {
        18: "4.5rem", // for a recurring 72px gap
      },
    },
  },
};
```

Then codemod:

```bash
npx jscodeshift -t scripts/codemods/swap-arbitrary.ts \
  --extensions=tsx \
  --from='text-\[#0284c7\]' --to='text-brand-500' \
  src
```

## Pass 3 — Lint

Add the official Tailwind ESLint plugin so future PRs flag arbitraries:

```bash
npm i -D eslint-plugin-tailwindcss
```

```jsonc
// .eslintrc.json
{
  "extends": ["plugin:tailwindcss/recommended"],
  "rules": {
    "tailwindcss/no-arbitrary-value": "error",
    "tailwindcss/classnames-order": "warn",
    "tailwindcss/no-custom-classname": "off"
  }
}
```

`no-arbitrary-value: error` is the policy lever — it forces every new arbitrary to come with a config change, which forces a design conversation.

## Component conventions

- One component = one file, one default export, one Tailwind class list per JSX element.
- For more than ~6 conditional classes, extract into a `cva` (class-variance-authority) variant. Block any ternary chain longer than 3.
- Use `cn()` from `class-variance-authority` (or `tailwind-merge`) — never raw string concat — so duplicate-utility merges happen.

## Anti-patterns

- Shipping pass 1 without pass 2. Block the PR — the arbitrary-value lint will be ignored forever otherwise.
- Adding `!important` (`!text-red-500`) to override a parent. The parent is the bug; fix it.
- Mixing `@apply` (Tailwind directives) with utility classes on the same element. Pick one per file.
- Abandoning the Tailwind config and building a parallel design-token system in a separate file. The config IS the design-token system.
- Writing a `globals.css` with 200+ rules that override Tailwind. If you need that, you're not using Tailwind.
