# PostHog Event Patterns

## Canonical Event Shape

Prefer this mental model when instrumenting:

- `event`: the business action or state change
- `distinct_id`: the stable actor identity
- `properties`: compact context for analysis
- `timestamp`: explicit only when replaying or backfilling
- `groups`: account, workspace, organization, or team context when supported

## Good Event Names

- `user_signed_up`
- `invite_accepted`
- `project_created`
- `subscription_upgraded`
- `report_exported`
- `webhook_delivery_failed`

## Avoid

- `clicked_button`
- `api_called`
- `did_thing`
- `success`
- `error`

These are usually too vague, too UI-specific, or too broad for useful backend analysis.

## Suggested Property Categories

### Actor

- `user_id`
- `account_id`
- `workspace_id`
- `role`

### Subject

- `project_id`
- `report_id`
- `subscription_id`
- `job_id`

### Flow Context

- `source`
- `channel`
- `plan`
- `entrypoint`
- `trigger_type`

### Outcome

- `status`
- `failure_reason`
- `duration_ms`
- `retry_count`

## Privacy Rules

- Do not send passwords, secrets, auth headers, raw tokens, payment credentials, or full request bodies.
- Avoid high-cardinality free text unless there is a strong product need.
- Prefer internal IDs over names or emails when analysis does not require human-readable values.
- If email is already an approved identity field in the codebase, keep usage consistent and minimal.

## Group Analytics

When analytics should roll up by company or workspace, attach group context consistently:

- account
- workspace
- organization
- team

Use only the grouping model already present in the codebase. Do not invent parallel group types.

## Feature Flags And Experiments

On the backend, track exposure when:

- the backend evaluates a flag that changes behavior
- an experiment branch affects business logic
- a job or webhook runs different code paths based on a flag

Avoid emitting extra exposure events if the existing flag system already captures them automatically.

