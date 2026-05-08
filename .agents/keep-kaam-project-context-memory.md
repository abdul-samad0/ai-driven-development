# Keep Kaam Project Context Memory

Date: 2026-05-08
Purpose: Preserve the working context established in recent chats without changing the finalized source-of-truth documents.

## Documentation Stage

- Documentation stage: `Planning complete`
- Repository stage on `2026-05-08`: `Docs-only; this note tracks aligned implementation context before build work begins`
- Role in the docs set: `Working memory only, kept in sync with the brief, plans, and ERD`

## Source Of Truth

Do not modify these files when using this memory note as reference:

- [.agents/keep-kaam-prd-clarification-log.md](/Users/mac/Documents/keep-kaam-training/.agents/keep-kaam-prd-clarification-log.md)
- [2026-05-08-keep-kaam-v1-final-project-brief.md](/Users/mac/Documents/keep-kaam-training/docs/superpowers/specs/2026-05-08-keep-kaam-v1-final-project-brief.md)

These remain the final scoped references for V1.

## Active Product Scope

Keep Kaam V1 is currently limited to:

- `Leave Management System`
- `Project Allocation`

Shared support requirements still in scope only where they directly support those modules:

- Google OAuth
- RBAC
- token-based authentication
- audit logs
- notifications
- operational UI support details

Out-of-scope modules such as onboarding, reimbursements, asset allocation, payslips, full sales pipeline, and broad V2 expansion should not be introduced into current V1 architecture work.

## Technical Direction

Implementation direction agreed in chat:

- backend: `Django + Django REST Framework`
- frontend: `React.js + TypeScript`

Architecture and modeling work should be aligned with this stack.

## Backend Implementation Plan Locked

Implementation planning has been completed and captured in:

- [2026-05-08-keep-kaam-v1-backend-implementation-plan.md](/Users/mac/Documents/keep-kaam-training/docs/superpowers/plans/2026-05-08-keep-kaam-v1-backend-implementation-plan.md)

This plan should now be treated as the active backend implementation reference unless a later approved revision replaces it.

## Frontend Implementation Plan Locked

Frontend implementation planning has been completed and captured in:

- [2026-05-08-keep-kaam-v1-frontend-implementation-plan.md](/Users/mac/Documents/keep-kaam-training/docs/superpowers/plans/2026-05-08-keep-kaam-v1-frontend-implementation-plan.md)

This plan should now be treated as the active frontend implementation reference unless a later approved revision replaces it.

## Backend Direction Agreed In Chat

- backend root should be `backend/`
- architecture should use modular Django apps:
  - `common`
  - `access`
  - `people`
  - `projects`
  - `leave`
  - `notifications`
  - `audit`
- business rules should live in service/domain layers, not only serializers or views
- execution should move phase by phase, with review/approval before advancing to the next phase

## Authentication Decision Locked

- API authentication should be token-based, not session-based
- preferred default implementation is `rest_framework_simplejwt`
- Phase 1 should include:
  - token obtain flow
  - token refresh flow
  - explicit decision on logout/refresh-token blacklisting
  - Google OAuth exchange path that issues first-party API tokens
- `django.contrib.sessions` may still exist for Django admin/framework compatibility, but not as the primary API auth strategy

## Important Backend Decisions Locked

- enforce critical integrity with database-backed constraints, not only service logic
- employee deactivation must be owned through lifecycle services, including:
  - deactivating future project assignments
  - cancelling pending leave
  - cancelling already-approved future leave
- leave workflows must support medical-certificate handling for long sick leave in V1
- leave and assignment mutations must use atomic service boundaries with concurrency protection
- leave-day counting must have one shared business-calendar source of truth
- audit and notification side effects must use explicit idempotency keys / dedupe strategy

## Phase Sequence Locked

- `Phase 1`: Platform Foundation and Access Control
- `Phase 2`: People, Customer, Project, and Assignment Domain Modeling
- `Phase 3`: Leave Domain Model and Policy Engine
- `Phase 4`: Leave APIs, Approval Workflow, and Employee/Manager UX Endpoints
- `Phase 5`: Project Allocation APIs, Capacity Views, and Intake Trigger
- `Phase 6`: Audit Logs, Notifications, and Accounts-Facing Reporting
- `Phase 7`: Hardening, Observability, Performance, and Release Readiness

## Frontend Repo Direction Locked

Implementation direction agreed in chat for the frontend plan:

- frontend repo type: `Vite + React + TypeScript`
- package manager: `yarn`
- router basename: `/app`
- import alias: `#/...`
- main frontend stack:
  - `React 19`
  - `React Router 7`
  - `TanStack Query`
  - `React Hook Form`
  - `Zod`
  - `Axios`
  - `Tailwind CSS`
  - `Sentry`
- important frontend runtime entry points:
  - `src/main.tsx`
  - `src/App.tsx`
  - `src/routes/createBrowserRouter.tsx`
  - `src/routes/PrivateRoute.tsx`
  - `src/context/AuthContext.tsx`
  - `src/services/axiosInstance.ts`
  - `src/services/apiClient.ts`
- important frontend wrapper stack to preserve:
  - `ErrorBoundary`
  - `AuthProvider`
  - `QueryClientProvider`
  - `RouterProvider`
  - `ToastContainer`
  - `ReactQueryDevtools`

## Frontend Placement Rules Locked

