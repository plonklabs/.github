# /implement — Execute an Agreed-Upon Stack of PRs Autonomously

## Description

Fires after a design discussion has settled an ordered list of PRs to ship. For each PR in order: implement → self-review and fix everything including nits → flip to ready → run the bot-review loop concurrently with the project's end-to-end verification → merge → move on. Pure-cleanup PRs skip the end-to-end verification.

Invoking this skill is the **explicit authorization** to flip PRs to ready and merge them autonomously — but only for the PRs on the agreed list, for this invocation. It does not authorize inventing new PRs or merging anything not on the list.

This skill is deliberately **repo-agnostic**. Wherever it says "the project's gate suite" or "the project's end-to-end verification," use the commands the repo actually defines (in its `CLAUDE.md` / `CONTRIBUTING` / CI config). Don't hard-code a toolchain.

## Arguments
- `$ARGUMENTS` — optional GitHub epic/issue number (e.g. `/implement 614`). If given, read that issue's `## Steps` checklist (the format produced by `/spec`) as a starting hint. If both a chat list and an epic are present, the **chat conversation is the source of truth**; the epic is only consulted when the chat list is implicit.

## Instructions

When the user runs `/implement` or `/implement <epic>`, execute the following phases.

### Phase 0: Resolve the PR list

1. **If `$ARGUMENTS` was passed**, fetch the epic body and extract unchecked `- [ ]` items under `## Steps`:
   ```bash
   gh issue view <n> --json body --jq '.body'
   ```
   These are the candidate PRs.
2. **Otherwise**, re-state the ordered PR list inferred from the immediately preceding chat conversation.
3. **Print** the numbered list back to the user before doing anything else.
4. **If the list is ambiguous** (no clear order, zero items, scope missing from one or more entries), STOP and ask the user to restate it. Never invent a PR not on the agreed list.
5. **State authorization once**: print one line confirming that this `/implement` invocation authorizes autonomous flip-to-ready and merge for the listed PRs only.

### Phase 1: Classify each PR

Annotate every PR with these fields and print the classification table before the loop starts:

- **`pure-cleanup: yes/no`** — yes if the entire PR is formatting / dead-code deletion / comment hygiene / dependency bumps with no behaviour change. Pure-cleanup PRs skip end-to-end verification.
- **`e2e: required/skip`** — `required` for any PR that changes runtime behaviour, public contracts, or anything the project's end-to-end / integration suite exercises; `skip` for pure-cleanup and docs-only PRs. The last non-cleanup PR in the stack is always `required` — even a thin wrapper gets exercised end-to-end once.
- **`branch-from: main | <prior-pr-branch>`** — default `main` (merge-first). Only stack when the user explicitly asked, or when a PR's diff cannot be built without the prior PR's code.

### Phase 2: Per-PR loop

For each PR in order:

#### 2a. Implement

1. Branch off `origin/main` (or the prior PR's branch when `branch-from` says so):
   ```bash
   git fetch origin main
   git switch -c feature/<descriptive-slug> origin/main
   ```
2. Implement **only** the changes scoped to this PR. Keep the scope mechanical — don't fold in unrelated cleanups.
3. Run the project's **local gate suite** — formatter, linter, and unit tests for the touched area (the exact commands are whatever the repo defines). All green before pushing.
4. Push and open a **draft** PR with a structured description (what changed and *why*, so a reader understands it without reading the diff). Link the epic when there is one. If the repo has a dedicated `/pr` skill, use it.

Push as soon as the code compiles and unit tests pass — the draft PR is a checkpoint. End-to-end verification doesn't gate the draft push (it runs in Lane B later).

#### 2b. Self-review

1. Self-review the full diff against the repo's quality standards (read its `CLAUDE.md` / style docs). Read **whole files**, not just diff hunks, and fix the whole class of any issue you find, not just the first instance.
2. Apply **every** finding — blocking, quality, **and nits**. Autonomous mode does not skip nits.
3. Re-run the gate suite after the fix commits.
4. Self-review is "clean" when a re-run would surface only cosmetic phrasing already addressed and no behavioural issues remain.

#### 2c. Flip to ready

```bash
gh pr ready <n>
```

This is the point the default "leave PRs as drafts" caution is overridden — the user pre-authorized ready/merge for the agreed list in Phase 0.

#### 2d. Concurrent gates — bot review + end-to-end verification in parallel

Run the two lanes **concurrently** against the same HEAD to minimize wall-clock; serial wastes minutes when both are slow.

**Lane A — bot review** (foreground): the repo's Claude review bot (the org `claude-review` workflow, run from the repo's `.github/workflows/claude-review.yml`) reviews every ready PR. Loop:
- Wait for the bot's review run on the **current HEAD** to finish.
- Apply every actionable finding (whole-class, whole-file — same rigor as 2b). Push the fix commit.
- Pushing advances HEAD → re-arm the wait on the new HEAD and repeat.
- Continue until the bot returns a clean/green review, or it surfaces a blocker. If it stays non-green on a real issue you can't mechanically resolve, STOP and surface to the user. **Never merge over a red bot review.**

