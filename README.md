# automate

A .NET backend with a React + TanStack frontend, built as a software factory. Every change
starts as a written intent and reaches `main` through reviewed pull requests, with Claude
drafting each artifact and a human accepting it.

There is no application code yet. The first work item, `0001-walking-skeleton`, creates it
by going through the process described below.

The process follows Anthropic's
[AI-native SDLC playbook](https://claude.com/blog/the-ai-native-sdlc-playbook). We use its
Plan, Design, Build and Test stages; Deploy and Maintain come later. The reasoning behind
each choice here is in [docs/factory-decisions.md](docs/factory-decisions.md).

## How work gets done

Each piece of work is a **work item**: a folder `intent/NNNN-slug/` that gains one file per
stage. Each stage is one pull request, and merging that PR is the acceptance.

```
idea ──/intent──▶ PR "intent: 0003-export-csv"  adds intent.md      merge = problem accepted
     ──/spec────▶ PR "spec: 0003-export-csv"    adds spec.md        merge = design accepted
     ──/build───▶ PR "build: 0003-export-csv"   adds plan.md        merge = done
                                                + code + tests
```

**Done** means the build PR is merged into `main` with every required check green.

### 1. Intent: what problem is worth solving

| | |
|---|---|
| Who starts it | Anyone with an idea or a bug |
| Command | `/intent <the idea in a sentence>` in Claude Code |
| Artifact | `intent/NNNN-slug/intent.md`: Problem, Outcome, Users and systems affected, Constraints, Open questions |
| Branch / PR title | `intent/NNNN-slug` / `intent: NNNN-slug` |
| Who accepts | The product owner, by merging. Closing the PR unmerged rejects the intent. |

Claude interviews you before it writes anything. The intent describes the problem, not the
solution.

### 2. Spec: what we will build

| | |
|---|---|
| Who starts it | Whoever picks up an accepted intent |
| Command | `/spec NNNN` |
| Artifact | `intent/NNNN-slug/spec.md`: Requirements, Acceptance criteria, Design, Out of scope, Concerns |
| Branch / PR title | `spec/NNNN-slug` / `spec: NNNN-slug` |
| Who accepts | The product owner, plus a tech lead for risky work, by merging |

Claude reads the intent and the code. Every intent constraint and open question has to be
covered, and every acceptance criterion has to be testable.

### 3. Build: plan it, then build it

| | |
|---|---|
| Who starts it | An engineer |
| Command | `/build NNNN` |
| Artifact | `intent/NNNN-slug/plan.md` (Files that change, Order of work, Risks, Proof), plus the code and tests |
| Branch / PR title | `build/NNNN-slug` / `build: NNNN-slug` |
| Who accepts | The reviewer, by merging once CI is green |

Claude proposes the plan in plan mode, and you approve it before any code is written. The
plan is the PR's first commit. If the work departs from the plan, `plan.md` is updated in
the same commit. Claude runs the tests and shows the output before it says it's done.

**Every skill ends by asking "Push and open the PR?"** If you say yes, it pushes the branch
and opens the PR with a drafted title and body. If you say no, it prints the two commands
for you to run. The push is a plain `git push`, which needs
`git config --global push.autoSetupRemote true` set once on your machine. Pushing with
`-u` would have Claude write to `.git/config`, which its sandbox doesn't allow.

### Bugs

Bugs are work items too. The spec's first acceptance criterion names a failing test. The
build PR commits that test before the fix, and the fix doesn't change the test.

### Chores

Changes that don't alter behaviour (docs, typos, dependency bumps, CI or process tweaks)
skip the work-item flow. Open one PR titled `chore: <summary>`. Chore PRs may not touch
`intent/`. CI can't tell whether behaviour changed, so that's the reviewer's call.

## Where things stand

There is no tracker. A work item's status is which of its files have merged to `main`, so
it can never drift:

```
$ scripts/status
0001-walking-skeleton                    done
0002-user-login                          ready-to-build
0003-export-csv                          intent-accepted

In flight (open PRs):
  #12 spec: 0003-export-csv
```

| On main | Status |
|---|---|
| `intent.md` | intent-accepted |
| `intent.md` + `spec.md` | ready-to-build |
| `intent.md` + `spec.md` + `plan.md` | done |

Rejected intents are closed PRs: `gh pr list --state closed --search "intent: in:title"`.

## What CI checks

| Check | Required to merge | What it does |
|---|---|---|
| `process` | yes | `scripts/check-work-items` checks that every work item folder is named `NNNN-slug`, that numbers are unique, that the artifacts exist in order, and that each artifact's `##` headings match its template in `.claude/skills/<stage>/template.md`. `scripts/check-pr` checks that the PR title names a stage, that the previous stage is already on `main`, and that the PR only changes files its stage allows. |
| `claude-review` | no (advisory) | Claude reviews the PR against [REVIEW.md](REVIEW.md) and the work item's intent, spec and plan, and posts inline comments. |

There are no build or test checks yet. Work item 0001 adds them and makes them required.

`main` is protected: every change arrives by squash-merged PR, the required checks must
pass, and branches must be up to date with `main`. That last rule is what catches two PRs
picking the same work item number. No approval count is required, because GitHub doesn't
let you approve your own PR, and the repo has one maintainer for now.

## Repository map

| Path | What it is |
|---|---|
| `intent/` | Work items, one folder each |
| `.claude/skills/{intent,spec,build}/` | The stage skills and their templates |
| `scripts/` | `status`, `check-work-items`, `check-pr` |
| `.github/workflows/` | `process.yml` (required), `claude-review.yml` (advisory) |
| `CLAUDE.md` | What Claude needs to know to work here: commands, conventions, known mistakes |
| `REVIEW.md` | Claude's review policy for PRs |
| `docs/factory-decisions.md` | Why the process is built this way, including what was rejected |

## One-time setup

These steps are needed once per repository, by someone with admin rights.

1. **Let Claude review PRs.** On a Claude Pro or Max plan, run `claude setup-token` and
   store the token:
   ```bash
   gh secret set CLAUDE_CODE_OAUTH_TOKEN
   ```
   To use API billing instead, set `ANTHROPIC_API_KEY` and change `claude_code_oauth_token`
   to `anthropic_api_key` in `.github/workflows/claude-review.yml`.
2. **Squash merges only.**
   ```bash
   gh repo edit --enable-squash-merge --enable-merge-commit=false \
     --enable-rebase-merge=false --delete-branch-on-merge
   ```
3. **Protect `main`.** Do this once the `process` check has run on at least one PR:
   ```bash
   gh api -X POST repos/journeybeforedestination/automate/rulesets --input - <<'JSON'
   {"name":"main","target":"branch","enforcement":"active",
    "conditions":{"ref_name":{"include":["~DEFAULT_BRANCH"],"exclude":[]}},
    "rules":[
     {"type":"pull_request","parameters":{"allowed_merge_methods":["squash"],"dismiss_stale_reviews_on_push":false,"require_code_owner_review":false,"require_last_push_approval":false,"required_approving_review_count":0,"required_review_thread_resolution":false}},
     {"type":"required_status_checks","parameters":{"strict_required_status_checks_policy":true,"required_status_checks":[{"context":"process"}]}},
     {"type":"non_fast_forward"},{"type":"deletion"}]}
   JSON
   ```
   When a new CI job must pass before merging, add its job name to
   `required_status_checks`. The name has to match the job id exactly, or every PR waits
   forever for a check that never reports.

## Not covered yet

- Deploy and Maintain (the playbook's stages 5 and 6).
- Evals for `CLAUDE.md` and the skills, and Claude Code hooks. They need code and a test
  command to act on.
- A second maintainer. When one joins, add `CODEOWNERS` and require 1 approval.

**Next:** run `/intent` for `0001-walking-skeleton`.