- `src/components/ui` is for atomic UI only, no business logic
- `src/components/forms` is for form UI only, typed from `src/schemas`
- `src/pages` is for page composition only, not business logic
- `src/hooks` contains logic-only hooks, and hook files must start with `use`
- `src/services` contains API client code only, built on `src/services/axiosInstance.ts`
- `src/schemas` contains Zod validation schemas
- `src/types` contains shared and API types
- `src/utils` contains pure helpers only
- use `#/` absolute imports

## Frontend And Backend Sync Rules Locked

- frontend and backend implementation plans must stay in sync
- future changes to one plan that affect payloads, workflow stages, permissions, filters, or execution order should be reflected in the other plan in the same review cycle
- frontend work should not begin for a dependent phase until the matching backend review gate is approved
- keep both implementation plans aligned with:
  - [2026-05-08-keep-kaam-v1-final-project-brief.md](/Users/mac/Documents/keep-kaam-training/docs/superpowers/specs/2026-05-08-keep-kaam-v1-final-project-brief.md)
  - [.agents/keep-kaam-prd-clarification-log.md](/Users/mac/Documents/keep-kaam-training/.agents/keep-kaam-prd-clarification-log.md)
  - [2026-05-08-keep-kaam-v1-erd.md](/Users/mac/Documents/keep-kaam-training/docs/architecture/erd/2026-05-08-keep-kaam-v1-erd.md)
  - this memory file

## Review-Driven Plan Fixes Applied

The frontend and backend plans were reviewed together and aligned further:

- frontend Phase 4 no longer depends on `allowed actions` / `availableActions`
- frontend approval visibility is now expected to derive from queue membership plus workflow stage metadata
- backend Phase 4 now explicitly includes:
  - `view own leave request detail`
  - scoped leave reporting endpoints for frontend operational reporting
- backend Phase 6 now explicitly includes notification APIs needed by the frontend plan:
  - paginated notification list
  - unread-only filter
  - mark-as-read mutation
  - optional unread-count summary if lightweight enough

## Working Guidance For Future Frontend Chats

- use the final scoped brief, clarification log, ERD, backend implementation plan, frontend implementation plan, and this memory file together
- do not invent frontend-visible workflow behavior that the backend plan does not guarantee
- do not regress from the agreed frontend repo rules, runtime entry points, or placement rules without explicit re-decision
- preserve the phase-by-phase review contract for frontend work just as strictly as backend work

## Working Guidance For Future Backend Chats

- use the final scoped brief, clarification log, and backend implementation plan together
- do not regress from token auth back to session auth for APIs unless explicitly re-decided
- do not skip phase review gates
- do not introduce out-of-scope modules while implementing backend phases
- preserve the current separation between source-of-truth docs, architecture docs, implementation plan, and memory notes

## ERD Artifact Created

Architecture documentation created:

- [docs/architecture/README.md](/Users/mac/Documents/keep-kaam-training/docs/architecture/README.md)
- [docs/architecture/erd/2026-05-08-keep-kaam-v1-erd.md](/Users/mac/Documents/keep-kaam-training/docs/architecture/erd/2026-05-08-keep-kaam-v1-erd.md)

## ERD Decisions Agreed In Chat

- Create both a conceptual ERD and a detailed ERD in Mermaid.
- Keep the ERD in a dedicated architecture docs area instead of mixing it into the final brief files.
- Do not add database constraints or constraint instructions into the ERD document.
- Include auth/access support entities in the ERD for visualization:
  - `USER_ACCOUNT`
  - `OAUTH_IDENTITY`
  - `AUTH_SESSION`
  - `ACCESS_ROLE`
  - `USER_ROLE_ASSIGNMENT`
- Still note that some or all of those auth/access entities may map to Django auth, sessions, permissions/groups, or OAuth integration tables rather than fully custom business models.
- Show `PK` and `FK` inside the detailed ERD cards.
- Mark bridge records directly in the cards:
  - `USER_ROLE_ASSIGNMENT` as the bridge between `USER_ACCOUNT` and `ACCESS_ROLE`
  - `PROJECT_ASSIGNMENT` as the bridge between `EMPLOYEE_PROFILE` and `PROJECT`
  - `LEAVE_BALANCE_ACCOUNT` as the junction between `EMPLOYEE_PROFILE` and `LEAVE_TYPE`
- Keep relationship hints in Mermaid lines such as `grants`, `opens`, `receives`, `influences`, `triggers`, and `progresses_through`.
- Keep the explanatory text beneath the ERD concise and easy to scan.

## Review-Driven Fixes Applied

The ERD was reviewed and then adjusted to preserve consistency:

- The auth/support note now clearly says those entities are intentionally shown in the ERD.
- Missing `USER_ACCOUNT` relationship lines were added for existing FK fields:
  - `USER_ROLE_ASSIGNMENT.assigned_by_user_id`
  - `CUSTOMER_COMMUNICATION_LOG.recorded_by_user_id`
  - `LEAVE_APPROVAL_STEP.approver_user_id`

## Working Guidance For Future Chats

- Use the two finalized source-of-truth files first when making scope decisions.
- Treat this file as working memory, not as a replacement for the final brief or clarification log.
- Avoid introducing out-of-scope V2 modules into ERD, planning, model, API, or UI work.
- Preserve the current architecture-doc structure unless there is a clear reason to expand it.
