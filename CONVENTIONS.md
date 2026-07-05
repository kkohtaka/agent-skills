# Skill Authoring Conventions

This document is the single source of truth for how the agent skills in this
package are written. Every skill under `.apm/skills/<name>/SKILL.md` MUST
follow it. When an existing skill and this document disagree, this document
wins and the skill should be updated.

These conventions are documentation for this repository only — they are not
deployed to consumer repositories.

## 1. File layout

- One skill per directory: `.apm/skills/<name>/SKILL.md`.
- `<name>` is lowercase kebab-case and matches the `name:` frontmatter key.
- Supporting files (scripts, templates) live next to `SKILL.md` in the same
  directory; reference them by relative path. APM copies the whole directory
  to the consumer.

## 2. Frontmatter

Required keys, in this order:

```yaml
---
name: <kebab-case, matches the directory>
description: <one sentence, imperative, says what the skill does and when to use it>
argument-hint: "[optional-arg]"        # omit if the skill takes no arguments
disable-model-invocation: <true|false> # see §4.2
allowed-tools: Bash(git *) Bash(gh *)  # see §4.1
---
```

- `description` is what the model matches on for auto-invocation — keep it
  precise and free of internal jargon.
- `allowed-tools` is a space-separated list. Scope `Bash` with a prefix matcher
  (`Bash(git *)`, `Bash(gh *)`). Never use a bare unscoped `Bash`.

## 3. Body structure

Two top-level sections, in this order:

### `## Context`

Gather the state the skill needs using `!`-inlined command blocks:

````markdown
**Working tree status:**
```
!`git status --short`
```
````

- **Read-only commands only.** Anything that mutates the repo or remote state
  belongs in a numbered step, never in a Context block — Context runs
  automatically every time the skill loads.
- Inline read-only file contents the skill depends on with `!`cat`` (templates,
  config) so the model sees them without an extra tool round-trip.

### `## Your Task`

- Open with: `Follow these steps in order. Stop and ask the user if anything is unclear.`
- Use numbered `### Step N — <imperative summary>` headings.
- Each step is concrete and checkable; show the exact commands to run.

## 4. Cross-cutting policies

### 4.1 Least privilege

List only the tools the skill actually uses, scoped as narrowly as possible.
If a skill needs to write files it relies on the `Write`/`Edit` tools; declare
them explicitly rather than shelling out.

### 4.2 Model invocation (`disable-model-invocation`)

- **`true` (explicit invocation only)** — any skill with side effects on the
  repo or external services: it commits, pushes, opens PRs/issues, deploys, or
  merges. The user must invoke these by name.
- **`false` (model may auto-invoke)** — read-only / advisory skills that only
  run, inspect, and report without mutating anything.

A skill that both inspects and fixes counts as having side effects → `true`.

### 4.3 Confirmation gates

Outward-facing or irreversible actions MUST stop and confirm with the user
before executing, even within an explicitly invoked skill. This always
includes: `git push`, opening a PR or issue, any deploy, and merging a PR.
State exactly what will happen, then wait.

### 4.4 Delegation, not duplication

Skills compose by name rather than re-implementing shared behavior. When a step
is "what another skill already does," say so and point to that skill instead of
copying its steps. Because consumers install skills independently, guard every
cross-skill pointer with "if available".

### 4.5 Project-agnosticism

These skills deploy into repositories with unknown stacks. A skill in this
package MUST NOT assume:

- a specific default branch name (`master`/`main`) — resolve it dynamically
  from `origin/HEAD` or `gh repo view`;
- a specific build system, linter, formatter, or test runner — formatting and
  quality fixes are the consumer repository's concern, never a step here;
- the presence of project files beyond git/GitHub itself — probe with
  `2>/dev/null` fallbacks and degrade gracefully (e.g. PR/issue templates).

### 4.6 Honest reporting

Report failures, skipped steps, and partial results plainly in the skill's
output. Never present an unverified or failed step as done.

### 4.7 Re-runnability

Design steps so the skill can be re-run after a mid-way failure without
corrupting state (check-before-create, detect already-committed work, etc.).

### 4.8 Language

Skill Markdown (`SKILL.md`) is written in **English**. Artifacts a skill
produces (issues, PRs, commits) follow the convention documented in that
skill — GitHub-facing text is English.

## 5. Authoring checklist

Before considering a skill done, verify:

- [ ] Frontmatter follows §2 (keys, order, scoped `allowed-tools`).
- [ ] `## Context` uses read-only commands only (§3).
- [ ] `disable-model-invocation` set per §4.2.
- [ ] Confirmation gates present for any outward/irreversible action (§4.3).
- [ ] No project-specific assumptions (§4.5).
- [ ] `apm audit` is clean and `apm install --dry-run --target claude` deploys it.
- [ ] Verified by manual invocation in at least one consumer repository.
