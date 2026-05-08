# Keep Kaam V1 Frontend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the React + TypeScript frontend for Keep Kaam V1, scoped to `Leave Management System` and `Project Allocation`, with strong client-side validation, safe API integration, role-aware UX, accessibility, and production-ready test coverage.

**Architecture:** Use the existing Vite frontend at repo root with `src/` as the app root, React Router under basename `/app`, the existing auth/query/error provider stack, and the established separation of `components`, `pages`, `hooks`, `services`, `schemas`, `types`, `utils`, and `data`. Keep policy-heavy logic on the backend; the frontend should guide users, validate inputs early, and render backend-driven workflow outcomes correctly.

**Tech Stack:** React 19, TypeScript, Vite, React Router v7, TanStack Query, React Hook Form, Zod, Axios, Tailwind CSS, Sentry, React Toastify, React Query Devtools, Vitest, React Testing Library, MSW, Playwright, axe-core.

---

## Documentation Stage

- Documentation stage: `Planning complete`
- Repository stage on `2026-05-08`: `Docs-only; frontend implementation has not started`
- Execution readiness: `Ready for implementation once the matching backend review gate is approved`

## Planning Notes

- This plan is updated against the provided frontend handoff, which is now the authoritative structure reference even though the actual app files are not present in the current workspace snapshot.
- Frontend root is the repository root, with application code in `src/`.
- Package manager: `yarn`
- Router basename: `/app`
- Import alias: `#/...`
- Runtime entry points that all implementation phases must preserve:
  - `src/main.tsx`
  - `src/App.tsx`
  - `src/routes/createBrowserRouter.tsx`
  - `src/routes/PrivateRoute.tsx`
  - `src/context/AuthContext.tsx`
  - `src/services/axiosInstance.ts`
  - `src/services/apiClient.ts`
- Existing runtime wrappers to preserve:
  - `ErrorBoundary`
  - `AuthProvider`
  - `QueryClientProvider`
  - `RouterProvider`
  - `ToastContainer`
  - `ReactQueryDevtools`
- Planned UI priorities:
  - role-aware navigation
  - safe form UX
  - approval workflows
  - allocation visibility
  - reporting-ready filters and tables
- Phase model follows the backend plan so frontend contracts do not race ahead of backend review gates.
- Frontend must follow the provided handoff rules plus `.agents/skills/frontend/AGENTS_ENGINEERING_STANDARDS.md`.
- Backend skill context intentionally retained while planning:
  - `schema-design`: strict typed contracts and enum-safe UI state
  - `django_constraints_indexing`: surface uniqueness/conflict errors cleanly instead of hiding them
  - `reusable_query_logic`: central query keys, reusable selectors, paginated/filter-aware data access
  - `unit-test-api-foundation`: strong unhappy-path coverage and exact error-shape assertions
  - `caching-performance-optimization`: measured caching, low refetch churn, list performance discipline

## Core Frontend Principles

- The frontend must not re-implement authoritative leave-policy decisions.
- Client-side validation should catch obvious mistakes early, but backend responses remain final.
- Every mutation path must handle:
  - loading
  - success
  - validation failure
  - permission failure
  - conflict/stale state
  - network retry/recovery
- Every screen must ship with role-aware empty states, skeletons, and error states.
- Every phase ends with review signoff before the next phase starts.

## Backend Sync Contract

- This frontend plan must stay in lockstep with [2026-05-08-keep-kaam-v1-backend-implementation-plan.md](/Users/mac/Documents/keep-kaam-training/docs/superpowers/plans/2026-05-08-keep-kaam-v1-backend-implementation-plan.md).
- The frontend must not treat a backend phase as implementation-ready until the matching backend review gate is approved.
- Shared source-of-truth alignment must be preserved with:
  - [2026-05-08-keep-kaam-v1-final-project-brief.md](/Users/mac/Documents/keep-kaam-training/docs/superpowers/specs/2026-05-08-keep-kaam-v1-final-project-brief.md)
  - [.agents/keep-kaam-prd-clarification-log.md](/Users/mac/Documents/keep-kaam-training/.agents/keep-kaam-prd-clarification-log.md)
  - [2026-05-08-keep-kaam-v1-erd.md](/Users/mac/Documents/keep-kaam-training/docs/architecture/erd/2026-05-08-keep-kaam-v1-erd.md)
  - [.agents/keep-kaam-project-context-memory.md](/Users/mac/Documents/keep-kaam-training/.agents/keep-kaam-project-context-memory.md)
