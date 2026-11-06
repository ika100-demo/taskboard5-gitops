---
plan_id: 001-persistent-tasks-and-web-task-board
spec: 001-persistent-tasks-and-web-task-board
feature: Tasks stored in Postgres so they survive restarts and two instances, and a web task board that lists and adds tasks
gitops_app: ika100-demo/taskboard5-gitops
status: draft
repos:
  - id: taskboard5-gitops
    shape: gitops-app
    summary: Postgres for the Task API, two API replicas in dev, and the web app's server-side API_URL
    acs: [AC-001.1, AC-001.9, AC-001.10]
    gitops:
      - {resource: postgres, type: database}
      - {uses: postgres, component: taskboard5-api}
      - {env: {API_URL: 'http://taskboard5-api'}, component: taskboard5-web}
    arguments: |
      Not expressible as a gitops operation: set taskboard5-api `instances.dev.replicas: 2` in applications/taskboard5/system.yaml (replacing the top-level `replicas: 1`)
      and drop the stale "exactly one until storage exists" comment, then `devbox run render`. Apply this step only
      after the taskboard5-api PR has merged and dev runs the Postgres-backed image (two in-memory replicas would
      break AC-001.9). Both components are already exposed; no expose operation is needed.
    depends_on: []
    done: false
  - id: taskboard5-api
    shape: service-python
    summary: Store tasks in Postgres so they survive restarts and are shared by every instance
    acs: [AC-001.1, AC-001.2, AC-001.9]
    criteria:
      - "Given tasks \"A\", \"B\" and \"C\" were created with POST /tasks in that order (one with due_date \"2020-01-01\", one with the title \"  B \"), when the service is restarted against the same database, then GET /tasks returns 200 with exactly those three tasks, each with the id, title and due_date returned on creation, oldest first (system AC-001.1)"
      - "Given the database named by DATABASE_URL has no tasks table yet, when the service starts, then it creates its schema itself and GET /tasks returns 200 with [] (system AC-001.1)"
      - "Given two service instances use the same database, when a task is created with POST /tasks on the first, then GET /tasks on the second lists it with the same id, title and due_date, in creation order (system AC-001.9)"
      - "Given the database is unreachable, when a caller sends POST /tasks with a valid body, then the response is 503 with {\"detail\": \"Task storage unavailable\"} within 5 seconds, and after the database is reachable again GET /tasks does not contain that task (system AC-001.2)"
      - "Given the database is unreachable, when a caller sends GET /tasks, then the response is 503 with {\"detail\": \"Task storage unavailable\"} within 5 seconds (system AC-001.2)"
      - "Given the database is unreachable, when GET /ready is called, then it answers 503 with {\"status\": \"not ready\"} while GET /health still answers 200; once the database is reachable again, GET /ready answers 200 with {\"status\": \"ready\"} without a restart (system AC-001.2)"
    arguments: |
      Withdraw AC-001.7 of this repo's spec 001 (restart empties the list) and its "Persistent storage" and
      "Consistency across several running instances" non-goals; every other criterion of spec 001 (shapes, 201/200/422,
      untrimmed titles, UUID ids) stays as it is and its tests must keep passing against Postgres.
      Connection settings come from DATABASE_URL (also PG* are injected). Creation order must not rely on clock time
      alone across instances: use a database sequence / identity column for ordering.
      Tests need a real Postgres (testcontainers or the devbox service); the unreachable case can stop the container
      or point DATABASE_URL at a closed port.
    depends_on: []
    done: false
  - id: taskboard5-web
    shape: web-nextjs
    summary: Task board page that lists tasks and adds a task through the web server
    acs: [AC-001.3, AC-001.4, AC-001.5, AC-001.6, AC-001.7, AC-001.8, AC-001.10]
    criteria:
      - "Given the Task API's GET /tasks returns tasks, when a person opens /, then the board lists every task in the order the API returned them (oldest first), each title exactly as returned (whitespace kept) and the due date as YYYY-MM-DD, or \"No due date\" when due_date is null (system AC-001.3)"
      - "Given the Task API's GET /tasks returns [], when a person opens /, then the page shows \"No tasks yet\" and the add-task form with a title field, a date picker for the due date and a submit button (system AC-001.4)"
      - "Given the board is open, when a person enters \"Buy milk\", optionally picks a due date and submits, then the web server sends POST /tasks with {\"title\": \"Buy milk\", \"due_date\": <date or null>}, the task appears at the end of the board without a manual reload, the form is cleared, and the task is still on the board after a reload (system AC-001.5)"
      - "Given the board is open, when a person submits an empty or whitespace-only title, then \"Title is required\" is shown next to the title; with a title longer than 200 characters \"Title must be at most 200 characters\" is shown; in both cases the entered values stay and no POST /tasks is sent (system AC-001.6)"
      - "Given the Task API is unreachable, does not answer within 5 seconds, or answers GET /tasks with anything but 200 and a JSON array, when a person opens /, then the page loads with status 200 and shows \"Tasks could not be loaded. Please try again later.\" instead of the list (system AC-001.7)"
      - "Given the Task API is unreachable, does not answer within 5 seconds, or answers POST /tasks with anything but 201 (including 503 and a 422 the form did not predict), when a person submits a valid task, then the form shows \"The task could not be saved. Please try again.\", keeps the entered values, and the board does not show the task (system AC-001.8)"
      - "Given a person opens the board and adds a task, when the browser's network requests are inspected, then every request goes to the web app's own origin and none to the Task API; API_URL is read only on the server and never sent to the browser (system AC-001.10)"
    arguments: |
      Replace the template starter page at app/page.tsx with the board. Read the Task API base URL from the
      server-only env var API_URL (in dev: http://taskboard5-api); for local runs and tests, stub the API
      (e.g. a fake HTTP server) rather than calling a real one. /api/ready must not depend on the Task API.
    depends_on: []
    done: false
gitops_pin: []
---

## taskboard5-gitops — Postgres for the Task API, two API replicas in dev, and the web app's server-side API_URL

**Shape:** `gitops-app` · **Criteria:** AC-001.1, AC-001.9, AC-001.10 · **Depends on:** []

The system declares no resource yet. This slice adds a `postgres` database resource, wires it into taskboard5-api
(`uses: postgres` injects `DATABASE_URL` and `PG*`), and gives taskboard5-web the server-side variable
`API_URL=http://taskboard5-api`. The rendered Service of taskboard5-api listens on port 80 (target port `http`, i.e.
the container's 8080), so the URL carries no port. The web app reads it on its server only, which is how AC-001.10
holds in the cluster.

AC-001.9 also needs `replicas: 2` for taskboard5-api in dev. That is not one of the four gitops operations, so it is a
manual step in this repo: edit `applications/taskboard5/system.yaml` (replicas 2, remove the stale one-replica
comment) and `devbox run render`. It must land only **after the taskboard5-api PR has merged** and dev runs the
Postgres-backed image; two in-memory replicas would show different lists. The resource, `uses` and `env` operations
can land first: the current API ignores `DATABASE_URL`, the current web page ignores `API_URL`.

## taskboard5-api — Store tasks in Postgres so they survive restarts and are shared by every instance

**Shape:** `service-python` · **Criteria:** AC-001.1, AC-001.2, AC-001.9 · **Depends on:** []

The in-memory `TaskStore` becomes a Postgres-backed store behind the same `POST /tasks` / `GET /tasks` contract; the
request and response shapes, validation and status codes of this repo's spec 001 do not change (system non-goal).
The service creates its own schema at startup (idempotent). Ordering uses a database-generated sequence so creation
order holds across instances. Storage failures map to `503` with a fixed JSON body; `/ready` checks the database
on every call so Kubernetes takes an instance out of the Service while storage is down, while `/health` stays
independent of storage so pods are not restarted for a database outage. This slice withdraws AC-001.7 and the
storage / multi-instance non-goals of spec 001. Test concerns: real Postgres in tests, a restart test that builds a
fresh app against the same database, a two-app test against one database, and an unreachable-database test.

## taskboard5-web — Task board page that lists tasks and adds a task through the web server

**Shape:** `web-nextjs` · **Criteria:** AC-001.3, AC-001.4, AC-001.5, AC-001.6, AC-001.7, AC-001.8, AC-001.10 · **Depends on:** []

The root page becomes the task board: a server-rendered list from `GET /tasks` and an add-task form whose submit
goes to the web app's own server (a Server Action or a route handler under `/api/`), which calls `POST /tasks`. The
browser never sees the API's address (AC-001.10). The form validates the title with the API's rules before sending
(1 to 200 characters, not only whitespace, not trimmed) and shows the confirmed wording; the due date comes from a
native date picker (`YYYY-MM-DD`). After a successful create the new task is appended to the list and the form is
reset without a full page reload. Load and save failures show the confirmed messages; the add form stays visible
when loading failed. It builds in parallel with the API against the contract below; tests stub the API.

## Contract

The only cross-repo calls are from the taskboard5-web **server** to the Task API. The browser talks only to
taskboard5-web. Base URL: env var `API_URL` on the web server (dev: `http://taskboard5-api`, no trailing slash).
All bodies are JSON (`Content-Type: application/json`), UTF-8.

**Task object**

| field | type | notes |
|---|---|---|
| `id` | string | server-assigned UUID |
| `title` | string | 1 to 200 characters, not only whitespace, stored and returned exactly as sent (no trimming) |
| `due_date` | string or null | calendar date `YYYY-MM-DD`; past dates allowed; `null` when none |

**`GET /tasks`**: no parameters.
- `200` — JSON array of Task objects, every task, oldest first (creation order). `[]` when there is none.
- `503` — `{"detail": "Task storage unavailable"}` when the database cannot be reached.

**`POST /tasks`**: body `{"title": "<string>", "due_date": "<YYYY-MM-DD>" | null}`; `due_date` may be omitted.
The web app sends `due_date: null` when no date is picked.
- `201` — the created Task object. It is durable and visible to `GET /tasks` on every instance once returned.
- `422` — FastAPI's standard validation body `{"detail": [{"loc": ["body", "title"|"due_date"], "msg": ..., "type": ...}, ...]}`; nothing stored.
- `503` — `{"detail": "Task storage unavailable"}`; nothing stored.

**`GET /ready`** (Kubernetes probe, not called by the web app): `200 {"status": "ready"}` when the database answers,
`503 {"status": "not ready"}` otherwise. **`GET /health`**: `200` regardless of the database.

**Web app (browser-facing, same origin):** `GET /` renders the board; the add-task submit goes to a same-origin
endpoint of the web app (Server Action or `/api/...` route; the web repo picks). The board's texts are exactly:
"No tasks yet", "No due date", "Title is required", "Title must be at most 200 characters",
"Tasks could not be loaded. Please try again later.", "The task could not be saved. Please try again."

### Errors

- Task API errors are JSON with a `detail` field: a string for `503`, FastAPI's validation list for `422`.
  Unhandled server errors are `500` with a JSON body; callers must not rely on its content.
- The web server treats `GET /tasks` as successful only on `200` with a JSON array; anything else (`503`, `500`,
  any other status, a non-JSON or malformed body, a connection error, a timeout) shows
  "Tasks could not be loaded. Please try again later." and the page itself still answers `200`.
- The web server treats `POST /tasks` as successful only on `201` with a Task object. A `422` whose `loc` names
  `title` shows the matching title message next to the field (normally prevented by the form's own check);
  every other outcome (`422` on `due_date`, `503`, `500`, any other status, a malformed body, a connection error,
  a timeout) shows "The task could not be saved. Please try again." In every failure case the entered values stay
  and the task is not added to the board. The web app does not retry automatically.
- The web app never forwards Task API error bodies or its URL to the browser.

### Timeouts

- Web server -> `GET /tasks`: 5 seconds total (connect + response). On timeout: the load-error message.
- Web server -> `POST /tasks`: 5 seconds total. On timeout: the save-error message. A create that the API completed
  after the timeout may show up on the next page reload; that is accepted (no idempotency key in this spec).
- Task API -> Postgres: connect timeout 2 seconds, statement timeout 3 seconds, so a request with storage down
  answers `503` within 5 seconds. `/ready` uses the same 2-second connect timeout.
- Task API down or not ready: Kubernetes removes not-ready pods from the Service; when no pod is ready, the web
  server gets a connection error and shows the load or save message. The web app's own `/api/ready` does not
  depend on the Task API, so the board stays up while the API is down.

## gitops-app PR

Dev only (system non-goal: staging and prod follow by the normal promotion), so no pins: dev tracks `main` of both
component repos. Order:

1. First gitops PR (any time): `resource: postgres` (database), `uses: postgres` on taskboard5-api, `env API_URL` on
   taskboard5-web, `devbox run render`. After Argo reconciles: the Postgres instance is ready, the taskboard5-api pod
   has `DATABASE_URL` / `PG*`, the taskboard5-web pod has `API_URL=http://taskboard5-api`.
2. After the taskboard5-api PR merges and dev runs its image: `replicas: 2` for taskboard5-api in `system.yaml`
   (remove the one-replica comment), render. After Argo reconciles: two ready taskboard5-api pods; a task created
   through `http://taskboard5-api.taskboard5-dev.localhost/tasks` is listed on repeated `GET /tasks` calls
   (served by both pods) with the same id, title and due_date; deleting one pod keeps the list.
3. After the taskboard5-web PR merges: open `http://taskboard5-web.taskboard5-dev.localhost/`, add a task, confirm
   it is listed after a reload and that the browser's network tab shows only requests to the web host.
