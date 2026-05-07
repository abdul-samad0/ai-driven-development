---
name: resilience-patterns
description: Circuit breaker, idempotency, and error handling for external API calls in a modular structure.
---

# Resilience Patterns

## Folder Structure

```text
integrations/
├── clients/
│   └── resilient_client.py
├── tasks/
│   └── retry_failed_requests_task.py
├── models/
│   └── IntegrationLog.py
└── api/v1/
    └── exceptions.py
```

## Circuit Breaker Pattern

Create `integrations/clients/resilient_client.py`:

```python
from django.core.cache import cache
from integrations.clients.base import BaseServiceClient, ServiceClientError
import logging

logger = logging.getLogger(__name__)


class CircuitBreakerMixin:
    """Mixin to add circuit breaker behavior to clients."""

    FAILURE_THRESHOLD = 5
    FAILURE_WINDOW = 60  # seconds
    COOLDOWN = 30  # seconds

    def _is_circuit_open(self) -> bool:
        """Check if circuit is open (service is down)."""
        return cache.get(f"circuit:{self.SERVICE_NAME}:open", False)

    def _record_failure(self):
        """Record a failure and potentially open the circuit."""
        key = f"circuit:{self.SERVICE_NAME}:failures"
        failures = cache.get(key, 0) + 1
        cache.set(key, failures, timeout=self.FAILURE_WINDOW)

        if failures >= self.FAILURE_THRESHOLD:
            cache.set(f"circuit:{self.SERVICE_NAME}:open", True, timeout=self.COOLDOWN)
            logger.warning(f"Circuit opened for {self.SERVICE_NAME} after {failures} failures")

    def _record_success(self):
        """Clear failures on successful request."""
        cache.delete(f"circuit:{self.SERVICE_NAME}:failures")
        cache.delete(f"circuit:{self.SERVICE_NAME}:open")


class ResilientServiceClient(CircuitBreakerMixin, BaseServiceClient):
    """Client with circuit breaker and resilience features."""

    def get(self, path: str, **kwargs):
        if self._is_circuit_open():
            raise ServiceClientError(self.SERVICE_NAME, "Circuit breaker is open (service unavailable)")
        try:
            result = super().get(path, **kwargs)
            self._record_success()
            return result
        except ServiceClientError as exc:
            self._record_failure()
            raise

    def post(self, path: str, **kwargs):
        if self._is_circuit_open():
            raise ServiceClientError(self.SERVICE_NAME, "Circuit breaker is open (service unavailable)")
        try:
            result = super().post(path, **kwargs)
            self._record_success()
            return result
        except ServiceClientError as exc:
            self._record_failure()
            raise
```

Usage in views:

```python
# integrations/api/v1/views/PaymentView.py
from rest_framework.views import APIView
from rest_framework.response import Response
from integrations.clients.stripe import StripeClient
from integrations.clients.base import ServiceClientError


class CreateChargeView(APIView):
    def post(self, request):
        try:
            client = StripeClient()
            result = client.create_charge(
                amount=request.data["amount"],
                currency="usd",
                source=request.data["source"]
            )
            client.close()
            return Response({"charge_id": result["id"]})
        except ServiceClientError as exc:
            return Response(
                {"detail": "Payment service temporarily unavailable"},
                status=503,
            )
```

## Idempotency

Add idempotency to prevent duplicate operations (payments, sends):

```python
# integrations/clients/resilient_client.py
import uuid
from typing import Any


class IdempotentMixin:
    """Mixin to add idempotency support."""

    def post_idempotent(self, path: str, **kwargs) -> Any:
        """Post request with idempotency key."""
        headers = kwargs.get("headers", {})
        headers["Idempotency-Key"] = str(uuid.uuid4())
        kwargs["headers"] = headers
        return self.post(path, **kwargs)


class StripeClient(IdempotentMixin, ResilientServiceClient):
    SERVICE_NAME = "Stripe"

    def get_base_url(self) -> str:
        return "https://api.stripe.com/v1"

    def get_auth_headers(self) -> dict[str, str]:
        from django.conf import settings
        return {"Authorization": f"Bearer {settings.STRIPE_SECRET_KEY}"}

    def create_charge(self, amount: int, currency: str, source: str) -> dict:
        """Create a charge with idempotency."""
        return self.post_idempotent(
            "/charges",
            json={"amount": amount, "currency": currency, "source": source}
        )
```

## Request Logging and Retry

Create `integrations/models/IntegrationLog.py`:

```python
from django.db import models


class IntegrationLog(models.Model):
    """Log all external API requests for debugging and retry."""

    STATUS_CHOICES = [
        ("pending", "Pending"),
        ("success", "Success"),
        ("failed", "Failed"),
        ("retrying", "Retrying"),
    ]

    service = models.CharField(max_length=64, db_index=True)
    method = models.CharField(max_length=10)  # GET, POST, etc.
    endpoint = models.CharField(max_length=255)
    request_body = models.JSONField(null=True, blank=True)
    response_status = models.IntegerField(null=True)
    response_body = models.JSONField(null=True, blank=True)
    error_message = models.TextField(blank=True)
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default="pending")
    retry_count = models.IntegerField(default=0)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        indexes = [
            models.Index(fields=["service", "status"]),
            models.Index(fields=["created_at"]),
        ]

    def __str__(self):
        return f"{self.service} {self.method} {self.endpoint} - {self.status}"
```