- Backend Phase 1 is the dependency for frontend auth shell assumptions.
- Backend Phases 2 and 3 are the dependency for frontend shared domain typing and leave-form policy hints.
- Backend Phase 4 is the dependency for frontend employee leave flows and approval UX.
- Backend Phase 5 is the dependency for frontend project allocation and capacity UX.
- Backend Phase 6 is the dependency for frontend notifications, audit visibility, and accounts-facing reporting.
- Backend Phase 7 is the dependency for frontend release-readiness, smoke coverage, and production hardening signoff.
- If backend contracts change, update frontend types, schemas, query hooks, route guards, and tests in the same review cycle.

## Proposed Frontend Layout

```text
react-js-boilerplate/
├── public/
├── src/
│   ├── components/
│   │   ├── errorFallback/
│   │   ├── forms/
│   │   ├── home/
│   │   ├── ui/
│   │   └── unauthorized/
│   ├── constants/
│   ├── context/
│   ├── data/
│   ├── hooks/
│   ├── pages/
│   ├── routes/
│   ├── schemas/
│   ├── services/
│   ├── types/
│   ├── utils/
│   ├── App.tsx
│   └── main.tsx
├── package.json
├── vite.config.ts
└── tsconfig.json
```

## Placement Rules To Enforce In Every Phase

- `src/components/ui` stays atomic and presentation-only.
- `src/components/forms` contains form UI only, typed from `src/schemas`.
- `src/pages` contains page composition only, not business logic.
- `src/hooks` contains logic-only hooks, and hook files must start with `use`.
- `src/services` contains API client code only, built on `src/services/axiosInstance.ts`.
- `src/schemas` contains Zod validation schemas.
- `src/types` contains shared and API types.
- `src/utils` contains pure helpers only.
- Use `#/` absolute imports.

## Phase Review Contract

Every phase must end with all of the following:

- unit and integration tests for that phase pass
- relevant Playwright coverage passes or is added to the deferred Phase 7 E2E batch with explicit tracking
- a11y checks pass for new critical flows
- loading, empty, error, and forbidden states are implemented
- mobile and desktop behavior are reviewed
- API contract assumptions are listed before the phase is marked complete
- you review and approve before the next phase starts

---

## Phase 1: Frontend Foundation, Auth Shell, and Route Guards

**Outcome:** A runnable frontend exists with app shell, providers, typed API client, auth bootstrap, role-aware routing, and consistent feedback primitives.

**Review Gate:** Do not start feature screens until auth/session/bootstrap behavior and route protection rules are approved.

**Backend Dependency:** Backend Phase 1 must be approved first because frontend auth, role guards, `/me`, and token behavior depend on stable access-control decisions.

**Files**

- Modify: `package.json`
- Modify: `src/main.tsx`
- Modify: `src/App.tsx`
- Modify: `src/routes/createBrowserRouter.tsx`
- Modify: `src/routes/PrivateRoute.tsx`
- Modify: `src/context/AuthContext.tsx`
- Modify: `src/services/axiosInstance.ts`
- Modify: `src/services/apiClient.ts`
- Create: `src/constants/routes.ts`
- Create: `src/constants/queryKeys.ts`
- Create: `src/hooks/useCurrentUser.ts`
- Create: `src/components/errorFallback/AppErrorState.tsx`
- Create: `src/components/ui/AppSkeleton.tsx`
- Create: `src/pages/{Login,Unauthorized,Dashboard}.tsx`
- Create: `src/test/msw/server.ts`
- Create: `src/test/integration/auth-shell.test.tsx`

**Key Build**

- preserve the existing Vite + React + TypeScript shell
- configure router basename `/app`, QueryClient, Axios instance, env handling, Sentry hooks, and global error normalization
- preserve the existing provider stack in `src/main.tsx` and `src/App.tsx`
- add authenticated layout and public layout without breaking `AuthProvider` or `ToastContainer`
- implement protected routes by actor role
- wire `GET /me` bootstrap query
- define token storage strategy compatible with backend token-based auth
- add global query defaults, retry policy, and unauthorized redirect handling

