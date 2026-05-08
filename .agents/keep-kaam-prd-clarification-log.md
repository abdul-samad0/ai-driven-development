# Keep Kaam PRD Clarification Log

Date: 2026-05-08
Active Scope:

- Module: `Leave Management System & Project Allocation`
- Version: `V1.0`
- Status: `Draft`
- Date: `February 2026`
- Prepared for: `Engineering, QA, People & Culture, Management`

Primary Source: [Keep Kaam PRD Updated (V2) (1).docx](/Users/mac/Documents/keep-kaam-training/docs/prd/Keep%20Kaam%20PRD%20Updated%20%28V2%29%20%281%29.docx)
Purpose: Preserve only the clarified decisions that still matter to the active scoped brief.

## Documentation Stage

- Documentation stage: `Planning complete`
- Repository stage on `2026-05-08`: `Docs-only; clarifications are locked before implementation`
- Role in the docs set: `Acts as scoped decision support for the final brief, plans, and architecture docs`

## Scope Rules

### 1. Active brief boundary

- Only `Leave Management System` and `Project Allocation` belong in the active brief.
- Extra modules and unrelated sections should be treated as noise and removed from the active brief.

### 2. Removed as noise

- Onboarding flows
- Profile-completion workflows
- General team-member document management
- Team-member setup / profile-creation / account-setup flows
- Reimbursements
- AI subscription approvals
- Payslips
- Asset allocation module
- Full sales pipeline stages
- Announcements
- Feedback / performance modules
- Broad V2 module expansion
- Competitive analysis
- Market comparison
- Broad changelog/history sections

### 3. Still allowed because directly related

- Medical certificate upload for sick-leave policy enforcement
- Public holiday calendar because it affects leave calculation
- Deal won -> create project trigger because it is a project-allocation entrypoint
- Customer/project planning details that directly support allocation decisions
- Audit logs, notifications, auth, and RBAC only as shared requirements for these two modules

## Leave Management Decisions

### 4. Leave approval route

- Leave approval is primarily tied to the Delivery Manager of the employee's most recently assigned active project.
- If no active project exists, HR becomes the approver.
- If the primary project has no assigned Delivery Manager, leave approval falls back to HR.

### 5. Leave types in scope

- `Annual`
- `Sick`
- `Casual`
- `Unpaid`
- `Wedding`
- `Special`

### 6. Special Leave categories

- Umrah
- Medical emergency/accident
- Death of parent, sibling, spouse, or child

### 7. Half-day rule

- Half-day leave is not supported.

### 8. Casual Leave rules

- Casual Leave is capped at 2 days per month.
- Casual Leave has no separate yearly cap.
- If Casual Leave exceeds 2 consecutive days, it auto-converts to Annual Leave.
- Casual Leave is not available to probation employees.
- Casual Leave is available to contractual/part-time employees.
- Casual Leave is available to interns.

### 9. Sick Leave rules

- Sick Leave still requires manager approval in normal cases.
- If Sick Leave exceeds 2 consecutive days, a medical certificate is required before submission.
- With a valid certificate, Sick Leave may exceed 2 consecutive days without converting to Annual Leave.

### 10. Annual Leave rules

- Annual Leave becomes eligible after 12 months.
- First-year entitlement is prorated by remaining full calendar months.
- Proration formula is `15 / 12 * remaining full calendar months`.
- Partial month at eligibility start does not count.
- Prorated entitlement is rounded up to the nearest 0.5 day.
- Already-eligible employees reset to the full 15-day balance on January 1.
- The 5-consecutive-day limit uses employee-selected days only.
- Sandwich/calendar expansion still affects balance deduction.
- Annual Leave is available to contractual/part-time employees after 12 months.

### 11. Wedding Leave rules

- Wedding Leave requires manager approval plus HR review when normal manager routing exists.
- Wedding Leave is fixed at 5 working days.
- Wedding Leave is available only once during an employee's tenure.
- Wedding Leave is available to all employee types unless another rule blocks it, such as Notice Period.

### 12. Special Leave rules

- Special Leave requires manager approval plus HR review when normal manager routing exists.
- Special Leave duration is decided case by case by HR.
- The same Special Leave category may be used multiple times if HR approves.
- Special Leave is available to all employee types unless another rule blocks it, such as Notice Period.
- Special Leave is exempt from the >2 consecutive day auto-conversion rule.

### 13. Unpaid Leave rules

