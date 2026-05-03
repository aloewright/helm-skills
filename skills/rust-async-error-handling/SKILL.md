---
name: rust-async-error-handling
description: Idiomatic error handling in async Rust — `thiserror` for library errors, `anyhow` for application code, `?` everywhere, never `.unwrap()` outside tests. Always preserve cause chains and never panic in a tokio task.
roles: [Worker, Reviewer]
tags: [rust, async, errors, tokio, thiserror, anyhow]
required_tools: [read_file, edit_file, cargo_check]
---

# rust-async-error-handling

The error-handling rules in this org for async Rust. Optimised for: cause chains survive, panics don't leak, retries are explicit, and `cargo clippy -W clippy::pedantic` stays clean.

## When to use

- Adding a new async function that can fail.
- Reviewing a PR that introduces `.unwrap()` / `.expect()` in production code.
- A `tokio::spawn` task is silently dying.
- You're tempted to `Box<dyn Error>` something — stop and read this skill first.

## Library code: `thiserror`

Each public crate defines an enum with one variant per failure mode. Wrap every external error type so the cause chain is preserved.

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum SkillRegistryError {
    #[error("io error reading {path}: {error}")]
    Io { path: String, error: String },

    #[error("yaml parse error in {path}: {error}")]
    Yaml { path: String, error: String },

    #[error("skill `{name}` not found")]
    NotFound { name: String },
}

impl From<std::io::Error> for SkillRegistryError {
    fn from(e: std::io::Error) -> Self {
        Self::Io {
            path: "<unknown>".into(),
            error: e.to_string(),
        }
    }
}
```

- One variant per *user-actionable* failure mode, not per call site.
- Include the offending path / id in the message so a stack-less log is still useful.
- Never `#[error("{0}")]` with a `String` — that strips the cause.

## Application code: `anyhow`

Top-level binaries and CLIs can use `anyhow::Result` and the `?` operator — the bin-level error message goes to stderr, the cause chain prints automatically with `RUST_LOG=trace` or `{:?}`.

```rust
use anyhow::{Context, Result};

pub async fn load_skills(path: &Path) -> Result<SkillRegistry> {
    let raw = tokio::fs::read_to_string(path)
        .await
        .with_context(|| format!("reading skills from {}", path.display()))?;
    let parsed: SkillRegistry = serde_yaml::from_str(&raw)
        .with_context(|| format!("parsing skills yaml at {}", path.display()))?;
    Ok(parsed)
}
```

- Always attach `with_context` at boundaries — bare `?` from a tokio task gives a one-line error with no cause.
- Use `.context("...")` for static strings, `.with_context(|| ...)` only when you need to format.

## `?` propagation, never `.unwrap()`

In production code:

- `.unwrap()` / `.expect()` are forbidden outside tests, build scripts, and `const`-eval contexts.
- `.unwrap_or_default()` is fine — it doesn't panic.
- `.unwrap_or_else(|| panic!(...))` is just `.unwrap()` with extra steps. Block in review.

## Spawned tasks must not panic

A panic in a `tokio::spawn`'d future is *swallowed* unless you `.await` the join handle. Always:

```rust
let handle = tokio::spawn(async move {
    // ... work ...
    anyhow::Ok::<()>(())
});

// Later:
match handle.await {
    Ok(Ok(())) => {}
    Ok(Err(err)) => tracing::error!(?err, "task failed"),
    Err(join_err) if join_err.is_panic() => {
        tracing::error!(?join_err, "task panicked");
    }
    Err(join_err) => tracing::error!(?join_err, "task cancelled"),
}
```

If you don't `await` the handle, the task panicking is invisible — the task just stops.

## Retries

- Explicit, with a budget. Use `tokio_retry` or a hand-rolled exponential backoff.
- Distinguish *retryable* (timeout, 5xx, transient I/O) from *terminal* (4xx, parse error). Add a `retryable(&self) -> bool` method on the error enum and gate retries on it.
- Never retry inside a `?` chain — retries belong at the call site.

## Logging vs returning

- Library code never logs — it returns. The caller decides whether the error is worth a log line.
- Application code logs once at the top-level `match` — no log-and-return-error duplicates in the chain.

## Anti-patterns

- `let x = future.await.unwrap();` in production — block.
- `Box<dyn Error>` in a public API — use `thiserror`.
- A `#[error("{0}")]` variant that swallows the underlying cause — use `#[error("...: {error}")]` and store the cause string.
- Catching a panic with `std::panic::catch_unwind` to "be robust." Fix the panic.
- Ignoring a `JoinHandle` (spawning then dropping). Make the task `'static`-bounded *and* await it, or use `JoinSet`.
- Returning `Result<(), String>` — strings don't compose. Use a typed error.
