# Backend Instrumentation Checklist

Use this checklist when implementing or reviewing PostHog analytics in backend code.

## Before Coding

- Find existing analytics helpers, clients, and test patterns.
- Confirm the canonical event name already exists or choose a name that fits the taxonomy.
- Confirm whether the path is synchronous, transactional, queued, retried, or idempotent.
- Confirm which identifiers and properties are safe to send.

## During Coding

- Emit on the success path, not before it.
- Keep analytics side effects non-blocking.
- Reuse existing wrappers instead of creating a second analytics abstraction.
- Add structured logging around analytics failures if it improves operability.

## Review Questions

- Will this event fire too early or too often?
- Can retries or duplicate webhooks emit the same event multiple times?
- Are any properties sensitive or unnecessarily high-cardinality?
- Should this be a group event instead of only a user event?
- Should the code also call `identify` when the system learns new identity data?
- Is there an existing frontend event with the same business meaning that should match this name?

## Testing

- Mock the analytics client or wrapper.
- Assert event name and key properties.
- Cover retry or failure behavior when duplicate emission is a risk.
- Verify analytics exceptions do not fail the main request or job unless explicitly intended.
