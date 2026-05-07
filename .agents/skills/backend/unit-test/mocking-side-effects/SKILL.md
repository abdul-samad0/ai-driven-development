---
name: unit-test-mocking-side-effects
description: >-
  Mock and stub external boundaries in Django tests: patch placement, call
  assertions, lightweight stubs (return_value, side_effect), async task dispatch,
  signals, and post-request DB state. Use when replacing real I/O or asserting
  side effects after API or unit tests.
---

# Unit Test Mocking, Stubs, and Side Effects

Use this skill when tests must **replace collaborators** (tasks, HTTP, email) or **assert state** after an action.

## Mocks vs stubs

- **Mock**: replace a callable or object and **assert how it was used** (`assert_called_once`, `assert_called_once_with`, `call_args`). Use when the contract is “this must be invoked with these arguments.”
- **Stub**: replace a dependency with a **canned implementation or return value**; often no call assertions, or only a light “was invoked” check. Use when you only need to block real side effects or return fixed data.

Prefer the **smallest stand-in**: if you do not care about arguments, a stub with `return_value=` (or a short `side_effect` function) is enough.

## Mocking rules

- Patch where the symbol is **used**, not where it is defined
- Use `@mock.patch` or `@unittest.mock.patch`
- Method args after `self` follow **reverse decorator order**
- For stricter stubs of callables, consider `autospec=True` or `spec=` so the mock matches the real signature

## Stub patterns (unittest.mock)

```python
@mock.patch("my_app.services.fetch_quota")
def test_uses_default_quota(self, fetch_quota):
    fetch_quota.return_value = {"used": 0, "limit": 10}
    ...

@mock.patch("my_app.tasks.send_invite.delay")
def test_invite_enqueued(self, send_invite):
    send_invite.side_effect = None  # no return; just prevent Celery
    ...
```

Use `MagicMock` when you need attribute sub-stubs without defining each nested name.

## Typical targets to mock or stub

- Async task dispatch (`*.delay` / `apply_async`) — often stubbed so nothing runs
- External services (email, payment, webhooks, HTTP clients)
- Heavy helpers with non-deterministic behavior

## Non-API tests

- Base class: `django.test.TestCase` or `unittest.TestCase`
- Verify side effects:
  - DB records created/updated
  - counters/totals changed
  - expected task/event dispatch occurred (mock/stub + assertion when relevant)

## Post-request state checks

- Reload instances with `refresh_from_db()`
- Assert final persisted values after update/delete
- Include soft-delete flag assertions when applicable

## Example (mock with assertion)

```python
@mock.patch("my_app.tasks.send_invite.delay")
def test_onboarding(self, mock_send_invite):
    with self.assertNumQueries(6):
        response = self.client.patch(self.base_url, payload)
    self.assertEqual(response.status_code, status.HTTP_200_OK)
    mock_send_invite.assert_called_once()
    self.obj.refresh_from_db()
    self.assertEqual(self.obj.is_active, True)
```

## Related skills in this repo

- `backend/unit-test/api-foundation` — request wrapping, assert order, query counts
- `backend/unit-test/auth-permissions` — real permission setup (prefer real wiring over mocking auth unless isolated)
