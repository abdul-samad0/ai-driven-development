# Keep Kaam V1 Final Project Brief

Date: 2026-05-08
Primary Source: [Keep Kaam PRD Updated (V2) (1).docx](/Users/mac/Documents/keep-kaam-training/docs/prd/Keep%20Kaam%20PRD%20Updated%20%28V2%29%20%281%29.docx)
Context Source: [.agents/keep-kaam-prd-clarification-log.md](/Users/mac/Documents/keep-kaam-training/.agents/keep-kaam-prd-clarification-log.md)

## Documentation Stage

- Documentation stage: `Planning complete`
- Repository stage on `2026-05-08`: `Docs-only; implementation has not started`
- Next action: `Use this brief, the synced plans, and the ERD to begin implementation review and Phase 1 execution`

## Product Scope

This brief is limited to:

- Module: `Leave Management System & Project Allocation`
- Version: `V1.0`
- Status: `Draft`
- Date: `February 2026`
- Prepared for: `Engineering, QA, People & Culture, Management`

Any unrelated V2 expansion content should be excluded from the active product brief.

## Product Purpose

Keep Kaam is an internal operations platform for a service-based software house. In this scoped brief, the platform focuses on:

- digitizing leave application, approval, policy enforcement, and leave reporting
- managing project assignment, allocation, capacity visibility, and related project/customer planning context

The goal is to replace spreadsheet-based and informal operational workflows with a structured, auditable, policy-aligned system.

## In Scope

### Leave Management System

- leave application and approval workflows
- leave-type policy enforcement
- leave balance calculation and deduction
- annual-leave eligibility, proration, reset, and deduction rules
- sick-leave medical certificate enforcement
- leave revision, resubmission, cancellation, and overlap handling
- leave reporting
- accounts-facing leave deduction reporting
- public holiday calendar as an input to leave calculations and planning
- notifications related only to leave events
- audit logging for leave-related state changes

### Project Allocation

- project assignment and reassignment
- project role per assignment
- allocation index / percentage behavior
- temporary and capped overallocation governance
- future-dated assignments
- allocation heatmaps and capacity visibility
- customer details related to project allocation
- project budget tracking
- deadline tracking
- milestone tracking and completion percentage
- customer health score
- customer communication log
- statement of work attachment
- automatic project creation trigger when a deal is won
- notifications related only to project-allocation events
- audit logging for project-allocation state changes

### Shared Platform Requirements Needed By These Modules

- Google OAuth
- RBAC
- session handling and related access-control behavior
- operational UI details required to use the scoped modules effectively

## Explicitly Out Of Scope

- onboarding flows
- profile-completion workflows
- general team-member document management
- team-member setup / profile-creation / account-setup flows
- reimbursements
- AI subscription approvals
- payslips
- asset allocation module
- full sales pipeline module
- announcements
- feedback / performance modules
- broad V2 module expansion
- competitive analysis
- market comparison
- broad changelog and history sections

## Actor Model

- `Team Member`
  Applies for leave and views personal leave/allocation context where relevant.

- `Delivery Manager`
  Primary approver for normal leave where a valid manager route exists. Uses allocation and capacity visibility.

- `People & Culture / HR`
  Participates in exception leave workflows, fallback approvals, policy enforcement, and reporting.

- `System Administrator`
  Owns administrative configuration and access-control controls needed by the active modules.

- `Leadership / Head of Delivery / Management`
  Uses project-allocation, customer, deadline, and capacity visibility.

- `Accounts`
  Real actor only where leave-deduction reporting is needed.

- `Cofounders`
  Recognized actor in the normalized model, though broader roadmap responsibilities are outside this brief.

- `Core`
  Separate actor/group from Admin; not automatically the same as System Administrator.

## Leave Management Rules

### Approval Routing

- leave approval is primarily tied to the Delivery Manager of the employee's most recently assigned active project
- if no active project exists, HR becomes the approver
- if the primary project has no assigned Delivery Manager, leave approval falls back to HR
- for normal leave types, HR alone is enough in fallback cases
- for dual-gate leave types, HR alone is also enough in fallback cases if no valid manager route exists
- backdated leave may be created by all users

### Leave Types

The active leave types are:

- `Annual`
- `Sick`
- `Casual`
- `Unpaid`
- `Wedding`
- `Special`

### Casual Leave

- capped at 2 days per month
- no separate yearly cap
- auto-converts to Annual Leave if it exceeds 2 consecutive days
- not available to probation employees
- available to contractual/part-time employees
- available to interns

### Sick Leave

- requires manager approval in normal cases
- if it exceeds 2 consecutive days, a medical certificate is required before submission
- with a valid certificate, it may exceed 2 consecutive days without converting to Annual Leave

### Annual Leave