**Short Code Direction**

```tsx
export function ProtectedRoute({
  allow,
  children,
}: ProtectedRouteProps): JSX.Element {
  const { user, isLoading } = useCurrentUser();

  if (isLoading) return <AppSkeleton />;
  if (!user) return <Navigate to="/login" replace />;
  if (!hasRequiredRole(user.roles, allow)) return <Navigate to="/unauthorized" replace />;

  return <>{children}</>;
}
```

```ts
export const queryKeys = {
  auth: {
    me: ["auth", "me"] as const,
  },
  leave: {
    mine: (params?: Record<string, string | number | boolean | undefined>) =>
      ["leave", "mine", params ?? {}] as const,
  },
};
```

```ts
export async function getCurrentUser(): Promise<CurrentUserResponse> {
  const response = await apiClient.get<CurrentUserResponse>(API_ENDPOINTS.auth.me);
  return response.data;
}
```

**Validation and Error Cases**

- expired/invalid token on bootstrap
- authenticated user with missing role claims
- 401 vs 403 routing behavior
- backend unavailable during app init
- corrupted local auth cache
- direct URL access to protected routes

**Tests**

- Unit:
  - role guard helpers
  - auth error normalizer
  - token persistence helpers
- Integration:
  - redirect unauthenticated user to login
  - render unauthorized page for wrong actor
  - recover from failed `/me` request
  - keep shell stable while bootstrap is loading
- E2E:
  - login bootstrap happy path
  - hard-refresh on protected route with valid token
  - invalid token sends user back to login

**Performance Notes**

- lazy-load route groups from `src/routes/createBrowserRouter.tsx`
- cache `/me` with controlled stale time
- prevent duplicate bootstrap requests on initial mount

**API Contract Assumptions**

- backend exposes authenticated current-user summary for bootstrap
- current-user payload includes stable role information for route guards
- backend returns deterministic 401 for unauthenticated and 403 for unauthorized access
- token refresh behavior is implemented or intentionally deferred, but the decision must be explicit before Phase 1 exits

---

## Phase 2: Shared UI System, Domain Types, and Form Infrastructure

**Outcome:** Shared typed contracts, reusable UI primitives, table/filter patterns, and form infrastructure exist before module-specific work begins.

**Review Gate:** Do not build leave or allocation workflows until shared field, table, modal, and validation patterns are approved.

**Backend Dependency:** Backend Phases 2 and 3 must be materially stable because shared frontend contracts depend on employee, project, leave-type, status, and policy-driven enum shapes.

**Files**

- Modify: `src/types/**/*`
- Modify: `src/schemas/**/*`
- Modify: `src/components/ui/**/*`
- Modify: `src/components/forms/**/*`
- Create: `src/components/ui/{Input,Select,Textarea,DateField,Badge,Modal}.tsx`
- Create: `src/components/forms/FormField.tsx`
- Create: `src/components/ui/DataTable.tsx`
- Create: `src/components/ui/TableToolbar.tsx`
- Create: `src/hooks/{usePagination,useDebouncedValue}.ts`
- Create: `src/utils/{apiErrors,date,formatters}.ts`
- Create: `src/test/integration/shared-ui.test.tsx`

**Key Build**

- define API response types and discriminated UI state shapes
- align all new contracts with `src/types`, not ad hoc feature-local typing
- standardize Zod form schemas and backend-error mapping
- add reusable paginated table and filter tabs
- add modal, confirmation, toast, and inline error conventions
- codify row truncation, copy-to-clipboard, alternating rows, and status badges

**Short Code Direction**

```ts
export const leaveRequestDraftSchema = z.object({
  leaveType: z.enum(["ANNUAL", "SICK", "CASUAL", "UNPAID", "WEDDING", "SPECIAL"]),
  startDate: z.string().min(1),
  endDate: z.string().min(1),
  reason: z.string().trim().min(3).max(500),
  specialCategory: z.string().optional(),
  medicalCertificate: z.instanceof(File).optional(),
}).superRefine((value, ctx) => {
  if (value.endDate < value.startDate) {
    ctx.addIssue({ code: "custom", path: ["endDate"], message: "End date must be after start date." });
  }
});
```

