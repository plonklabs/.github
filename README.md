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

Inputs: `runs-on` (default `ubuntu-latest`), `model` (default `claude-sonnet-4-6`), `max-turns` (default `50`), `collaborators-only` (default `true`), `review-file` (default `REVIEW.md`). Secret: `CLAUDE_CODE_OAUTH_TOKEN`.

## Claude skills

`.claude/skills/` holds repo-agnostic Claude Code skills:

- **`implement`** — execute an agreed-upon stack of PRs autonomously (implement → self-review → ready → bot review + e2e → merge → next).
- **`spec`** — turn a feature idea into a spec and a tracking epic issue.
- **`topr`** — rebase a stacked PR onto `origin/main` (squash-merge aware).

> ⚠️ GitHub's `.github`-repo mechanism does **not** auto-distribute `.claude/skills/` to other repos — that magic only covers profiles, issue templates, and workflows. These live here as a canonical, version-controlled copy. To use them in another repo, copy the skill into that repo's `.claude/skills/` or into your user-level `~/.claude/skills/`.

## Contributing

`main` is protected — changes land via pull request.
