# Factory decisions

This is the plan that set up the software factory, kept as the record of why the process
is built the way it is. The steps describe the setup as it was planned; the README
describes how things are now. Read "Considered and rejected" before proposing a change to
the process.

## What changes

Today the repo has only a README, LICENSE and a .NET `.gitignore`. When this plan is done, it holds the process for delivering work, but no application code. That process follows Anthropic's AI-Native SDLC playbook (https://claude.com/blog/the-ai-native-sdlc-playbook), which takes work from an idea to merged code like this:

- **Three artifacts and three PRs per work item.** Every work item lives in `intent/NNNN-slug/` and moves forward through three PRs. Each PR merges one artifact: `intent.md`, then `spec.md`, then `plan.md` together with the code and tests.
- **Skills drive each stage.** Each stage is run by a project skill (`/intent`, `/spec`, `/build`).
- **CI enforces the order.** A required CI check, `process`, rejects any PR that skips a stage or touches files outside its stage.
- **Claude reviews every PR, but can't block it.** It reviews against `REVIEW.md` and leaves advisory comments only.
- **The README explains the whole flow** and describes only what really exists.

Application code is out of scope. The .NET + React/TanStack walking skeleton becomes work item `0001`, and it is the first thing delivered *through* this process.

## The load-bearing decision: state is the files on main, and CI guards how they get there

We have no tracker. A work item's status is simply which of its artifacts are on `main`:

| On main | Status |
|---|---|
| `intent.md` | accepted |
| `+ spec.md` | ready-to-build |
| `+ plan.md` | done (plan.md merges in the same PR as the code) |

Work that is still in progress is open PRs. A closed intent PR that was never merged means the intent was rejected.

That only works if **no PR can put an artifact on main out of order.** Here is what goes wrong without the `process` check:

1. Someone opens `spec: 0003-export-csv`, adding `intent/0003-export-csv/spec.md` and also `src/Export.cs`.
2. Main now says 0003 is *ready-to-build*. In fact code is already shipped that no one reviewed against an accepted spec.
3. Later, `build: 0004-login` adds `plan.md` for 0004 while 0004's spec PR is still open. `scripts/status` reports 0004 as *done*, even though its design was never accepted.

The status table is now false, and nothing about it looks wrong. So `scripts/check-pr` (Step 2) is not optional polish. It is the thing that makes "status = files on main" true. Everything else in this plan builds on it.

## Decisions (already settled with the user; don't reopen)

- **Three PRs per work item, with the PR title stating its stage:**
  - `intent: NNNN-slug`
  - `spec: NNNN-slug`
  - `build: NNNN-slug`

  The `build` PR's first commit is `plan.md`. That plan has already been approved interactively in Claude plan mode before any code is written.
- **Folder IDs are sequential, four digits:** `intent/0001-walking-skeleton/`. CI rejects duplicate numbers.
- **Status is derived, never stored.** There is no `status:` field anywhere.
- **`chore:` PRs skip the work-item flow.** They are for changes that don't alter behaviour: docs, typos, dependency bumps, CI and process tweaks. They may not touch `intent/`. Whether a change is "no behaviour change" is up to the reviewer, because CI can't tell.
- **Bugs take the full path.** The bug's spec names a failing test, which the build PR commits before the fix (a playbook recommendation).
- **Approvals: solo setup.** Changes reach main only through PRs, the `process` check must pass, squash merge only, no force pushes or deletions, and **0 required approvals**. GitHub won't let the author approve their own PR. The user acts as product owner and tech lead.
- **AI review: `REVIEW.md` + `anthropics/claude-code-action@v1`.** It runs on every PR and is read-only: it comments, never pushes, and is **not** a required check. There is no `@claude` responder workflow.
- **CI on day one checks process artifacts only.** Build, test and lint arrive with 0001.
- **New third-party dependency:** `anthropics/claude-code-action` (plus `actions/checkout`). The scripts use only bash, git and grep. Tell the user about each one.

## The work

Do all of this on one branch, `chore/software-factory`, as one `chore: set up software factory` PR. Use one commit per step, in this order. **Why this order:** the templates are the single source of truth for section headings, so the checker that reads them comes after them. The workflow that runs the checker comes after the checker. The README comes last, because it has to describe only what already exists.

The user's global instructions say not to write to GitHub without asking. The implementer **commits locally and stops**. The user pushes, opens the PR and does Step 6.

### Step 1: stage templates and skills

