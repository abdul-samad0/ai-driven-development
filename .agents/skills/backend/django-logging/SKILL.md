---
name: django-logging
description: Complete reference for adding, configuring, and using logging in any Django backend project. Use this skill whenever the user wants to set up logging in Django, add a logger to a file, configure the LOGGING dict in settings, add structured/JSON logging, create logging middleware, log Celery tasks, log external API calls, or debug missing/duplicate log output. Trigger this skill even if the user just mentions "Django logs", "log something in my view", or "set up file logging" — you almost certainly want this skill for any Django observability task.
---

# Django Logging Skill

A complete reference for adding, configuring, and using logging in any Django backend project.

---

## Architecture Overview

Django uses Python's built-in `logging` module, configured via a `LOGGING` dictionary in settings.

| Component | Role |
|-----------|------|
| **Logger** | Entry point — the object you call in code (`logger.info(...)`) |
| **Handler** | Decides where a log record goes (file, console, external service) |
| **Formatter** | Controls the string format of the log record |
| **Filter** | Optional gate that can suppress or enrich records |

Log records propagate up the logger hierarchy by default (`myapp.users.tasks` → `myapp.users` → `myapp` → root), unless `propagate: False` is set.

---

## Settings Configuration (`config/settings/base.py`)

Place the full `LOGGING` dict in `base.py` so all environments inherit it. Override only what differs per environment.

### Minimal Production-Ready Setup

```python
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent.parent

LOGGING = {
    "version": 1,
    "disable_existing_loggers": True,
    "formatters": {
        "simple": {
            "format": "{asctime} {levelname} {message}",
            "style": "{",
        },
    },
    "handlers": {
        "file": {
            "level": "INFO",
            "class": "project.core.logging.handler.LoggingHandler",
            "filename": BASE_DIR / "debug.log",
            "formatter": "simple",
            "backupCount": 10,
            "maxBytes": 1 * 1024 * 1024,  # 1 MB
        },
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "simple",
        },
    },
    "loggers": {
        "django.request": {
            "handlers": [
                "file",
                "console",
            ],
            "level": "ERROR",
            "propagate": False,
        },
        "project": {
            "handlers": [
                "file",
                "console",
            ],
            "level": "INFO",
        },
    },
}
```

**Key decisions:**
- `"disable_existing_loggers": True` — disables all default Django loggers so that logging is fully controlled by this configuration.
- `propagate: False` on app logger prevents double-logging.

### Structured JSON Handler (for Datadog, ELK, CloudWatch)

```python
# myapp/core/logging/handler.py
import json

from logging.handlers import RotatingFileHandler


class LoggingHandler(RotatingFileHandler):
    def emit(self, record):
        msg = self.format(record)

        data = {
            "logger": {
                "name": record.name,
            },
            "error": {},
            "http": {},
            "user": {},
            "message": msg,
            "code": {
                "file_path": record.pathname,
                "function_name": record.funcName,
                "line_no": record.lineno,
            },
            "process": {
                "id": record.process,
                "name": record.processName,
            },
            "thread": {
                "id": record.thread,
                "name": record.threadName,
            },
        }

        if record.exc_info:
            data["error"]["type"] = record.exc_info[0].__name__
            data["error"]["value"] = str(record.exc_info[1])
            data["error"]["traceback"] = record.exc_text

        try:
            data["http"]["uri"] = record.request_uri
            data["http"]["method"] = record.request_method
            data["http"]["referer"] = record.http_referer
            data["http"]["useragent"] = record.http_user_agent
            data["user"]["ip"] = record.ip_address
            data["http"]["status_code"] = record.status_code
        except AttributeError:
            pass

        record.msg = json.dumps(data)
        record.args = ()

        try:
            if self.shouldRollover(record):
                self.doRollover()

            super().emit(record)
        except Exception:
            self.handleError(record)
```

Reference it in settings by replacing the `file` handler class:
```python
"class": "myapp.core.logging.handler.LoggingHandler",
```

---

## Adding a Logger to Any File

Always declare a module-level logger using `__name__`:

```python
import logging
logger = logging.getLogger(__name__)
```

Never hardcode a logger name — `__name__` resolves dynamically and stays accurate if the file is moved.

---

## Log Level Guidelines

| Level | Method | When to use |
|-------|--------|-------------|
| `DEBUG` | `logger.debug()` | Fine-grained tracing; off in production by default |
| `INFO` | `logger.info()` | Normal expected events: record created, task started/finished |
| `WARNING` | `logger.warning()` | Unexpected but recovered: retry, missing optional config |
| `ERROR` | `logger.error()` | Operation failed and could not recover |
| `CRITICAL` | `logger.critical()` | System-level failure requiring immediate intervention |

**Always use `logger.exception()` inside `except` blocks** — it's identical to `logger.error(..., exc_info=True)` and auto-captures the full traceback.

```python
try:
    result = external_service.call()
except ExternalServiceError:
    logger.exception("External service call failed")
    raise
```

---

## Common Patterns

### Log Body Size Limit (Constance Config)
Add `MAX_LOG_BODY_CHARS_LIMIT` in Constance to limit and control the size of response bodies stored in logs.

