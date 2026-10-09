---
name: intent
description: Start a new work item. Interview the originator and draft intent/NNNN-slug/intent.md, the first of the three stage PRs (intent, spec, build).
argument-hint: "[idea in a sentence]"
disable-model-invocation: true
---

# /intent: capture what problem is worth solving

You are running the Plan stage. The output is one file, `intent/NNNN-slug/intent.md`,
merged by a PR titled `intent: NNNN-slug`. Merging it means the product owner accepts
the problem. Closing the PR unmerged means it was rejected.

The idea to start from: $ARGUMENTS

## 1. Start on main

```bash
git status --short --branch
```

If the branch isn't `main`, or there are uncommitted changes, stop and tell the user:

> You're on `<branch>`. Commit or stash that work, run `git switch main && git pull`,
> then run `/intent` again.

Don't switch for them. The usual cause is still sitting on another stage's branch, and
its PR may still need their fixes. Checking now saves an interview that can't be
committed.

## 2. Interview

Talk with the originator like an analyst before writing anything. Ask about who is
affected, what they do today, what "solved" looks like, and what must not change. Use
`AskUserQuestion` for choices, and offer concrete options. Stop when you could fill every
section of [template.md](template.md) without guessing.

Intent is the problem, not the solution. If the conversation drifts into design, write
that down as an open question or a constraint and move on. The spec stage decides how.

## 3. Pick the number and branch

```bash
git fetch origin
git ls-tree --name-only origin/main intent/ 2>/dev/null
gh pr list --state open --json title --jq '.[].title'
```

The next number is the highest four-digit number in either list, plus one (`0001` if
there are none). The slug is short, lowercase and hyphenated. Then:

```bash
git switch -c intent/NNNN-slug origin/main
```

## 4. Write and commit

Copy [template.md](template.md) to `intent/NNNN-slug/intent.md`, set the title, and fill
every section. Keep the `##` headings exactly as they are, because CI compares them with
the template. Show the draft to the originator and fix what they correct. Commit only
that file:

```bash
scripts/check-work-items
git add intent/NNNN-slug/intent.md
git commit -m "Propose intent NNNN-slug"
```

## 5. Handoff

Show the branch, the PR title `intent: NNNN-slug`, and a PR body: two to four lines
summarising the problem and outcome, plus a link to the file. Then ask
**"Push and open the PR?"**

- Yes: `git push -u origin intent/NNNN-slug`, then
  `gh pr create --title "intent: NNNN-slug" --body-file <tmpfile>`, and print the URL.
- No: print those two commands for the user to run.

Never push or open a PR without that yes.

End by telling the user that once the PR is merged, `git switch main && git pull`
brings the intent onto their main, ready for `/spec NNNN`.
