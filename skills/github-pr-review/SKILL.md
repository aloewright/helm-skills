---
name: github-pr-review
description: Review a GitHub pull request from the CLI with `gh pr view` + `gh pr diff`, then post line-anchored comments via `gh pr review`. Always read the diff before commenting and never approve without running the failing CI logs.
roles: [Reviewer]
tags: [github, pr, code-review, gh, ci]
required_tools: [gh_pr_view, gh_pr_diff, gh_pr_review, gh_run_view]
---

# github-pr-review

Reviews a PR end-to-end from the terminal so the same flow works for solo devs and agents. The four `gh` invocations below are the entire happy path — keep the sequence stable so reviewers can scan a transcript at a glance.

## When to use

- A teammate (or an agent) opened a PR and asked for a review.
- A scheduled review job kicks in for every PR with the `needs-review` label.
- CI failed and the PR description does not yet acknowledge why.

## Read before you write

```bash
gh pr view <NUMBER> --json title,body,labels,reviewDecision,statusCheckRollup
gh pr diff <NUMBER>
```

`statusCheckRollup` tells you which checks are red — read those first. A passing review on a PR with red CI is a review smell.

If CI is red, get the failing run's logs:

```bash
gh run view --log-failed <RUN_ID>
```

## Comment on a specific line

```bash
gh pr review <NUMBER> \
  --comment \
  --body "Suggest extracting this into `parse_*` so the test in `mod_tests.rs` can target it directly."
```

For a multi-comment review use `--body-file` so the review reads as one cohesive message instead of N tiny comments.

## Approve / request changes

```bash
gh pr review <NUMBER> --approve --body "LGTM. Ran the failing test locally; passes after the fix."
gh pr review <NUMBER> --request-changes --body "Block on the data-migration question in line 42 of `db/schema.ts`."
```

Never `--approve` while CI is red unless the PR description names the unrelated infra issue and links to a tracking ticket.

## Hard rules

- Read the diff before posting a single comment.
- Never approve a PR you have not run locally for non-trivial changes (>50 LOC).
- Never `--approve` without reading `gh run view --log-failed` for each red check.
- Block on missing tests for any code path you would have written tests for.
- Block on hardcoded secrets, even in test fixtures.

## Anti-patterns

- "LGTM" reviews on >50-LOC PRs without a single inline comment. Force yourself to flag at least one concrete spot — even a praise comment counts ("nice extraction here").
- Approving while CI is red because "the failure is unrelated." Either fix it on this PR or open a follow-up issue and link it from the PR before approving.
- Reviewing the PR title + description without running `gh pr diff`.
- Suggesting a refactor that doubles the diff size. Open a follow-up PR instead and link it.