```ts
export function mapApiErrorsToForm<TFieldValues extends FieldValues>(
  error: ApiError,
  setError: UseFormSetError<TFieldValues>,
): void {
  if (error.fieldErrors) {
    Object.entries(error.fieldErrors).forEach(([field, message]) => {
      setError(field as Path<TFieldValues>, { type: "server", message });
    });
    return;
  }

  setError("root", { type: "server", message: error.message });
}
```

**Validation and Error Cases**

- invalid enum values from stale UI state
- date parsing and timezone-safe display
- optional file upload rules for certificate flows
- long text truncation with accessible full-value reveal
- filter state restoring from URL params

**Tests**

- Unit:
  - Zod schemas
  - formatters
  - query-string helpers
- Integration:
  - shared form field error rendering
  - modal focus trap and escape handling
  - filter toolbar syncing with URL state
- E2E:
  - reusable table keyboard navigation
  - copy-to-clipboard interaction

**Performance Notes**

- keep table primitive render-light
- debounce search/filter input updates
- centralize date formatting to avoid repeated expensive transforms

**API Contract Assumptions**

- backend enums for leave status, leave type, employment type, project status, and role names are stable enough to encode in frontend types
- backend date fields are consistently serialized in a predictable format
- backend validation payloads are stable enough to map into React Hook Form without per-endpoint special cases

---

## Phase 3: Leave Employee Flows

**Outcome:** Team members can create, edit, resubmit, cancel, and review their leave requests with strong frontend validation and clear policy guidance.

**Review Gate:** Do not build manager/HR approval tools until employee leave creation and detail flows are approved against backend API contracts.

**Backend Dependency:** Backend Phase 4 employee leave endpoints must be reviewed and approved before this frontend phase starts.

**Files**

- Create: `src/data/leave/queries/*.ts`
- Create: `src/data/leave/mutations/*.ts`
- Create: `src/components/forms/{LeaveRequestForm}.tsx`
- Create: `src/components/leave/{LeaveRequestList,LeaveRequestTimeline,LeaveBalanceCards}.tsx`
- Create: `src/hooks/leave/{useLeaveFilters,useLeaveForm}.ts`
- Create: `src/pages/{LeaveList,LeaveCreate,LeaveDetail}.tsx`
- Create: `src/test/integration/leave-employee-flows.test.tsx`
- Create: `src/test/e2e/leave-employee.spec.ts`

**Key Build**

- personal leave dashboard with balances, request list, and statuses
- create/edit/resubmit flow with conditional fields
- cancellation flow for allowed statuses
- detail timeline showing preserved approval/rejection comments
- attachments UI for medical certificate
- UX messaging for backend-driven policy results such as auto-conversion and notice-period override rejection

**Short Code Direction**

```tsx
const selectedLeaveType = form.watch("leaveType");
const shouldShowMedicalUpload =
  selectedLeaveType === "SICK" && consecutiveDays > 2;

const submitLeave = useMutation({
  mutationFn: createLeaveRequest,
  onSuccess: () => queryClient.invalidateQueries({ queryKey: queryKeys.leave.mine() }),
});
```

```tsx
const onSubmit = form.handleSubmit(async (values) => {
  try {
    await submitLeave.mutateAsync(values);
    navigate(APP_ROUTES.leave.list);
  } catch (error) {
    mapApiErrorsToForm(normalizeApiError(error), form.setError);
  }
});
```

**Validation and Error Cases**

- start date after end date
- same-day invalid duration if half-day is attempted
- unsupported attachment type or oversize file
- special leave without category
- sick leave longer than 2 days without certificate
- revision flow preserving same reference ID
- cancel action on already-started or already-cancelled request
- overlap conflicts returned by backend
- notice period blocked leave with override messaging

**Tests**

- Unit:
  - leave form conditional logic
  - attachment validators
  - status-to-badge mapping
- Integration:
  - create leave happy path
  - render backend field errors and non-field errors
  - resubmit revised request with preserved timeline
  - hide forbidden actions by status