Files:
- `.claude/skills/intent/SKILL.md`, `.claude/skills/intent/template.md`
- `.claude/skills/spec/SKILL.md`, `.claude/skills/spec/template.md`
- `.claude/skills/build/SKILL.md`, `.claude/skills/build/template.md`

The template headings (H2 only; the checker compares them exactly) are:

- **intent:** `## Problem`, `## Outcome`, `## Users and systems affected`, `## Constraints`, `## Open questions`. This is the playbook's intent.md contents.
- **spec:** `## Requirements`, `## Acceptance criteria`, `## Design`, `## Out of scope`, `## Concerns`. The rules for filling them:
  - Number the requirements `R1…`.
  - Each acceptance criterion must be testable and name the requirement it proves.
  - Concerns are policy or constraint conflicts the spec author flags for the reviewer.
- **build (plan):** `## Files that change`, `## Order of work`, `## Risks`, `## Proof`. These are the playbook's headings, verbatim.

Each file starts with `# NNNN-slug: Intent` (or `Spec` or `Plan`). Headings below H2 are free-form.

Skill frontmatter: `name`, `description`, `disable-model-invocation: true` (stages are started by a human, never auto-loaded), and `argument-hint`. Here is what each skill does:

- **`/intent [idea]`**
  1. Interviews the originator, analyst style.
  2. Picks the next number: the max of the numbers in `intent/` on `origin/main` and the numbers in open PR titles (`gh pr list --state open --json title`), plus 1.
  3. Creates branch `intent/NNNN-slug`, writes `intent.md` from the template and commits.
  4. Runs the **handoff** with title `intent: NNNN-slug` (see below).
- **`/spec NNNN`**
  1. Requires `intent.md` on `origin/main`.
  2. Reads it and the codebase, and writes `spec.md` on branch `spec/NNNN-slug`.
  3. Must resolve or carry forward every intent open question, and cover every intent constraint.
  4. For bugs, the acceptance criteria name the failing test.
  5. Runs the handoff with title `spec: NNNN-slug`.
- **`/build NNNN`**
  1. Requires `spec.md` on `origin/main`.
  2. Produces the plan in plan mode and waits for approval.
  3. Then, on branch `build/NNNN-slug`: commits `plan.md` first, then implements. Bugs get the failing test committed before the fix.
  4. Runs the tests and shows the output before saying it's done (the playbook's definition of done).
  5. If the implementation drifts from the plan, it updates `plan.md` in the same commit.
  6. Runs the handoff with title `build: NNNN-slug`.

**The handoff** is the same final step in all three skills. Each skill describes it inline rather than sharing a file, so each one stays readable on its own.
1. Show the branch, the PR title and a drafted PR body. The body is a two-to-four-line summary of the artifact, plus links to the work item's files. For `build`, it also includes the test output.
2. Ask: "Push and open the PR?"
3. **Yes:** run `git push -u origin <branch>`, then `gh pr create --title "<title>" --body-file <tmpfile>`, and print the PR URL.
4. **No:** print those two commands for the user to run.

Never push or open a PR without that yes. This follows the user's rule that GitHub writes are confirmed first, and it was their explicit choice over "print only" and "fully automatic". Claude Code's own permission prompt on `git push` and `gh pr create` is a second check on top of this, not a replacement for it.

Gate: none (no CI yet). This step falsifies no docs.

### Step 2: check scripts

Files: `scripts/check-work-items`, `scripts/check-pr` and `scripts/status`. All are executable bash with `set -euo pipefail`.

**`scripts/check-work-items`** checks the whole tree; run it on the PR merge commit. It succeeds if `intent/` is missing.
- Every directory matches `^[0-9]{4}-[a-z0-9]+(-[a-z0-9]+)*$`, and no number repeats.
- Every folder has `intent.md`. `spec.md` requires `intent.md`, and `plan.md` requires `spec.md`.
- For each artifact, `grep '^## '` must equal the `^## ` lines of its template in `.claude/skills/<stage>/template.md`, where `plan.md` maps to the `build` template.

**`scripts/check-pr "<title>"`** checks the diff `git diff --name-only --no-renames "$base"...HEAD`. In CI, HEAD is the PR merge commit and `base` defaults to `HEAD^1`. Locally, `BASE=origin/main` checks a whole branch.

| Title | Allowed changes | Must already be on base (`git cat-file -e $base:path`) |
|---|---|---|
| `intent: <id>` | only `intent/<id>/intent.md` | nothing |
| `spec: <id>` | only `intent/<id>/{intent,spec}.md` | `intent/<id>/intent.md` |
| `build: <id>` | anything outside `intent/`, plus `intent/<id>/*`; `intent/<id>/plan.md` must exist at HEAD | `intent/<id>/spec.md` |
| `chore: …` | anything not under `intent/` | nothing |

