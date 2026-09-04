# plonklabs/.github

The organization-level `.github` repository for **Plonk Labs**. It holds three things: the public org profile, reusable CI workflows shared across repos, and a set of shared Claude Code skills.

> Looking for the public profile text? That's [`profile/README.md`](profile/README.md) — the page rendered at [github.com/plonklabs](https://github.com/plonklabs). This file documents the repo itself.

## Contents

| Path | What it is |
|------|------------|
| [`profile/README.md`](profile/README.md) | The org profile page shown on `github.com/plonklabs`, plus its logo assets in `profile/assets/`. |
| [`.github/workflows/`](.github/workflows/) | Reusable workflows other repos call (see below). |
| [`.claude/skills/`](.claude/skills/) | Shared Claude Code skills: `implement`, `spec`, `topr`. |

## Reusable workflows

Other repos call these as `plonklabs/.github/.github/workflows/<file>.yml@main`. They run with the **caller's** event context, so they read `github.event.*` directly.

> For a **private** repo to call these, enable *Settings → Actions → General → "Accessible from repositories in the organization"* on this repo.

### Discord notifications

`discord-pr`, `discord-release`, `discord-issue`, and `discord-notify-failure`. Each takes a `runs-on` input (default `ubuntu-latest`) and a `DISCORD_WEBHOOK` secret.

```yaml
# .github/workflows/pr-notifications.yml in a caller repo
on:
  pull_request:
    types: [closed]
jobs:
  notify:
    uses: plonklabs/.github/.github/workflows/discord-pr.yml@main
    secrets: inherit
```

`discord-notify-failure` is meant to be called from a failed job:

```yaml
  notify-failure:
    needs: [build, test]
    if: failure()
    uses: plonklabs/.github/.github/workflows/discord-notify-failure.yml@main
    with:
      workflow_name: My CI
    secrets: inherit
```

### Claude code review

`claude-review` owns the scaffolding (checkout, `anthropics/claude-code-action`, the tool allowlist, model, and the collaborators-only gate). The **project-specific review instructions live in a `REVIEW.md` at the calling repo's root** — the workflow prepends a `REPO`/`PR NUMBER` header and feeds that file to Claude as the prompt.

```yaml
# .github/workflows/claude-review.yml in a caller repo
on:
  pull_request:
    types: [opened, synchronize, ready_for_review]
concurrency:
  group: claude-review-${{ github.event.pull_request.number }}
  cancel-in-progress: true
jobs:
  review:
    uses: plonklabs/.github/.github/workflows/claude-review.yml@main
    secrets: inherit
```

Inputs: `runs-on` (default `ubuntu-latest`), `model` (default `claude-sonnet-4-6`), `max-turns` (default `50`), `collaborators-only` (default `true`), `review-file` (default `REVIEW.md`), `max-rounds` (default `5`), `rereview-max-turns` (default `20`). Secret: `CLAUDE_CODE_OAUTH_TOKEN`.

**Rounds are counted in content, not pushes.** Each round leaves behind a hidden marker holding `git patch-id` for every commit of the PR — the reviewer is asked to end its verdict comment with the line, and the workflow posts it itself if the reviewer omits it. A rebase rewrites every SHA but preserves patch-ids, so a force-push that changes nothing runs no model round, does not advance the round counter toward `max-rounds`, and posts a short carried-over comment repeating the previous round's `Verdict:` line. A rebase that also adds a commit is re-reviewed as exactly that commit.

Only `[bot]` logins may supply a marker, and only the reviewer (or a carried-over comment repeating one verbatim) may supply a verdict, so a PR author cannot talk the guards into skipping a round.

> **Consumer merge gates** that tie a verdict to the head SHA must accept the carried-over comment: it is posted by the workflow's own token (`github-actions[bot]`), not by `claude[bot]`, and carries the `<!-- claude-review-carried-over -->` marker.

**A clean verdict enables auto-merge.** When a round ends with `Verdict: clean` and no unresolved bot review thread, the workflow runs `gh pr merge --auto --squash` on the PR; a carried-over clean verdict does the same. `Verdict: findings` never touches auto-merge — a later clean round is what enables it — and a round-ceiling run never does either, having produced no verdict. Auto-merge still waits on every required check and on branch protection, so this removes the button press, not a gate. A repository with auto-merge disabled logs a warning instead of failing the check.

The workflow needs no permissions beyond the `contents: read`, `pull-requests: write`, `id-token: write` it already had — a reusable workflow that requests more than its caller grants fails the run at startup, so the marker is written by *creating* a comment, never by editing one.

The guards are covered by `.github/tests/test_cost_guards.py`, run by the `tests` workflow.

## Claude skills

`.claude/skills/` holds Claude Code skills **scoped to this repo** — same archetype as the skills in the other plonklabs repos, but their gates and examples are this repo's (Actions YAML, profile, templates; `actionlint`/markdown validation; squash-merge into protected `main`):

- **`implement`** — execute an agreed-upon stack of PRs autonomously (implement → validate → self-review → bot review → squash-merge → next).
- **`spec`** — design a change to this repo and open a tracking epic issue whose `## Steps` feed `/implement`.
- **`topr`** — rebase a stacked PR onto `origin/main` (squash-merge aware).

> ⚠️ GitHub's `.github`-repo mechanism does **not** auto-distribute `.claude/skills/` to other repos — that magic only covers profiles, issue templates, and workflows. These live here as a canonical, version-controlled copy. To use them in another repo, copy the skill into that repo's `.claude/skills/` or into your user-level `~/.claude/skills/`.

## Contributing

`main` is protected — changes land via pull request.
