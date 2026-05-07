---
name: unit-test-auth-permissions
description: >-
  Authorization and Django permission wiring for DRF tests: 403 cases,
  alternate users, groups, and codenames. Use when endpoints are permission-gated
  beyond simple authentication. For 401 and unauthenticated request mechanics
  (logout, assertNumQueries(0)), use unit-test-api-foundation.
---

# Unit Test Auth and Permissions

Use this skill when the endpoint is **permission-gated** (403) or you need to wire **Django permissions** via groups.

Unauthenticated (401) tests follow the standard pattern in `unit-test-api-foundation`: `logout()`, optional `assertNumQueries(0)`, then assert status and canonical 401 body.

## Unauthorized case (403)

- `self.client.logout()` then create an alternate user with a factory
- `self.client.force_authenticate(user=other_user)` (user lacks the required permission or role)
- Expected response:
  - status: `status.HTTP_403_FORBIDDEN`
  - body: `{"detail": "You do not have permission to perform this action."}`

## Permission wiring (Django auth)

When endpoint requires a codename:

1. Create group from project factory
2. Add user to group
3. Fetch permission by codename
4. Add permission to group

```python
permission = Permission.objects.get(codename="add_team")
group.permissions.add(permission)
```

## Factory guidance

- Use project factories rather than direct model construction
- For passwords, use `factory.PostGenerationMethodCall("set_password", "...")`
- Keep organization/team relationships realistic to project models

## Related skills in this repo

- `backend/unit-test/api-foundation` — APITestCase setup, 401 pattern, response assertions
- `backend/unit-test/mocking-side-effects` — mocking permission backends or external auth only when needed