- becomes eligible after 12 months
- first-year entitlement is prorated by remaining full calendar months
- proration formula is `15 / 12 * remaining full calendar months`
- a partial month at eligibility start does not count
- prorated entitlement is rounded up to the nearest 0.5 day
- already-eligible employees reset to the full 15-day balance on January 1
- the 5-consecutive-day limit uses employee-selected days only
- sandwich/calendar expansion still affects deduction from annual balance
- available to contractual/part-time employees after 12 months

### Wedding Leave

- fixed at 5 working days
- available once during an employee's tenure
- requires manager approval plus HR review when a valid manager route exists
- available to all employee types unless blocked by another rule such as Notice Period

### Special Leave

- fixed categories:
  - Umrah
  - Medical emergency/accident
  - Death of parent, sibling, spouse, or child
- duration is decided case by case by HR
- repeat usage is allowed with HR approval
- available to all employee types unless blocked by another rule such as Notice Period
- exempt from the `>2 consecutive days -> Annual Leave` auto-conversion rule

### Unpaid Leave

- allowed even when Annual Leave balance remains
- available to all employee types unless blocked by another rule such as Notice Period
- exempt from the `>2 consecutive days -> Annual Leave` auto-conversion rule

### Half-Day Rule

- half-day leave is not supported

### Notice Period

- notice period blocks all leave by default
- HR and System Administrator may override the restriction
- leave created via notice-period override still requires manager approval where a valid manager route exists
- entering notice period does not automatically cancel already-approved future leave

### Revision, Resubmission, and Cancellation

- rejected leave requests may be edited and resubmitted as the same request
- the same reference ID is preserved
- previous approval and rejection comments remain visible
- `Needs Revision` requests continue blocking overlap on their updated dates
- in `Needs Revision`, the employee may change the leave dates
- if leave dates change in `Needs Revision`, earlier manager approval remains valid
- if approved leave is cancelled before its start date, approval history remains visible
- a request auto-converted from `Casual` to `Annual` may not be cancelled just to revert to a shorter casual request

### Dual-Gate Outcomes

For `Backdated`, `Wedding`, `Special`, and `Unpaid` leave:

- both manager approval and HR approval are required when a valid manager route exists
- if the manager rejects, final status becomes `Rejected`
- if the manager approves but HR rejects, final status becomes `Needs Revision`
- after an HR-driven revision, manager re-review is not required

### Overlap and Deactivation

- overlap is blocked against `Pending`, `Approved`, and `Needs Revision` requests
- if an employee is deactivated:
  - pending leave requests are automatically cancelled
  - already approved future leave is automatically cancelled

## Project Allocation Rules

### Assignment Authority

Project assignment and reassignment authority belongs to:

- `System Administrator`
- `People & Culture Manager`
- `Leadership / Head of Delivery`

### Assignment Structure

- team member role per project remains in scope
- one employee may have multiple active assignments to the same project at the same time
- multiple same-project assignments must represent different roles or purposes
- identical parallel assignments are not allowed
- parallel same-project assignment allocations are summed into total allocation

### Allocation Governance

- allocation above 100% and up to 125% requires mandatory justification
- temporary overallocation above 100% must have an end date
- temporary overallocation above 100% is capped at 21 consecutive days
- when temporary overallocation ends, only the temporary portion is removed
- no additional allocation constraint is required beyond the current 125% limit and temporary-overallocation governance

### Future-Dated Assignments

- future-dated assignments are allowed
- future-dated assignments count toward total allocation even before their start date
- if a future-dated assignment causes allocation to exceed 100%, it is allowed only if it fits temporary-overallocation rules

### Deactivation

- when an employee is deactivated, future-dated project assignments are automatically deactivated

### Project And Customer Planning Context

The following remain core parts of `Project Allocation`:

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

### Project Creation Trigger

- `deal won -> create project` remains in scope as a project-allocation entrypoint
- full sales pipeline stages remain out of scope

## Shared Platform Requirements

### Authentication And Access Control

- Google OAuth
- RBAC
- session handling
- related access-control behavior

### Audit Logs And Notifications

- audit logs remain in scope only for actions inside `Leave Management` and `Project Allocation`
- notifications remain in scope only for `Leave Management` and `Project Allocation` events

### Operational UI Details

Operational UI details remain in scope where they support these modules, including:

- alternating row colors
- truncated long emails
- copy-to-clipboard behavior
- filters tabs

## Final Notes

- This brief intentionally excludes unrelated V2 expansion content.
- The long-form memory of clarified decisions is preserved in:
  [.agents/keep-kaam-prd-clarification-log.md](/Users/mac/Documents/keep-kaam-training/.agents/keep-kaam-prd-clarification-log.md)
- That file should continue to be treated as the context source for future revisions of this scoped brief.