- E2E:
  - employee submits normal leave
  - employee hits overlap validation
  - employee resubmits `Needs Revision`
  - employee uploads required medical certificate

**Performance Notes**

- optimistic UI only for safe local states like modal close, not for authoritative leave status changes
- paginate leave history
- avoid refetching all leave screens after every mutation

**API Contract Assumptions**

- backend provides endpoints for create, revise, cancel, list own requests, and view balances
- leave detail payload includes status, comments history, reference ID, attachments metadata, and approval history
- backend enforces overlap, notice-period, certificate, and revision semantics and returns user-displayable errors
- revised leave requests preserve the same reference ID as specified in the backend plan

---

## Phase 4: Leave Approval, HR Review, and Leave Reporting UX

**Outcome:** Delivery Managers and HR can review, approve, reject, request revision, and inspect leave workflows with role-specific queues and reporting filters.

**Review Gate:** Do not move to project allocation screens until approval queues, dual-gate behavior, and fallback-routing UX are approved.

**Backend Dependency:** Backend Phase 4 manager/HR endpoints and approval workflow semantics must be approved before this frontend phase starts.

**Files**

- Create: `src/components/leave/{ApprovalQueue,ApprovalActionPanel,LeaveReportFilters}.tsx`
- Create: `src/pages/{LeaveApprovalQueue,LeaveReport}.tsx`
- Modify: `src/data/leave/**/*`
- Create: `src/test/integration/leave-approval-flows.test.tsx`
- Create: `src/test/e2e/leave-approval.spec.ts`

**Key Build**

- manager queue, HR queue, and role-specific action visibility
- approve/reject/request-revision actions with comment capture
- dual-gate explanation UI for Wedding, Special, Unpaid, and Backdated leave
- fallback-route messaging when HR is sole approver
- reporting filters for type, status, employee, date range, and actor scope

**Short Code Direction**

```tsx
const isInMyQueue = queueItem.assignedApproverId === user.id;
const requiresManagerStage = queueItem.currentStage === "MANAGER";
const requiresHrStage = queueItem.currentStage === "HR";

const canManagerAct = isInMyQueue && requiresManagerStage;
const canHrAct = isInMyQueue && requiresHrStage;
```

```tsx
const approveMutation = useMutation({
  mutationFn: approveLeaveRequest,
  onSuccess: async () => {
    await queryClient.invalidateQueries({ queryKey: queryKeys.leave.approvalQueue() });
    toast.success("Leave request updated.");
  },
  onError: (error) => {
    toast.error(normalizeApiError(error).message);
  },
});
```

**Validation and Error Cases**

- approval action without required comment where backend requires one
- stale queue item already processed by another approver
- hidden action buttons for ineligible role
- dual-gate status explanation after HR rejection
- empty queue, empty report result, and forbidden report scope

**Tests**

- Unit:
  - action visibility helpers
  - queue filter param builders
- Integration:
  - manager approves eligible request
  - HR sends request to revision
  - stale item error toast and query refresh
  - report page preserves filters in URL
- E2E:
  - manager approval flow
  - HR revision flow
  - report filtering and export trigger if scoped for V1

**Performance Notes**

- poll or refresh approval queues conservatively
- split queue summary list from detail panel queries
- prefetch selected request detail on row focus/hover only if measured useful

**API Contract Assumptions**

- backend exposes approval queue endpoints with filters for status, leave type, date range, employee, and approver context
- leave payload includes current approval state, approval history, and enough workflow metadata to derive stage-aware visibility from queue membership and current workflow stage
- backend preserves fallback HR-only semantics and dual-gate workflow outcomes exactly as defined in the brief
- stale approval actions return deterministic conflict/already-processed responses suitable for toast + refresh UX

---

## Phase 5: Project Allocation CRUD, Assignment Workflow, and Capacity UX

**Outcome:** Authorized actors can manage customers, projects, assignments, and allocation states with strong conflict handling and usable capacity views.

**Review Gate:** Do not move to notifications/reporting hardening until allocation workflows and permission boundaries are approved.

**Backend Dependency:** Backend Phase 5 allocation, project, assignment, and capacity endpoints must be approved before this frontend phase starts.