**Lane B — end-to-end verification** (background, started right after 2c when `e2e: required`):
- Run the project's end-to-end / integration suite via a background task; watch it.
- If it needs an exclusive resource (a shared cluster, database, or device), run its internal steps **sequentially** — but the whole pipeline still runs in parallel with Lane A.
- Use **only the project's sanctioned tooling** to mutate any environment. Read-only inspection for diagnosis is fine; never reach around the sanctioned path to "finish" something, that hides real bugs.
- If `e2e: skip`, Lane B is a no-op for this PR.

**HEAD-pinning**: tag every Lane B run with the commit SHA it started against. When Lane A pushes a fix:
- Lane B still running against the old SHA → kill it and restart against the new SHA.
- Lane B already finished against an old SHA → discard its result and restart.

The merge precondition is "Lane A green AND Lane B green (where required) against the **same** SHA." Drift means re-run.

**Failure handling**:
- **Flake** (transient infra/network/port): one automatic retry. A second failure is real.
- **Real failure**: push a fix commit — Lane A picks it up via re-review; Lane B restarts (HEAD-pinning).
- **Design-level failure** (the failure reveals a problem in the agreed PR's design, not a mechanical bug): STOP and surface to the user.

#### 2e. Merge

Precondition (all true at the **same** HEAD): Lane A green, Lane B green where required, local gate suite green.

Squash-merge per the repo's policy. Pin the merge subject to the PR **title** and the body to the PR **description**, rather than letting GitHub concatenate every in-PR commit message — the PR description is the canonical record reviewers actually saw, and pinning it keeps stray commit-message text (including anything a fix commit introduced) out of `main`:

```bash
TITLE=$(gh pr view <n> --json title --jq '.title')
BODY=$(gh pr view <n> --json body --jq '.body')
gh pr merge <n> --squash --delete-branch --subject "$TITLE" --body "$BODY"
git fetch origin main && git log -1 origin/main   # verify the merge landed
```

#### 2f. Prep next PR

- `branch-from: main` → `git fetch origin main` and start 2a fresh from main.
- Stacked on this one → run `/topr <next-pr#>` to drop the just-squashed commits and rebase onto fresh main.

### Phase 3: Final report

When the loop completes (all PRs merged) OR stops on a blocker:
- Print the merged PR list with URLs and commit SHAs.
- For each PR, state which gates ran: bot review (always), end-to-end (yes/no + result).
- If stopped, name the PR, the failing step, and the surfaced output.

## Rules

- Never invent PRs not on the agreed list.
- Never skip nits in self-review under this skill — autonomous mode applies them all.
- Keep each PR's scope mechanical — don't smuggle in unrelated cleanups.
- Pure-cleanup PRs (formatting / dead-code / dependency bumps, no behaviour change) skip end-to-end verification — even when last in the stack.
- Use only the project's sanctioned tooling to mutate any test environment; read-only inspection for diagnosis is fine.
- HEAD-pinning: every Lane B run is tagged with the SHA it started on; a Lane A fix kills and restarts it. Merge requires both lanes green at the **same** SHA.
- Merge gates: bot review green AND end-to-end green (where required) AND the local gate suite green at HEAD. All must hold — no exceptions.
- Flake policy: one automatic retry; a second failure is real — don't paper over it.
- Merge form: `gh pr merge <n> --squash --delete-branch --subject "<PR title>" --body "<PR description>"`.
- Run `/topr <pr#>` before re-opening review on a stacked PR.
- Every merged PR must leave the system in a working state.
