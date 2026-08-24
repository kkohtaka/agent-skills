# agent-skills

Project-agnostic VCS/GitHub workflow skills for [Claude Code](https://claude.com/claude-code),
packaged and distributed with [APM (Agent Package Manager)](https://microsoft.github.io/apm/).

## Skills

| Skill | What it does | Auto-invocable |
| --- | --- | --- |
| `commit` | Create incremental conventional commits on a properly named branch (no push) | No — invoke explicitly |
| `create-pr` | Push the branch and open a draft PR following conventional-commit conventions | No — invoke explicitly |
| `create-issue` | File a well-structured GitHub issue using the repo's templates and labels | No — invoke explicitly |
| `debug-ci` | Read-only triage of a red CI check: failed jobs, root step, log excerpts | Yes |

All skills resolve the repository's default branch dynamically and assume
nothing about the build system — they work in any git/GitHub repository.
Authoring rules live in [CONVENTIONS.md](CONVENTIONS.md).

### Environments

Each skill works both in a local checkout and in a Claude Code cloud session,
which has no `gh` CLI:

- `commit` uses plain `git` only, so it needs no GitHub transport at all.
- `create-pr`, `create-issue`, and `debug-ci` probe for `gh` at load time and
  fall back to the **GitHub MCP server** (observed as `github`) when it is
  absent. `git push` works in both environments, so only the GitHub API calls
  differ. Each step names the call for both routes; if neither is available the
  skill stops and says so rather than reporting the action as done.

In a cloud session the GitHub connector must be enabled for the session —
without it, the three GitHub-facing skills have no route. See
[CONVENTIONS.md §4.9](CONVENTIONS.md) for the authoring rule behind this.

## Consuming

Add the dependency to your repository's `apm.yml`:

```yaml
targets:
  - claude
dependencies:
  apm:
    - kkohtaka/agent-skills#v0.3.0
```

Then install:

```bash
apm install --target claude
```

APM deploys each skill to `.claude/skills/<name>/` and pins the exact content
hash in `apm.lock.yaml`. Commit both `apm.yml` and `apm.lock.yaml`; add
`apm_modules/` to `.gitignore`.

### Source of truth

This repository is the **single source of truth** for these skills. Do not
hand-edit the deployed copies in a consumer repository — the next
`apm install` overwrites them. Change the skill here, tag a new version, then
bump the pin in each consumer.

## Versioning

Releases are semver git tags (`vMAJOR.MINOR.PATCH`):

- **PATCH** — wording fixes, no behavioral change.
- **MINOR** — new skill, or a backward-compatible behavior change.
- **MAJOR** — a skill is removed/renamed, or its arguments/behavior change
  incompatibly.

Consumers pin an exact tag (`#v0.3.0`) and upgrade deliberately.

## Development

Validate before tagging a release:

```bash
apm install --dry-run --target claude   # preview deployment
apm audit                               # scan for hidden Unicode characters
```

## License

MIT
