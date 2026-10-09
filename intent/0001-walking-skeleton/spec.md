# 0001-walking-skeleton: Spec

## Requirements

- **R1** Opening the app at `/` shows a single page with the text **status OK** when the
  frontend, backend and PostgreSQL are all up.
- **R2** The backend exposes `GET /api/status`. It returns `200` when a `SELECT 1` against
  PostgreSQL succeeds, and `503` when it fails. It does not crash or hang when the
  database is unreachable.
- **R3** The page shows **status OK** only after it receives a `200` from
  `/api/status`. Any other response or a network error shows **status unavailable**.
- **R4** One Docker Compose file defines the stack: `db` (PostgreSQL), `app` (the .NET
  service, which also serves the built frontend) and `e2e` (the Playwright tests).
- **R5** A developer runs the stack and the Playwright tests locally with the same
  commands CI uses, and needs only Docker installed. `CLAUDE.md` lists those commands.
- **R6** A GitHub Actions job with id `e2e` runs the Playwright tests on every pull request,
  with no path filter. It fails when any test fails.
- **R7** `e2e` becomes a required status check on `main` next to `process`, and the
  README's ruleset instructions include it.
- **R8** `CLAUDE.md` and `README.md` describe the new architecture, the build and test
  commands, and the new CI check.
- **R9** Each tool uses its current LTS line (.NET 10, Node 24). Tools with no LTS line
  (PostgreSQL, Playwright, React, TanStack, Vite) use the latest stable release when the
  build starts, pinned to that exact version.

## Acceptance criteria

All criteria except AC6 and AC7 are Playwright tests run by `docker compose run --build --rm e2e`.

- **AC1 (R1, R4):** Given the Compose stack is up, when the test opens `/`, then the
  page shows "status OK".
- **AC2 (R2):** Given the stack is up, when the test sends `GET /api/status` to `app`,
  then the response is `200`.
- **AC3 (R2):** Given an `app` instance whose connection string points at a database host
  that doesn't exist, when the test sends `GET /api/status` to it, then the response is
  `503` within 10 seconds.
- **AC4 (R3):** Given `/api/status` is intercepted to return `503`, when the page loads,
  then it shows "status unavailable" and never "status OK".
- **AC5 (R3):** Given `/api/status` is intercepted to fail at the network level, when the
  page loads, then it shows "status unavailable" and never "status OK".
- **AC6 (R5, R6):** On the build PR, the `e2e` job is green. The same command run
  locally on a clean clone passes. The build PR body shows the local output.
- **AC7 (R6, R7):** With `e2e` added to the ruleset, a PR whose test fails can't be
  merged. To check, temporarily change the expected text in AC1's test on a throwaway PR.
  The user runs this check (see Concerns).

## Design

**Layout.** There are four new top-level entries: `backend/`, `frontend/`, `e2e/` and
`compose.yaml`. Nothing existing moves.

**Backend.** `backend/` holds one ASP.NET Core minimal-API project on .NET 10. It has one
endpoint, `GET /api/status`, which opens an Npgsql connection and runs `SELECT 1` with a
short timeout. It returns `200 {"status":"ok"}` on success and `503
{"status":"unavailable"}` on any failure. The connection string comes from configuration,
so Compose sets it through an environment variable. The backend also serves the
frontend's built files as static files, with a fallback to `index.html`. There is no ORM
and no migration tool. The first work item that adds a table chooses one, and can judge
it against that table.

**Frontend.** `frontend/` holds a Vite + React + TypeScript SPA. TanStack Router defines
the one route, `/`. TanStack Query fetches `/api/status` and turns the result into the
page text. Only a resolved `200` shows "status OK". Loading, error and non-200 responses
show "status unavailable". Retries are turned off, so a failure shows immediately rather
than after Query's default backoff. In local development, the Vite dev server proxies
`/api` to the backend. TanStack Start isn't used, so no Node server runs in production.

**Image.** One multi-stage Dockerfile builds the frontend in a Node 24 stage and the
backend in a .NET SDK stage. Its final stage is the ASP.NET runtime image with the
frontend files copied into the web root. Because the frontend and API share one origin,
there is no CORS and no proxy in the deployed stack.

