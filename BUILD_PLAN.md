# BUILD PLAN

## Current Phase

_(No active phase — Phase 1A, 1B, and 1C all shipped. Awaiting next decision.)_

The eligible candidates are listed under **Future Phases** below. Promote one to Current Phase when starting work.

## Objectives

_(Set when a phase is active.)_

## Decisions Already Made (carried forward)

These remain canonical regardless of which phase comes next:

- **`documents.project_id` is the primary project FK.** Never altered by attach/unlink — those operate on `document_project_links` only.
- **`documents.task_id` is the primary task FK.** Never altered by attach/unlink — those operate on `document_task_links` only.
- **Idempotent attach** (both project and task variants): returns 204 whether or not the link existed. Includes the case where the document is already primary-linked (no row inserted in association table).
- **Unlink rule**: 400 if attempting to unlink the document's primary owner. Already-unlinked is a no-op.
- **Owner-scoped everything**: every project/task/document query filters by `owner_id == current_user.id` and `deleted_at IS NULL`. 404 on miss to avoid leaking existence.
- **Soft-delete filter** on every read.
- **Frontend api.ts is the canonical layer for new document/task calls.** New methods route through `documentsApi`/`projectsApi`/`tasksApi`; raw `fetch` is acceptable only for the upload presign flow.
- **Service-layer tenancy** — no DB-level RLS; missing `owner_id` filter = data leak.
- **`AttachDocumentsModal` is reused across project and task contexts** via a discriminated `mode: 'project' | 'task'` prop. `AttachToTaskModal` is the parallel inverse picker (used from the documents-library page).

## Constraints

- **Auto-deploy on master** — every push goes to production. No staging.
- **Single EC2** — every deploy reboots the frontend. ~13 G free post-resize.
- **CLAUDE.md rules** — read system docs first; update STATE_OF_PLAY in same commit.
- **No new dependencies** unless strictly necessary.
- **Auto-snapshot in CI** — `sitrep.yml` regenerates `TECH_SNAPSHOT.md` on every push. Don't hand-edit that file.
- **Migration drift recurrence**: every autogenerate surfaces the same six pre-existing index/constraint deltas (`uq_admin_users_email`, `ix_admin_users_email`, `ix_ai_usage_owner_week`, `ix_documents_status_created`, `ix_outputs_owner_created`, `ix_tasks_owner_created`). Trim manually each time, or fix the underlying drift (declare on models or remove from prod).

## Next Steps (Ordered)

_(Set when a phase is active.)_

## Future Phases (eligible)

- **Phase 1D** — Search/filter inside the picker modals (both project and task pickers). Becomes useful when users accumulate many documents or tasks. Could also fix the documents-library `excludeTaskIds=[]` UX gap by tracking per-doc linked tasks.
- **Phase 2 — Cross-context document features.** Document tagging, doc-level chat with Ava, cross-project/task search, folder hierarchy. Bigger scope; needs sub-decomposition.
- **Operational track** — durable extraction queue (Celery or pg-based), staging environment, Stripe webhook body retention, RLS pilot on a small table, declare the 6 prod indexes on models (currently surface as autogenerate drift), consolidate the two parallel link mechanisms (drop `documents.task_id` / `documents.project_id` FKs in favour of association tables).
- **Tech debt** — `completed_at` on Task; standardise datetime TZ; resolve documents↔tasks circular FK; deletion of `/dashboard/chat` dead route file.

## Recently Completed Phases

- **Phase 1A** (`e40c8b8`) — Backend foundation for project linking: `document_project_links`, `Document.linked_projects` relationship, `GET /api/v1/projects/{project_id}/documents` union endpoint, project page data-source swap.
- **Phase 1B** (`beba5a0`) — Picker UI for project linking: `AttachDocumentsModal`, "Add existing document" button on project page, per-row Unlink button, `POST`/`DELETE` endpoints for project attach/unlink.
- **Phase 1C** (this commit) — Multi-task document linking + allocation UI: `document_task_links` association table, `Document.linked_tasks` relationship, `attach_document_to_task` / `unlink_document_from_task` service functions, `POST`/`DELETE` endpoints at `/api/v1/tasks/{task_id}/documents/{document_id}`. Existing `GET /api/v1/documents/tasks/{task_id}/documents` refactored to return the union via `list_documents_for_task_union` with task-ownership 404 verification. `AttachDocumentsModal` extended with discriminated `mode` prop (project + task contexts share one component). New `AttachToTaskModal` component. Task view (`TaskDocumentUpload`) gains "+ Add existing" button + per-row Unlink. Documents-library page rows gain "Link to task" button.
