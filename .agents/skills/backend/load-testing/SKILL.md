---
name: backend-load-testing
description: >-
  Write Locust load tests for Django/DRF REST APIs. Covers two-part workflow:
  setup (base.in + make cr) and writing authenticated multi-endpoint tests
  following project conventions (make_request helper, on_start setup,
  task-per-endpoint style). Use when writing, generating, or reviewing load
  tests or performance/stress tests for any Django API endpoint.
---

# Load Testing Django/DRF APIs with Locust

Use Locust to simulate concurrent users hitting API endpoints. This skill has two parts — complete **Part 1 (Setup)** before **Part 2 (Writing Tests)**.

Pair with `backend/unit-test/api-foundation` for endpoint contracts and response shapes.

---

## Part 1 — Setup

### 1. Add the dependency

Open `requirements/base.in` and add:

```
locust
gevent
```

Do **not** run `pip install` directly. The project manages dependencies through compiled requirement files.

### 2. Install

```bash
make cr
```

This recompiles and installs all dependencies, including the newly added packages.

### 3. Create the load tests directory

```
<project_root>/
└── load_tests/
    └── <feature>_load_test.py
```

If the project already has a `load_tests/` directory, place new files alongside existing ones and follow the naming convention already in use.

---

## Part 2 — Writing Tests

### Discovering endpoints before writing tasks

Before writing any tasks, read the project's URL configuration:

- Django DRF: look for `<app>/api/v<N>/urls.py`
- Map each route to its HTTP method and path
- Prioritise by traffic profile:

| Priority | What to include |
|---|---|
| High | High-traffic reads — list, detail, search, dashboard |
| Medium | Key writes in the normal user flow (create, update) |
| Low / skip | DELETE, hard-reset, admin/internal, or irreversible operations |

---

### Pattern A — Authenticated multi-endpoint test

Use when endpoints require a Bearer/JWT token and you want to simulate realistic user sessions hitting several endpoints.

```python
import logging

from decouple import config
from django.conf import settings
from gevent import monkey
from locust import HttpUser, between, task


monkey.patch_all()

logging.basicConfig(level=logging.INFO)


class LoadTestUser(HttpUser):
    wait_time = between(1, 2)
    host = settings.<BASE_URL_SETTING>   # read from Django settings, not hardcoded

    access_token = config("ACCESS_TOKEN")  # reads from .env; never hardcode credentials
    user_id = None
    # declare any other IDs the tasks will need as class-level None

    def on_start(self):
        """Called once per simulated user before tasks begin."""
        self.get_user_detail()

    def get_user_detail(self):
        """Retrieves user details and updates instance variables."""
        endpoint = "/users/detail/"
        headers = {"Authorization": f"Bearer {LoadTestUser.access_token}"}
        response = self.client.get(endpoint, headers=headers)

        if response.status_code == 200:
            data = response.json()
            self.user_id = data["id"]
            logging.info("User details retrieved successfully.")
        else:
            logging.error(
                f"Failed to get user detail: {response.status_code}, {response.text}"
            )
            self.stop()  # must stop — tasks must not run with broken state

    def make_request(self, endpoint, method="GET", headers=None):
        """Generic function to make API requests and handle responses."""
        if headers is None:
            headers = {"Authorization": f"Bearer {self.access_token}"}
        response = self.client.request(method, endpoint, headers=headers)
        if response.status_code == 200:
            logging.info(f"Request to {endpoint} succeeded.")
        else:
            logging.error(
                f"Failed request to {endpoint}: {response.status_code}, {response.text}"
            )

    @task
    def get_user_addresses(self):
        """Fetches all addresses of the authenticated user."""
        self.make_request("/core/address/")

    @task
    def get_user_profile(self):
        """Fetches profile for the authenticated user."""
        self.make_request(f"/users/{self.user_id}/profile/")

    @task
    def get_public_categories(self):
        """Fetches publicly available categories — no auth needed."""
        self.make_request("/catalog/categories/", headers={})
```

