# 0001-walking-skeleton: Intent

## Problem

The repository has a delivery process but nothing for it to deliver. There is no
application code, no build, and no automated test, so CI can only check process
artifacts. Every future work item would have to settle the stack, the local setup and the
test harness before it could show its own behaviour. Nothing proves that the frontend,
backend and database can run together, either on a developer's machine or in CI.

The people affected are the developers (and Claude) building work items, and the reviewer
merging them. Today a reviewer has no green or red signal that a change still works end to
end.

## Outcome

- Opening the app shows a single page that says **status OK**.
- "OK" is shown only when the UI has reached the backend and the backend has reached
  PostgreSQL. If any of the three is down, the page doesn't say OK.
- A Playwright test checks that page against the stack started by Docker Compose.
- That test runs in CI on every pull request and is a **required** check on `main`, so a
  red run blocks the merge.
- A developer can run the same stack and the same Playwright test locally with documented
  commands, and gets the same result CI does. `CLAUDE.md` lists those commands.
- Nothing else: no features, no auth, no domain data.

## Users and systems affected

- Developers and Claude, who build every later work item on this skeleton.
- The reviewer and product owner, who rely on the new required check when merging.
- New: a React frontend, one .NET backend service, and a PostgreSQL database.
- GitHub Actions: a new e2e workflow next to `process`.
- `main` branch protection: the new job is added to `required_status_checks`.
- `CLAUDE.md` and `README.md`: build and test commands, and the architecture section.

## Constraints

- Frontend: React with TanStack, as the README says.
- Backend: exactly one .NET service.
- Database: PostgreSQL.
- End-to-end tests: Playwright, run against the stack started by Docker Compose, both in
  CI and locally.
- CI runs on GitHub Actions. The new job's name must match the name added to the
  ruleset's required checks exactly, or every PR waits forever for it.
- The new check, like `process`, must report on every PR (no path filter), because it's
  required.
- Use the current LTS release of each tool that has one (.NET, Node). For tools without an
  LTS line, use the latest stable release.
- Keep it as small as possible. The skeleton proves the wiring and nothing more.

## Open questions

- Which TanStack pieces (Router, Query, Start)?
- How does the backend check PostgreSQL? A plain connection query, or a migration tool
  set up now so later work items have one?
- Does the frontend get served by its own container, or by the backend?
- How long can the e2e job take in CI before it's a problem? Is image or dependency
  caching needed now?
- Do unit tests for the backend and frontend belong in this item, or only the Playwright
  test?