**Files**

- Create: `src/data/projects/queries/*.ts`
- Create: `src/data/projects/mutations/*.ts`
- Create: `src/components/forms/{ProjectForm,AssignmentForm}.tsx`
- Create: `src/components/projects/{AllocationTable,CapacityView,ProjectHealthPanel}.tsx`
- Create: `src/pages/{ProjectList,ProjectDetail,AssignmentManagement,Capacity}.tsx`
- Create: `src/test/integration/project-allocation-flows.test.tsx`
- Create: `src/test/e2e/project-allocation.spec.ts`

**Key Build**

- customer/project list and detail screens
- assignment create/edit/reassign flows
- future-dated assignment handling
- allocation summary per employee and per project
- justification flow for >100% to 125% allocation
- temporary overallocation end-date enforcement
- communication log and SOW/document presentation

**Short Code Direction**

```ts
export const assignmentSchema = z.object({
  employeeId: z.string().min(1),
  projectId: z.string().min(1),
  roleName: z.string().min(1),
  allocationPercentage: z.number().min(1).max(125),
  startDate: z.string().min(1),
  endDate: z.string().optional(),
  overallocationJustification: z.string().optional(),
}).superRefine((value, ctx) => {
  if (value.allocationPercentage > 100 && !value.overallocationJustification?.trim()) {
    ctx.addIssue({ code: "custom", path: ["overallocationJustification"], message: "Justification is required above 100% allocation." });
  }
});
```

```tsx
const totalAllocation = assignments.reduce((sum, assignment) => {
  return sum + assignment.allocationPercentage;
}, 0);

const requiresJustification = totalAllocation > 100;
const exceedsHardLimit = totalAllocation > 125;
```

**Validation and Error Cases**

- duplicate parallel assignment with identical role/purpose
- >125% allocation attempt
- temporary overallocation without end date
- temporary overallocation longer than 21 days
- invalid future-dated range
- deactivated employee selection
- project document missing/unavailable
- conflict after concurrent assignment update

**Tests**

- Unit:
  - assignment schema conditions
  - allocation summary helpers
  - heatmap color threshold helpers
- Integration:
  - create assignment with justification
  - render backend conflict error cleanly
  - block forbidden actor actions
  - capacity filters and totals update correctly
- E2E:
  - leadership creates future assignment
  - over-allocation justification flow
  - deactivated employee cannot be assigned

**Performance Notes**

- virtualize high-row-count capacity tables if data size justifies it
- keep heatmap computation memoized and selector-based
- fetch table summaries separately from heavy detail payloads

**API Contract Assumptions**

- backend exposes customers, projects, milestones, communication logs, project document metadata, assignments, and capacity-summary endpoints
- capacity endpoints return heatmap-ready or summary-ready data without requiring frontend business recomputation
- assignment conflict and over-allocation responses are explicit enough to render field or banner errors
- intake-trigger-related project creation is backend-owned and not modeled as a broad sales workflow in the frontend

---

## Phase 6: Notifications, Audit Visibility, and Operational Reporting UI

**Outcome:** Users can see scoped notifications, action history, and operational reporting surfaces needed for V1 without leaking unrelated modules.

**Review Gate:** Do not start release hardening until operational visibility screens are approved as in-scope and sufficiently lightweight.

**Backend Dependency:** Backend Phase 6 audit, notification, and accounts-report endpoints must be approved before this frontend phase starts.

**Files**

- Create: `src/data/notifications/queries/*.ts`
- Create: `src/data/notifications/mutations/*.ts`
- Create: `src/components/notifications/{NotificationCenter,NotificationList}.tsx`
- Create: `src/components/reporting/{AuditTimeline,LeaveDeductionReport}.tsx`
- Create: `src/pages/{Notifications,Audit,AccountsLeaveReport}.tsx`
- Create: `src/test/integration/operational-reporting.test.tsx`

**Key Build**

- notification center for leave/allocation events only
- audit/event visibility on detail pages where backend exposes it
- accounts-facing leave deduction report UI
- export trigger UI only if backend confirms V1 support

**Short Code Direction**

```tsx
const { data, isLoading, isError } = useNotificationsQuery({
  unreadOnly,
  page,
});
```