- Unpaid Leave requires manager approval plus HR review when normal manager routing exists.
- Unpaid Leave is allowed even when Annual Leave balance remains.
- Unpaid Leave is available to all employee types unless another rule blocks it, such as Notice Period.
- Unpaid Leave is exempt from the >2 consecutive day auto-conversion rule.

### 14. Notice Period rules

- Notice Period blocks all leave types by default.
- HR and System Administrator may override the restriction.
- Leave created through a Notice Period override still requires manager approval where a valid manager route exists.
- Entering Notice Period does not automatically cancel already-approved future leave.

### 15. Resubmission and revision rules

- Rejected leave requests may be edited and resubmitted as the same request.
- The same reference ID is preserved.
- Previous approval/rejection comments remain visible.
- `Needs Revision` requests continue to block overlap on their updated dates.
- In `Needs Revision`, the employee may change the leave dates.
- If dates are changed in `Needs Revision`, earlier manager approval remains valid.

### 16. Dual-gate approval behavior

- `Backdated`, `Wedding`, `Special`, and `Unpaid` leave need both manager approval and HR approval when a valid manager route exists.
- If the manager rejects, final status becomes `Rejected`.
- If the manager approves but HR rejects, final status becomes `Needs Revision`.
- When a dual-gate request is revised and resubmitted after HR rejection, the manager does not need to review it again.
- In fallback cases where no valid manager route exists, HR alone is enough even for these dual-gate leave types.

### 17. Overlap, cancellation, and deactivation

- Overlap is blocked against `Pending`, `Approved`, and `Needs Revision` requests.
- If an employee cancels an approved leave before the start date, approval history remains visible.
- If an employee is deactivated:
  - pending leave requests are automatically cancelled
  - already approved future leave is automatically cancelled

### 18. Accounts-facing leave reporting

- Accounts-facing leave deduction reporting remains in scope.
- This does not imply broader salary or payroll modules are in scope.

## Project Allocation Decisions

### 19. Project assignment authority

- Project assignment and reassignment may be performed by:
  - System Administrator
  - People & Culture Manager
  - Leadership / Head of Delivery

### 20. Project role and assignment structure

- Team member role per project remains in scope.
- One employee may have multiple active assignments to the same project at the same time.
- If multiple active assignments exist on the same project, they must represent different roles or purposes.
- Identical parallel assignments should not be allowed.
- Parallel same-project assignment allocations are summed into the employee's total allocation.

### 21. Over-allocation rules

- Any allocation above 100% and up to 125% requires mandatory justification.
- Temporary overallocation above 100% retains the 21 consecutive day cap.
- Temporary overallocation above 100% always requires an end date.
- When temporary overallocation ends, only the temporary portion is removed.

### 22. Future-dated assignment rules

- Future-dated project assignments are allowed.
- Future-dated assignments count toward total allocation even before their start date.
- If a future-dated assignment causes allocation to exceed 100%, it is allowed only if it fits temporary-overallocation rules.

### 23. Deactivation effects on allocation

- When an employee is deactivated, future-dated project assignments are automatically deactivated.

### 24. Project/customer planning details

- The following remain core parts of `Project Allocation`:
  - customer details
  - project budget tracking
  - deadline tracking
  - milestone tracking
  - completion percentage
  - customer health score
  - customer communication log
  - SOW attachment
  - allocation heatmaps
  - capacity visibility reporting

### 25. Project creation trigger

- `Deal won -> create project` remains in scope as a project-allocation entrypoint.
- Full sales pipeline stages do not remain in scope.

## Shared Platform Requirements Still In Scope

### 26. Authentication and access control

- Google OAuth
- RBAC
- session handling
- related access-control behavior

### 27. Audit logs and notifications

- Audit logs remain in scope only for actions inside `Leave Management` and `Project Allocation`.
- Notifications remain in scope only for `Leave Management` and `Project Allocation` events.

### 28. Operational UI details

- Operational UI details remain in scope where they support these modules, including:
  - alternating row colors
  - truncated long emails
  - copy-to-clipboard behavior
  - filters tabs

## Remaining Open Questions

- None.

## Notes

- This file is the active scoped memory artifact.
- The active scoped spec is:
  [2026-05-08-keep-kaam-v1-final-project-brief.md](/Users/mac/Documents/keep-kaam-training/docs/superpowers/specs/2026-05-08-keep-kaam-v1-final-project-brief.md)
- Extra historical or broad-V2 clarification noise has been intentionally removed.
- Related decisions were kept even when they touch shared platform behavior, as long as they directly support `Leave Management System` or `Project Allocation`.
