# STATE OF PLAY

_Last reviewed: 2026-05-02. Production live on `master`. This commit hardens `deploy.yml` after a silent-deploy-failure incident on `f2956ea`. Two-line change: (a) `set -euo pipefail` at top of the SSH script — prior script had no `set -e`, so a failed `git pull origin master` was logged-but-ignored and the rest of the script restarted the stale code; (b) `npm install --legacy-peer-deps` → `npm ci --legacy-peer-deps` — eliminates the lockfile-drift class entirely (npm install was mutating `package-lock.json` on every deploy, leaving uncommitted changes that blocked the next deploy's git pull). Prod lockfile drift cleaned out-of-band before this commit's push so the auto-deploy lands cleanly. Prior commits: security CVE upgrades (`f2956ea`), anchor-mirror workflow (`0dc655c`), chat-email markdown strip (`dc0d9ac`), chat output quality (`58ed050`), sidebar restore (`28a7d99`), TaskDetailModal migration (`4e1c372`), CompleteWithAvaModal migration (`187f27b`), TrialExpiredModal migration (`bd3fb6d`), widened `ui/Modal` title prop (`0027c64`), deleted dead `CompleteTaskModal` (`5e4048a`), task-level AI context-builder (`b20c3f4`), per-task linked-emails card (`274b812`), project Reactivate button (`f3b7f2e`), housekeeping (`aa224ce`), Modal accessibility (`7f6c9ff`), nginx vhost version control (`be8d08c`), Make me Happy reset modal (`a8b45a9`), linked-emails card on project detail page (`5d581a2`), chat context-builder fix (`eae74f3`), email↔task M:N (`4b89fbc`), project lifecycle UI (`f938d4f`), ops doc update (`caa9794`), six-indexes drift cleanup (`eb2fcb9`), Phase 1C (`89e9803`), Phase 1B (`beba5a0`), Phase 1A (`e40c8b8`), Gold System (`228ad2b`)._

## Current Status

Production is up on `ordoai.co.uk`. **Phase 1C — multi-task document linking — is complete in this commit.** Documents can now be attached to multiple tasks (in addition to the existing multi-project capability from Phase 1B). New `document_task_links` association table; `documents.task_id` (single primary FK) is unchanged by design. Allocation UI works from three entry points: project page (Phase 1B), task view (Phase 1C), and documents library page (Phase 1C). No active feature workstream after this lands.

## What Works

End-to-end:
- Auth — register, login, logout, password reset, email verification, account delete.
- Projects — full CRUD, status filter, project detail page (tasks + Ava context + documents card + **linked-emails card**). **Archive (sets `status='archived'`) and Delete (soft-delete via `deleted_at`) buttons on the project detail page header. List page defaults to active-only with Active/Archived tab toggle. Per-card "Reactivate" button on Archived rows (PATCH `status='active'`).** Linked-emails card lists primary-FK + M:N-via-task emails returned by `GET /api/v1/projects/{id}/emails`; primary-FK rows have an Unlink button (PATCH `project_id=null`); task-linked rows show a "via task: <title>" badge instead.
- Tasks — full CRUD, notes, share-link tokens, RACI assignments, recurrence, dependencies, source-email/source-doc tracing, complete-with-Ava modal. **Documents section now reads union (primary `task_id` ∪ `document_task_links`) and supports attach/unlink existing docs. Linked-emails card in `TaskDetailModal` lists emails attached via `email_task_links`; rows deep-link to `/inbox?id=...` and have per-row Unlink (DELETE on `/api/v1/tasks/{task_id}/emails/{email_id}`).**
- Documents — direct-to-S3 presigned upload, async text extraction, list + delete + download URL, dual-write from email attachments.
  - **Many-to-many links to projects** via `document_project_links` (Phase 1B).
  - **Many-to-many links to tasks** via `document_task_links` (Phase 1C).
  - **Picker modal on project page** lets you attach existing documents to additional projects; per-row Unlink button on linked-but-not-primary docs.
  - **Picker modal on task view** (extended `AttachDocumentsModal` with `mode="task"`) for attaching existing docs to additional tasks; per-row Unlink button on non-primary linked docs.
  - **Documents library page** — each row has a "Link to task" button → opens `AttachToTaskModal` to pick any task to associate.
- Inbox — SendGrid inbound webhook, parsed emails listed in `/inbox`, AI categorisation, suggested actions/projects, convert-to-task, complete-with-Ava, delegate flow. **Emails can be linked to multiple existing tasks via the new "Link to task" picker in EmailDetail; per-row unlink available. M:N via `email_task_links`.**
- Chat with Ava — general + project-scoped, SSE streaming, history persistence, message-level ActionBar. **Project-chat context now includes documents (primary FK + M:N via `document_project_links`), emails (primary FK + emails linked via `email_task_links` to any task in this project), tasks. All capped to 5 docs / 5 emails / 10 tasks per request, ~2000 chars per item.**
- Outputs — drafts, briefings, RNS-style notes, internal comms, presentations (PPTX). v3 voice prompts.
- Reminders — in-app + email, recurrence, daily Ava email, weekly digest.
- Admin panel — login, change password, stats, user mgmt (delete/plan/verify/CSV export), subscriptions, AI usage, audit log, revenue, admin-user CRUD.
- Stripe — checkout, customer portal, webhook idempotency. Pro upgrades flip plan correctly.
- Dashboard — Hero with 2×2 guided action grid (Go to tasks / Open a project / **Make me Happy** reset modal / Upload & summarise document), DecisionRow with 4 live metric cards, Today's Priorities, Today's Wins, Projects (deadline-sorted). **Make me Happy** opens a frontend-only modal with a random reset prompt (16 items across quotes, breathing, physical, mindset).
- Auto-deploy — push to `master` → GitHub Actions → EC2 pull + alembic + build + pm2 restart. Auto-snapshot regeneration via `sitrep.yml`.
- Project documents endpoint — `GET /api/v1/projects/{project_id}/documents` returns deduplicated union (primary + linked).
- Task documents endpoint — `GET /api/v1/documents/tasks/{task_id}/documents` now returns deduplicated union of primary `task_id` + `document_task_links`.
- Document↔project linking endpoints — `POST` (idempotent attach) and `DELETE` (unlink, 400 on primary) at `/api/v1/projects/{project_id}/documents/{document_id}`.
- **Document↔task linking endpoints — `POST` (idempotent attach, 204 if already primary or already linked) and `DELETE` (unlink, 400 on primary) at `/api/v1/tasks/{task_id}/documents/{document_id}`.**

## What Is Broken / Not Working

Nothing currently live-broken. Latent (visible in code, not breaking flows):
- `/dashboard/chat` route file still exists (no longer linked from anywhere).

## In Progress

_(No active feature workstream. Phase 1D and other backlog items eligible — see BUILD_PLAN.md.)_

## Recently Completed

Last 5 (newest first):
1. **(this commit)** — `chore(ops): harden deploy.yml — set -euo pipefail + npm ci`. Direct response to a silent-deploy-failure on `f2956ea` where the auto-deploy reported `success` in GitHub Actions but the box stayed on `0dc655c` with old packages. Two-line change: (a) `set -euo pipefail` at top of the inline SSH script — without it, `git pull origin master` failures (e.g. blocked by uncommitted local changes) were logged but the script kept running, restarting the stale code with no visible failure; (b) `npm install --legacy-peer-deps` → `npm ci --legacy-peer-deps` — `npm install` mutates `package-lock.json` on every run, which then blocks the next deploy's git pull. `npm ci` uses the lockfile verbatim and never mutates it, eliminating the drift class. Prod lockfile drift was cleaned out-of-band before this push (so the very deploy of THIS commit lands cleanly). The `if`/`else` nginx-sync block intentionally tolerates `nginx -t` failures via its own internal logic; `set -e` doesn't change that because the conditionals are exempt and the in-block `sudo cp` / `systemctl reload` should fail loudly anyway. Single commit, 2 files: `.github/workflows/deploy.yml`, `STATE_OF_PLAY.md`. No app code, schema, or migration.
2. **`f2956ea`** — `chore(security): upgrade deps to close 5 known CVEs`. Outcome of a routine cheap-set security check. Backend: `python-jose[cryptography]` 3.3.0 → 3.4.0 (PYSEC-2024-232/233 — JWT cookie auth path), `pdfplumber` 0.11.4 → 0.11.9 + explicit `pdfminer-six==20251230` pin (CVE-2025-64512, CVE-2025-70559 — user-uploaded PDF extraction surface; pdfplumber 0.11.4 hard-pinned the vulnerable pdfminer-six version, requiring pdfplumber to be bumped first), `python-multipart` 0.0.9 → 0.0.26 (CVE-2024-53981, CVE-2026-24486, CVE-2026-40347 — multipart upload DoS). Frontend: `marked` → 18.0.3 (GHSA-6v9c-7cg6-27q7 OOM DoS), `hono` → 4.12.16 (GHSA-458j-xx4x-4375 JSX SSR HTML injection). Skipped: `starlette` (would require fastapi upgrade — separate work; flagged as tech debt), `postcss` (build-time only, would force Next downgrade; flagged as tech debt), `pytest` (dev-only). Initial deploy silently failed (see commit 1 above for root cause + fix); manual recovery completed before this STATE_OF_PLAY entry was written.
3. **`dc0d9ac`** — `fix(email-export): strip markdown before mailto/Gmail/Outlook URL encoding`. Chat-generated emails were arriving in recipient mail clients with raw markdown markers (`**The Business**` literal asterisks instead of bold). `mailto:`/Gmail compose/Outlook web compose URLs all carry plain text only; markdown source ships verbatim. Three-layer fix: (a) backend `CHAT_ADDENDUM` and `outputs/prompts.py` `email_draft` template both now forbid markdown emphasis in email bodies — forward fix; (b) `lib/emailLinks.ts` gains pure `stripMarkdown` helper applied to body AND subject inside all three URL builders — defence-in-depth; (c) two Group B inline-mailto sites refactored to use the helpers — `EmailAvaPanel.tsx` `buildMailto` and `OrdoOnboarding.tsx` Gmail+Outlook buttons. `buildOutlookDesktopUrl` widened to optional `to?: string`. Strip rules deliberately skip `_italic_` (collides with identifiers like `2026_Q1_Plan`). 10/10 transformation smoke tests pass.
4. **`58ed050`** — `fix(chat): three quality fixes for project chat output panel`. (1) Fresh just-generated outputs auto-expand instead of starting collapsed; older history still loads collapsed per design. The `handleSend` flow now adds the newest assistant message id to `expandedMessageIds` after `fetchHistory()` returns the enriched list. (2) Email-shape detection broadened: backend `CHAT_ADDENDUM` now instructs Ava to start email-shaped replies with `Subject:` (forward fix); `detectAvaOutputType` adds a greeting+sign-off fallback so historical emails saved before the addendum still classify correctly. ActionBar's existing `case 'email'` branch renders Outlook/Gmail buttons unchanged. (3) Removed the unconditional nudge that rendered on every output type including `general` where the buttons it promised didn't exist.
5. **`28a7d99`** — `fix(layout): restore Calendar and People links to sidebar nav`. Both were removed in commit `924945f` (2026-04-27 "simplified sidebar" sweep), but on review the removal wasn't intended for either — both routes still existed and were reachable only by URL. Single-file frontend fix to `Sidebar.tsx`: adds `CalendarDays`/`Users` to lucide imports; appends `/people` to `PRIMARY_NAV` (with original `tourId='tour-people'`); inserts `/calendar` to `SECONDARY_NAV` between Inbox and Documents. No backend or routing change.

Out-of-band ops (not commits): EBS resize 8 G → 20 G; full disk-full recovery; nginx `/inbound/*` location block restored on prod (this commit documents it).

## Known Technical Debt

- **Two parallel link mechanisms for document↔task**: legacy `documents.task_id` (single primary FK) and `document_task_links` (M:N additional). Future consolidation may remove the FK column entirely once the M:N path is the canonical one across all callers. Same shape as document↔project (primary `documents.project_id` + `document_project_links`).
- **Documents page "Link to task" picker shows ALL tasks**, including ones already linked to that document. Idempotent backend means re-linking is a no-op, so this is harmless but visually noisy. The page doesn't currently track per-doc linked-task IDs to populate `excludeTaskIds`. Phase 1D could fix by either (a) tracking the linked-task list per doc, or (b) marking already-linked tasks with an inline indicator.
- **Mixed datetime TZ conventions.** `tasks` and `ai_usage` are naive UTC; newer tables are TZ-aware. Both new association tables (`document_project_links`, `document_task_links`) are TZ-aware. Easy to mismatch on new joins.
- **No row-level security.** Service-layer filters only.
- **Soft delete is partial.** Doc/Project/Task only.
- **Document extraction is non-durable.** `asyncio.create_task` — process restart loses in-flight extractions.
- **No background queue.**
- **Single-worker scheduler assumption** baked into reminders, daily/weekly emails.
- **Stripe webhook stores no body** — only `id` + `event_type`. No replay.
- **`AdminAuditLog.admin_id` has no FK constraint** to `admin_users`.
- **Denormalised `source_email_*`** on Document/Task — updates don't propagate.
- **`Email.(user_id, message_id)` unique was removed** to allow re-sends; dedup is application-layer only.
- **Partial SQLAlchemy `relationship()` declarations.** Document has `linked_projects` (Phase 1A) and `linked_tasks` (Phase 1C); Project has `linked_documents` via backref; Task has `linked_documents` via backref. Other models still rely on explicit joins.
- ~~**Six prod indexes/constraints not declared on models**~~ — **Resolved 2026-04-29**: declared via `__table_args__` on AdminUser/AIUsageRecord/Document/GeneratedOutput/Task; `--autogenerate` now produces empty noop. The DB has both `uq_admin_users_email` (UniqueConstraint) AND `ix_admin_users_email` (non-unique Index) on the same column — declared both to match exactly.
- **Circular FK between `documents.task_id` and `tasks.source_doc_id`** triggers SAWarning during alembic operations.
- **`/dashboard/chat`** dead route file not deleted.
- **`UserProfile` has no timestamps**.
- **No `completed_at` on Task.** `updated_at` is the proxy.
- **Documents upload zone still uses raw `fetch`** for the presigned URL + `confirm` calls. The list/attach/unlink path is now typed via api.ts, but the upload presign-PUT-confirm flow is intentionally untouched.
- **Task-level chat surface absent (data layer ready).** `backend/app/tasks/context_builder.py` now exists (`build_task_context`); when/if a task-level chat surface is added, it can pass-through to the same builder. Only the streaming chat surface itself is missing.
- **`tasks/strategy.py` is one-shot at task creation.** It generates `task.ava_strategy` once via `asyncio.create_task` after task create, before any link could exist. Strategy reflects link state at creation time only — subsequent emails/documents linked to a task don't trigger re-generation. Re-firing on link/unlink is bigger work (queue + dedup + cost capping); deferred until usage signal warrants. `tasks/strategy.py` was intentionally NOT migrated to `build_task_context` for the same reason.
- **`TaskDetailModal` is overdue for splitting** — at 1,469 lines (post-migration to `ui/Modal`) it's the largest single component file in the repo. The view-state machine (7 states) + per-view rendering + per-action handlers could all be separate components. Out of scope for the modal migration just completed; flag for future refactor when the next substantial change lands in this file. (`FirstUsePrompt` is a corner toast, `UsageNudge` is an inline banner — different primitives, intentionally not on `ui/Modal`.)
- **Certbot writes back to the vhost on cert renewal.** Lines marked `# managed by Certbot` in `/etc/nginx/sites-available/ordoai` get re-written by Certbot every ~60 days. The repo's `ops/nginx/ordoai.conf` is the source of truth for routing config but doesn't see those edits — until the next deploy that touches the vhost re-syncs from repo (potentially overwriting Certbot's drift). Acceptable for v1; the routing-critical content (`location` blocks) is fully protected. Future cleanup: split SSL config into a separate Certbot-owned `include` outside the repo.
- **No frontend test infra.** `lib/emailLinks.ts:stripMarkdown` is a pure regex transformation, ideal first candidate for unit tests when Vitest/Jest is set up. Manual verification today (10 transformation cases smoke-tested via Node REPL on initial commit); future regex tweaks risk silent regression.
- **`ExecutionBar.tsx` weekly-summary email and a few user-typed `mailto:` sites still construct URLs inline** (`EmailDelegatePanel.tsx:33`, `TaskDetailModal.tsx:1311` simple-delegate, `people/page.tsx:413,423` invites, `app/onboarding/page.tsx:282-298`). Low risk (no Ava markdown source — bodies are user-typed or templated) but worth refactoring to `lib/emailLinks.ts` for consistency next time they're touched.
- **`starlette` 0.38.6 has 2 known CVEs** (CVE-2024-47874 multipart DoS, CVE-2025-54121 content-type confusion) — fix is starlette 0.47.2 but it requires bumping `fastapi` past 0.115.0, which is broader than a security commit should be. Defer to a planned FastAPI upgrade window. Mitigation today: nginx is in front of the multipart path; rate limits at infra level reduce DoS surface.
- **`postcss` <8.5.10 (transitive via Next.js)** has GHSA-qx2v-qp2m-jg93 (XSS via unescaped `</style>` in stringify output). Build-time only; production bundle does not invoke the vulnerable codepath at runtime. Fix requires a Next.js downgrade — defer until a newer Next ships an upgrade.

## UX / Product Issues

- **Picker modals list ALL the user's documents/tasks, no search/filter.** Phase 1D scope.
- **Documents-library page picker** doesn't visibly mark already-linked tasks (excludeTaskIds is empty by design — see tech debt above).
- **Extraction status is opaque** — `⏳`/`✓` icons; failed status renders the same as processing.
- **Hero "Create with Ava" button removed** — replaced by "Make me Happy" reset modal. The future Create-with-Ava flow is being redesigned separately; the reset modal is the placeholder until that scope lands. (This also resolves the prior duplicate-destination concern where Go-to-tasks and Create-with-Ava both navigated to `/tasks`.)
- **Wins card uses `tasks.updated_at` as completion time.** Editing a done task bumps its position in the wins list.
- **No breadcrumbs.** Sub-routes only navigable back via sidebar.
- **No "Mark completed" from the Archived tab.** Reactivate (flip back to `active`) is now wired (this commit); marking an archived project `completed` directly is not. PATCH endpoint accepts both transitions — only the second button is missing. Add when usage signal warrants. **Soft-deleted project recovery still needs a backend endpoint** (separate problem — no undelete endpoint exists).

## Risks

- **Single EC2 = SPOF.**
- **Schedulers co-located with API.**
- **Free-plan limit hardcoded** — changes require deploy.
- **Stripe webhook idempotency is the only mechanism** preventing duplicate plan flips on redelivery.
- **Email ingestion dual-writes** with no transaction wrap visible.
- **`pm2 dump.pm2`** must be saved after ad-hoc pm2 changes or they vanish on reboot.
- **`.next/` corruption mid-build** = app down. Mitigated by EBS resize but still a class of failure.
- **Auto-deploy on master = no staging.**
- **AI cost not capped** for pro users.

## Next Likely Steps

1. **Phase 1D** — Search/filter in picker modals (both project and task pickers). Becomes useful as users accumulate many docs/tasks. Could also fix the documents-library `excludeTaskIds=[]` tradeoff by tracking per-doc linked tasks and marking them.
2. **Phase 2** — Cross-project/task document features (tagging, doc-level chat with Ava, cross-context search, folders).
3. Operational cleanup: resolve hung apt-get; add logrotate; consider deleting `/dashboard/chat`; decide on `mind-systems/` (~581 M on prod).
4. Tech debt candidates: declare the 6 prod indexes on models (or trim them); resolve documents↔tasks circular FK; add `completed_at` on Task; standardise datetime TZ; consider consolidating the two parallel link mechanisms (drop `documents.task_id` and `documents.project_id` FKs in favour of association tables).
