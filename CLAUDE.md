# CLAUDE.md

## Commands

- `scripts/status`: each work item's status, plus open PRs. Healthy output:
  ```
  0001-walking-skeleton                    done
  0002-user-login                          ready-to-build

  In flight (open PRs):
    #7 spec: 0003-export-csv
  ```
- `scripts/check-work-items`: validates `intent/`. Healthy output is silent, with exit code 0.
- `BASE=origin/main scripts/check-pr "<pr title>"`: validates the branch's changes
  against the stage in the title. Healthy output is silent, with exit code 0. (CI runs
  it without `BASE` on the PR merge commit.)

There is no build or test command yet. Work item 0001 adds them here.

## Conventions

- All work flows through the process in [README.md](README.md#how-work-gets-done):
  `/intent`, then `/spec`, then `/build`, one PR each. Changes that don't alter behaviour
  go in a `chore:` PR.
- Never edit another work item's folder under `intent/`.
- Never push or open a PR without asking the user first.

## Architecture

None yet. Work item 0001 (the walking skeleton) defines it: a .NET backend and a
React + TanStack frontend.

## Things Claude gets wrong

When Claude makes the same mistake twice, add the correction here.
