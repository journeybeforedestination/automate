---
name: build
description: Implement an accepted spec. Agree on intent/NNNN-slug/plan.md in plan mode, then write the code and tests in the same PR, the last of the three stage PRs.
argument-hint: "NNNN"
disable-model-invocation: true
---

# /build: plan it, then build it

You are running the Build stage for work item $ARGUMENTS. The output is one PR titled
`build: NNNN-slug` containing `intent/NNNN-slug/plan.md`, the code, and the tests.
Merging it with CI green means the work item is done.

## 1. Start from main

```bash
git status --short --branch
```

If the branch isn't `main`, or there are uncommitted changes, stop and tell the user:

> You're on `<branch>`. Commit or stash that work, run `git switch main && git pull`,
> then run `/build` again.

Don't switch for them. The usual cause is still sitting on the last stage's branch, and
its PR may still need their fixes.

```bash
git fetch origin
git ls-tree --name-only origin/main intent/ | grep '^intent/NNNN-'
git switch -c build/NNNN-slug origin/main
```

If `spec.md` for this number is not on `origin/main`, stop. Its spec PR has not been
accepted.

## 2. Plan, and get it approved before any code

Read `intent.md`, `spec.md`, `CLAUDE.md` and the code involved. Ask the engineer about
the repo where the code doesn't answer. Then propose a plan shaped like
[template.md](template.md): Files that change, Order of work, Risks, Proof. Every
acceptance criterion in the spec must be covered in Proof.

Present it for approval with plan mode (`ExitPlanMode`), or, if not in plan mode, ask
explicitly. Make no edits until the user approves.

## 3. Commit the plan first

Write the approved plan to `intent/NNNN-slug/plan.md`. Keep the `##` headings exactly as
they are.

```bash
scripts/check-work-items
git add intent/NNNN-slug/plan.md
git commit -m "Plan NNNN-slug"
```

## 4. Build

Follow the plan's order of work.

- **Bugs:** commit the failing test on its own first, then fix it without editing that test.
- When the work departs from the plan, update `plan.md` in the same commit as the change
  that departs from it. Reviewers compare the diff with the plan.
- Do not edit any other work item's folder.

## 5. Prove it

Run the repo's test command from `CLAUDE.md` and show the output. The task isn't done
until the tests have been run and their output shown. If something fails, fix it or
report it; never call it done.

## 6. Handoff

Show the branch, the PR title `build: NNNN-slug`, and a PR body: two to four lines on
what was built, links to the work item's files, and the test output summary. Then ask
**"Push and open the PR?"**

- Yes: `git push`, then
  `gh pr create --title "build: NNNN-slug" --body-file <tmpfile>`, and print the URL.
- No: print those two commands for the user to run.

Never push or open a PR without that yes.

End by telling the user that once the PR is merged, `git switch main && git pull`
puts them back on main, ready for the next work item.
