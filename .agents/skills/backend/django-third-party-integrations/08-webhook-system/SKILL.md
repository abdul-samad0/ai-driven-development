---
name: webhook-system
description: Receive, validate, log, and process webhooks from external services in a modular structure.
---

# Webhook System

## Folder Structure

```text
integrations/
├── models/
│   └── WebhookEvent.py
├── api/v1/
│   ├── views/
│   │   └── WebhookReceiveView.py
│   ├── serializers/
│   │   └── WebhookEventSerializer.py
│   └── urls.py
├── tasks/
│   └── sync_webhook_task.py
├── filters/
│   └── WebhookEventFilter.py
├── admin/
│   └── WebhookEventAdmin.py
└── tests/
    └── test_webhooks.py
```

## Webhook Event Model

Create `integrations/models/WebhookEvent.py`:

```python
from django.db import models


class WebhookEvent(models.Model):
    """Log received webhook events for audit trail and replay."""

    STATUS_CHOICES = [
        ("received", "Received"),
        ("processing", "Processing"),
        ("processed", "Processed"),
        ("failed", "Failed"),
    ]

    provider = models.CharField(max_length=64, db_index=True)  # "stripe", "twilio", etc.
    event_id = models.CharField(max_length=255, unique=True, db_index=True)
    event_type = models.CharField(max_length=128, db_index=True)
    payload = models.JSONField()
    signature = models.CharField(max_length=512, blank=True)  # For verification
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default="received")
    error_message = models.TextField(blank=True)
    processed_at = models.DateTimeField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)

    class Meta:
        indexes = [
            models.Index(fields=["provider", "status"]),
            models.Index(fields=["created_at", "provider"]),
        ]

    def __str__(self):
        return f"{self.provider} - {self.event_type} ({self.status})"
```

Update `integrations/models/__init__.py`:

```python
from integrations.models.OAuthToken import OAuthToken
from integrations.models.StripeCustomer import StripeCustomer
from integrations.models.IntegrationLog import IntegrationLog
from integrations.models.WebhookEvent import WebhookEvent

__all__ = ["OAuthToken", "StripeCustomer", "IntegrationLog", "WebhookEvent"]
```

## Webhook Signature Verification

Create `integrations/api/v1/verifiers.py`:

```python
import hashlib
import hmac
import time
import stripe
from django.conf import settings
from typing import Callable


def verify_stripe_signature(payload: bytes, sig_header: str) -> bool:
    """Verify Stripe webhook signature."""
    try:
        stripe.Webhook.construct_event(
            payload,
            sig_header,
            settings.STRIPE_WEBHOOK_SECRET
        )
        return True
    except (ValueError, stripe.error.SignatureVerificationError):
        return False


def verify_twilio_signature(request, url: str, params: dict) -> bool:
    """Verify Twilio webhook signature."""
    from twilio.request_validator import RequestValidator
    validator = RequestValidator(settings.TWILIO_AUTH_TOKEN)
    sig = request.META.get("HTTP_X_TWILIO_SIGNATURE", "")
    return validator.validate(url, params, sig)


def verify_slack_signature(timestamp: str, sig: str, body: str) -> bool:
    """Verify Slack webhook signature."""
    # Check timestamp is recent (within 5 minutes)
    if abs(time.time() - int(timestamp)) > 60 * 5:
        return False

    # Compute signature
    base = f"v0:{timestamp}:{body}"
    expected = "v0=" + hmac.new(
        settings.SLACK_SIGNING_SECRET.encode(),
        base.encode(),
        hashlib.sha256
    ).hexdigest()

    return hmac.compare_digest(expected, sig)


def verify_github_signature(payload: bytes, sig_header: str) -> bool:
    """Verify GitHub webhook signature."""
    expected_sig = "sha256=" + hmac.new(
        settings.GITHUB_WEBHOOK_SECRET.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected_sig, sig_header)


VERIFIERS: dict[str, Callable] = {
    "stripe": verify_stripe_signature,
    "twilio": verify_twilio_signature,
    "slack": verify_slack_signature,
    "github": verify_github_signature,
}


def verify_webhook(provider: str, request=None, **kwargs) -> bool:
    """Dispatch to appropriate verifier."""
    verifier = VERIFIERS.get(provider)
    if not verifier:
        return False
    return verifier(request, **kwargs) if request else verifier(**kwargs)
```

## DRF Webhook View

Create `integrations/api/v1/views/WebhookReceiveView.py`:

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.permissions import AllowAny
from integrations.models import WebhookEvent
from integrations.api.v1.verifiers import verify_webhook
from integrations.tasks.sync_webhook_task import process_webhook
import json


