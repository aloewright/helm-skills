# helm-skills

Community skills for Helm agents — a flat collection of self-contained `SKILL.md` files
that the `warp skills` CLI can discover, install, update, and contribute back to.

```bash
warp skills search "drizzle d1"
warp skills install drizzle-migration
warp skills update
warp skills contribute ./.agents/skills/my-new-skill
```

Skills installed via `warp skills install <name>` land in `~/.warp/skills/<name>/`, where
the in-process `crates/skills` loader picks them up automatically — they appear in the
agent's available-skill list on the next turn with no extra config.

## Layout

```
helm-skills/
├── OWNERS                 # who can merge to main (see "Governance" below)
├── README.md              # this file
└── skills/
    ├── ai-gateway-routing/SKILL.md
    ├── doppler-secret-fetch/SKILL.md
    ├── drizzle-migration/SKILL.md
    ├── wrangler-deploy/SKILL.md
    ├── warp-new-scaffolding/SKILL.md
    ├── github-pr-review/SKILL.md
    ├── cloudflare-workers-debugging/SKILL.md
    ├── tailwind-css-migration/SKILL.md
    ├── rust-async-error-handling/SKILL.md
    └── typescript-strict-mode-adoption/SKILL.md
```

Every skill is one directory containing one `SKILL.md`. The directory name MUST match
the `name:` in front matter; the `warp skills` CLI uses the directory name as the
install slug.

## Skill format

Each `SKILL.md` is markdown with YAML front matter:

```yaml
---
name: my-skill                       # required, matches the directory name
description: One-line summary.       # required, used by `warp skills search`
roles: [Worker, Reviewer]            # optional; empty = applies to every role
tags: [cloudflare, deploy, secrets]  # optional; free-form filter tags
required_tools: [wrangler_deploy]    # optional; trust-model gate (see below)
---

# Skill body in markdown
...
```

The schema matches `SkillFrontMatter` in
[`crates/skills/src/lib.rs`](https://github.com/aloewright/warp/blob/master/crates/skills/src/lib.rs)
of the warp repo.

### Trust model: `required_tools`

A skill that needs specific tools to run safely should declare them in front matter.
The agent runner refuses to invoke a skill whose `required_tools` are not all
granted in the current session. Example:

```yaml
required_tools: [wrangler_deploy, doppler_secret_fetch]
```

Use this for skills that exec real side-effecting commands (deploy, migrate, push) —
not for read-only documentation skills.

## Authoring conventions

The starter skills set the bar — match it when you contribute.

1. **Specific behaviors over platitudes.** Good: "Block any PR that adds a
   `process.env.X` read without a corresponding `doppler secrets set X`." Bad:
   "Be careful with secrets."
2. **Front matter has `name`, `description`, `roles`, `tags`** — `name` matches
   the directory.
3. **Every skill ends with an "Anti-patterns" section** that lists the exact things
   to block in review. Reviewers and agents both consume this.
4. **Copy-pasteable code blocks**, not pseudocode.
5. **One skill = one pattern.** Don't bundle "all of Cloudflare" into a single skill.

## Contributing

Open a PR! Anyone can. The flow:

1. Write your skill locally — usually inside your project's `.agents/skills/` so you
   can iterate against the real agent.
2. Run `warp skills contribute path/to/your-skill`. This shells out to `gh pr create`
   with a sane title and body.
3. An owner reviews, asks for tightening if needed, and merges.

You can also do it by hand:

```bash
gh repo fork aloewright/helm-skills --clone
cp -R path/to/your-skill helm-skills/skills/
cd helm-skills
git checkout -b add-my-skill
git add skills/your-skill && git commit -m "skill(your-skill): add"
gh pr create
```

### What gets merged

- Skills that match the conventions above.
- Skills that are reasonably general — they're worth installing for more than one project.
- Skills with a clear "Anti-patterns" section.

### What does NOT get merged

- Project-specific skills ("how we deploy at $COMPANY"). Keep those in your repo's
  `.agents/skills/` instead.
- Skills that duplicate an existing one. Open an issue suggesting an edit to the
  existing skill instead.
- Skills that hardcode a secret, API key, account ID, or org URL.

## Governance

The `OWNERS` file at the repo root lists who can merge to `main`. To become an owner,
contribute three merged skills, then open a PR adding yourself to `OWNERS` and ping an
existing owner.

The repo is MIT-licensed (see `LICENSE`). All contributions are accepted under the
MIT license; by opening a PR you certify you wrote the content (or had permission to
contribute it) and accept that license.
