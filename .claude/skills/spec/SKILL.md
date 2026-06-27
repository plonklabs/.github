# /spec — Create a Change Spec with a GitHub Epic Issue

## Description

Guided workflow for designing a change to **this repo** (`plonklabs/.github`) and creating a GitHub epic issue to track it. Use it for anything bigger than a one-off edit: a new reusable workflow, a profile redesign, a new shared skill, issue/PR templates, or a multi-PR refactor of the existing workflows. The output is one epic issue whose `## Steps` checklist feeds `/implement`.

## Arguments
- `$ARGUMENTS` — the change name or a short description (e.g. `/spec add stale-issue workflow`).

## Instructions

When the user runs `/spec <name-or-description>`, follow this workflow:

### Phase 1: Gather Information

Walk the user through each section, one at a time, offering to draft content. The sections:

1. **Context** — What problem does this solve? What's the current state in the repo?
2. **Proposal** — High-level solution (1–2 paragraphs).
3. **User Flows** — Concrete end-to-end scenarios from the relevant actor's perspective. For this repo the "user" is usually a **caller repo**, a **maintainer**, or **the automation itself** (e.g. "a PR is merged in `plonk` → the `discord-pr` reusable workflow posts an embed"). Number the steps.
4. **Acceptance Criteria** — Checkbox list of observable, testable outcomes (e.g. "`actionlint` is clean", "the profile renders the logo in dark mode", "a caller at `@main` resolves the new input"). Verifiable without reading the diff.
5. **Design** — The concrete changes: workflow `inputs:`/`secrets:`/job steps, README/profile structure, skill phases, or template fields. Include YAML/markdown snippets where they clarify.
6. **Changes Required** — The specific files/paths affected (`.github/workflows/…`, `profile/…`, `.claude/skills/…`, templates).
7. **Dependencies** — GitHub features relied on, **caller repos affected** (a reusable-workflow interface change ripples to `plonk`/`flatbed`/…), and any **org settings** (e.g. Actions "Accessible from repositories in the organization", branch protection).

For each section, present what you have and ask the user to confirm or revise before moving on.

### Phase 2: Break Down into Steps

Once the spec is complete:

1. Propose implementation steps as a **checkbox list** in the epic body. Each step ≈ one PR with a clear imperative description.
2. For steps that change a **reusable workflow's interface**, note the affected callers in the step text — `/implement` keys its downstream caller check off this.
3. Include enough detail that someone picking up a step knows what to do.
4. Ask the user to confirm, reorder, split, or merge steps.

**Do NOT create sub-issues.** All steps live as checkboxes in the epic body — avoids notification spam and keeps tracking in one place.

### Phase 3: Review & Create

1. Present the **full epic issue** for final review.
2. Only after explicit approval, create it with `gh issue create`.
3. Apply an `epic` label (create it if the repo doesn't have one) and any relevant labels.

### Issue Format

**Epic issue:**
```
## Context
...

## Proposal
...

## User Flows

**<Caller repo / maintainer / automation> does <action>:**
1. Step from that actor's perspective
2. What happens next
3. Observable outcome

## Acceptance Criteria

- [ ] Testable outcome 1
- [ ] Testable outcome 2

## Design
...

## Changes Required
...

## Dependencies
...

## Steps
- [ ] **Step 1: <imperative description>** — files changed, callers affected, how to verify
- [ ] **Step 2: <imperative description>** — ...
```

Each step checkbox is self-contained: bold title, then a dash and enough context to work from. When a step is done, check the box and reference the PR in a comment.

User flows define the "what" from the actor's perspective. Acceptance criteria define "done" with testable outcomes. Steps define the "how" PR by PR.

### Rules

- Never create issues without explicit user approval.
- Use imperative mood for step titles ("Add X", "Refactor Y", not "Added X").
- Keep steps small enough for a single squash-merged PR.
- Do NOT create separate sub-issues — use checkbox lists in the epic body.
- One epic per change area; fold related work into existing epics rather than spawning new ones.
- Always surface caller-repo and org-setting dependencies — they're the easiest things to forget in a `.github` repo.