class WebhookReceiveView(APIView):
    """Generic webhook receiver for multiple providers."""

    authentication_classes = []
    permission_classes = [AllowAny]

    def post(self, request, provider: str):
        # Verify signature based on provider
        if not self._verify_signature(request, provider):
            return Response({"detail": "Invalid signature"}, status=401)

        # Parse event data
        payload = request.data if isinstance(request.data, dict) else json.loads(request.body)
        event_id = payload.get("id") or payload.get("event_id")
        event_type = payload.get("type") or payload.get("event_type")

        # Check for duplicate events
        if WebhookEvent.objects.filter(provider=provider, event_id=event_id).exists():
            return Response({"detail": "Event already processed"}, status=200)

        # Log the webhook event
        webhook_event = WebhookEvent.objects.create(
            provider=provider,
            event_id=event_id,
            event_type=event_type,
            payload=payload,
            signature=request.META.get("HTTP_STRIPE_SIGNATURE", ""),
        )

        # Offload processing to Celery
        process_webhook.delay(webhook_event.id)

        # Return 200 immediately
        return Response({"status": "received"}, status=200)

    def _verify_signature(self, request, provider: str) -> bool:
        """Verify webhook signature based on provider."""
        if provider == "stripe":
            payload = request.body
            sig = request.META.get("HTTP_STRIPE_SIGNATURE", "")
            return verify_webhook("stripe", payload=payload, sig_header=sig)
        elif provider == "twilio":
            url = request.build_absolute_uri()
            params = request.POST.dict()
            return verify_webhook("twilio", request=request, url=url, params=params)
        elif provider == "slack":
            timestamp = request.META.get("HTTP_X_SLACK_REQUEST_TIMESTAMP", "")
            sig = request.META.get("HTTP_X_SLACK_SIGNATURE", "")
            body = request.body.decode() if isinstance(request.body, bytes) else request.body
            return verify_webhook("slack", timestamp=timestamp, sig=sig, body=body)
        elif provider == "github":
            payload = request.body
            sig = request.META.get("HTTP_X_HUB_SIGNATURE_256", "")
            return verify_webhook("github", payload=payload, sig_header=sig)
        return False
```

## Webhook Serializer

Create `integrations/api/v1/serializers/WebhookEventSerializer.py`:

```python
from rest_framework import serializers
from integrations.models import WebhookEvent


class WebhookEventSerializer(serializers.ModelSerializer):
    class Meta:
        model = WebhookEvent
        fields = ["id", "provider", "event_id", "event_type", "status", "created_at", "processed_at"]
        read_only_fields = ["id", "created_at", "processed_at"]
```

## Webhook Processing Task

Create `integrations/tasks/sync_webhook_task.py`:

```python
from celery import shared_task
from django.utils import timezone
from integrations.models import WebhookEvent
import logging

logger = logging.getLogger(__name__)


@shared_task
def process_webhook(webhook_event_id: int):
    """Process a received webhook event."""
    try:
        webhook_event = WebhookEvent.objects.get(id=webhook_event_id)
    except WebhookEvent.DoesNotExist:
        return {"error": "Webhook event not found"}

    webhook_event.status = "processing"
    webhook_event.save()

    try:
        # Dispatch to provider-specific handler
        handler = get_webhook_handler(webhook_event.provider)
        handler(webhook_event)

        webhook_event.status = "processed"
        webhook_event.processed_at = timezone.now()
    except Exception as exc:
        webhook_event.status = "failed"
        webhook_event.error_message = str(exc)
        logger.exception(f"Failed to process webhook {webhook_event.id}: {exc}")

    webhook_event.save()
    return {"status": webhook_event.status}


def get_webhook_handler(provider: str):
    """Get the handler function for a provider."""
    handlers = {
        "stripe": handle_stripe_webhook,
        "twilio": handle_twilio_webhook,
        "slack": handle_slack_webhook,
        "github": handle_github_webhook,
    }
    return handlers.get(provider, lambda event: None)


def handle_stripe_webhook(webhook_event: WebhookEvent):
    """Handle Stripe webhook events."""
    event_type = webhook_event.event_type
    data = webhook_event.payload.get("data", {}).get("object", {})

    if event_type == "checkout.session.completed":
        on_checkout_completed(data)
    elif event_type == "customer.subscription.created":
        on_subscription_created(data)
    elif event_type == "customer.subscription.deleted":
        on_subscription_deleted(data)
    elif event_type == "invoice.payment_succeeded":
        on_invoice_paid(data)
    elif event_type == "invoice.payment_failed":
        on_invoice_failed(data)


def on_checkout_completed(data):
    """Handle successful checkout."""
    session_id = data.get("id")
    amount = data.get("amount_total")
    logger.info(f"Checkout completed: {session_id} - ${amount}")


def on_subscription_created(data):
    """Handle new subscription."""
    subscription_id = data.get("id")
    logger.info(f"Subscription created: {subscription_id}")


