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

## Consuming

Add the dependency to your repository's `apm.yml`:

```yaml
targets:
  - claude
dependencies:
  apm:
    - kkohtaka/agent-skills#v0.1.0
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

Consumers pin an exact tag (`#v0.1.0`) and upgrade deliberately.

## Development

Validate before tagging a release:

```bash
apm install --dry-run --target claude   # preview deployment
apm audit                               # scan for hidden Unicode characters
```

## License

MIT