**Writing tasks for Pattern A:**

- **One `@task` per endpoint** — keep tasks small and focused
- **Name tasks after what they fetch**: `get_user_addresses`, `get_patient_cards`
- **Public endpoints** — pass `headers={}` to skip auth explicitly (not `None`)
- **Parameterised paths** — use instance variables set during `on_start`:
  ```python
  @task
  def get_patient_records(self):
      """Fetches records for the authenticated patient."""
      self.make_request(f"/vaults/{self.patient_id}/records/")
  ```
- **POST/PUT tasks** — generate unique values per request to avoid uniqueness/idempotency failures. A fixed payload will succeed once and then return 400/409 for every subsequent iteration, which measures validation errors rather than write performance:
  ```python
  import uuid

  @task(1)
  def create_draft(self):
      """Creates a draft record with a unique title per request."""
      self.client.post(
          "/vaults/drafts/",
          json={"title": f"load-test-{uuid.uuid4().hex}"},
          headers={"Authorization": f"Bearer {self.access_token}"},
      )
  ```
  Prefer `uuid.uuid4().hex` for free-form fields and `int(time.time() * 1000)` for numeric IDs. If the endpoint is idempotent by design, a fixed payload is fine — document why.
- **Task weighting** — use `@task(weight)` to model realistic traffic (reads far outnumber writes):
  ```python
  @task(3)
  def get_items(self): ...

  @task(1)
  def create_item(self): ...
  ```

---

### Pattern B — Unauthenticated stress / single-endpoint test

Use when hammering one endpoint at maximum throughput — e.g. a login endpoint, token exchange, or public search.

```python
import logging

from decouple import config
from gevent import monkey
from locust import HttpUser, constant, task


monkey.patch_all()

logging.basicConfig(level=logging.INFO)


class StressUser(HttpUser):
    wait_time = constant(0)  # no pause → maximum throughput
    host = config("BASE_URL", default="https://your-api.example.com")

    def on_start(self):
        self.username = config("TEST_USERNAME", default="testuser")
        self.password = config("TEST_PASSWORD", default="testpass")

    @task
    def login(self):
        """Simulates a login request."""
        response = self.client.post(
            "/api/v1/auth/token/",
            json={"username": self.username, "password": self.password},
        )
        if response.status_code == 200:
            logging.info("Login succeeded.")
        else:
            logging.error(f"Login failed: {response.status_code}, {response.text}")
```

---

## Running load tests

### Interactive (web UI)

```bash
locust -f load_tests/<feature>_load_test.py
# Opens http://localhost:8089 — set users/spawn rate in the browser
```

### Headless (CI / scripted)

Ensure the `.env` file contains the required variables (e.g. `ACCESS_TOKEN`), then:

```bash
locust -f load_tests/<feature>_load_test.py \
  --headless \
  --users 50 \
  --spawn-rate 5 \
  --run-time 60s
```

---

## Key rules

| Rule | Why |
|---|---|
| `monkey.patch_all()` at the very top | Locust uses gevent; patching makes I/O non-blocking so thousands of users run efficiently |
| `on_start` must call `self.stop()` on failure | Without it, tasks run with `None` IDs and produce misleading results |
| Credentials via `decouple.config()`, never hardcoded | Reads from `.env`; load test files get committed, so hardcoded secrets are a security risk |
| `headers={}` for public endpoints (not `None`) | `None` triggers the default auth path inside `make_request`; `{}` explicitly skips it |
| `logging.basicConfig(level=logging.INFO)` | Locust captures stdout — structured logging is the only reliable way to trace what happens during a run |
| Avoid destructive endpoints (DELETE, reset) | Load tests run many iterations; accidental mass-deletes in staging are hard to undo |
| Add to `base.in`, then `make cr` — never `pip install` | Keeps dependency graph consistent and reproducible across environments |

---