def on_subscription_deleted(data):
    """Handle subscription cancellation."""
    subscription_id = data.get("id")
    logger.info(f"Subscription deleted: {subscription_id}")


def on_invoice_paid(data):
    """Handle successful payment."""
    invoice_id = data.get("id")
    amount = data.get("amount_paid")
    logger.info(f"Invoice paid: {invoice_id} - ${amount}")


def on_invoice_failed(data):
    """Handle failed payment."""
    invoice_id = data.get("id")
    logger.warning(f"Invoice failed: {invoice_id}")


def handle_twilio_webhook(webhook_event: WebhookEvent):
    """Handle Twilio webhook events."""
    # Implementation


def handle_slack_webhook(webhook_event: WebhookEvent):
    """Handle Slack webhook events."""
    # Implementation


def handle_github_webhook(webhook_event: WebhookEvent):
    """Handle GitHub webhook events."""
    # Implementation
```

## URL Configuration

Create `integrations/api/v1/urls.py`:

```python
from django.urls import path
from integrations.api.v1.views.WebhookReceiveView import WebhookReceiveView

app_name = "integrations_api"

urlpatterns = [
    path("webhooks/<str:provider>/", WebhookReceiveView.as_view(), name="webhook-receive"),
]
```

Add to main `urls.py`:

```python
urlpatterns = [
    ...
    path("api/integrations/", include("integrations.api.v1.urls")),
]
```

Register webhooks with providers:
- **Stripe**: `https://yourdomain.com/api/integrations/webhooks/stripe/`
- **Twilio**: `https://yourdomain.com/api/integrations/webhooks/twilio/`
- **Slack**: `https://yourdomain.com/api/integrations/webhooks/slack/`
- **GitHub**: `https://yourdomain.com/api/integrations/webhooks/github/`

## Webhook Filter

Create `integrations/filters/WebhookEventFilter.py`:

```python
import django_filters
from integrations.models import WebhookEvent


class WebhookEventFilter(django_filters.FilterSet):
    created_at_from = django_filters.DateTimeFilter(field_name="created_at", lookup_expr="gte")
    created_at_to = django_filters.DateTimeFilter(field_name="created_at", lookup_expr="lte")

    class Meta:
        model = WebhookEvent
        fields = ["provider", "event_type", "status"]
```

## Admin Customization

Create `integrations/admin/WebhookEventAdmin.py`:

```python
from django.contrib import admin
from django.utils.html import format_html
from integrations.models import WebhookEvent
from integrations.filters import WebhookEventFilter
import json


@admin.register(WebhookEvent)
class WebhookEventAdmin(admin.ModelAdmin):
    list_display = ("provider", "event_type", "status_badge", "event_id", "created_at")
    list_filter = ("provider", "status", "created_at")
    search_fields = ("event_id", "event_type")
    readonly_fields = ("payload_formatted", "created_at")

    def payload_formatted(self, obj):
        """Display JSON payload nicely."""
        return format_html("<pre>{}</pre>", json.dumps(obj.payload, indent=2))

    def status_badge(self, obj):
        """Display status with color."""
        colors = {
            "received": "#FFA500",
            "processing": "#0099FF",
            "processed": "#00CC00",
            "failed": "#FF0000",
        }
        color = colors.get(obj.status, "#999999")
        return format_html(
            '<span style="background-color: {}; color: white; padding: 5px 10px; border-radius: 3px;">{}</span>',
            color,
            obj.status
        )
```

## Testing

Create `integrations/tests/test_webhooks.py`:

```python
from django.test import TestCase, Client as DjangoClient
from integrations.models import WebhookEvent
import json


class TestStripeWebhook(TestCase):
    def setUp(self):
        self.client = DjangoClient()
        self.webhook_url = "/api/integrations/webhooks/stripe/"

    def test_webhook_received_and_logged(self):
        payload = {
            "id": "evt_test",
            "type": "checkout.session.completed",
            "data": {
                "object": {
                    "id": "cs_test",
                    "amount_total": 5000,
                }
            }
        }

        # Mock stripe signature verification
        with patch("integrations.api.v1.verifiers.verify_stripe_signature", return_value=True):
            response = self.client.post(
                self.webhook_url,
                data=json.dumps(payload),
                content_type="application/json",
                HTTP_STRIPE_SIGNATURE="test_sig"
            )

        self.assertEqual(response.status_code, 200)
        self.assertTrue(WebhookEvent.objects.filter(event_id="evt_test").exists())
```

## Key Principles

1. **Return 200 immediately** — Don't block the webhook provider
2. **Be idempotent** — Deduplicate by `event_id` before processing
3. **Log raw payloads** — Store the complete webhook for audit and debugging
4. **Verify signatures** — Always authenticate webhook sources
5. **Process asynchronously** — Use Celery for event processing
6. **Handle failures gracefully** — Log errors and allow manual replay
