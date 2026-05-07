---
name: posthog-event-analytics
description: Implements or updates backend product analytics instrumentation with PostHog. Use it when adding, reviewing, or refactoring server-side event tracking, identity calls, group analytics, feature-flag exposure tracking, or analytics-aware logging in APIs, workers, webhooks, and background jobs.
---

# PostHog Backend Event Analytics

## Overview

Use this skill when backend code needs reliable PostHog instrumentation.

This skill is for:

- server-side event capture
- `identify` and alias flows
- group analytics
- feature flag and experiment exposure events
- analytics-aware logging for jobs, APIs, and webhook handlers
- auditing analytics for event quality, privacy, and consistency

**Keywords**: PostHog, product analytics, event tracking, backend analytics, server-side capture, identify, alias, groups, feature flags, experiments, observability, logging, event taxonomy

## Workflow

1. Find the business action that matters.
2. Confirm the canonical event name and properties before editing code.
3. Instrument as close as possible to the successful state change.
4. Reuse existing analytics wrappers or client factories if the codebase already has them.
5. Keep analytics non-blocking and never let capture failures break the main flow.
6. Add or update logs so production debugging can correlate app behavior with analytics emission.
7. Verify tests, idempotency, and privacy constraints.

## Event Design Rules

### Naming

- Prefer clear, business-level names such as `order_completed` or `report_exported`.
- Use past-tense or state-change naming for completed actions.
- Avoid UI-only names in backend systems unless the backend is explicitly mirroring frontend telemetry.
- Keep naming consistent with the existing event taxonomy.

### Properties

- Include only properties needed for analysis, debugging, segmentation, or experimentation.
- Prefer stable identifiers and low-cardinality dimensions.
- Exclude secrets, tokens, passwords, raw access credentials, and unnecessary personal data.
- Normalize enums and status values instead of sending free-form text when possible.

### Identity

- Use `identify` when a stable user identity becomes known or changes.
- Use alias or merge flows only when the product already uses them and identity stitching is intentional.
- For account or workspace analytics, use group analytics if the codebase supports it.

## Implementation Rules

### Capture Location

- Emit events after the durable success path, such as after commit, enqueue success, or confirmed external completion.
- Do not emit success events before the operation actually succeeds.
- For failure analytics, send a distinct failure event only when the product truly analyzes failures as product behavior.

### Reliability

- Analytics must be best-effort.
- Wrap capture calls so exceptions are logged and swallowed unless the codebase explicitly treats analytics as critical.
- Avoid duplicate events in retries, webhooks, and background jobs. Use idempotency keys or existing dedupe guards where available.

### Logging

- Log enough context to trace emission attempts and failures.
- Include event name, actor or subject identifier when safe, and correlation IDs or request IDs when available.
- Never log full sensitive payloads just because they were sent to analytics.

## Validation

- Check existing wrappers, naming conventions, and tests before adding new helpers.
- Add or update tests around event emission when the repo already tests analytics behavior.
- If there is no direct analytics assertion pattern, test the wrapper call boundary or mock the PostHog client.
- Confirm local or staging configuration does not silently disable the path you are validating.

## References

- For event taxonomy, naming, and payload guidance, read [references/posthog-event-patterns.md](references/posthog-event-patterns.md).
- For backend implementation and review checklist items, read [references/backend-instrumentation-checklist.md](references/backend-instrumentation-checklist.md).

