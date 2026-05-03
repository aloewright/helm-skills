---
name: cloudflare-workers-debugging
description: Debug a deployed Cloudflare Worker with `wrangler tail`, structured logging, AI Gateway logs, and Logpush — never with `console.log` left in production code. Always reproduce locally with `wrangler dev` before paging the on-call.
roles: [Worker]
tags: [cloudflare, workers, debugging, wrangler, observability]
required_tools: [wrangler_tail, wrangler_dev, wrangler_logs, cf_aig_logs]
---

# cloudflare-workers-debugging

A Worker misbehaves in production. Don't grep prod logs blindly — bisect fast: reproduce locally, tail prod, then look at AI Gateway logs if the call went through the gateway.

## When to use

- A Worker is returning 500s or wrong data in production.
- A Worker silently drops requests under load.
- An AI Gateway call from a Worker times out or returns an unexpected shape.
- A scheduled (cron) Worker stops firing.

## Step 1 — Reproduce locally

```bash
doppler run --scope . -- npx wrangler dev --remote
```

`--remote` runs against real bindings (D1, R2, KV, AI) so the bug surfaces in the same code path. `--local` uses Miniflare and is faster but can hide binding-specific bugs.

If you cannot reproduce locally, the bug is environmental — go to step 3 before guessing.

## Step 2 — Add structured logs (and remove them after)

Use `console.log(JSON.stringify({ event, ...ctx }))` so log lines are queryable. Avoid stringly-typed messages — they don't survive a search across a million lines.

```ts
console.log(
  JSON.stringify({
    event: "pr_review_dispatched",
    pr_number: prNumber,
    duration_ms: Date.now() - start,
  }),
);
```

Logs you add for a specific debug session must come out before merge — block any PR that ships a new `console.log("debugging X")`.

## Step 3 — Tail production

```bash
npx wrangler tail --format=pretty
npx wrangler tail --status=error              # only failed requests
npx wrangler tail --search='"event":"foo"'    # filter by structured field
```

For a long-tail "happens 1% of the time" bug, enable Logpush to R2 / S3 instead so you can grep over a wider window:

```bash
npx wrangler logpush create --dataset workers_trace_events --destination r2://logs/...
```

## Step 4 — AI Gateway logs

If the Worker calls the gateway, the request/response is in the AI Gateway dashboard regardless of whether the Worker logged it. Filter by the `cf-aig-metadata` tag the Worker set:

```ts
await env.AI.run(
  "dynamic/text_gen",
  { messages },
  {
    gateway: {
      id: "x",
      metadata: { request_id: ctx.requestId },
    },
  },
);
```

Then in the dashboard, search by `request_id`. This survives a full Worker crash because the gateway logs are server-side.

## Common failure modes

- **`Error 1101`**: a thrown exception escaped the Worker. Read `wrangler tail` for the stack — the line number maps to the bundled output, run `npx wrangler deploy --dry-run` to inspect.
- **`Error 1102`**: CPU time exceeded. The Worker is doing too much work in one invocation. Move to Durable Objects or break the work into Queue messages.
- **D1 timeouts under load**: usually missing index. `EXPLAIN QUERY PLAN <sql>` from the wrangler shell.
- **AI Gateway 503**: provider upstream is down. The dynamic route should fall back — if not, check the route's fallback config in the dashboard.

## Anti-patterns

- Shipping `console.log` lines added "for debugging" to production. Block in review.
- Disabling logpush "to save money" without measuring. Logpush is cheap; lost incidents are not.
- Calling the provider directly to "rule out the gateway." If the gateway has a bug, file it — don't bypass. See `ai-gateway-routing`.
- Bumping `compatibility_date` to "fix" a runtime bug. The bug is in your code, not the runtime.