```tsx
function handleMarkAsRead(notificationId: string): void {
  markAsReadMutation.mutate(
    { notificationId },
    {
      onSuccess: () => {
        queryClient.invalidateQueries({ queryKey: queryKeys.notifications.list() });
      },
    },
  );
}
```

**Validation and Error Cases**

- unauthorized report access
- stale notification mark-as-read request
- empty audit history
- partially available reporting data
- unsupported export format from backend

**Tests**

- Unit:
  - notification grouping helpers
  - report filter mappers
- Integration:
  - unread toggle and pagination
  - accounts-only report route guard
  - audit timeline renders event ordering correctly
- E2E:
  - user opens notification center
  - accounts user filters leave deduction report

**Performance Notes**

- incremental pagination for notifications
- avoid fetching audit timelines until detail panels are opened
- separate unread count query from full notification feed

**API Contract Assumptions**

- backend notification records are scoped only to leave and project-allocation events
- backend audit payloads are append-only and ordered consistently for timeline rendering
- accounts-facing leave deduction reporting is available without expanding into payroll
- export/download behavior is frontend-visible only if backend explicitly exposes it in scoped V1

---

## Phase 7: Hardening, Accessibility, Performance, and Release Readiness

**Outcome:** The frontend is stable enough for staging with complete regression coverage, accessibility checks, performance tuning, and production safeguards.

**Review Gate:** This phase closes the frontend implementation cycle and should be approved before deployment promotion.

**Backend Dependency:** Backend Phase 7 hardening and smoke flows should be approved before frontend release signoff is finalized.

**Files**

- Modify: `src/**/*`
- Create: `src/test/e2e/smoke.spec.ts`
- Create: `src/test/integration/error-boundaries.test.tsx`
- Modify: `README.md`
- Modify: `CONTRIBUTION.md`
- Review: `.github/pull_request_template.md`

**Key Build**

- error boundary and generic crash fallback
- exhaustive empty/loading/error/forbidden states audit
- form retry and offline-ish recovery polish
- bundle review and route-level code splitting verification
- keyboard navigation and screen-reader pass on critical workflows
- analytics/monitoring hooks if approved

**Short Code Direction**

```tsx
<ErrorBoundary fallback={<AppErrorState title="Something went wrong" />}>
  <RouterProvider router={router} />
</ErrorBoundary>
```

```ts
export function normalizeApiError(error: unknown): ApiError {
  if (isAxiosError<ApiErrorResponse>(error) && error.response?.data) {
    return {
      message: error.response.data.message ?? "Request failed.",
      fieldErrors: error.response.data.errors,
      statusCode: error.response.status,
    };
  }

  return {
    message: "Something went wrong. Please try again.",
  };
}
```

**Validation and Error Cases**

- network timeout and retry exhaustion
- 500 responses on list and detail pages
- malformed backend payload guardrails
- browser back/forward behavior with filters and forms
- unsaved changes prompts where appropriate

**Tests**

- Unit:
  - error-state mappers
  - fallback UI helpers
- Integration:
  - error boundary rendering
  - malformed payload handling
  - route transition retains filter state
- E2E:
  - full smoke test by actor-critical flow
  - mobile viewport regression checks
  - accessibility checks on key pages

**Performance Notes**

- main route bundle budget target under organizational standards
- profile Query cache churn on list/detail screens
- prefetch only high-confidence next routes
- verify no unnecessary rerender storms in large tables/forms

**API Contract Assumptions**

- backend smoke scenarios for leave and allocation can be mirrored in frontend E2E flows
- backend logging, permission matrix, and query regressions are stable enough that frontend failures can be triaged against trustworthy backend behavior
- no late backend contract changes are introduced after frontend smoke coverage is frozen without updating this plan and the backend plan together

---

## Cross-Phase Validation Matrix

### Authentication and Authorization

- unauthenticated user redirected correctly
- authenticated but unauthorized user shown 403-safe UX
- actor-role visibility for Team Member, Delivery Manager, HR, System Admin, Leadership, Accounts
- token refresh and logout state cleanup handled consistently
- basename `/app` routing works for direct deep links and reloads

### Data Validation

