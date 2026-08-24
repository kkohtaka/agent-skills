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
them explicitly rather than shelling out. A skill that talks to an MCP server
grants it by server name (`mcp__github`); see §4.9 for the caveat that comes
with that.

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
  from `origin/HEAD`, falling back to `git ls-remote --symref origin HEAD`
  (which works in a fresh clone where `origin/HEAD` is unset, and needs no
  GitHub CLI). Resolving the *name* is not the same as having the *ref*: a
  cloud session's clone fetches only the branches it needs, so
  `refs/remotes/origin/<default>` frequently does not exist at load time.
  Anything comparing against it must check with `git rev-parse --verify` and
  say the ref is unfetched, rather than reporting an empty diff;
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

### 4.9 Dual-environment support (`gh` may be absent)

These skills run in two kinds of environment, and both are first-class:

| Environment | GitHub transport |
| --- | --- |
| Local checkout / devcontainer | the `gh` CLI |
| Claude Code on the web (cloud session) | the GitHub MCP server |

`git` itself — including `push` — works in both, so anything a skill can do
with plain `git` needs no branching at all. Only the GitHub API calls do.
A skill that depends on a tool which may be missing MUST:

1. **Probe, do not guess.** Detect the route with a read-only check
   (`command -v gh >/dev/null 2>&1 && gh auth status >/dev/null 2>&1`) rather
   than inferring the environment from anything else.
2. **Guard every `## Context` command** that could be unavailable, so the skill
   loads with a usable message instead of a command error. Context runs on every
   load, before the model can decide anything (§3).
3. **Keep the judgement in one place.** Only the fetch/write route differs
   between environments — the analysis, the drafting, and the confirmation gates
   (§4.3) are written once and are identical on both routes. Do not fork a step
   into two near-duplicate steps when only the command differs.
4. **Fail loudly when neither route is available.** Say which routes were tried
   and stop; never silently skip the step or report it as done (§4.6).

Each route MUST state which environment it is for, so a reader is not left
wondering why there are two.

**Verify every MCP tool name and response field against a live session.** The
server's surface is not a mirror of the CLI's, and a plausible name is often
absent: the GitHub server has no tool that lists a repository's labels, and its
`issue_read` response carries no numeric `id`. A route written from a guessed
tool name fails at the moment it is needed. Where the MCP surface has no
equivalent, fall back to the REST endpoint over `curl` — a cloud session's
egress proxy authenticates `api.github.com` requests — and grant `Bash(curl *)`.

**MCP server names are not stable identifiers.** They derive from the
connector's display name, so an `allowed-tools` grant written against a name
breaks silently if the connector is renamed. The GitHub server is observed as
`github` (granted as `mcp__github`); a skill referring to it should say so and
note that the tool names are the same under any server name.

## 5. Authoring checklist

Before considering a skill done, verify:

- [ ] Frontmatter follows §2 (keys, order, scoped `allowed-tools`).
- [ ] `## Context` uses read-only commands only (§3).
- [ ] `disable-model-invocation` set per §4.2.
- [ ] Confirmation gates present for any outward/irreversible action (§4.3).
- [ ] No project-specific assumptions (§4.5).
- [ ] Dual-environment rule followed (§4.9): the route is probed, `## Context`
      is guarded, the judgement is not duplicated per route, each route says
      which environment it is for, and a missing route fails loudly.
- [ ] `allowed-tools` grants every MCP server the skill now uses (§4.1, §4.9).
- [ ] `apm audit` is clean and `apm install --dry-run --target claude` deploys it.
- [ ] Verified by manual invocation in at least one consumer repository.