```python
#config/settings/constance.py

CONSTANCE_CONFIG = {
    "MAX_LOG_BODY_CHARS_LIMIT": (2048, "Maximum body size limit to keep for the logfile."),
}
```

### HTTP Request/Response Middleware

```python

import json
import logging

from copy import deepcopy
from uuid import uuid4

from constance import config
from ipware import get_client_ip


logger = logging.getLogger("project")


class LoggingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        logging_attributes = {
            "request_uri": request.build_absolute_uri(),
            "request_method": request.method,
            "http_referer": request.META.get("HTTP_REFERER", ""),
            "http_user_agent": request.META.get("HTTP_USER_AGENT", ""),
            "ip_address": get_client_ip(request)[0],
            "status_code": None,
        }

        uuid = uuid4()
        logger.info(f"REQ {uuid} {request.path}", extra=logging_attributes)

        response = self.get_response(request)

        response_body = "-"
        if "application/json" in response.get("content-type", "") and hasattr(response, "data"):
            response_body = self._truncate_large_response_body(deepcopy(response.data))

        status_code = response.status_code

        logging_attributes["status_code"] = status_code

        logger.info(f"RESP {uuid} {response_body}", extra=logging_attributes)

        return response

    def _truncate_large_response_body(self, value):
        """
        We are truncating the response body after a certain number of characters to
        keep the logs cleanable, understandable and easy to navigate.
        """

        response_body = json.dumps(value, default=str)
        response_body = f"{response_body[:config.MAX_LOG_BODY_CHARS_LIMIT]}..."
        return response_body
```

Register in settings: `"myapp.core.middleware.logging.LoggingMiddleware"`

### Celery Background Task

```python
logger = logging.getLogger(__name__)

@shared_task
def sync_records(user_id: int) -> None:
    logger.info(f"Starting record sync for user {user_id}")
    try:
        count = _do_sync(user_id)
        logger.info(f"Synced {count} records for user {user_id}")
    except Exception:
        logger.exception(f"Record sync failed for user {user_id}")
        raise   # re-raise so Celery marks the task as FAILURE
```

### External API Call

```python
def fetch_user_profile(user_id: str) -> dict:
    response = requests.get(f"{API_BASE}/users/{user_id}/")
    if response.ok:
        logger.info(f"Fetched profile for user {user_id}")
        return response.json()
    logger.warning(f"Failed to fetch profile for user {user_id}. Status: {response.status_code}")
    return {}
```

### Database / ORM Operation

```python
def deactivate_stale_accounts(days: int) -> int:
    cutoff = timezone.now() - timedelta(days=days)
    updated = User.objects.filter(last_login__lt=cutoff).update(is_active=False)
    logger.info(f"Deactivated {updated} accounts inactive for {days}+ days")
    return updated
```

### Serializer Validation Error

```python
class OrderSerializer(serializers.ModelSerializer):
    def validate(self, attrs):
        if attrs["quantity"] > MAX_ORDER_QUANTITY:
            logger.warning(
                f"Order quantity {attrs['quantity']} exceeds limit {MAX_ORDER_QUANTITY} "
                f"for user {self.context['request'].user.id}"
            )
            raise serializers.ValidationError("Quantity exceeds allowed maximum.")
        return attrs
```

---

## Best Practices

**Do:**
- Use `__name__` always — keeps logger names aligned with your module hierarchy.
- Pass structured context via `extra=` — machine-queryable; e.g. `extra={"user_id": user.id, "ip": ip}`.
- Use `logger.exception()` inside `except` blocks — captures the full traceback automatically.
- Log at I/O boundaries: request in/out, external call made/failed, task start/end.
- Include identifiers in messages: `"Order 4821 created"` not `"Order created"`.

**Don't:**
- Use `print()` for observability — unstructured and unroutable.
- Log sensitive data: passwords, tokens, full credit card numbers, PII.
- Log inside tight loops at INFO or above — use `DEBUG` or aggregate first.
- Catch-and-swallow exceptions silently — at minimum call `logger.exception(...)`.

---

## Common Gotchas

| Symptom | Cause | Fix |
|---------|-------|-----|
| Logs appear twice | `propagate` not `False` while root logger also has handlers | Set `"propagate": False` |
| No logs appear | Logger namespace doesn't match `LOGGING` keys | Verify `__name__` resolves to a prefix matching a configured logger |
| No tracebacks in exceptions | Used `logger.error()` not `logger.exception()` | Switch to `logger.exception()` or add `exc_info=True` |
| `extra` key conflicts | Using reserved `LogRecord` keys like `message`, `name` | Namespace your keys: `request_uri` not `uri` |

---

## Quick Reference

```python
# 1. Declare (top of any file)
import logging
logger = logging.getLogger(__name__)

# 2. Log levels
logger.debug("Detailed trace: %s", value)
logger.info(f"Record {obj.id} created")
logger.warning(f"Retry {attempt}/3 for {job_id}")
logger.error(f"Payment {payment_id} failed")
logger.critical("Database unreachable — aborting")

# 3. Exceptions (inside except blocks)
try:
    risky_call()
except SomeError:
    logger.exception("risky_call failed")   # auto-captures traceback

# 4. Structured context
logger.info("User logged in", extra={"user_id": user.id, "ip": ip})
```