- required fields
- enum mismatches
- date-range validation
- numeric boundary validation
- attachment validation
- duplicate/conflict response handling
- server-field-error to RHF mapping
- URL filter param validation and recovery from malformed query strings

### Workflow Integrity

- leave revision preserves history
- approval actions respect backend state
- cancellation visibility matches status rules
- allocation concurrency conflicts trigger refresh/recovery
- redirect-after-success flows preserve correct actor landing pages
- repeated submit clicks do not create duplicate mutations

### Error Handling

- invalid payload shape
- validation error envelope mapping
- stale object conflict
- network failure
- backend 500 fallback
- missing attachment/document reference
- 429/rate-limit handling for retry-safe views
- offline or timed-out mutation recovery messaging

### Accessibility

- keyboard-only navigation
- visible focus states
- screen-reader labels on forms, tables, modals, and toasts
- color contrast and status communication not relying on color alone
- focus return after modal close and route-guard redirect

### Performance

- paginated lists by default
- route-level code splitting
- debounced filters
- virtualization only where row counts justify it
- low duplicate-query churn on dashboard screens
- stable query keys and scoped invalidation after mutations
- avoid unnecessary rerenders in form-heavy screens

## Coverage Review

Current plan coverage is strong across:

- role-based auth and route protection
- client validation plus backend-authoritative error mapping
- leave creation, revision, cancellation, approval, and reporting
- assignment conflicts, over-allocation, and capacity views
- notifications, audit visibility, and accounts reporting
- unit, integration, E2E, accessibility, and performance checks
- frontend-backend phase dependency alignment
- document-level sync with brief, clarification log, ERD, backend plan, and context memory

Remaining items to keep explicitly visible during execution:

- confirm actual test runner setup in `package.json` before Phase 1, because the workspace snapshot did not expose existing test tooling files
- confirm exact backend error envelope shape before finalizing `normalizeApiError` and `mapApiErrorsToForm`
- confirm whether token refresh is silent, proactive, or interceptor-driven inside `src/services/axiosInstance.ts`
- confirm whether export/report download is in V1 backend scope before building frontend export actions
- confirm final payload shape for approval queue items, leave timelines, capacity summaries, and audit events before locking UI types

---

## Suggested Phase Sequence for Review

1. Backend Phase 1 approved -> Frontend Phase 1: auth shell and route guards
2. Backend Phases 2-3 approved -> Frontend Phase 2: shared UI and typed form contracts
3. Backend Phase 4 employee endpoints approved -> Frontend Phase 3: employee leave flows
4. Backend Phase 4 approval endpoints approved -> Frontend Phase 4: leave approval and reporting UX
5. Backend Phase 5 approved -> Frontend Phase 5: allocation workflows and capacity views
6. Backend Phase 6 approved -> Frontend Phase 6: notifications, audit, and accounts reporting
7. Backend Phase 7 approved -> Frontend Phase 7: hardening and release readiness

This sequence keeps the frontend aligned with backend dependencies: auth first, shared contracts second, leave workflows before manager tooling, and allocation after backend summaries are stable.

---

## Assumptions to Keep Visible During Execution

- Frontend code lives at repo root with application code in `src/`.
- The backend remains the source of truth for leave-policy decisions and conflict detection.
- React Router v7, TanStack Query, React Hook Form, Zod, Axios, Tailwind, and Sentry are approved frontend defaults based on the handoff and engineering standards.
- File upload support is needed for medical certificates and project/SOW-related document display, even if storage details remain backend-led.
- Notifications remain limited to leave and project-allocation events only.
- Implementation must use `yarn` commands: `yarn dev`, `yarn build`, `yarn lint`, `yarn preview`.

---

## Handoff

Plan complete and saved to [2026-05-08-keep-kaam-v1-frontend-implementation-plan.md](/Users/mac/Documents/keep-kaam-training/docs/superpowers/plans/2026-05-08-keep-kaam-v1-frontend-implementation-plan.md).

Recommended execution mode for this plan:

1. Subagent-driven per phase, because each phase has a clear review gate and maps cleanly to your approval-first workflow.
2. Inside each approved phase, execute in order: failing tests, minimal implementation, integration verification, E2E pass for the critical path, then phase review summary.
