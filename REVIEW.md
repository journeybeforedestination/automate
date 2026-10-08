# Review policy

How Claude reviews pull requests in this repo. The findings are advice for the human who
merges. They never approve or block a PR on their own.

## Find the stage first

Read the PR title (`gh pr view`). It names the stage and the work item:
`intent: NNNN-slug`, `spec: NNNN-slug`, `build: NNNN-slug`, or `chore: ...`. The work
item's files are in `intent/NNNN-slug/`. Read every one that exists before reviewing.

## What to check, by stage

**intent:** Is the problem stated with concrete evidence, not adjectives? Is the outcome
observable without knowing the solution? Are the constraints real limits rather than
design choices made too early? Are the open questions the ones that actually matter?

**spec:**
- Does every constraint in `intent.md` appear as a requirement, a design decision or a
  concern?
- Is every open question in `intent.md` either answered or carried into Concerns with a
  reason?
- Is every acceptance criterion testable, and does it name a requirement?
- Does the design fit the code that exists today?

**build:**
- Does the diff do what `plan.md` says, and no more? Is any drift reflected in `plan.md`?
- Is every acceptance criterion in `spec.md` covered by a test?
- For a bug, was the failing test added before the fix and left unchanged by it?
- Then the usual passes: correctness bugs, then security.

**chore:** Does it really change no behaviour? If it does, say that it needs a work item.

## Severity

- **Important:** wrong behaviour, a security problem, a spec or plan that's missed or
  contradicted, a missing test for an acceptance criterion. Say what breaks and when.
- **Nit:** anything else worth saying. Post at most 3 nits per PR. If there are more,
  keep the 3 most useful.

End with one top-level summary comment that lists the Important findings, or says there
are none.

## Do not report

- Anything CI already enforces: artifact headings, folder names, which files a stage
  may change, PR title format.
- Generated files and lockfiles.
- Style preferences that no convention in `CLAUDE.md` backs up.
- Praise.