Any other title fails, and the error message says what the allowed forms are.

**`scripts/status`** lists each `intent/*` folder on `origin/main` (via `git ls-tree`) with its derived status. If `gh` is on PATH, it then lists open PRs. It does a `git fetch` first.

Gate: the user runs the scripts locally (see Proof). This step falsifies no docs.

### Step 3: CI workflow `.github/workflows/process.yml`

- Triggers on `pull_request` with types `[opened, edited, synchronize, reopened]` (see Traps for why `edited` is needed).
- Has one job with the id **`process`**, which becomes the required check's name. Permissions are `contents: read`.
- Uses `actions/checkout@v7` with `fetch-depth: 2`.
- Runs `scripts/check-work-items`, then `scripts/check-pr "$TITLE"`, with `TITLE: ${{ github.event.pull_request.title }}` passed through `env:` and never interpolated into the script (see Traps).

Gate: the setup PR itself must pass. Its title, `chore: set up software factory`, is a valid chore that touches nothing in `intent/`.

### Step 4: AI review: `REVIEW.md` + `.github/workflows/claude-review.yml`

**`REVIEW.md`** follows the playbook's recommendation:
- **Passes:** bugs, security, and compliance with `intent.md`/`spec.md`/`plan.md`.
- **Severity:** findings are either **Important** or **Nit**, with at most 3 nits.
- **Don't report** anything CI already enforces (artifact shape, stage paths) or generated files.
- **Checklists by stage**, worked out from the PR title:
  - **intent:** is the problem concrete, is the outcome observable, are the constraints real?
  - **spec:** does every intent constraint and open question have a home (this closes the playbook's weak spec gate), and is every acceptance criterion testable?
  - **build:** does the diff match `plan.md` and meet every acceptance criterion with a test, and is any drift reflected in `plan.md`?
  - **chore:** does it really change no behaviour?

**`claude-review.yml`** is adapted from the action's `examples/pr-review-comprehensive.yml`:
- `on: pull_request` with types `[opened, synchronize, ready_for_review, reopened]`.
- `if: github.event.pull_request.head.repo.full_name == github.repository` (see Traps: forks).
- Permissions `contents: read`, `pull-requests: write`, `id-token: write`.
- `actions/checkout@v7` with `fetch-depth: 1`.
- Action inputs:
  - `claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}`
  - `github_token: ${{ secrets.GITHUB_TOKEN }}`
  - a `prompt` telling it to follow `REVIEW.md`
  - `claude_args: --allowedTools "mcp__github_inline_comment__create_inline_comment,Bash(gh pr comment:*),Bash(gh pr diff:*),Bash(gh pr view:*)"` (taken from the example)

We pass `GITHUB_TOKEN` rather than installing the Claude GitHub App. The App requests Contents: write, while `GITHUB_TOKEN` is limited to the job's read-only contents permission. That fits the user's caution about agents writing. **Verify step:** on the first real PR, confirm inline comments post when using `github_token` with no App installed. If they don't, the fallback is installing the App (https://github.com/apps/claude), and the user decides that.

Gate: the workflow runs on the setup PR. Its failure does not block merging until the secret exists.

### Step 5: `CLAUDE.md` + `README.md`

**`CLAUDE.md`** stays under a page and uses the playbook's sections:
- **Commands:** `scripts/status`, `scripts/check-work-items`, `scripts/check-pr`, each with an example of healthy output.
- **Conventions:** a one-paragraph pointer to the README's process section, plus "never edit another work item's folder".
- **Architecture:** "none yet; 0001 defines it".
- **Things Claude gets wrong:** empty. The rule is that when Claude makes the same mistake twice, the correction goes here.

**`README.md`** replaces the current two lines (`README.md:1-2`, which are falsified by this change). It must cover:
1. One paragraph saying what this repo is: a .NET backend + React/TanStack frontend, built through the process below.
2. The flow at a glance: intent → spec → build, one PR each, and "done = build PR merged with `process` (and later build/test) green".
3. Per stage: who starts it, the command, the artifact, the branch and PR title, who approves, and what CI checks. Also say that every skill ends by asking before it pushes and opens the PR.
4. `scripts/status` and how status is derived; that a closed intent PR means rejected.
5. Bugs (a failing test first) and `chore:` PRs.
6. What CI runs today and that AI review is advisory.
7. One-time setup (Step 6), written as instructions for the user.
8. What isn't covered yet: deploy, maintain, evals, hooks. The next work item is `0001-walking-skeleton`.
9. A link to the playbook.

Gate: a reader of the README alone could take an idea to a merged build PR. This step falsifies `README.md`, which it rewrites.

### Step 6: user-run GitHub setup (not run by the implementer)

Do these after the setup PR is pushed and `process` has run once:

1. **Secret:** run `claude setup-token` locally (Pro/Max), then `gh secret set CLAUDE_CODE_OAUTH_TOKEN`.
2. **Repo setting:** `gh repo edit --enable-squash-merge --enable-merge-commit=false --enable-rebase-merge=false --delete-branch-on-merge`.
3. **Ruleset:** `gh api -X POST repos/journeybeforedestination/automate/rulesets --input ruleset.json`, using this body, which is the shape verified against the GitHub REST docs:
   ```json
   {"name":"main","target":"branch","enforcement":"active",
    "conditions":{"ref_name":{"include":["~DEFAULT_BRANCH"],"exclude":[]}},
    "rules":[
     {"type":"pull_request","parameters":{"allowed_merge_methods":["squash"],"dismiss_stale_reviews_on_push":false,"require_code_owner_review":false,"require_last_push_approval":false,"required_approving_review_count":0,"required_review_thread_resolution":false}},
     {"type":"required_status_checks","parameters":{"strict_required_status_checks_policy":true,"required_status_checks":[{"context":"process"}]}},
     {"type":"non_fast_forward"},{"type":"deletion"}]}
   ```
   Merge the setup PR **before** creating the ruleset, or merge it under the ruleset once `process` is green. Either works.

## Proof

These steps are for the user to run. Nothing below is done by the implementer unless asked.

1. **Locally, before pushing:** `scripts/check-work-items` passes with no `intent/` folder. Then make a scratch `intent/0001-x/intent.md` copied from the template and check it passes. Delete a heading and check it fails with a message naming the file and the heading.
2. **Locally, the diff check:** `git commit` a scratch change, then run `scripts/check-pr "spec: 0001-x"` against `HEAD^1`/`HEAD`. It must fail because there is no intent on the base.
3. **On GitHub:** the setup PR shows `process` green and a Claude review comment (once the secret is set).
4. **End-to-end test of the process:** run `/intent` for `0001-walking-skeleton` and open its PR. Then retitle the PR to `spec: 0001-walking-skeleton` and check that `process` re-runs and fails. That proves the `edited` trigger works.

## Traps

- **The required check name must match the job id exactly.** If the ruleset requires `process` but the job is named anything else, every PR waits forever on "Expected — Waiting for status to be reported", and nothing says why. Rename both or neither.
- **Never add `paths:` filters to `process.yml`.** A required check that doesn't run on a PR never reports, and that PR can't merge. This is the same silent wait as above.
- **The `edited` trigger.** Without `types: [..., edited]`, retitling a PR (for example to fix `spec:` → `build:`) doesn't re-run `process`, so the check result belongs to the old title.
- **The PR title is untrusted input.** Interpolating `${{ github.event.pull_request.title }}` directly into `run:` allows shell injection on a public repo. Pass it through `env:` and quote `"$TITLE"`.
- **Check the merge commit, not the head.** `actions/checkout` on `pull_request` checks out the PR merge commit, so `HEAD^1` is the base, and `check-work-items` sees main's folders plus the PR's. Duplicate-number detection depends on that. **Verify** in the first run's logs that `git log -1 --format=%P` shows two parents. With `fetch-depth: 1`, `HEAD^1` doesn't exist and the script fails with a confusing git error, so it must be `fetch-depth: 2`.
- **"Require branches to be up to date" (`strict_required_status_checks_policy: true`) is deliberate.** Two intent PRs can each pick `0005` and both pass on their own. Strict mode forces the second one to rebase onto main and re-run, and at that point the duplicate is caught.
- **Fork PRs get no secrets.** On this public repo a fork PR would run `claude-review` with an empty token and fail confusingly. The `if:` on the head repo skips it instead. `claude-review` must never be a required check.
- **When 0001 adds build/test jobs, add them to the ruleset's `required_status_checks`.** Otherwise "done = tests passing" is not enforced, and PRs merge with red tests. 0001's spec must include this.
- **Don't name the build skill `/plan`.** Claude Code has a built-in plan-mode command by that name. **Verify step:** after Step 1, type `/` in Claude Code and confirm `/intent`, `/spec` and `/build` appear and don't shadow built-ins.
- **Squash merging replaces the PR's commits with one commit.** "plan.md is the first commit" only holds inside the PR. On main, the plan and the code arrive in one squash commit titled `build: NNNN-slug`. That's intended, and it makes `git log --oneline` read as a stage history.

## Considered and rejected

- **One PR per work item:** nothing would be accepted before code exists, which removes the playbook's gates.
- **Four PRs (plan.md as its own PR):** the plan is already approved interactively in plan mode, so a fourth gate adds wait time for a solo developer without a second reviewer.
- **A `status:` field in intent.md:** duplicated state that drifts. The files on main are already the status.
- **Date-prefixed or slug-only folder IDs:** the user preferred short, sortable, speakable numbers. Collisions are handled by strict mode plus the duplicate check.
- **Scaffolding the .NET/React skeleton in this setup:** the skeleton would skip the process it's meant to prove. It is 0001 instead.
- **Process docs with no CI:** the README would describe enforcement that doesn't exist.
- **An `@claude` responder that pushes fixes:** it gives an agent write access in CI. The user chose read-only review.
- **Skills only print the push and PR commands:** I first planned this, and it also left out the `git push`. The user picked asking before pushing instead, which avoids manual steps at every stage without any unconfirmed GitHub writes.
- **Skills push and open the PR with no question:** this would rely only on Claude Code's permission prompt, which disappears once those commands are allow-listed.
- **Installing the Claude GitHub App:** its token has Contents: write. `GITHUB_TOKEN` scoped by job permissions is enough for comments (pending the verify step in Step 4).
- **Required approvals / CODEOWNERS:** this is a solo developer, and GitHub forbids self-approval, so it would block every merge. Revisit when a second person joins.
- **A bug fast lane (one PR):** a second process to document. Bugs use the full path with a short spec.
- **bats or another test framework for the scripts:** a new dependency for three small scripts. Proof steps 1–2 and the 0001 run cover them. Revisit if the scripts grow.
- **A separate spec-review agent:** the playbook notes the spec gate is weak. `REVIEW.md`'s spec checklist covers it without adding another moving part.
- **My own wrong turn:** I first sketched the third skill as `/plan`, matching the artifact name. It collides with Claude Code's built-in plan-mode command, so it's `/build`, which is also more accurate because it covers plan + code.

## Accepted with known risk

- **0 required approvals.** The `process` check plus the user's own judgement is the whole gate. *Revisit when* a second contributor joins: add CODEOWNERS (`intent/**` → product owner, everything else → tech lead) and set the approval count to 1.
- **`claude-code-action@v1` is a moving tag on a public repo.** *Revisit when* the repo holds anything sensitive: pin to a commit SHA.

## Environment constraints (not visible in the code)

- **Remote:** `git@github.com:journeybeforedestination/automate.git`, **public**, default branch `main`. The repo currently allows both squash and merge commits; Step 6 changes that.
- **Local toolchain (via mise):** dotnet 10.0.400, node v24.21.0, gh 2.102.0 (authenticated as `journeybeforedestination`), Claude Code 2.1.293.
- **The user's global CLAUDE.md** forbids GitHub writes without asking: no push, no PR, no secret, no ruleset by the implementer.
- **`CLAUDE_CODE_OAUTH_TOKEN` requires a Pro/Max plan** (`claude setup-token`). If the user prefers API billing, use `ANTHROPIC_API_KEY` with the `anthropic_api_key` input instead.

## Out of scope, and where it went

- **The application skeleton:** becomes `intent/0001-walking-skeleton/`, the first `/intent` run after setup. It's named in the README's "next" section.
- **Deploy, Maintain, evals (`evals/`, `agent-evals.yml`), hooks, managed settings:** listed in the README's "not covered yet" section. Evals and hooks have nothing to act on until code and a test command exist.

## Reading that was partial

- **The playbook:** read once through a summarising fetch, not line by line. The quoted plan.md headings and the "Things Claude gets wrong" rule came through that summary. Re-read the source if the wording matters.
- **claude-code-action:** read `action.yml` inputs, `examples/pr-review-comprehensive.yml`, and the auth and fork sections of `docs/setup.md` and `docs/security.md`. I did not read how the action's default allowed tools interact with `--allowedTools` (for example, whether Read and Grep stay available). Check the first review's log to see whether it read the spec and plan files.
