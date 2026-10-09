---
name: spec
description: Turn an accepted intent into intent/NNNN-slug/spec.md (requirements, acceptance criteria, design), the second of the three stage PRs.
argument-hint: "NNNN"
disable-model-invocation: true
---

# /spec: decide what to build

You are running the Design stage for work item $ARGUMENTS. The output is
`intent/NNNN-slug/spec.md`, merged by a PR titled `spec: NNNN-slug`. Merging it means
the design is accepted and Build may start.

## 1. Start from main

```bash
git fetch origin
git ls-tree --name-only origin/main intent/ | grep '^intent/NNNN-'
git switch -c spec/NNNN-slug origin/main
```

If `intent.md` for this number is not on `origin/main`, stop. Its intent PR has not been
accepted, and CI would reject the spec PR anyway.

## 2. Read before writing

Read `intent.md`, `CLAUDE.md`, and the code the intent touches. The spec has to fit the
system that exists, not an imagined one.

## 3. Write the spec

Copy [template.md](template.md) to `intent/NNNN-slug/spec.md` and fill every section.
Keep the `##` headings exactly as they are.

- Every intent **constraint** shows up as a requirement, a design decision, or a concern.
- Every intent **open question** is either answered here or listed under Concerns with
  the reason it can wait. Ask the user when only they can answer.
- Every acceptance criterion is testable and names its requirement. These become the
  tests in the build PR.
- **Bugs:** the first acceptance criterion names the failing test that reproduces the bug.
- If a policy or constraint conflicts with the design, list it under Concerns. Do not
  decide it silently.

Fixing a mistake in `intent.md` in this same PR is allowed. Say so in the PR body.

```bash
scripts/check-work-items
git add intent/NNNN-slug/
git commit -m "Specify NNNN-slug"
```

## 4. Handoff

Show the branch, the PR title `spec: NNNN-slug`, and a PR body: two to four lines
summarising the requirements and any concerns that need a decision, plus a link to the
file. Then ask **"Push and open the PR?"**

- Yes: `git push -u origin spec/NNNN-slug`, then
  `gh pr create --title "spec: NNNN-slug" --body-file <tmpfile>`, and print the URL.
- No: print those two commands for the user to run.

Never push or open a PR without that yes.
