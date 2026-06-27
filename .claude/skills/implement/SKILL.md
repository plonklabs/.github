# /implement — Execute an Agreed-Upon Stack of PRs Autonomously

## Description

Fires after a discussion has settled an ordered list of PRs to ship **in this repo** (`plonklabs/.github`). For each PR in order: implement → validate → self-review and fix everything including nits → open PR → run the bot-review loop → merge → move on.

Invoking this skill is the **explicit authorization** to merge the listed PRs autonomously — only the PRs on the agreed list, only for this invocation. It does not authorize inventing PRs not on the list.

This repo holds **GitHub Actions YAML, the org profile (`profile/`), Claude skills (`.claude/skills/`), issue/PR templates, and docs** — no compiled code. So the gates here are static validation (actionlint, YAML, markdown/link checks), not a build/test/cluster pipeline. `main` is protected: everything lands via squash-merged PR.

## Arguments
- `$ARGUMENTS` — optional GitHub epic/issue number. If given, read that issue's `## Steps` checklist (the `/spec` format) as a starting hint. If both a chat list and an epic exist, the **chat conversation is the source of truth**.

## Instructions

When the user runs `/implement` or `/implement <epic>`, execute the following phases.

### Phase 0: Resolve the PR list

1. **If `$ARGUMENTS` was passed**, fetch the epic and extract unchecked `- [ ]` items under `## Steps`:
   ```bash
   gh issue view <n> --json body --jq '.body'
   ```
2. **Otherwise**, re-state the ordered PR list inferred from the preceding chat.
3. **Print** the numbered list back before doing anything else.
4. **If the list is ambiguous** (no clear order, zero items, missing scope), STOP and ask the user to restate it. Never invent a PR not on the list.
5. **State authorization once**: print one line confirming this `/implement` run authorizes autonomous merge for the listed PRs only.

### Phase 1: Classify each PR

Print a classification table before the loop. For each PR note:

- **`area`** — `workflow` (`.github/workflows/`), `profile` (`profile/`), `skill` (`.claude/skills/`), `template` (issue/PR templates), or `docs` (`README.md` etc.). Drives which checks apply in 2a.
- **`affects-callers: yes/no`** — `yes` if the PR changes a **reusable workflow's interface or behaviour** (inputs, secrets, `on:` contract, or the embed/review output). Caller repos (`plonk`, `flatbed`, …) invoke these at `@main`, so an interface change is a breaking change for them — flag it for the end-to-end note in 2d.
- **`branch-from: main | <prior-pr-branch>`** — default `main` (merge-first). Only stack when the user asked, or when a PR can't be built without the prior one.

### Phase 2: Per-PR loop

For each PR in order:

#### 2a. Implement + validate

1. Branch off `origin/main`:
   ```bash
   git fetch origin main
   git switch -c <descriptive-slug> origin/main
   ```
2. Make **only** the changes scoped to this PR. Keep scope mechanical — don't fold in unrelated cleanups.
3. Validate, by `area`:
   - **workflow** — `actionlint` on every changed workflow (download it if not installed; pass a config listing self-hosted labels like `prod-std`/`prod-dind` so they don't false-positive). Confirm the YAML parses (`ruby -ryaml -e 'YAML.load_file(ARGV[0])' <file>`). For reusable workflows, re-check the `inputs:`/`secrets:` block matches how callers pass them.
   - **profile / docs** — markdown is well-formed and every referenced asset/link resolves (e.g. `profile/README.md` logo paths point at real files in `profile/assets/`). Remember cross-repo image links don't render on GitHub — assets must live in this repo.
   - **skill** — the `SKILL.md` frontmatter/heading is intact and any `/skill` or `[[link]]` references it makes actually exist in this repo.
   All checks clean before pushing.
4. Push and open a **draft** PR with a structured description (what changed and *why*). Link the epic when there is one.

#### 2b. Self-review

1. Self-review the full diff against this repo's conventions. Read **whole files**, and fix the whole class of any issue, not just the first instance (e.g. if one workflow gains a `runs-on` input, check the sibling workflows).
2. Apply **every** finding — blocking, quality, **and nits**. Autonomous mode applies them all.
3. Re-run the 2a validation after the fix commits.

#### 2c. Flip to ready

```bash
gh pr ready <n>
```
This is the point the default "leave PRs as drafts" caution is overridden — the user pre-authorized it in Phase 0.

#### 2d. Bot-review loop

This repo is reviewed by the Claude bot via its own `claude-review` caller (the dogfooded org workflow). Loop:
- Wait for the bot's review run on the **current HEAD** to finish.
- Apply every actionable finding (whole-class, whole-file — same rigor as 2b). Push the fix commit; HEAD advances → re-arm the wait and repeat.
- Continue until the bot returns a clean review, or it surfaces a blocker you can't mechanically resolve — then STOP and surface to the user. **Never merge over a red bot review.**

*(If the bot isn't configured on this repo, the 2b self-review is the review gate — say so explicitly before merging.)*

**End-to-end note for `affects-callers: yes` PRs:** static checks can't prove a reusable-workflow change works for callers — only a caller run does. After merge, exercise it: open a throwaway PR (or re-run the relevant event) in `plonk`/`flatbed` and confirm the shared workflow still resolves and behaves. Call this out in the final report.

#### 2e. Merge

Precondition at the **same** HEAD: bot review green (or self-review clean if no bot), all 2a validation green.

Squash-merge, pinning the subject to the PR **title** and body to the PR **description** (keeps stray commit-message text off `main`):
```bash
TITLE=$(gh pr view <n> --json title --jq '.title')
BODY=$(gh pr view <n> --json body --jq '.body')
gh pr merge <n> --squash --delete-branch --subject "$TITLE" --body "$BODY"
git fetch origin main && git log -1 origin/main   # verify it landed
```

#### 2f. Prep next PR

- `branch-from: main` → `git fetch origin main` and start 2a fresh from main.
- Stacked → run `/topr <next-pr#>` to drop the just-squashed commits and rebase onto fresh main.

### Phase 3: Final report

When the loop completes OR stops on a blocker:
- Print the merged PR list with URLs and commit SHAs.
- Note which PRs were `affects-callers` and whether the downstream caller check was run + its result.
- If stopped, name the PR, the failing step, and the surfaced output.

## Rules

- Never invent PRs not on the agreed list.
- Never skip nits in self-review under this skill.
- Keep each PR's scope mechanical — no smuggled cleanups.
- Always `actionlint` changed workflows; always confirm referenced assets/links resolve before merging a profile/docs change.
- A reusable-workflow interface change (`affects-callers: yes`) isn't done until a real caller run confirms it — static checks alone don't close it.
- Merge gate: bot review green (or self-review clean, stated) AND all validation green at HEAD.
- Merge form: `gh pr merge <n> --squash --delete-branch --subject "<PR title>" --body "<PR description>"`.
- Run `/topr <pr#>` before re-opening review on a stacked PR.
- Every merged PR leaves `main` in a working state (workflows lint-clean, profile renders).
