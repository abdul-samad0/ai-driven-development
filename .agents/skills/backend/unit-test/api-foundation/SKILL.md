---
name: unit-test-api-foundation
description: >-
  Django/DRF API test scaffolding (APITestCase, setUpTestData, naming), standard
  request patterns including unauthenticated calls (logout, assertNumQueries),
  response assertions (status, validation payloads, ErrorDetail, 404), and
  create/update/delete follow-ups. Use when writing or reorganizing API view
  tests or expected_result payloads.
---

# Unit Test API Foundation (Django / DRF)

Default skill for **structure and contract** of API tests. Pair with `unit-test-auth-permissions` for permission wiring and `unit-test-mocking-side-effects` for patches, stubs, and task/signal boundaries.

## Class and method conventions

- API view test classes: `{ViewName}APIViewTestCase`
- Base class: `rest_framework.test.APITestCase`
- Method names start with `test_`
- Use one class per endpoint or view feature

## Setup pattern

- Put shared objects in `setUpTestData(cls)` (users, orgs, base URL)
- Put per-test auth or mutable setup in `setUp(self)`
- Prefer `cls.base_url = reverse("url_name", kwargs={...})` when URL is stable

## Standard request patterns

### Authenticated requests (default)

In `setUp`, use `self.client.force_authenticate(user=self.user)` when the endpoint requires an authenticated user.

### Unauthenticated requests (401 coverage)

`force_authenticate` persists on the client until cleared. For tests that expect **401 Unauthorized**:

1. Call `self.client.logout()` before the request so the client is not authenticated.
2. When the view should do **no database work** before rejecting (typical for permission classes that deny unauthenticated users early), wrap the request in `with self.assertNumQueries(0):` to lock that invariant.

Then assert status and body (see canonical bodies below).

### Query counting on success paths

- Wrap the request in `with self.assertNumQueries(N):` when you want to guard against N+1 or accidental extra queries.
- Keep `N` strict enough to catch regressions without being brittle.

## Request / assertion order

1. Optionally wrap the request in `assertNumQueries(N)` (including `0` for no-DB cases above).
2. Capture `response`.
3. Define `expected_result` when comparing bodies.
4. Assert `response.status_code` first, then `response.data`.
5. Optionally pass `response.content` as failure context in `assertEqual` for clearer diffs.

## Create / update / delete follow-up checks

- For create: assert object existence or key fields
- For update/delete: reload object with `refresh_from_db()`
- For soft delete: assert inactive flag (`is_active` or project equivalent)

## Canonical response bodies

- 401: `{"detail": "Authentication credentials were not provided."}`
- 403: `{"detail": "You do not have permission to perform this action."}`
- 404: `{"detail": "Not found."}`

## Validation errors

- Match serializer output shape exactly
- Use exact field keys and message format
- For DRF structured errors, use `ErrorDetail` where strict equality is required

```python
expected_result = {
    "team": [ErrorDetail(string='Invalid pk "1000" - object does not exist.', code="does_not_exist")]
}
```

## Ordered responses

- Use `OrderedDict` when response order is meaningful and the API returns ordered keys
- Prefer deterministic expected payloads over partial assertions when feasible

## Base skeleton

```python
class TeamListAPIViewTestCase(APITestCase):
    @classmethod
    def setUpTestData(cls):
        cls.user = UserFactory(...)
        cls.base_url = reverse("team_list")

    def setUp(self):
        self.client.force_authenticate(user=self.user)

    def test_teams_listing(self):
        with self.assertNumQueries(2):
            response = self.client.get(self.base_url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data["results"], expected_result)

    def test_unauthenticated(self):
        self.client.logout()
        with self.assertNumQueries(0):
            response = self.client.get(self.base_url)
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)
        self.assertEqual(
            response.data,
            {"detail": "Authentication credentials were not provided."},
        )
```

## Related skills in this repo

- `backend/unit-test/auth-permissions` — 403, groups, codenames
- `backend/unit-test/mocking-side-effects` — patches, stubs, tasks, signals