**Compose.** `compose.yaml` defines these services:
- `db`: PostgreSQL pinned to an exact version, with a `pg_isready` healthcheck.
- `app`: the image above. It depends on `db` being healthy and publishes a port, so
  developers can open the page in a browser.
- `app-nodb`: the same image with its connection string pointed at a host that doesn't
  exist. AC3 uses it. It's in the `e2e` profile, so `docker compose up` doesn't start it.
- `e2e`: the official Playwright image, whose tag matches the `@playwright/test` version
  in `e2e/package.json`. It mounts `e2e/` and runs `npx playwright test`. It depends on
  `app` and `app-nodb`, and waits until `app` answers before the tests start. It's in the
  `e2e` profile.

The Playwright tests reach the app by its Compose service name, so no host ports are
involved in CI. Browsers come with the image and never install on the host.

**Commands** (go into `CLAUDE.md`):
- `docker compose up --build` runs the stack at `http://localhost:<port>`.
- `docker compose run --build --rm e2e` runs the Playwright tests. Its exit code is the
  test result.
- `docker compose down` stops the stack.

**CI.** `.github/workflows/e2e.yml` triggers on `pull_request` with the same types as
`process.yml` and has no `paths:` filter. It has one job, id `e2e`, with `contents: read`
permissions. The job checks out the code, runs `docker compose run --build --rm e2e`, and
always runs `docker compose down` afterwards. It has no image or dependency caching. The
budget is 10 minutes. If the build PR's run takes longer, caching becomes a `chore:` PR
sized against the measured time.

**Required check.** The ruleset is a GitHub setting, not a file, so the build PR can't
change it. The build PR updates the README's ruleset JSON and one-time setup steps to list
`{"context":"e2e"}`. The user adds the check to the live ruleset after `e2e` has reported
once on the build PR.

**Docs.** `CLAUDE.md` gains the commands above and replaces "There is no build or test
command yet" and the Architecture placeholder with a short description of the stack. The
README's "What CI checks" table, repository map, one-time setup and "Not covered yet"
section are updated so they describe only what exists.

**New third-party dependencies** (the build PR lists them again with exact versions):
- Backend: Npgsql.
- Frontend: React, React DOM, TanStack Router, TanStack Query, Vite and its React
  plugin, TypeScript.
- Tests: `@playwright/test`.
- Images: `postgres`, `mcr.microsoft.com/dotnet/sdk` and `aspnet`, `node`,
  `mcr.microsoft.com/playwright`.

## Out of scope

- Features, auth, domain data, and any table or schema.
- A migration tool or ORM. The first item that adds a table picks it.
- Backend and frontend unit tests and their harnesses. The first item with logic worth
  unit-testing adds them.
- Image or dependency caching in CI, unless the measured run exceeds the 10-minute budget.
- Linting, formatting and type-check jobs.
- Deployment, environments and production configuration.
- Running Playwright on the host, or its UI mode. Developers can still do it, but no
  command for it is documented or supported.
- A separate container for the frontend.

## Concerns

- **The required check is a manual step that can be skipped.** No file in the repo can
  enforce R7. If the user never adds `e2e` to the live ruleset, PRs with red tests can
  merge and the README describes protection that doesn't exist. The order matters too.
  Adding the check before `e2e` has reported once leaves the build PR itself waiting
  forever. The build PR's handoff will remind the user of this step and its order, and
  AC7 checks that it was done.
- **"Frontend down" can't be a separate state.** The backend serves the UI, so if the UI
  is down then the backend is down too, and the page doesn't load at all. The intent's
  rule that the page "doesn't say OK if any of the three is down" still holds. There's
  just no state where the UI is up and the backend isn't, so AC4 and AC5 cover that rule
  by intercepting the request in the browser instead.
- **The 10-minute CI budget is a guess until it's measured.** Building the .NET and Node
  stages and pulling the Playwright image with no cache could come close. It can wait,
  because the build PR's run measures it and a `chore:` PR can add caching if needed.
- **"Latest stable" moves.** R9 pins exact versions when the build starts. Later upgrades
  are `chore:` dependency bumps, and nothing updates these versions automatically.