Update `integrations/models/__init__.py`:

```python
from integrations.models.OAuthToken import OAuthToken
from integrations.models.StripeCustomer import StripeCustomer
from integrations.models.IntegrationLog import IntegrationLog

__all__ = ["OAuthToken", "StripeCustomer", "IntegrationLog"]
```

Extend client to log requests:

```python
# integrations/clients/base.py
from integrations.models import IntegrationLog

class BaseServiceClient(ABC):
    # ... existing code ...

    def _log_request(self, method: str, path: str, status: str, status_code: int = None, error: str = ""):
        """Log the API request."""
        IntegrationLog.objects.create(
            service=self.SERVICE_NAME,
            method=method,
            endpoint=path,
            response_status=status_code,
            status=status,
            error_message=error,
        )
```

## Retry Task

Create `integrations/tasks/retry_failed_requests_task.py`:

```python
from celery import shared_task
from django.utils import timezone
from datetime import timedelta
from integrations.models import IntegrationLog


@shared_task
def retry_failed_requests():
    """Retry failed requests that should be retried."""
    failed_logs = IntegrationLog.objects.filter(
        status="failed",
        retry_count__lt=3,
        created_at__gte=timezone.now() - timedelta(hours=24),
    )

    for log in failed_logs:
        try:
            # Get the appropriate client for this service
            client_class = get_client_for_service(log.service)
            client = client_class()

            # Retry the request
            method = log.method.lower()
            request_method = getattr(client, method)
            response = request_method(log.endpoint, json=log.request_body)

            log.status = "success"
            log.response_body = response
            log.response_status = 200
            log.retry_count += 1
            log.save()
        except Exception as exc:
            log.error_message = str(exc)
            log.retry_count += 1
            log.save()


def get_client_for_service(service_name: str):
    """Get the client class for a service."""
    clients = {
        "Stripe": "integrations.clients.stripe.StripeClient",
        "SendGrid": "integrations.clients.sendgrid.SendGridClient",
    }
    module_path, class_name = clients.get(service_name, "").rsplit(".", 1)
    module = __import__(module_path, fromlist=[class_name])
    return getattr(module, class_name)
```

Configure periodic task:

```python
# settings.py
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    "retry-failed-requests": {
        "task": "integrations.tasks.retry_failed_requests_task.retry_failed_requests",
        "schedule": crontab(minute="*/15"),  # Every 15 minutes
    },
}
```

## DRF Exception Handler

Create `integrations/api/v1/exceptions.py`:

```python
from rest_framework.views import exception_handler as drf_exception_handler
from rest_framework.response import Response
from integrations.clients.base import ServiceClientError


def integration_exception_handler(exc, context):
    """Handle integration-specific exceptions."""
    if isinstance(exc, ServiceClientError):
        # 502 = Bad Gateway (external service error)
        return Response(
            {
                "detail": "External service unavailable. Please try again later.",
                "service": exc.service,
            },
            status=502,
        )
    return drf_exception_handler(exc, context)
```

Register in `settings.py`:

```python
REST_FRAMEWORK = {
    "EXCEPTION_HANDLER": "integrations.api.v1.exceptions.integration_exception_handler",
}
```

## Admin Customization

Create `integrations/admin/IntegrationLogAdmin.py`:

```python
from django.contrib import admin
from integrations.models import IntegrationLog


@admin.register(IntegrationLog)
class IntegrationLogAdmin(admin.ModelAdmin):
    list_display = ("service", "method", "endpoint", "status", "response_status", "retry_count", "created_at")
    list_filter = ("service", "status", "created_at")
    search_fields = ("endpoint", "error_message")
    readonly_fields = ("request_body", "response_body", "error_message", "created_at", "updated_at")

    def has_add_permission(self, request):
        return False

    def has_delete_permission(self, request, obj=None):
        return False
```

## Testing

Create `integrations/tests/test_circuit_breaker.py`:

```python
from django.test import TestCase
from django.core.cache import cache
from integrations.clients.resilient_client import ResilientServiceClient
from integrations.clients.base import ServiceClientError


class TestCircuitBreaker(TestCase):
    def setUp(self):
        cache.clear()

    def test_circuit_opens_after_failures(self):
        class TestClient(ResilientServiceClient):
            SERVICE_NAME = "Test"

            def get_base_url(self):
                return "https://test.com"

            def get_auth_headers(self):
                return {}

        client = TestClient()

        # Simulate failures
        for _ in range(5):
            client._record_failure()

        # Circuit should be open
        self.assertTrue(client._is_circuit_open())
```
