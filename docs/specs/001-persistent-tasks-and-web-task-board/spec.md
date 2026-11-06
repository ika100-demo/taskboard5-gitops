---
spec_id: 001-persistent-tasks-and-web-task-board
title: Persistent tasks and web task board
status: approved
priority: P1
shape: gitops-app
---

# 001 — Persistent tasks and web task board

## Problem

The taskboard5 Task API can create and list tasks, but it holds them in memory: every restart or redeploy empties the list, and the API must run as exactly one instance. The taskboard5 website shows only a starter page, so nobody can see or add tasks without calling the API by hand. Tasks need to survive restarts and people need a page where they can read the board and add to it; only then is the system usable as a task board.

## Stories

As a person tracking work, I want the tasks I have recorded to still be there after the system restarts or is redeployed, so that I can rely on the board.
**Repos:** taskboard5-api, taskboard5-gitops

As a person tracking work, I want to open the task board in my browser and see all tasks with their due dates, so that I know what is recorded.
**Repos:** taskboard5-web, taskboard5-api, taskboard5-gitops

As a person tracking work, I want to add a task with a title and an optional due date from the board, so that I do not need to call the API by hand.
**Repos:** taskboard5-web, taskboard5-api, taskboard5-gitops

## Acceptance criteria

- **AC-001.1** Given tasks "A", "B" and "C" were created in that order, when the Task API is restarted or redeployed, then `GET /tasks` returns `200` with the same three tasks, each with the same `id`, `title` and `due_date` as returned on creation, in creation order (oldest first).
- **AC-001.2** Given the task storage is unreachable, when a caller sends `POST /tasks` or `GET /tasks`, then the response is `503` with a JSON error body, no task is added, and the Task API's readiness check reports not ready until storage is reachable again.
- **AC-001.3** Given tasks exist, when a person opens the web app's root URL, then the task board lists every task in creation order (oldest first), each showing its title exactly as entered and its due date as `YYYY-MM-DD`, or "No due date" when it has none.
- **AC-001.4** Given no task exists, when a person opens the web app's root URL, then the board shows the text "No tasks yet" and the form for adding a task.
- **AC-001.5** Given the board is open, when a person enters the title "Buy milk", optionally picks a due date, and submits the form, then the new task appears at the end of the board without a manual page reload, the form is cleared, and the task is returned by `GET /tasks` and is still on the board after a page reload.
- **AC-001.6** Given the board is open, when a person submits the form with an empty or whitespace-only title or a title longer than 200 characters, then the form shows "Title is required" or "Title must be at most 200 characters" next to the title, keeps the entered values, and no task is created.
- **AC-001.7** Given the Task API is unreachable or answers with an error, when a person opens the board, then the page still loads and shows "Tasks could not be loaded. Please try again later." instead of the list.
- **AC-001.8** Given the Task API is unreachable or answers with an unexpected error, when a person submits a valid task, then the form shows "The task could not be saved. Please try again.", keeps the entered values, and the board does not show the task.
- **AC-001.9** Given the Task API runs as 2 instances in dev, when a task is created through one instance, then `GET /tasks` answered by the other instance lists it, with the same `id`, `title` and `due_date`.
- **AC-001.10** Given a person opens the board and adds a task, when the browser's network requests are inspected, then none of them goes to the Task API: the browser only talks to the web app, which calls the Task API from its server.

## Non-goals

- Migrating existing in-memory tasks: none survive a restart, so there is nothing to carry over.
- Editing, completing, deleting or reordering tasks, in the API or on the board.
- Filtering, sorting options, search or pagination of the board.
- Sign-in, users or per-user boards; anyone who can reach the web app can see and add tasks.
- Live updates of tasks added by other people while the board is open (a page reload shows them).
- Running in staging or prod as part of this spec: it is built for dev, and the normal promotion takes it further.
- Backups of the task database.
- Any change to the API's request and response shapes, validation rules or status codes from taskboard5-api spec 001, other than the storage behaviour in AC-001.1 and AC-001.2.

## Open questions

- ~~Should the one-replica pin on the Task API be lifted?~~ Answered: yes, 2 replicas in dev, see AC-001.9.
- ~~Should the web app call the Task API from its server or from the browser?~~ Answered: from its server, see AC-001.10.
- ~~Confirm: title and due date per task, oldest first, "No due date" when none?~~ Answered: yes, see AC-001.3.
- ~~Confirm the user-facing wording?~~ Answered: confirmed exactly, see AC-001.4, AC-001.6, AC-001.7, AC-001.8.
- ~~Confirm: the form checks the title with the API's rules and the due date comes from a date picker?~~ Answered: yes, see AC-001.6.
- ~~Confirm: storage unreachable gives `503` and not ready?~~ Answered: yes, see AC-001.2.
- ~~Confirm: no migration of existing tasks?~~ Answered: yes, see Non-goals.
- ~~In which environments should this run?~~ Answered: dev only for now, then the normal promotion, see Non-goals.
- ~~Should tasks in the database be backed up?~~ Answered: not in this spec, see Non-goals.

## Changelog

- 2026-11-06 created
- 2026-11-06 amended: open questions answered; added AC-001.9 (2 Task API instances in dev share the tasks) and AC-001.10 (the web app calls the Task API from its server); backups and staging/prod rollout recorded under Non-goals.
- 2026-11-06 approved
